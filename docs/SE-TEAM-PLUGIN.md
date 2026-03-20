# Software Engineering Team Plugin — NanoClaw Integration

This document covers how the `software-engineering-team` Claude Code plugin is integrated into NanoClaw, what agents are available, and how to use them from your messaging channel.

---

## What the Plugin Is

The `software-engineering-team` plugin (installed at `~/.claude/plugins/cache/uni-plugin-local/software-engineering-team/1.0.0/`) provides 7 specialist Claude Code sub-agents, each focused on a distinct engineering discipline. They were originally translated from GitHub Copilot CLI agents (see `translation-report.json`).

An 8th agent — **SE: Team Lead** — was created locally as an orchestrator that triages requests and dispatches the right specialists.

---

## Agents Reference

### SE: Team Lead (`se-team-lead.md`) — Orchestrator
**Use this one for most tasks.** It triages the request, dispatches specialists in parallel, and synthesizes findings into a single prioritized report. It knows when to call Security vs. Architect vs. DevOps, and it has full NanoClaw tool access (`mcp__nanoclaw__*`) to send you notifications and post PR comments.

### SE: Security (`se-security-reviewer.md`)
Security-focused code review. Covers OWASP Top 10, Zero Trust architecture, LLM/ML-specific threats (OWASP LLM Top 10), secrets handling, authentication flows, and access control. Prioritizes findings by risk: Critical (auth, payments, AI models) → Medium (user data, external APIs) → Low (UI, utilities).

### SE: Architect (`se-system-architecture-reviewer.md`)
Architecture review and design validation. Applies Well-Architected frameworks scaled to system size: full frameworks for enterprise (>100K users), fundamentals for smaller systems. Handles traditional web, AI/agent systems, data pipelines, and microservices patterns.

### SE: DevOps/CI (`se-gitops-ci-specialist.md`)
CI/CD pipeline design, deployment failure debugging, and GitOps workflows. Focuses on making deployments reliable and automatic. Triages failures by: what changed → when it broke → scope of impact.

### SE: Product Manager (`se-product-manager-advisor.md`)
Creates GitHub issues with full business context — not just implementation tasks, but user stories, success criteria, and value alignment. Always asks: who is the user, what problem are they solving, how is success measured.

### SE: Responsible AI (`se-responsible-ai-code.md`)
Reviews AI/ML code for bias, accessibility compliance, fairness, and ethical concerns. Checks recommendations engines, content filters, and automated decision systems. Tests inputs across demographic groups and validates WCAG accessibility.

### SE: Tech Writer (`se-technical-writer.md`)
Writes and reviews developer documentation, READMEs, changelogs, tutorials, and technical blog posts. Adapts tone by audience: conversational for blogs, direct and precise for reference docs, encouraging for tutorials.

### SE: UX Designer (`se-ux-ui-designer.md`)
Jobs-to-be-Done analysis and user journey mapping. Creates research artifacts (personas, journey maps, JTBD analyses) intended to inform Figma designs. Does not produce UI designs directly.

---

## How Integration Works

### The Problem

The plugin agents live on the host at:
```
~/.claude/plugins/cache/uni-plugin-local/software-engineering-team/1.0.0/agents/
```

NanoClaw containers mount `data/sessions/{group}/.claude` as `/home/node/.claude` inside each container. The host plugin directory is not inside this path, so agents are not available to NanoClaw agents by default.

### The Solution: Agent Sync (mirrors the Skills Sync)

NanoClaw already syncs `container/skills/` into every group's `.claude/skills/` at container startup. The same pattern was extended to agents.

**Files in the repo:**
```
container/agents/
  se-team-lead.md              ← created locally (orchestrator)
  se-security-reviewer.md      ← copied from plugin cache
  se-system-architecture-reviewer.md
  se-gitops-ci-specialist.md
  se-product-manager-advisor.md
  se-responsible-ai-code.md
  se-technical-writer.md
  se-ux-ui-designer.md
```

**Sync logic added to `src/container-runner.ts`** (after the skills sync block at line ~149):
```typescript
// Sync agents from container/agents/ into each group's .claude/agents/
const agentsSrc = path.join(process.cwd(), 'container', 'agents');
const agentsDst = path.join(groupSessionsDir, 'agents');
if (fs.existsSync(agentsSrc)) {
  fs.mkdirSync(agentsDst, { recursive: true });
  for (const agentFile of fs.readdirSync(agentsSrc)) {
    if (!agentFile.endsWith('.md')) continue;
    fs.copyFileSync(
      path.join(agentsSrc, agentFile),
      path.join(agentsDst, agentFile),
    );
  }
}
```

This runs on the host before each container launch. No container rebuild is needed when agent files change — just rebuild the TypeScript (`npm run build`) and restart NanoClaw.

**Result:** All agents are available at `/home/node/.claude/agents/` inside every container, and the Claude Agent SDK loads them automatically.

