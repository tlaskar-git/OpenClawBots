# OpenClaw BOT — Deployment Guide

Complete end-to-end guide to deploy an OpenClaw BOT instance on a dedicated Ubuntu VM.

> **Prerequisites:** A fresh Ubuntu 22.04+ VM, a Microsoft 365 account with OneDrive, a Telegram bot token, and API keys for your chosen AI models (Gemini and/or Claude).

---

## Phase 1: Infrastructure

This phase prepares the OS, installs the OpenClaw engine, and establishes the cloud connection.

### 1.1 Update and Install Core Tools

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install python3.12 rclone fuse3 ufw curl -y
```

### 1.2 Security and Networking

> **Important:** Replace the example subnets (`10.10.1.0/24`, `10.20.1.0/24`) with your own trusted network ranges.

```bash
sudo ufw default deny incoming
sudo ufw allow from 10.10.1.0/24 to any port 22
sudo ufw allow from 10.20.1.0/24 to any port 22
sudo ufw allow from 10.10.1.0/24 to any port 3000
sudo ufw allow from 10.20.1.0/24 to any port 3000
sudo ufw allow from 10.10.1.0/24 to any port 3389
sudo ufw allow from 10.20.1.0/24 to any port 3389
sudo ufw enable
```

### 1.3 XFCE and XRDP Installation (Optional)

Only needed if you want a desktop GUI on the VM. The rclone OneDrive authentication step requires a browser, so this helps if you are authenticating directly on the VM.

```bash
sudo apt install xfce4 xfce4-goodies -y
sudo apt install xrdp -y
sudo adduser xrdp ssl-cert
echo "startxfce4" > ~/.xsession
sudo systemctl restart xrdp
sudo apt install qemu-guest-agent -y
sudo systemctl enable --now qemu-guest-agent
```

### 1.4 Mount OneDrive Vault

**Before starting:** Create the `bot` folder in your OneDrive root first.

```bash
mkdir -p ~/OneDrive/bot
rclone config
```

Follow the on-screen instructions to connect your OneDrive account. Then mount:

```bash
rclone mount OneDrive:bot ~/OneDrive/bot --vfs-cache-mode full --daemon
```

### 1.5 Install Node.js 22

Required for the OpenClaw engine.

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
```

### 1.6 Install OpenClaw Engine

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

During setup, choose **Manual Config** and set:
- **Workspace port:** Use a nonstandard port (e.g., 3000)
- **Workspace path:** Set to the following for OneDrive sync to work:

```
/home/bot/OneDrive/bot/instances/[BOT-NAME]/workspace
```

---

## Phase 2: BOT Sync to OneDrive

This phase migrates secrets to OneDrive and sets up the sync job.

### 2.1 Migrate Secrets

```bash
mkdir -p ~/OneDrive/bot/instances/[BOT-NAME]/secrets/
mv ~/.openclaw/openclaw.json ~/OneDrive/bot/instances/[BOT-NAME]/secrets/secrets.json
ln -s ~/OneDrive/bot/instances/[BOT-NAME]/secrets/secrets.json ~/.openclaw/openclaw.json
```

### 2.2 Lock Down Permissions

```bash
chmod 600 ~/OneDrive/bot/instances/[BOT-NAME]/secrets/secrets.json
```

Verify:

```bash
ls -l ~/OneDrive/bot/instances/[BOT-NAME]/secrets/secrets.json
# Expected output: -rw------- (Permission 600)
```

### 2.3 Set Up Sync Job (Cron)

```bash
crontab -e
```

Add this line at the bottom:

```
*/10 * * * * rclone sync /var/lib/openclaw/ ~/OneDrive/bot/instances/[BOT-NAME]/memory/
```

This syncs local databases to OneDrive every 10 minutes.

---

## Phase 3a: Initial Configuration

Run these commands to expose the gateway on your LAN:

```bash
openclaw config set gateway.bind lan
openclaw config set gateway.controlUi.allowInsecureAuth true
```

---

## Phase 3b: Master Bot Instruction Block

Copy and paste the following into your OpenClaw BOT as the system prompt. Replace `[USERID]` with your Telegram User ID.

> **How to get your Telegram User ID:** Message the `@userinfobot` on Telegram. It will reply with your numeric User ID.

---

### System Prompt

```
Your role is an independent agent on this VM. You operate on a dedicated Ubuntu VM
with full local rights but follow strict operational guardrails set below.
```

### Rule 1: Telegram Lockdown

Lock bot access to your Telegram User ID only.

```
Change config to allowlist (lock to my ID [USERID]).
```

### Rule 2: Model Selection Logic

> Note: This configuration uses Claude and Gemini models. Adjust as needed.

| Category | Primary | Fallback | When |
|----------|---------|----------|------|
| Heartbeats | Gemini 1.5 Flash | Claude 3 Haiku | Every 60 minutes for health checks |
| Sub-agents | Gemini 1.5 Pro | Claude 3.5 Sonnet | "Advanced Reasoning" requests only |
| Simple Queries | Gemini 1.5 Flash | Claude 3 Haiku | Calendar, file reads, basic chat |

### Rule 3: Secret File Security

- Set permissions for `secrets.json` to `600` during onboarding.
- Ensure only your specific bot process reads these keys.
- Confirm the restricted file status before attempting to migrate tokens.

### Rule 4: Secret Access and Approval

- Secrets remain isolated. The bot has no permission to share or read keys from other bots.
- Request access to API keys for every new session.
- To unlock a key, send a Telegram message to User ID `[USERID]`: *"Requesting [Key Name]. Reply /Approve or /Deny"*.
- Keys remain in process memory for 300 seconds (5 minutes).
- Wipe keys from process memory immediately after this window expires.

### Rule 5: Expensive Model Usage (Sonnet or Opus)

When a task requires an expensive model:

1. Send a Telegram message to User ID `[USERID]`: *"Requesting to use [Model Name] for [Task]. Reply /Approve or /Deny"*.
2. Proceed with the expensive model only if you receive `/approve`.
3. Otherwise, continue with the default model.

### Rule 6: Elevated Privileges (Sudo / Admin Tasks)

1. Send a Telegram message to User ID `[USERID]`: *"Requesting to perform [Task Details] for [Reason]. Reply /Approve or /Deny"*.
2. Proceed only if you receive `/approve`. Otherwise, abort.

### Rule 7: Efficiency and Token Control

- Apply the QMD Skill for all local searching.
- Index Markdown files using keywords and vector search.
- Send only the most relevant snippets to AI models instead of full documents.
- Apply the Session Initialisation Rule at startup.
- Load only `MEMORY.md` and `USER.md` to keep startup context at ~8KB.

### Rule 8: Folder Hierarchy and Storage

| Path | Use |
|------|-----|
| `/opt/bot/` | Engine logic and setup scripts |
| `/var/lib/openclaw/` | Active databases for high-speed search |
| `~/OneDrive/bot/` | Cloud vault for secrets, shared memory, and backups |

### Rule 9: Safety and Monitoring

- Monitor the global KILL switch at `~/OneDrive/bot/global/KILL` every 5 seconds.
- Shut down the instance within 5 seconds if this file exists.
- Send a heartbeat check to the user every 60 minutes to verify system health.

### Rule 10: Guardrail Change Approval

Any changes that modify, disable, or bypass any of these rules require approval:

1. Send a Telegram message to User ID `[USERID]`: *"Guardrails change required: [Task Details] for [Reason]. Reply /approve or /Deny"*.
2. Proceed only if you receive `/approve`. Otherwise, abort.
