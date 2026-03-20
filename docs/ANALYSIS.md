# NanoClaw: In-Depth Analysis

## What It Is

NanoClaw is a **personal AI assistant orchestration platform**. It is a single Node.js process that listens for messages across messaging channels, routes trigger messages to AI agents running inside isolated Linux containers, and returns responses back to the originating channel. It also manages persistent memory, scheduled tasks, and multi-group context isolation.

The primary value proposition: **real OS-level isolation** instead of application-level permission checks. Every agent runs in its own container — Bash access is safe because commands run inside the container, not on your host.

The project was built as a minimal alternative to OpenClaw (formerly ClawBot), which had grown to ~500k lines of code, 53 config files, and 70+ dependencies. NanoClaw does the same core job in ~30 TypeScript source files.

---

## Architecture Overview

### Complete Data Flow

```
Messaging Platform (WhatsApp, Telegram, Slack, Discord, Gmail)
         │
         ▼
    Channel Layer  (src/channels/)
    Self-registering adapters; standardized interface
         │
         ▼
    SQLite Database  (src/db.ts)
    Messages, groups, sessions, tasks stored here
         │
         ▼
    Poll Loop  (src/index.ts — every 2 seconds)
    Detects new trigger messages in registered groups
         │
         ▼
    GroupQueue  (src/group-queue.ts)
    Per-group concurrency control (max 5 containers globally)
         │
         ▼
    Container Runner  (src/container-runner.ts)
    Spawns Docker/Apple Container with specific filesystem mounts
         │
         ▼
    Agent Runner  (container/agent-runner/src/index.ts)
    Runs Claude Agent SDK inside the container
    Communicates back to host via stdout markers + IPC filesystem
         │
         ▼
    Credential Proxy  (src/credential-proxy.ts)
    HTTP proxy on localhost:3001; injects real API keys so containers never see secrets
```

### Key Design Decisions

| Decision | Why |
|---|---|
| Single Node.js process | Simplicity; no inter-process coordination |
| SQLite for everything | No external DB dependency; portable; simple |
| Polling loop (not webhooks) | Works without a public URL; simpler operational model |
| File-based IPC | Container→host communication via JSON files in mounted directories |
| Credential proxy | Containers get a `placeholder` API key; real keys never leave the host |
| Per-group filesystem isolation | Each group can only read/write its own folder |

---

## The Three Critical Layers

### 1. Host Orchestrator (`src/`)

The Node.js process that never touches Claude directly. It:
- Connects channels (WhatsApp, Telegram, etc.)
- Polls SQLite for new trigger messages every 2 seconds
- Spawns containers with the right filesystem mounts
- Reads streaming output from containers (via `---NANOCLAW_OUTPUT_START---` / `---NANOCLAW_OUTPUT_END---` markers)
- Forwards agent responses back to the right channel
- Runs the task scheduler (checks for due tasks every 60 seconds)
- Watches IPC directories for agent-initiated actions (send message, schedule task, register group)

### 2. Container Agent (`container/agent-runner/`)

Runs *inside* the container. It:
- Receives the message prompt via stdin as JSON
- Calls the Claude Agent SDK (`@anthropic-ai/claude-agent-sdk`)'s `query()` function
- Provides Claude with a set of allowed tools: Bash, file tools, web tools, and a custom `nanoclaw` MCP server
- Polls `/workspace/ipc/input/` for follow-up messages (enabling multi-turn conversations without restarting the container)
- Archives conversations to `/workspace/group/conversations/` before context compaction

### 3. NanoClaw MCP Server (`container/agent-runner/src/ipc-mcp-stdio.ts`)

A stdio MCP server that runs *inside* the container alongside the agent. It gives Claude structured tools to interact with the host:

| Tool | What it does |
|---|---|
| `send_message` | Writes a JSON file to the IPC messages dir; host picks it up and sends via channel |
| `schedule_task` | Writes to IPC tasks dir; host creates a DB record |
| `list_tasks` | Reads `current_tasks.json` that the host pre-wrote to the IPC dir |
| `pause_task` / `resume_task` / `cancel_task` / `update_task` | Task management via IPC |
| `register_group` | Main-only; registers a new chat for the assistant to respond to |

All host-side actions are **authorized** based on which group's IPC directory the file came from — a non-main group can only send to itself, only main can register new groups.

---

## Security Model

