---
name: hermes-provision
description: Deploy Hermes Agent (Nous Research) on a Hetzner / Ubuntu VPS. Combines OnlyTerp bootstrap (bare-metal Hermes) with a Camofox-VNC Docker container for browser automation, then wires Discord and OAuth-based providers (ChatGPT subscription via openai-codex, Gemini, etc.). Triggers include "deploy hermes", "hermes on hetzner", "hermes setup", "install hermes agent", "hermes camofox vnc".
allowed-tools:
  - Bash
---

# Hermes Agent Hetzner Deploy

Provision and deploy a Hermes Agent instance on a Hetzner Cloud (or any Ubuntu) VPS, with optional Camofox-VNC browser stack and Discord integration.

## Safety Rules

- **NEVER execute delete commands.** `hcloud server delete`, `rm -rf`, `docker rm`, `systemctl disable --now` on other services are forbidden without explicit user approval.
- **NEVER expose or log API tokens, OAuth secrets, Discord bot tokens.**
- **ALWAYS ask for confirmation** before create/modify operations on a live server. Present the command and wait for approval.
- **ALWAYS suggest a snapshot** before changes on an existing Hetzner box:

```bash
hcloud server create-image <server> --type snapshot --description "Backup before Hermes install"
```

- **WARN before running bootstrap on a server with other services** — bootstrap resets UFW and may collide with existing reverse proxies on 80/443. Verify in Step 3.

## Prerequisites

- Ubuntu 22.04 / 24.04 fresh box (Debian 12 also works) with root or sudo
- SSH connectivity from the local machine
- Outbound 443 reachable (GitHub, OpenAI, Discord, Anthropic, Camoufox releases)
- RAM ≥ 4 GB recommended (Hermes ~150 MB + Camofox idle ~120 MB + browser active overhead)
- For new server creation only: `hcloud` CLI configured (see `clawpod-provision` skill Step 0)

## Workflow

### Step 1: Server Selection / Creation

Branch:

- **A: Existing server** → ask for `<ip>` and SSH key path, then Step 2
- **B: New Hetzner server** → defer to `clawpod-provision` Step 2 to create a `cpx22 ubuntu-24.04 nbg1` box, then continue here

```bash
hcloud server list
```

### Step 2: Server Diagnostics

Before any change, inspect the box. **This is critical** — bootstrap resets UFW and installs Caddy; conflicts with existing services must be surfaced first.

```bash
ssh -o StrictHostKeyChecking=no root@<ip> "\
  echo '=== OS ===' && cat /etc/os-release | head -3 && \
  echo '=== Memory ===' && free -h | head -2 && \
  echo '=== Disk ===' && df -h / | tail -1 && \
  echo '=== Listening ports (public) ===' && ss -tlnp 2>/dev/null | awk '\$4 ~ /^0\\.0\\.0\\.0|^\\[::\\]/' && \
  echo '=== UFW ===' && (ufw status 2>/dev/null || echo 'ufw not active') && \
  echo '=== Existing services ===' && systemctl is-active caddy nginx docker hermes hermes-gateway clawpod 2>/dev/null && \
  echo '=== Node ===' && (node --version 2>/dev/null || echo 'not installed')"
```

Report findings to the user. Pay attention to:

- **Other services on 80/443** → bootstrap's Caddy install will conflict
- **Existing UFW rules beyond 22** → bootstrap will reset and only allow 22/80/443
- **Existing `hermes` user** → may be from prior install
- **RAM < 4 GB** → flag risk

Pause and get explicit user approval before proceeding.

### Step 3: OnlyTerp Bootstrap

Bootstrap installs system packages, creates the `hermes` user, locks down with UFW + fail2ban, and tries to install Hermes (which is expected to fail — handled in Step 4).

```bash
ssh root@<ip> "curl -sSL https://raw.githubusercontent.com/OnlyTerp/hermes-optimization-guide/main/scripts/vps-bootstrap.sh \
    -o /tmp/vps-bootstrap.sh && bash /tmp/vps-bootstrap.sh"
```

Timeout: 600s.

What bootstrap does:

