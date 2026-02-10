# OpenClaw BOT — Design Document

## Overview

OpenClaw BOT uses a **Distributed Autonomous VM Fleet** architecture. Every bot runs as an independent worker on its own dedicated Linux VM. If one bot fails, the rest of the fleet keeps running.

Each agent communicates with the user via Telegram and shares knowledge through a "Global Brain" stored on OneDrive, while keeping secrets completely isolated per instance.

## Architecture Diagram

![OpenClaw BOT Architecture](./assets/architecture.png)

## Model Usage and Performance Logic

Every bot embeds this model selection logic to keep costs low and responses fast.

| Task Category | Primary Model | Fallback Model | Trigger |
|--------------|---------------|----------------|---------|
| Heartbeats | Gemini 1.5 Flash | Claude 3 Haiku | Runs every 60 minutes to check system health |
| Sub-agents | Gemini 1.5 Pro | Claude 3.5 Sonnet | Triggered only for "Advanced Reasoning" requests |
| Simple Queries | Gemini 1.5 Flash | Claude 3 Haiku | Calendar tasks, local file reads, basic chat |

## Memory and Secret Architecture

Data is partitioned so each bot instance has complete privacy. **Secrets are never shared between bots.**

| Memory Type | Location | Access | Purpose |
|------------|----------|--------|---------|
| Global Brain | `~/OneDrive/bot/global/` | All Bots | Shared knowledge, master prompts, global KILL switch |
| Isolated Memory | `~/OneDrive/bot/instances/[BOT]/` | This Bot Only | Private chat history and bot-specific habits |
| Isolated Secrets | `~/OneDrive/bot/instances/[BOT]/secrets/` | This Bot Only | Private API keys and tokens |

## Network Security (UFW)

The firewall blocks all incoming traffic by default. Access is limited to trusted subnets of your choice.

| Service | Port | Protocol | Trusted Source Subnets | Action |
|---------|------|----------|----------------------|--------|
| SSH | 22 | TCP | 10.10.1.0/24, 10.20.1.0/24 | ALLOW |
| Bot UI / API | 3000 | TCP | 10.10.1.0/24, 10.20.1.0/24 | ALLOW |
| Outbound | Any | Any | All Destinations (Internet) | ALLOW |

> **Note:** Replace the example subnets with your own trusted network ranges.

## Directory Structure

The file system separates the engine logic from permanent data storage.

| Path | Purpose | Persistence |
|------|---------|-------------|
| `/opt/bot/` | **The Engine** — Bot logic and setup scripts | Ephemeral (Local) |
| `/var/lib/openclaw/` | **Active Memory** — Databases for high-speed search | Ephemeral (Local) |
| `~/OneDrive/bot/` | **The Vault** — Secrets, shared memory, and backups | Permanent (Cloud) |

## Deployment Process

New bots are provisioned using two templates:

**Step 1 — Infrastructure (Template 1):**
Updates Ubuntu, installs Python 3.12, configures the UFW firewall, and mounts the OneDrive Vault.

**Step 2 — Onboarding (Template 2):**
1. Run the terminal script and enter the Telegram token.
2. **Secret Migration:** Move the configuration from `~/.openclaw/openclaw.json` to the isolated secret folder at `~/OneDrive/bot/instances/[BOT-NAME]/secrets/secrets.json`.
3. **Symbolic Linking:** Create a symlink from the local application folder to the OneDrive secret file to maintain a single source of truth.

**Step 3 — Instructions:**
Provide specific instructions. Set up Claude/Gemini tokens during this stage. The bot begins its heartbeat cycle.

> See [DEPLOYMENT.md](./DEPLOYMENT.md) for the full step-by-step commands.

## Operational Guardrails

**The Approval Rule:**
If a bot needs an expensive API key or model (e.g., Claude 3.5 Sonnet), it sends a Telegram message to User ID `[USERID]`. The bot verifies this specific ID and proceeds only after receiving an `/approve` response.

**The Rule Change Approval:**
Any attempt by the bot to modify, disable, or bypass established guardrails requires explicit Telegram approval from the user.

**Secret Lockdown:**
During onboarding, the bot migrates secrets to `~/OneDrive/bot/instances/[BOT]/secrets/secrets.json` and applies `chmod 600` (read/write for the owner only).

**Ephemeral-to-Cloud Persistence:**
The bot operates on local NVMe storage for speed (`/var/lib/openclaw/`). A cron job executes an `rclone sync` every 10 minutes to back up all active databases to the permanent OneDrive vault.

**Global Kill Switch:**
The bot monitors `~/OneDrive/bot/global/KILL` every 5 seconds. If this file is detected, the instance shuts down within 5 seconds for fleet-wide safety.

## Efficiency Skills (Cost Reduction)

### QMD Skill (Local Searching)

The QMD Skill avoids expensive full document reads. It indexes local Markdown files using keywords and vector search, then retrieves only the most relevant sentences and sends these snippets to the AI.

**Result:** Reduces the token bill by ~95%.

### Session Initialisation Rule

Standard bots often load large (~50KB) histories at the start of every chat. This rule restricts the bot to loading only `MEMORY.md` and `USER.md` at startup.

**Result:** Startup size drops to ~8KB, reducing costs per session.

---

## Appendix A: Directory and Path Reference

| Component | Local Path (Ephemeral) | Cloud Path (Permanent) | Purpose |
|-----------|----------------------|----------------------|---------|
| Secrets | `~/.openclaw/openclaw.json` (symlink) | `~/OneDrive/bot/instances/[BOT]/secrets/secrets.json` | API keys, Telegram tokens, auth |
| Databases | `/var/lib/openclaw/` | `~/OneDrive/bot/instances/[BOT]/memory/` | Search indices and high-speed memory |
| Workspace | `/opt/bot/workspace/` | `~/OneDrive/bot/instances/[BOT]/workspace/` | Active project files |
| Logs | `/var/log/openclaw/` | `~/OneDrive/bot/instances/[BOT]/logs/` | Private conversation history |
| Global Brain | N/A | `~/OneDrive/bot/global/` | Shared fleet knowledge and KILL switch |

## Appendix B: Critical Command Reference

| Action | Command | Expected Result |
|--------|---------|-----------------|
| Security Check | `ls -l ~/OneDrive/bot/instances/[BOT]/secrets/secrets.json` | Must show `-rw-------` (Permission 600) |
| Sync Check | `crontab -l` | Confirm the rclone sync job is listed |
| Service Status | `sudo systemctl status openclaw-gateway.service` | Status should be `active (running)` |
| Manual Sync | `rclone sync /var/lib/openclaw/ ~/OneDrive/bot/instances/[BOT]/memory/` | Forces an immediate database backup |
| Emergency Stop | `touch ~/OneDrive/bot/global/KILL` | All bots shut down within 5 seconds |

## Appendix C: Memory File Definitions

These files live in your `workspace/` and define how the bot interacts with you:

- **USER.md** — Stores your specific preferences, formatting rules, and professional persona requirements.
- **MEMORY.md** — Contains curated, long-term project context that survives across sessions.
- **secrets.json** — The primary store for your Telegram token and model provider keys.

## Appendix D: User ID Guardrails

All administrative and high-cost requests are locked to **User ID `[USERID]`**:

- **Sudo Access:** Requires `/approve` from this ID.
- **Model Upgrades:** Requires `/approve` to use Claude 3.5 Sonnet or Opus.
- **Rule Changes:** Any modification to the Master Instruction Block requires a Telegram request and approval from this ID.
