# Installing NanoClaw on Raspberry Pi 5 (8GB)

The Raspberry Pi 5 with 8GB RAM is a solid choice for running NanoClaw as a lightweight, always-on personal assistant. It runs Linux with Docker — both fully supported by NanoClaw.

## Prerequisites

- **Raspberry Pi 5 (8GB)** with Raspberry Pi OS (64-bit, Bookworm) or Ubuntu 24.04 LTS arm64
- MicroSD card (32GB+) or NVMe SSD (recommended for performance)
- Network connection (Ethernet recommended for reliability)
- An **Anthropic API key** (from [console.anthropic.com](https://console.anthropic.com))
- A **Claude Code subscription** (Claude Max, Pro, or Team plan)

## Step 1: Update the System

```bash
sudo apt update && sudo apt upgrade -y
```

## Step 2: Install Node.js 22+

NanoClaw requires Node.js 20+. Install v22 (LTS) via NodeSource:

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
```

Verify:

```bash
node --version   # Should show v22.x.x
npm --version
```

## Step 3: Install Build Tools

`better-sqlite3` (a NanoClaw dependency) compiles native C++ code. You'll need build essentials:

```bash
sudo apt install -y build-essential python3 git
```

## Step 4: Install Docker

Docker is the container runtime on Linux. Install it via the official convenience script:

```bash
curl -fsSL https://get.docker.com | sudo sh
```

Add your user to the `docker` group so you don't need `sudo`:

```bash
sudo usermod -aG docker $USER
```

**Log out and back in** (or run `newgrp docker`) for the group change to take effect.

Verify:

```bash
docker run --rm hello-world
```

## Step 5: Install Claude Code

Claude Code is the CLI that drives NanoClaw's setup and customization:

```bash
npm install -g @anthropic-ai/claude-code
```

Verify:

```bash
claude --version
```

Log in to Claude Code:

```bash
claude auth login
```

Follow the prompts to authenticate with your Anthropic account.

## Step 6: Fork and Clone NanoClaw

Fork the repo on GitHub first (so you can customize it), then clone your fork:

```bash
# With GitHub CLI:
gh repo fork qwibitai/nanoclaw --clone
cd nanoclaw

# Or manually:
git clone https://github.com/<your-username>/nanoclaw.git
cd nanoclaw
```

## Step 7: Install Dependencies

```bash
npm install
```

This will compile `better-sqlite3` natively for ARM64. It takes a bit longer on the Pi than on a desktop — this is normal.

## Step 8: Create the `.env` File

```bash
cat > .env << EOF
ANTHROPIC_API_KEY=sk-ant-xxxxxxxxxxxxxxxxxxxxx
ASSISTANT_NAME=Andy
EOF
```

Replace the API key with your actual key. Change `Andy` to whatever trigger name you prefer.

## Step 9: Run Setup with Claude Code

This is the AI-native part — Claude Code handles the rest of the setup interactively:

```bash
claude
```

Inside the Claude Code prompt, run:

```
/setup
```

Claude will:
1. Build the TypeScript project (`npm run build`)
2. Build the agent container image (`container/build.sh`) — this takes a while on the Pi (10-20 min on first build due to Chromium and npm packages on ARM64)
3. Configure the systemd service for auto-start
4. Guide you through channel setup (WhatsApp, Telegram, etc.)

> **Note:** The container image build (`docker build`) pulls `node:22-slim` for ARM64 and installs Chromium. The ARM64 Chromium package is larger and slower to install than on x86. Be patient.

## Step 10: Add a Messaging Channel

Still inside `claude`, add whichever channel you want:

- **WhatsApp:** `/add-whatsapp` — authenticates via QR code or pairing code
- **Telegram:** `/add-telegram` — uses a BotFather token
- **Discord:** `/add-discord`
- **Slack:** `/add-slack`

Follow Claude's prompts to complete channel authentication.

## Step 11: Start NanoClaw

If `/setup` configured the systemd service, start it:

```bash
systemctl --user start nanoclaw
systemctl --user enable nanoclaw   # Auto-start on boot
```

Or run manually for testing:

```bash
npm start
```

Check that it's running:

```bash
systemctl --user status nanoclaw
```

## Step 12: Enable Lingering (Important for Headless Use)

By default, systemd user services stop when you log out. Enable lingering so NanoClaw runs even without an active SSH session:

```bash
sudo loginctl enable-linger $USER
```

## Troubleshooting

### Container build fails with out-of-memory

The 8GB Pi can struggle if Chromium compilation or npm install is too aggressive:

```bash
# Limit Docker build memory
docker build --memory=4g -t nanoclaw-agent:latest container/
```

Or increase swap:

```bash
sudo dphys-swapfile swapoff
sudo sed -i 's/CONF_SWAPSIZE=.*/CONF_SWAPSIZE=2048/' /etc/dphys-swapfile
sudo dphys-swapfile setup
sudo dphys-swapfile swapon
```

### `better-sqlite3` fails to compile

Ensure build tools are installed:

```bash
sudo apt install -y build-essential python3
npm rebuild better-sqlite3
```

### Docker permission denied

Make sure your user is in the `docker` group and you've logged out/back in:

```bash
groups   # Should include 'docker'
```

### Container agent is slow

ARM64 containers are slower than x86 for CPU-bound tasks. The Pi 5 handles it fine for typical assistant workloads, but complex multi-step agent tasks may take longer than on a desktop. Consider:

- Using an NVMe SSD instead of a microSD card (huge I/O improvement)
- Setting a longer container timeout in `src/config.ts` if agents time out

### Claude Code can't authenticate

If you're running headless (SSH only), Claude Code's browser-based auth won't work. Authenticate on a machine with a browser first, then copy the credentials:

```bash
# On your desktop after authenticating:
scp -r ~/.claude pi@<pi-ip>:~/.claude
```

### Checking logs

```bash
journalctl --user -u nanoclaw -f          # Live systemd logs
cat groups/<group-folder>/logs/*.log       # Container run logs
```

## Performance Tips

1. **Use an NVMe SSD** — The biggest single improvement. MicroSD I/O is the bottleneck for SQLite, Docker, and container launches.
2. **Keep swap enabled** — Agent containers with Chromium can spike memory usage.
3. **Use Telegram over WhatsApp** — Telegram's bot API is lighter weight than WhatsApp's web protocol, using less memory and CPU.
4. **Disable browser automation if unused** — If you don't need web browsing in agents, you can create a slimmer Dockerfile without Chromium, saving ~500MB RAM and significant build time.

## Architecture on the Pi

```
Raspberry Pi 5 (ARM64, 8GB)
+-- systemd user service
|   +-- NanoClaw (Node.js process, ~100MB RSS)
|       +-- Channel adapter (WhatsApp/Telegram/etc.)
|       +-- SQLite database
|       +-- Task scheduler
|       +-- Container spawner
|           +-- Docker
|               +-- nanoclaw-agent containers (ARM64 Linux)
|                   +-- Claude Agent SDK -> Anthropic API
```

The Pi acts as a thin orchestrator. The heavy lifting (LLM inference) happens on Anthropic's servers via the API. The Pi just manages messaging, scheduling, and container lifecycle — all well within its capabilities.