- apt: `curl ca-certificates gnupg jq git python3-venv age rclone ufw fail2ban unattended-upgrades` + Node.js 20 + Caddy
- Creates user `hermes` (password disabled)
- Writes `/home/hermes/.hermes/{config.yaml,.env,skills,logs,lightrag}` and `/etc/caddy/Caddyfile.hermes.reference`
- Installs systemd units `hermes.service`, `hermes-dashboard.service` (broken — replaced in Step 5)
- Resets UFW, allows 22/80/443 only
- Enables fail2ban and unattended-upgrades

> **Important**: The bootstrap's "Installing Hermes" step always fails with `curl: (35) ... tlsv1 unrecognized name` because the upstream `install.hermes.nous.ai` host returns SNI errors. Continue to Step 4.

### Step 4: Hermes Install (official path)

Use the canonical Nous Research installer as the `hermes` user, from `hermes`'s home (uv needs write access):

```bash
ssh root@<ip> 'sudo -u hermes -i bash -c "cd ~ && curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash"'
```

Timeout: 600s.

The installer:
- Installs `uv` package manager
- Installs Python 3.11 via uv
- Installs Hermes binary at `~/.local/bin/hermes`
- Lays out `~/.hermes/hermes-agent/` (project code) and links bundled skills

Verify:

```bash
ssh root@<ip> "sudo -u hermes -i hermes --version"
```

### Step 5: Replace Broken systemd Unit

OnlyTerp's `hermes.service` calls `hermes run` (a non-existent subcommand in 0.12+). Remove it and install the official `hermes-gateway.service`:

```bash
ssh root@<ip> "systemctl stop hermes hermes-dashboard 2>/dev/null; \
  systemctl disable hermes hermes-dashboard 2>/dev/null; \
  rm -f /etc/systemd/system/hermes.service /etc/systemd/system/hermes-dashboard.service; \
  systemctl daemon-reload; \
  ln -sf /home/hermes/.local/bin/hermes /usr/local/bin/hermes; \
  hermes gateway install --system --run-as-user hermes --force"
```

Verify the unit was generated:

```bash
ssh root@<ip> "systemctl cat hermes-gateway.service | head -20"
```

### Step 6: Configure .env (Discord + Camofox placeholders)

Append Discord and Camofox env vars to the `.env` that bootstrap created. Existing keys (Anthropic/Google/Telegram from bootstrap) are preserved.

```bash
ssh root@<ip> "sudo -u hermes tee -a /home/hermes/.hermes/.env > /dev/null <<'EOF'

# Discord (fill in before starting hermes-gateway)
DISCORD_BOT_TOKEN=
DISCORD_ALLOWED_USERS=
DISCORD_FREE_RESPONSE_CHANNELS=

# Camofox (set after the container is running)
CAMOFOX_URL=http://127.0.0.1:9377
EOF"
```

> **Important**: Never paste real tokens into this skill's commands. Ask the user to add them via `sudo -u hermes -i nano /home/hermes/.hermes/.env` in a later interactive step, or via the Hermes setup wizard.

### Step 7: Provider Auth (OAuth — ChatGPT subscription)

Hermes supports OAuth providers that do **not** require a paid API key:

| Provider | Subscription |
|---|---|
| `openai-codex` | ChatGPT Plus / Pro |
| `gemini` (oauth_enabled) | Google account (free tier) |
| `qwen-oauth` | Qwen Portal |
| `copilot` | GitHub Copilot |

The setup wizard is interactive (device flow OAuth). The user must run it themselves in their SSH terminal:

```bash
# (User runs this from their own terminal)
ssh root@<ip>
sudo -u hermes -i hermes setup model
# → choose openai-codex (or gemini / qwen-oauth / copilot)
# → open the printed URL in a local browser and approve
```

Verify auth completion:

```bash
ssh root@<ip> "sudo -u hermes -i hermes auth status openai-codex"
```

Expect `openai-codex: logged in`.

### Step 8: Install Docker (for Camofox)

```bash
ssh root@<ip> "curl -fsSL https://get.docker.com | sh && docker --version && systemctl is-active docker"
```

Timeout: 300s.

### Step 9: Build Camofox Image (with VNC patch)

`jo-inc/camofox-browser` has no public Docker image. Source-build it. The upstream Dockerfile does NOT install the VNC plugin's apt deps — patch it.