```
Host Machine
├── .env (API keys) — never mounted into containers
├── ~/.config/nanoclaw/mount-allowlist.json — tamper-proof, never mounted
│
└── Container (per agent invocation)
    ├── /workspace/group/     <- group folder (read-write)
    ├── /workspace/global/    <- global CLAUDE.md (read-only for non-main)
    ├── /workspace/ipc/       <- IPC filesystem (write-only from container perspective)
    ├── /home/node/.claude/   <- isolated Claude sessions per group
    └── /app/src/             <- per-group agent-runner source (customizable)
```

Three security boundaries:
1. **Filesystem isolation**: container can only see explicitly mounted paths
2. **Credential isolation**: credential proxy injects API keys; containers use a placeholder
3. **IPC authorization**: host validates the source group directory before acting on any IPC command

---

## Memory System

Each group has its own `groups/{name}/CLAUDE.md`. When Claude runs, the Agent SDK automatically loads the `CLAUDE.md` from the working directory (`/workspace/group`), giving the agent persistent per-group memory.

There's also `groups/global/CLAUDE.md` — mounted read-only in non-main groups, read-write for main. This lets the main channel set global context (e.g., "this assistant's name is Andy, always respond in English").

Conversations are archived to `groups/{name}/conversations/` as Markdown files before context compaction.

---

## How You Can Use It

### Basic Usage Pattern

Once running, you talk to it from your messaging platform using the trigger word:
```
@Andy summarize the last week of commits in ~/projects/myapp
@Andy every Monday at 9am, pull HackerNews top stories and send me a briefing
@Andy join the Family Chat group    (from main channel)
```

### Channel Setup

The base install has no channels. You add them by running skills inside Claude Code:
- `/add-whatsapp` — QR code or pairing code authentication
- `/add-telegram` — Bot token setup
- `/add-slack` — Socket mode (no public URL needed)
- `/add-discord` — Bot token + guild ID
- `/add-gmail` — OAuth via GCP

Each skill modifies your fork's code to add the channel and self-registers it via the `registerChannel()` function in `src/channels/registry.ts`.

---

## How to Extend It

There are two levels of extension:

### Level 1: Fork + Ask Claude Code to Modify

Since the codebase is intentionally small, you can just describe what you want:
- "Change the trigger word to @Bob"
- "Add a custom system prompt to all agents in the family-chat group"
- "Mount my Obsidian vault at ~/Documents/Obsidian for the main group"
- "Don't respond after 10pm in the family-chat group"

### Level 2: Add a New Channel (skill pattern)

A channel is a TypeScript class implementing the `Channel` interface (`src/types.ts:82`):

```typescript
interface Channel {
  name: string;
  connect(): Promise<void>;
  sendMessage(jid: string, text: string): Promise<void>;
  isConnected(): boolean;
  ownsJid(jid: string): boolean;
  disconnect(): Promise<void>;
  setTyping?(jid: string, isTyping: boolean): Promise<void>;  // optional
  syncGroups?(force: boolean): Promise<void>;                  // optional
}
```

The channel self-registers at startup via:
```typescript
registerChannel('mychannel', (opts) => new MyChannel(opts));
```

The factory receives `opts.onMessage` (call this when a message arrives), `opts.onChatMetadata`, and `opts.registeredGroups`. Return `null` if credentials are missing — the orchestrator skips unconfigured channels gracefully.

### Level 3: Extend What the Agent Can Do

The agent-runner source is **per-group customizable** — the host copies `container/agent-runner/src/` to `data/sessions/{group}/agent-runner-src/` on first run. You can modify that copy to give one specific group different capabilities.

To add a new tool available to *all* agents, add it to the MCP server in `container/agent-runner/src/ipc-mcp-stdio.ts`. For example, to let the agent query a database, add a `query_db` tool that writes an IPC file, then handle it in `src/ipc.ts`.

### Level 4: Add Container Skills

Skills placed in `container/skills/` are synced into every group's `.claude/skills/` at startup. These are Claude Code skills available inside the agent container — useful for giving the agent access to specialized tools like browser automation (already included as `agent-browser`).

### Level 5: Contribute a Skill Branch

For larger additions (new channels, new integrations), the project's contribution model is to fork, make changes on a branch, and submit a PR. The maintainers create a `skill/{name}` branch that other users can merge via `/add-{name}`.

---

## Available Extension Points (Summary)