### Adding New Agents in the Future

To add agents from another plugin or create a custom one:

```bash
# Copy from a plugin
cp ~/.claude/plugins/cache/uni-plugin-local/<plugin>/<version>/agents/*.md container/agents/

# Or create one from scratch
cat > container/agents/my-agent.md << 'EOF'
---
name: 'My Agent'
description: 'What it does'
tools: ['Bash', 'Read', 'Write', 'Edit', 'Glob', 'Grep']
---

# My Agent

...instructions...
EOF

npm run build
# Restart NanoClaw — agents sync on next container start
```

---

## How to Use from Your Messaging Channel

### Using the Team Lead (recommended)

The Team Lead handles triage and parallelism for you. Use it for any request that might involve multiple disciplines.

**General requests:**
```
@Andy use the SE Team Lead to review PR #142
@Andy use the SE Team Lead to review the auth module in /workspace/extra/myapp/src/auth.ts
@Andy use the SE Team Lead to scope out the new payment feature and create a GitHub issue
@Andy use the SE Team Lead to do a weekly code health check on /workspace/extra/myapp
```

**The Team Lead will automatically:**
1. Identify which specialists are relevant (e.g., Security + Architect for a code review)
2. Dispatch them in parallel using the `Task` tool
3. Wait for all results
4. Synthesize findings into a single prioritized report (Critical → Important → Suggestions)
5. Post results (PR comment, report file, or message back to you)

### Calling Specialists Directly

For focused, single-discipline tasks you can skip the Team Lead:

```
@Andy use the SE Security agent to review /workspace/extra/myapp/src/auth.ts
@Andy use the SE Architect agent to review the design doc at /workspace/extra/myapp/DESIGN.md
@Andy use the SE Tech Writer to update the README for /workspace/extra/myapp to match recent changes
@Andy use the SE DevOps agent to debug why the GitHub Actions workflow is failing
@Andy use the SE Product Manager to create a GitHub issue for the login flow redesign
```

### Scheduled Tasks

From your main channel, you can schedule recurring engineering reviews:

**Weekly security scan:**
```
@Andy every Friday at 4pm, use the SE Security agent to review any files committed
this week in /workspace/extra/myapp and send me a summary of findings
```

**Weekly code health report:**
```
@Andy every Monday at 9am, use the SE Team Lead to do a code health check on
/workspace/extra/myapp — security scan on new code, docs drift check,
and architecture review of any new modules. Save the report to
/workspace/group/reports/ and send me the summary.
```

**Automated PR review:**
```
@Andy check every 30 minutes if any new PRs have been opened in my-org/myapp.
For each new one, use the SE Team Lead to review it, then post the findings
as a PR comment with gh pr review.
```

**Post-deploy architecture review:**
```
@Andy every Wednesday at 2pm, review the git log from the past week in
/workspace/extra/myapp. If significant new modules were added, run the
SE Architect agent on them and send me a design assessment.
```

---

## Report Format

The Team Lead produces reports in this structure:

```markdown
## Engineering Review: <subject>

### Critical (fix before merge)
- [Security] SQL injection risk in UserRepository.findByEmail() — line 42
- [Architect] Session state stored in-process; will break at >1 instance

### Important (address soon)
- [Security] Missing rate limiting on /api/login endpoint
- [Architect] No circuit breaker on third-party payment API calls

### Suggestions (optional improvements)
- [Tech Writer] Auth module has no inline documentation

### Specialist Notes
**Security**: Reviewed auth.ts and payment.ts. Two critical findings...
**Architecture**: Session management design will not scale horizontally...

---
Recommendation: REQUEST CHANGES — resolve Critical items before merging.
```

---

## Files Changed / Created

| File | Change |
|---|---|
| `container/agents/se-team-lead.md` | Created — orchestrator agent |
| `container/agents/se-security-reviewer.md` | Copied from plugin cache |
| `container/agents/se-system-architecture-reviewer.md` | Copied from plugin cache |
| `container/agents/se-gitops-ci-specialist.md` | Copied from plugin cache |
| `container/agents/se-product-manager-advisor.md` | Copied from plugin cache |
| `container/agents/se-responsible-ai-code.md` | Copied from plugin cache |
| `container/agents/se-technical-writer.md` | Copied from plugin cache |
| `container/agents/se-ux-ui-designer.md` | Copied from plugin cache |
| `src/container-runner.ts` | Added agent sync block (mirrors skills sync at line ~149) |

---

## Keeping Agents Up to Date

The agent files in `container/agents/` are snapshots copied from the plugin cache. If the plugin is updated, re-copy:

```bash
cp ~/.claude/plugins/cache/uni-plugin-local/software-engineering-team/1.0.0/agents/*.md container/agents/
npm run build
# Restart NanoClaw
```

The Team Lead (`se-team-lead.md`) is maintained locally and should not be overwritten by this copy.