```bash
ssh root@<ip> "apt-get install -y make git >/dev/null 2>&1 && \
  cd /opt && git clone https://github.com/jo-inc/camofox-browser.git && \
  cd camofox-browser && \
  sed -i '/python3-minimal \\\\\$/i\\    # VNC stack (x11vnc + noVNC + websockify) for browser remote access\\n    x11vnc \\\\\\n    novnc \\\\\\n    python3-websockify \\\\\\n    procps \\\\' Dockerfile"
```

Fetch upstream binaries (Camoufox zip ~680 MB + yt-dlp ~35 MB) and build the image:

```bash
ssh root@<ip> "cd /opt/camofox-browser && make fetch"
```

Timeout: 600s.

```bash
ssh root@<ip> "cd /opt/camofox-browser && make build"
```

Timeout: 900s.

Verify the image:

```bash
ssh root@<ip> "docker images | grep camofox-browser"
```

### Step 10: Run Camofox Container

Bind every port to **127.0.0.1**. External access is via SSH tunnel only.

```bash
ssh root@<ip> "IMG=\$(docker images --format '{{.Repository}}:{{.Tag}}' | grep '^camofox-browser:' | head -1) && \
  docker run -d --restart unless-stopped --name camofox-browser \
    -p 127.0.0.1:9377:9377 \
    -p 127.0.0.1:6080:6080 \
    -p 127.0.0.1:5900:5900 \
    -e ENABLE_VNC=1 \
    -e VNC_BIND=0.0.0.0 \
    -e VNC_RESOLUTION=1920x1080 \
    -e MAX_OLD_SPACE_SIZE=1024 \
    -v /home/hermes/.camofox-data:/root/.camofox \
    \$IMG"
```

Smoke test:

```bash
ssh root@<ip> "sleep 5 && \
  curl -s http://127.0.0.1:9377/health && echo && \
  curl -s -o /dev/null -w 'noVNC HTTP %{http_code}\\n' http://127.0.0.1:6080/vnc.html"
```

Expect Camofox health `ok:true, browserConnected:true` and noVNC HTTP 200.

### Step 11: Configure Discord Gateway

The user must create a Discord application + bot first:

1. https://discord.com/developers/applications → New Application → Bot → Reset Token (copy the long token)
2. Enable **Privileged Gateway Intents**: Message Content Intent, Server Members Intent
3. OAuth2 → URL Generator → scopes `bot` + `applications.commands`, Bot Permissions `Send Messages / Read Message History / Use Slash Commands`. Open the URL to invite the bot to a server.
4. Capture their own Discord user ID (Discord app → Settings → Advanced → Developer Mode → right-click avatar → Copy User ID).

User runs the gateway setup interactively:

```bash
# (User runs this from their own terminal)
ssh root@<ip>
sudo -u hermes -i hermes setup gateway
# → choose `discord` (space to select, Enter)
# → paste bot token
# → paste allowed user IDs
```

### Step 12: Start hermes-gateway and Sanity Check

```bash
ssh root@<ip> "systemctl restart hermes-gateway && sleep 6 && \
  systemctl status hermes-gateway --no-pager | head -10 && \
  journalctl -u hermes-gateway -n 20 --no-pager"
```

Expect `Active: active (running)` and the only WARNING line about Opus codec (irrelevant for text bots).

### Step 13: Prevent system/user systemd Conflict (Critical)

After `hermes setup gateway` runs as `hermes` user with `Linger=yes`, a **duplicate user-level** `hermes-gateway.service` can spawn. Both then fight via `gateway run --replace`, producing Discord replies like `Gateway shutting down — Your current task will be interrupted.` on every message.

Disable lingering and kill any duplicate process:

```bash
ssh root@<ip> "loginctl disable-linger hermes && \
  pkill -u hermes -f 'hermes_cli.main gateway' 2>/dev/null; \
  systemctl restart hermes-gateway && sleep 5 && \
  echo === processes === && ps -ef | grep 'hermes_cli.main gateway' | grep -v grep"
```

Verify exactly **one** gateway process. If `hermes gateway status --system` warns `Both user and system gateway services are installed`, run the above again.

### Step 14: Report

Present to the user:

1. Server info (IP, RAM, disk free)
2. Hermes version + active provider (e.g. `openai-codex: logged in`)
3. Service status: `hermes-gateway` active, `camofox-browser` Docker container Up
4. UFW rules
5. Next steps:

```
Discord:
  Bot is online if /home/hermes/.hermes/.env has DISCORD_BOT_TOKEN set.
  DM the bot or @-mention it in a channel.

VNC (Camofox browser view) — from your local machine:
  ssh -N -L 6080:127.0.0.1:6080 root@<ip>
  Then open http://127.0.0.1:6080/vnc.html

Camofox API:
  ssh -N -L 9377:127.0.0.1:9377 root@<ip>
  curl http://127.0.0.1:9377/health

Useful commands on the server:
  systemctl status hermes-gateway
  journalctl -u hermes-gateway -f
  docker logs -f camofox-browser
  sudo -u hermes -i hermes auth status <provider>
  sudo -u hermes -i hermes update            # update Hermes
  cd /opt/camofox-browser && make reset      # rebuild Camofox
```

## Update Workflow

### Update Hermes

```bash
ssh root@<ip> "sudo -u hermes -i hermes update && systemctl restart hermes-gateway"
```

### Update Camofox

```bash
ssh root@<ip> "cd /opt/camofox-browser && git pull && make reset"
```

`make reset` stops + removes the container, deletes the image, then rebuilds. After rebuild, re-run the Step 10 `docker run` command (since `make reset` does not relaunch with our env flags).

## Troubleshooting

| Issue | Root Cause | Solution |
|-------|-----------|----------|
| Bootstrap exits with `tlsv1 unrecognized name` on "Installing Hermes" | Upstream `install.hermes.nous.ai` SNI broken | Expected. Continue to Step 4 (official installer). |
| `hermes` command not found in systemd unit | OnlyTerp unit hardcodes `/usr/local/bin/hermes`, installer puts it in `~/.local/bin/` | Step 5 creates the symlink. |
| `hermes: error: argument command: invalid choice: 'run'` | OnlyTerp unit calls obsolete `hermes run` | Step 5 replaces unit with `hermes gateway install --system`. |
| Discord replies `Gateway shutting down` on every message | Both system + user `hermes-gateway` instances active, fighting via `--replace` | Step 13: `loginctl disable-linger hermes` + kill duplicate. |
| Camofox container starts but VNC unreachable | Upstream Dockerfile missing `x11vnc / novnc / websockify` | Step 9 patches Dockerfile before build. |
| `hermes setup model` says "no terminal available" | Run inside SSH multiplexed / non-TTY session | User must run interactively from their own terminal. |
| `hermes auth status openai-codex` shows `not logged in` | OAuth device flow not completed | Re-run `hermes setup model` and approve in browser. |
| RAM exhausted during Camofox build | LTO + Camoufox unzip on 2 GB box | Use `cpx22` (4 GB) or larger; do not run on `cpx11`. |
| Caddy fails to bind 80 | Existing reverse proxy on the box | Skip Caddy use, or stop the other proxy first. The bootstrap installs Caddy but does not start an active reverse-proxy unless you place a real `Caddyfile`. |

## Provider Reference (OAuth no-key options)

Hermes plugin paths under `~/.hermes/hermes-agent/plugins/model-providers/`:

- `openai-codex/` → `base_url=https://chatgpt.com/backend-api/codex`, `auth_type=oauth_external`, `env_vars=()`. Uses the ChatGPT subscription via Codex CLI device flow.
- `gemini/` → set `providers.google.oauth_enabled: true` in `config.yaml`. Uses Google account free tier.
- `qwen-oauth/` → Qwen Portal OAuth.
- `copilot/` → GitHub Copilot OAuth.

For paid API providers (Anthropic, OpenAI raw API), set the corresponding `*_API_KEY` in `~/.hermes/.env` instead of using a wizard.

## Reference

- OnlyTerp bootstrap: https://github.com/OnlyTerp/hermes-optimization-guide
- Hermes docs: https://hermes-agent.nousresearch.com/docs/
- Camofox upstream: https://github.com/jo-inc/camofox-browser
- Companion runbook (this repo): `docs/20260507_hermes_hetzner_deployment_runbook.md`