| Extension Point | File/Location | Use Case |
|---|---|---|
| New messaging channel | `src/channels/` + `registerChannel()` | Signal, SMS, Matrix |
| New MCP tool for agents | `container/agent-runner/src/ipc-mcp-stdio.ts` | Custom agent actions |
| New IPC command type | `src/ipc.ts` → `processTaskIpc()` | Host-side effects triggered by agents |
| Per-group agent customization | `data/sessions/{group}/agent-runner-src/` | Different tool sets per group |
| Container skills | `container/skills/` | Shared agent capabilities |
| Global agent instructions | `groups/global/CLAUDE.md` | Cross-group context/persona |
| Mount external data | `groups/{name}/CLAUDE.md` → `containerConfig.additionalMounts` | Give agents access to files |
| Custom scheduler triggers | Modify `src/task-scheduler.ts` | Event-driven vs time-driven tasks |

---

## Extending for Autonomous Coding Tasks

This section explains how to use NanoClaw as an orchestration layer for scheduling and dispatching autonomous coding agents — Claude Code CLI, Gemini CLI, GitHub Copilot CLI, Jules, and Kiro.

### Why NanoClaw Is Well-Suited for This

The container already has `@anthropic-ai/claude-code` installed globally (see `container/Dockerfile:34`). Every agent invocation IS a Claude Code session via the Agent SDK. The agent has:

- `Bash` tool — can run any CLI installed in the container
- `Read`, `Write`, `Edit`, `Glob`, `Grep` — full file I/O
- `WebSearch`, `WebFetch` — internet access
- `Task` / `TeamCreate` / `SendMessage` — agent swarms (sub-agent orchestration)
- `mcp__nanoclaw__*` — schedule tasks, send messages back, register groups
- `agent-browser` — Chromium-based browser automation (via Bash)

Installing additional coding CLIs is a one-line Dockerfile change. The scheduler fires them on any cron schedule. Results come back to your phone.

---

### 1. Claude Code CLI (Built-in — No Changes Needed)

Claude Code is already installed in every container. The agent runner itself _is_ the Claude Agent SDK. No additional setup is required.

**Use it as a scheduled autonomous coder:**

From your main channel or a dedicated coding group:
```
@Andy every weekday at 9am, check the failing tests in /workspace/extra/myapp,
fix what you can, and open a draft PR with gh pr create when done
```

For this to work, mount the target repo via `additionalMounts`:
```json
{
  "hostPath": "~/projects/myapp",
  "containerPath": "/workspace/extra/myapp",
  "readonly": false
}
```

Claude Code will run the tests via Bash, use its Edit tool to fix failures, commit, and push. The `send_message` MCP tool delivers a summary to your phone.

**Key config** (`container/agent-runner/src/index.ts:402`):
```typescript
allowedTools: [
  'Bash', 'Read', 'Write', 'Edit', 'Glob', 'Grep',
  'WebSearch', 'WebFetch',
  'Task', 'TaskOutput', 'TaskStop',
  'TeamCreate', 'TeamDelete', 'SendMessage',
  'TodoWrite', 'ToolSearch', 'Skill',
  'NotebookEdit',
  'mcp__nanoclaw__*'
],
```

To add a new tool (e.g., a database query tool), add its name here and implement it in the MCP server.

---

### 2. Gemini CLI

**Installation** — edit `container/Dockerfile`, add after the existing `npm install -g` line:
```dockerfile
RUN npm install -g @google/gemini-cli
```

Rebuild the container:
```bash
./container/build.sh
```

**Authentication** — Gemini CLI reads from `~/.config/gemini/` or `GOOGLE_API_KEY`. Mount your pre-authenticated config read-only:

Register the group with an additional mount (from main channel):
```
@Andy for the coding-workspace group, add a mount:
  hostPath: ~/.config/gemini
  containerPath: /home/node/.config/gemini
  readonly: true
```

Or set `GOOGLE_API_KEY` in your `.env` and pass it to containers by adding a line in `src/container-runner.ts:buildContainerArgs()`:
```typescript
if (process.env.GOOGLE_API_KEY) {
  args.push('-e', `GOOGLE_API_KEY=${process.env.GOOGLE_API_KEY}`);
}
```

**Usage** — agents call Gemini CLI from Bash for tasks where Gemini's long context (2M tokens) is advantageous:
```bash
# Inside container Bash:
git diff HEAD~30 | gemini -p "Review this large diff and identify architectural regressions"
```

**Scheduled task example:**
```
@Andy every Monday at 8am, run a full code review of last week's changes
in /workspace/extra/myapp using Gemini CLI (better for large diffs),
save the report to /workspace/group/reports/week-$(date +%Y%W).md,
and send me the summary
```

---

### 3. GitHub Copilot CLI (`gh copilot`)

**Installation** — add to `container/Dockerfile`:
```dockerfile
RUN apt-get update && apt-get install -y gh && rm -rf /var/lib/apt/lists/*
RUN gh extension install github/gh-copilot
```

**Authentication** — mount your GitHub CLI config read-only:
```json
{
  "hostPath": "~/.config/gh",
  "containerPath": "/home/node/.config/gh",
  "readonly": true
}
```

`gh` reads from this directory and finds your stored OAuth token.

**Usage** — inside container Bash:
```bash
gh copilot suggest "write a GitHub Actions workflow that runs tests on PR open"
gh copilot explain "$(cat /workspace/extra/myapp/src/auth.ts)"
```

**Scheduled task example — automated code explanation:**
```
@Andy every Friday, for each new file added to /workspace/extra/myapp this week,
use gh copilot explain to generate inline documentation and append it
to /workspace/group/docs/code-explanations.md
```

**PR automation example:**
```
@Andy check every 15 minutes if any PR in my-org/my-repo has been open
for more than 2 days with no review. If so, use gh copilot explain
on the diff and post a summary as a PR comment to help reviewers.
```

---

### 4. Jules (Google's Async Coding Agent)

Jules is a GitHub-integrated async coding agent. It does not have a standalone headless CLI — it operates by picking up GitHub issues assigned to it in connected repositories. The integration with NanoClaw is therefore **via the GitHub API and `gh` CLI**.

**Integration pattern:**

1. Connect Jules to your GitHub repo in the Jules dashboard
2. Create a NanoClaw scheduled task that generates and assigns GitHub issues to Jules
3. Optionally, poll for Jules' PRs and report back

**Setup** — install `gh` CLI in the container (see above). Jules uses its own GitHub identity, so you just need `gh` auth for your own account.

**Scheduled task example — escalation pipeline:**
```
@Andy every day at 9am, run the test suite in /workspace/extra/myapp.
If tests fail and you cannot fix them within 10 minutes:
  1. Create a GitHub issue with gh issue create --title "Auto: failing tests"
     and include the full stack traces in the body
  2. Label the issue "jules-task" so Jules picks it up
  3. Send me the issue URL
```

The agent runs tests, attempts fixes using its own Edit/Bash tools, and if it can't resolve them escalates to Jules — creating a two-tier autonomous coding loop.

**Polling for Jules' output:**
```
@Andy check every 30 minutes if any PR from Jules has been opened
in my-org/myapp. When one appears, review the diff and leave a
comment with your assessment, then notify me.
```

**Key `gh` commands available inside the container:**
```bash
gh issue create --title "..." --body "..." --label "jules-task"
gh pr list --author "@github-app-jules[bot]"
gh pr view <number> --json body,files
gh pr review <number> --comment --body "..."
```

---

### 5. Kiro (AWS Agentic IDE)

Kiro is a VS Code-based agentic IDE that operates on spec files (`.kiro/specs/`). It does not have a standalone headless CLI. The integration approaches are:

**Approach A — Spec file pipeline (recommended):**

Have NanoClaw agents generate Kiro-compatible spec files and commit them to a branch that Kiro monitors.

Kiro spec structure (`.kiro/specs/<feature>/`):
```
requirements.md   — user stories in EARS format
design.md         — technical design
tasks.md          — implementation tasks checklist
```

**Scheduled task example:**
```
@Andy every Monday, check GitHub issues labeled "kiro-task" in my-org/myapp.
For each one, generate a Kiro spec with requirements.md, design.md, and tasks.md
files based on the issue description. Commit them to the kiro-specs branch
and notify me with the list of spec files created.
```

The agent uses its file tools to write the specs, then `git commit && git push` via Bash.

**Approach B — Agent-browser automation:**

NanoClaw already includes `agent-browser` (Chromium in the container). If Kiro exposes a web interface, an agent can interact with it via browser automation:

```bash
# Available inside any container:
agent-browser --url "https://kiro.aws/..." --task "Create a new spec for feature X"
```

**Approach C — Watch for Kiro CLI releases:**

AWS is actively developing Kiro. As of 2026, watch for a `kiro` CLI package. When available, add to Dockerfile:
```dockerfile
RUN npm install -g @aws/kiro-cli   # when released
```

And mount your AWS credentials:
```json
{
  "hostPath": "~/.aws",
  "containerPath": "/home/node/.aws",
  "readonly": true
}
```

---

### Multi-Agent Coding Pipeline Blueprint

This is a complete autonomous coding pipeline using NanoClaw's existing primitives. It coordinates multiple coding agents in a single scheduled task:

```
Schedule: 0 9 * * 1-5  (9am weekdays)
Group: coding-workspace
Mounts:
  - ~/projects/myapp  →  /workspace/extra/myapp  (read-write)
  - ~/.config/gh      →  /home/node/.config/gh   (read-only)

Prompt:
  You are an autonomous software engineer. Every morning, do the following:

  1. Run: cd /workspace/extra/myapp && npm test 2>&1 | tee /workspace/group/today-test.log
  2. If all tests pass:
     - Run: git log --oneline --since="1 day ago" to summarize yesterday's work
     - Send me a green status summary via send_message
     - Done.
  3. If tests fail:
     a. Analyze the failures
     b. Attempt to fix them (you have up to 15 minutes)
     c. Re-run tests
     d. If green: git commit -am "Auto-fix: failing tests" && git push && gh pr create --draft
        Send me the PR URL.
     e. If still failing:
        gh issue create \
          --title "Auto: unresolved test failures $(date +%Y-%m-%d)" \
          --body "$(cat /workspace/group/today-test.log)" \
          --label "jules-task,needs-review"
        Send me the issue URL.
  4. Save a summary to /workspace/group/daily-log/$(date +%Y-%m-%d).md
```

This single task creates a complete autonomous coding loop:

```
9am every weekday
       │
       ▼
Claude Code agent runs tests
       │
  ┌────┴────┐
  │         │
pass      fail
  │         │
  ▼         ▼
Summary  Try to fix (Claude Code)
sent         │
         ┌───┴───┐
         │       │
       fixed   still failing
         │       │
         ▼       ▼
       PR      Jules issue
      opened   created
         │       │
         └───┬───┘
             ▼
        Notification
        sent to phone
```

---

### Key Extension Points for Coding Workflows

| Extension Point | File | What to Change |
|---|---|---|
| Install a new CLI | `container/Dockerfile` | Add `RUN npm install -g <tool>` |
| Pass API keys to containers | `src/container-runner.ts:buildContainerArgs()` | Add `-e KEY=value` args |
| Mount credentials read-only | Group `containerConfig.additionalMounts` | Add `~/.config/<tool>` mount |
| Add a new MCP tool | `container/agent-runner/src/ipc-mcp-stdio.ts` | Add tool definition |
| Change what tools agents have | `container/agent-runner/src/index.ts:402` | Edit `allowedTools` array |
| Per-group agent customization | `data/sessions/<group>/agent-runner-src/` | Modify the per-group copy |
| Schedule a coding task | From your messaging channel | Ask agent to schedule it |
| Mount a code repository | Group `containerConfig.additionalMounts` | Add `~/projects/<repo>` mount |

### Security Considerations for Coding Tasks

**Scope mounts tightly.** Mount only the specific project directory, not all of `~/projects`. Use read-only unless the agent needs to write.

**Use deploy keys.** For `git push` access, generate a repo-specific deploy key and mount `~/.ssh/deploy_key_<repo>`. Do not mount your main `~/.ssh` — it contains keys to other systems.

**Scope GitHub tokens.** When mounting `~/.config/gh`, ensure your `gh auth` token only has permissions to the repos the coding group needs.

**Set task timeouts.** Long-running coding tasks need higher `containerConfig.timeout` values. The default is 30 minutes. Complex refactors may need more.

**Never mount `.env`.** The container-runner already shadows `.env` with `/dev/null` for the main group mount. But if you add a custom mount that includes a directory containing secrets, those will be readable by the agent.

**Audit IPC escalations.** When the agent calls `register_group` or `schedule_task` via IPC, the host validates the source directory. Non-main groups can only schedule for themselves. Review the IPC authorization logic in `src/ipc.ts:processTaskIpc()` before adding new IPC command types.
