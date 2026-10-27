# Autonomous Agent Workspace & VM Bootstrap Template

This document is a **standalone, zero-assumption executable specification**. You can hand this file directly to any CLI-capable AI agent (`claude`, `codex`, `hermes`, or a fresh bootstrap runner) on a clean Ubuntu Server VM, or run its embedded scripts directly from the command line.

---

## 1. Execution Contract for the AI Agent

When executing this specification to build a VM workspace or provision a new agent, follow these five operational rules:

1. **Zero Hardcoded Assumptions**: Never assume a username (`v`, `root`), home path, hostname (`rcmprox`), IP address, or Proxmox storage pool (`local-zfs`). Always resolve paths and host targets dynamically from `/etc/agent-workspace/workspace.env` or `$WORKSPACE_ROOT/.env`.
2. **Pull-First, Generate-Fallback**: If `CENTRAL_TEMPLATE_REPO` is provided in the environment, clone/pull scripts and reference docs from that Git repository first. If offline or unset, generate all required scripts, configs, and reference docs verbatim from Section 4 and Section 5 of this file.
3. **Interactive Pre-Flight Only for Missing Secrets**: Inspect `.env` before running setup steps. Only prompt the human operator for values that are currently blank or set to `CHANGE_ME`. Never halt execution for paths or system variables that can be auto-detected.
4. **Enforce Progressive Disclosure (`AGENTS.md` < 2 KB)**: Keep always-on system instructions (`AGENTS.md` and `soul.md`) compact (< 500 tokens). Place detailed schemas, API docs, and troubleshooting procedures into dedicated files under `docs/` and `runbooks/`, linked via explicit conditional pointers (`WHEN [task] -> READ [path]`).
5. **Idempotent Execution**: Every script and command must be safe to re-run without duplicating entries, overwriting existing user vaults, or corrupting running databases.

---

## 2. Phase 0: Pre-Flight Environment & Secret Discovery

Run this script first on any target Ubuntu machine. It auto-detects system parameters, prompts only for missing repository URLs and API keys, and writes the master environment file to `$WORKSPACE_ROOT/.env`.

```bash
#!/usr/bin/env bash
# File: preflight.sh
# Purpose: Zero-assumption environment discovery and secret collection
set -euo pipefail

export TARGET_USER="${SUDO_USER:-$USER}"
export TARGET_HOME="$(eval echo "~$TARGET_USER")"
export WORKSPACE_ROOT="${WORKSPACE_ROOT:-$TARGET_HOME/workspace}"
ENV_FILE="$WORKSPACE_ROOT/.env"

mkdir -p "$WORKSPACE_ROOT"
touch "$ENV_FILE"
chmod 600 "$ENV_FILE"

# Helper to read or prompt for a required configuration value
ensure_var() {
  local var_name="$1"
  local prompt_text="$2"
  local default_val="${3:-}"
  local is_secret="${4:-false}"

  # Check if already present in ENV_FILE
  if grep -q "^${var_name}=" "$ENV_FILE"; then
    local current_val
    current_val="$(grep "^${var_name}=" "$ENV_FILE" | cut -d'=' -f2-)"
    if [[ -n "$current_val" && "$current_val" != "CHANGE_ME" ]]; then
      return 0
    fi
  fi

  local input_val=""
  if [[ "$is_secret" == "true" ]]; then
    read -rsp "$prompt_text: " input_val
    echo ""
  else
    if [[ -n "$default_val" ]]; then
      read -rp "$prompt_text [$default_val]: " input_val
      input_val="${input_val:-$default_val}"
    else
      read -rp "$prompt_text: " input_val
    fi
  fi

  # Remove old placeholder line if exists and write new value
  sed -i "/^${var_name}=/d" "$ENV_FILE"
  echo "${var_name}=\"${input_val}\"" >> "$ENV_FILE"
}

echo "=== Agent Workspace Pre-Flight Discovery ==="
ensure_var "WORKSPACE_ROOT" "Workspace root directory" "$WORKSPACE_ROOT"
ensure_var "DATA_PLANE_ROOT" "Persistent data plane directory (/srv)" "/srv"
ensure_var "CENTRAL_TEMPLATE_REPO" "Central Git repo for shared scripts/templates (leave blank to generate locally)" ""
ensure_var "OBSIDIAN_VAULT_REPO" "Git SSH/HTTPS URL for existing Obsidian Vault (leave blank to init fresh vault)" ""
ensure_var "PRIMARY_LLM_PROVIDER" "Primary reasoning provider (anthropic, openai, openrouter)" "openrouter"
ensure_var "PRIMARY_LLM_MODEL" "Primary reasoning model ID" "anthropic/claude-3.5-sonnet"
ensure_var "AUXILIARY_LLM_MODEL" "Cheap auxiliary model ID for titles/compression" "groq/llama-3.3-70b-versatile"
ensure_var "PRIMARY_API_KEY" "Primary LLM API Key" "" "true"
ensure_var "GROQ_API_KEY" "Groq API Key (for Whisper voice & fast auxiliary tasks)" "" "true"
ensure_var "TELEGRAM_BOT_TOKEN" "Telegram Bot Token (from @BotFather, optional)" "" "true"
ensure_var "TELEGRAM_ALLOWED_USERS" "Comma-separated numeric Telegram User IDs allowed to command agent" ""
ensure_var "PROXMOX_SSH_TARGET" "Optional Proxmox SSH user@host (e.g., admin@100.x.y.z, leave blank if standalone)" ""
ensure_var "PROXMOX_STORAGE_POOL" "Default Proxmox storage pool name if using PVE" "local-lvm"

echo "Pre-flight configuration saved to $ENV_FILE"
```

---

## 3. Phase 1: Core OS Dependencies & CLI Toolchain Installer

This script installs all required runtimes (`docker`, `python3`, `uv`, `node`, `npm`, `pnpm`, `bun`) and CLI utilities (`cloudflared`, `wrangler`, `claude-code`, `codex`, `agent-reach`) on Ubuntu Server or Desktop.

```bash
#!/usr/bin/env bash
# File: bootstrap-dependencies.sh
# Purpose: Idempotent installation of OS packages, runtimes, and AI CLIs
set -euo pipefail

export DEBIAN_FRONTEND=noninteractive

echo "=== 1. Updating Ubuntu & Installing Base Packages ==="
sudo apt-get update -y
sudo apt-get upgrade -y
sudo apt-get install -y \
  git curl wget jq unzip tar build-essential \
  ca-certificates gnupg lsb-release software-properties-common \
  python3 python3-pip python3-venv ffmpeg netcat-openbsd arp-scan nmap

echo "=== 2. Installing Docker & Docker Compose v2 ==="
if ! command -v docker >/dev/null 2>&1; then
  sudo apt-get install -y docker.io docker-compose-v2
fi
sudo systemctl enable --now docker
sudo usermod -aG docker "${SUDO_USER:-$USER}"

echo "=== 3. Installing Node.js 22 LTS, NPM, and PNPM ==="
if ! command -v node >/dev/null 2>&1; then
  curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
  sudo apt-get install -y nodejs
fi
if ! command -v pnpm >/dev/null 2>&1; then
  sudo npm install -g pnpm
fi

echo "=== 4. Installing Astral UV (Python Package & Tool Manager) ==="
if ! command -v uv >/dev/null 2>&1; then
  curl -LsSf https://astral.sh/uv/install.sh | sh
fi
export PATH="$HOME/.local/bin:$PATH"

echo "=== 5. Installing Bun Runtime ==="
if ! command -v bun >/dev/null 2>&1; then
  curl -fsSL https://bun.sh/install | bash
fi
export BUN_INSTALL="$HOME/.bun"
export PATH="$BUN_INSTALL/bin:$PATH"

echo "=== 6. Installing Cloud & Coding Agent CLIs ==="
# Cloudflare Wrangler & cloudflared
if ! command -v wrangler >/dev/null 2>&1; then
  sudo npm install -g wrangler
fi
if ! command -v cloudflared >/dev/null 2>&1; then
  curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
  echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/cloudflared.list
  sudo apt-get update -y && sudo apt-get install -y cloudflared || true
fi

# Claude Code CLI & OpenAI Codex CLI
sudo npm install -g @anthropic-ai/claude-code @openai/codex || true

# Agent-Reach (Zero-API Media & Web Extraction Layer)
if ! command -v agent-reach >/dev/null 2>&1; then
  uv tool install "git+https://github.com/Panniantong/Agent-Reach.git" || pip3 install --user "git+https://github.com/Panniantong/Agent-Reach.git" || true
fi

# Enable systemd user lingering so background agent gateways survive logout
sudo loginctl enable-linger "${SUDO_USER:-$USER}" || true

echo "=== Dependency Bootstrap Complete ==="
```

---

## 4. Phase 2: Workspace Architecture & Progressive Disclosure Scaffolding

To prevent context bloat, the workspace separates **always-on identity files** (`AGENTS.md`, `soul.md`) from **on-demand reference files** (`docs/*.md`, `runbooks/*.md`).

Run this script to build the folder hierarchy, pull or initialize your GitHub Obsidian vault, and generate all reference documents and helper scripts.

```bash
#!/usr/bin/env bash
# File: scaffold-workspace.sh
# Purpose: Create directory layout, clone repos, and write progressive disclosure docs
set -euo pipefail

TARGET_USER="${SUDO_USER:-$USER}"
TARGET_HOME="$(eval echo "~$TARGET_USER")"
WORKSPACE_ROOT="${WORKSPACE_ROOT:-$TARGET_HOME/workspace}"
ENV_FILE="$WORKSPACE_ROOT/.env"

if [[ -f "$ENV_FILE" ]]; then
  # shellcheck disable=SC1090
  source "$ENV_FILE"
fi

DATA_PLANE_ROOT="${DATA_PLANE_ROOT:-/srv}"

echo "=== 1. Creating Control Panel, Agent Area, and Data Plane Directories ==="
mkdir -p "$WORKSPACE_ROOT"/{control-panel/{config,scripts,docker},agents/pool,docs,brain/{vault,raw-ingest,vector-store}}
sudo mkdir -p "$DATA_PLANE_ROOT"/agent-bus/{inbox,working,outbox,archive,failed}
sudo mkdir -p "$DATA_PLANE_ROOT"/second-brain/{qdrant_storage,postgres_data,redis_data}
sudo chown -R "$TARGET_USER":"$TARGET_USER" "$WORKSPACE_ROOT" "$DATA_PLANE_ROOT"/agent-bus "$DATA_PLANE_ROOT"/second-brain
chmod -R 2775 "$DATA_PLANE_ROOT"/agent-bus "$DATA_PLANE_ROOT"/second-brain

# Pull central template repo if specified
if [[ -n "${CENTRAL_TEMPLATE_REPO:-}" ]]; then
  echo "=== Pulling scripts and configs from CENTRAL_TEMPLATE_REPO ==="
  git clone "$CENTRAL_TEMPLATE_REPO" "/tmp/central-template" || true
  cp -rn /tmp/central-template/* "$WORKSPACE_ROOT/control-panel/" 2>/dev/null || true
  rm -rf /tmp/central-template
fi

echo "=== 2. Initializing or Cloning Functioning Brain (Obsidian Vault) ==="
VAULT_DIR="$WORKSPACE_ROOT/brain/vault"
if [[ -n "${OBSIDIAN_VAULT_REPO:-}" && ! -d "$VAULT_DIR/.git" ]]; then
  git clone "$OBSIDIAN_VAULT_REPO" "$VAULT_DIR"
else
  mkdir -p "$VAULT_DIR"/{00-Meta,01-Profile,02-Knowledge/{People,Companies,Ideas,Research,Work},03-Transcripts,04-Insights}
  if [[ ! -d "$VAULT_DIR/.git" ]]; then
    git -C "$VAULT_DIR" init -b main
  fi
fi

# Ensure core profile seed file exists
if [[ ! -f "$VAULT_DIR/01-Profile/core_context.md" ]]; then
  cat << 'EOF' > "$VAULT_DIR/01-Profile/core_context.md"
# Principal Context & TELOS

## Active Objectives
- Maintain a sovereign, self-improving second brain and multi-agent workspace.
- Automate infrastructure provisioning, media ingestion, and synthesis.

## Working Preferences
- Concise, direct, technical responses with zero filler.
- Ground all recommendations in verified tool outputs and documented sources.
EOF
fi
```

---

## 5. Phase 3: Progressive Disclosure Pointer Files (`AGENTS.md` & Reference Docs)

The root `AGENTS.md` file is kept under **1.5 KB (~350 tokens)**. It acts strictly as a routing map telling any agent where to look up detailed syntax only when needed.

```bash
#!/usr/bin/env bash
# File: generate-docs-and-pointers.sh
# Purpose: Generate root AGENTS.md index and on-demand reference documentation
set -euo pipefail

WORKSPACE_ROOT="${WORKSPACE_ROOT:-$HOME/workspace}"

# 1. Master Progressive Disclosure Router (AGENTS.md)
cat << 'EOF' > "$WORKSPACE_ROOT/AGENTS.md"
# Workspace Agent Operating Index (Progressive Disclosure Map)

You are operating inside a structured AI Agent Workspace. To protect context window budget, DO NOT load all documentation upfront. Follow the pointers below based on your active task:

## Environment & Paths
- **Workspace Root**: `$WORKSPACE_ROOT` (contains `control-panel/`, `agents/`, `brain/`, `docs/`)
- **Data Plane & Task Bus**: `/srv/agent-bus/` (`inbox/`, `working/`, `outbox/`, `archive/`, `failed/`)
- **Knowledge Vault (Obsidian)**: `$WORKSPACE_ROOT/brain/vault/`

## Conditional Reference Pointers (Read ONLY When Needed)
1. **WHEN routing, claiming, or completing asynchronous tasks**:
   -> READ `$WORKSPACE_ROOT/docs/task-bus-protocol.md`
2. **WHEN provisioning a new agent or updating agent roles/personalities**:
   -> READ `$WORKSPACE_ROOT/docs/agent-provisioning-guide.md`
   -> RUN `$WORKSPACE_ROOT/control-panel/scripts/init-agent.sh`
3. **WHEN ingesting YouTube, TikTok, audio, web pages, or pre-existing files**:
   -> READ `$WORKSPACE_ROOT/docs/media-and-file-ingestion.md`
4. **WHEN reading, writing, linking, or syncing the Second Brain / Obsidian Vault**:
   -> READ `$WORKSPACE_ROOT/docs/second-brain-taxonomy.md`
5. **WHEN managing Proxmox VMs, Docker stacks, or running system updates**:
   -> READ `$WORKSPACE_ROOT/docs/infrastructure-ops.md`

## Core Safety Rules
- Never commit `.env` files, raw API keys, or tokens to Git.
- Always request human confirmation before executing destructive commands (`qm destroy`, `rm -rf`, `docker volume prune`).
EOF

# 2. On-Demand Doc: Task Bus Protocol
cat << 'EOF' > "$WORKSPACE_ROOT/docs/task-bus-protocol.md"
# Asynchronous Task Bus Protocol

All multi-agent delegation occurs via atomic file operations in `/srv/agent-bus/`.

## Lifecycle States
1. **Create Task**: Write JSON payload to `/srv/agent-bus/inbox/task_<timestamp>_<slug>.json`.
2. **Claim Task**: Worker atomically moves file to `/srv/agent-bus/working/`.
3. **Complete Task**: Worker writes output path to JSON and moves task to `/srv/agent-bus/outbox/`, then `/srv/agent-bus/archive/`.
4. **Error Handling**: If execution fails, append error trace to JSON and move to `/srv/agent-bus/failed/`.

## Task Payload Schema
```json
{
  "task_id": "task_20261027_yt_ingest",
  "target_agent": "researcher",
  "type": "media_synthesis",
  "input_uri": "https://youtube.com/watch?v=...",
  "output_path": "/home/v/workspace/brain/vault/04-Insights/note.md",
  "status": "pending"
}
```
EOF

# 3. On-Demand Doc: Second Brain Taxonomy & Obsidian Sync
cat << 'EOF' > "$WORKSPACE_ROOT/docs/second-brain-taxonomy.md"
# Second Brain & Obsidian Vault Taxonomy

The vault lives at `$WORKSPACE_ROOT/brain/vault/` and uses standard Obsidian WikiLinks (`[[Note Title]]`) and YAML frontmatter.

## Folder Routing Rules
- `00-Meta/`: Templates, tag indexes, and system logs.
- `01-Profile/`: `core_context.md` and `telos.md` (read before strategic planning).
- `02-Knowledge/People/`: Profiles of individuals, contacts, and collaborators.
- `02-Knowledge/Companies/`: Vendors, clients, and competitor entities.
- `02-Knowledge/Ideas/`: Raw concepts, architectures, and hypotheses.
- `02-Knowledge/Research/`: Deep technical syntheses and multi-source briefs.
- `02-Knowledge/Work/`: Active project logs, SOPs, and deliverables.
- `03-Transcripts/`: Raw verbatim transcripts named `YYYY-MM-DD_<source-id>.md`.
- `04-Insights/`: Extracted takeaways, mental models, and action items linking back to `03-Transcripts/`.

## Required Frontmatter Schema
```yaml
---
title: "Note Title"
date: "YYYY-MM-DD"
type: "research | insight | transcript | entity"
source_url: "https://..."
tags: ["topic-a", "topic-b"]
---
```

## Syncing with GitHub
Run `$WORKSPACE_ROOT/control-panel/scripts/sync-brain.sh` after creating or modifying vault notes.
EOF

# 4. On-Demand Doc: Media & Pre-Existing File Ingestion
cat << 'EOF' > "$WORKSPACE_ROOT/docs/media-and-file-ingestion.md"
# Media & Pre-Existing File Ingestion

## 1. YouTube, TikTok & Web Extraction
Use `agent-reach` or `yt-dlp` via `$WORKSPACE_ROOT/control-panel/scripts/ingest-media.sh <URL>`.
- Always attempt native subtitle extraction first (`yt-dlp --write-auto-sub --sub-lang en --skip-download`).
- If native subtitles are missing, extract `.mp3` to `/tmp/` and transcribe via Groq Whisper API (`whisper-large-v3-turbo`), then delete the temporary audio file immediately.

## 2. Pre-Existing File Upload & Batch Ingestion
Place existing PDFs, Markdown archives, `.docx`, or `.txt` files into `$WORKSPACE_ROOT/brain/raw-ingest/`.
- Run `$WORKSPACE_ROOT/control-panel/scripts/ingest-docs.sh` to convert files into clean frontmatter-tagged Markdown inside `02-Knowledge/` and move processed originals to `raw-ingest/processed/`.
EOF

# 5. On-Demand Doc: Infrastructure & Proxmox Operations
cat << 'EOF' > "$WORKSPACE_ROOT/docs/infrastructure-ops.md"
# Infrastructure & Proxmox Operations

## Local Maintenance
- Run `$WORKSPACE_ROOT/control-panel/scripts/sys-update.sh` to update OS, Node, UV, Bun, and Docker stacks.
- Docker Compose manifest is located at `$WORKSPACE_ROOT/control-panel/docker/docker-compose.yml`.

## Remote Proxmox Management (When PROXMOX_SSH_TARGET is set)
- Use `$WORKSPACE_ROOT/control-panel/scripts/pve.sh {list|status|start|stop|clone}`
- Never run `qm destroy` or `qm stop` on production VMs without explicit operator confirmation.
EOF
```

---

## 6. Phase 4: Operational Scripts & Guided Agent Provisioning Wizard

This script populates `control-panel/scripts/` with the **Guided Agent Provisioner** (`init-agent.sh`), **Vault Git Sync** (`sync-brain.sh`), **Media/Doc Ingestor** (`ingest-media.sh`), and **Lean Docker Stack** (`docker-compose.yml`).

```bash
#!/usr/bin/env bash
# File: install-control-scripts.sh
# Purpose: Install operational scripts and guided agent provisioning wizard
set -euo pipefail

WORKSPACE_ROOT="${WORKSPACE_ROOT:-$HOME/workspace}"
SCRIPTS_DIR="$WORKSPACE_ROOT/control-panel/scripts"
mkdir -p "$SCRIPTS_DIR"

# 1. Guided Agent & Team Provisioning Script (init-agent.sh)
cat << 'EOF' > "$SCRIPTS_DIR/init-agent.sh"
#!/usr/bin/env bash
# Usage:
#   Interactive: ./init-agent.sh
#   Non-interactive: ./init-agent.sh --name "researcher" --role "Deep synthesis" --archetype 2 --model "openrouter/deepseek-chat"
set -euo pipefail

WORKSPACE_ROOT="${WORKSPACE_ROOT:-$HOME/workspace}"
AGENTS_DIR="$WORKSPACE_ROOT/agents/pool"
REGISTRY_FILE="$WORKSPACE_ROOT/agents/registry.yaml"

AGENT_NAME=""
AGENT_ROLE=""
ARCH_CHOICE=""
AGENT_MODEL=""

while [[ $# -gt 0 ]]; do
  case "$1" in
    --name) AGENT_NAME="$2"; shift 2 ;;
    --role) AGENT_ROLE="$2"; shift 2 ;;
    --archetype) ARCH_CHOICE="$2"; shift 2 ;;
    --model) AGENT_MODEL="$2"; shift 2 ;;
    --bootstrap-default-team)
      "$0" --name "orchestrator" --role "Front-door router, task dispatcher, and operator liaison" --archetype 1 --model "anthropic/claude-3.5-sonnet"
      "$0" --name "researcher" --role "Deep technical researcher, media transcriber, and knowledge synthesizer" --archetype 2 --model "openrouter/deepseek-chat"
      "$0" --name "hr-ops" --role "Agent governance, prompt auditor, skill curator, and health monitor" --archetype 3 --model "openai/gpt-4o-mini"
      exit 0
      ;;
    *) echo "Unknown arg: $1"; exit 1 ;;
  esac
done

if [[ -z "$AGENT_NAME" ]]; then
  echo "=== Guided Agent Provisioning Questionnaire ==="
  read -rp "1. Agent Identifier (slug, e.g., orchestrator, researcher, hr-ops): " AGENT_NAME
  read -rp "2. Core Role & Mission (1 sentence): " AGENT_ROLE
  read -rp "3. Primary Model [anthropic/claude-3.5-sonnet]: " AGENT_MODEL
  AGENT_MODEL="${AGENT_MODEL:-anthropic/claude-3.5-sonnet}"
  echo "4. Select Personality & Operational Archetype:"
  echo "   [1] Direct Operator (Concise, zero filler, execution & routing focused)"
  echo "   [2] Analytical Researcher (Deep synthesis, citation-strict, multi-source verification)"
  echo "   [3] Governance & Systems Architect (Audits prompts, tracks drift, optimizes workflows)"
  read -rp "Select (1-3) [1]: " ARCH_CHOICE
  ARCH_CHOICE="${ARCH_CHOICE:-1}"
fi

case "$ARCH_CHOICE" in
  1)
    PERSONA_TITLE="Direct Technical Operator"
    TONE_RULES="Communicate with high signal and zero filler. Lead with outcomes, status, and exact commands. Never speculate without checking tool output."
    ;;
  2)
    PERSONA_TITLE="Analytical Synthesis Specialist"
    TONE_RULES="Prioritize depth, primary-source citations, mental models, and structural clarity. Cross-link every new note to existing vault entities using [[WikiLinks]]."
    ;;
  3)
    PERSONA_TITLE="Governance & Systems Architect"
    TONE_RULES="Audit agent outputs, maintain progressive disclosure pointers, prune redundant token overhead, and propose skill upgrades based on failed task logs."
    ;;
  *)
    PERSONA_TITLE="Standard Specialist"
    TONE_RULES="Direct, objective, and accurate."
    ;;
esac

TARGET_DIR="$AGENTS_DIR/$AGENT_NAME"
mkdir -p "$TARGET_DIR"
 sudo mkdir -p "/srv/agents/$AGENT_NAME/data"
sudo chown -R "$USER":"$USER" "/srv/agents/$AGENT_NAME"

# Generate soul.md with explicit Progressive Disclosure pointers
cat <<EOF > "$TARGET_DIR/soul.md"
# Agent Identity: ${AGENT_NAME}

## Core Mission
${AGENT_ROLE}

## Persona & Communication Rules (${PERSONA_TITLE})
${TONE_RULES}

## Progressive Disclosure Pointers (Where to Look)
- **Master Workspace Index**: Read \`$WORKSPACE_ROOT/AGENTS.md\`
- **User Context & Goals**: Read \`$WORKSPACE_ROOT/brain/vault/01-Profile/core_context.md\`
- **My Permitted Tools & Models**: Read \`$TARGET_DIR/inventory.md\`
- **My Step-by-Step SOPs**: Read \`$TARGET_DIR/runbook.md\`
- **Task Bus Format**: Read \`$WORKSPACE_ROOT/docs/task-bus-protocol.md\`
- **Vault Filing Rules**: Read \`$WORKSPACE_ROOT/docs/second-brain-taxonomy.md\`
EOF

# Generate inventory.md
cat <<EOF > "$TARGET_DIR/inventory.md"
# Inventory & Tool Permissions: ${AGENT_NAME}

## Assigned Models
- **Primary Reasoning**: \`${AGENT_MODEL}\`
- **Auxiliary / Compression**: \`groq/llama-3.3-70b-versatile\`

## Filesystem & Runtime Paths
- **Agent Definition**: \`${TARGET_DIR}/\`
- **Runtime State**: \`/srv/agents/${AGENT_NAME}/data/\`
- **Task Queue**: \`/srv/agent-bus/\`
- **Brain Vault**: \`$WORKSPACE_ROOT/brain/vault/\`

## Permitted Scripts
- \`$WORKSPACE_ROOT/control-panel/scripts/sync-brain.sh\`
- \`$WORKSPACE_ROOT/control-panel/scripts/ingest-media.sh\`
EOF

# Generate runbook.md
cat <<EOF > "$TARGET_DIR/runbook.md"
# Operational Runbook: ${AGENT_NAME}

## Standard Execution Loop
1. **Context Check**: Inspect \`/srv/agent-bus/inbox/\` for tasks assigned to \`${AGENT_NAME}\`.
2. **Claim**: Move claimed task JSON to \`/srv/agent-bus/working/\`.
3. ** Progressive Lookup**: Consult only the relevant doc in \`$WORKSPACE_ROOT/docs/\` required for the current task.
4. **Execute & Verify**: Write deliverable to \`$WORKSPACE_ROOT/brain/vault/\` with valid YAML frontmatter.
5. **Sync & Archive**: Run \`sync-brain.sh\` and move task JSON to \`/srv/agent-bus/archive/\`.
EOF

# Register in agents/registry.yaml if not present
touch "$REGISTRY_FILE"
if ! grep -q "^  ${AGENT_NAME}:" "$REGISTRY_FILE"; then
  cat <<EOF >> "$REGISTRY_FILE"
  ${AGENT_NAME}:
    role: "${AGENT_ROLE}"
    archetype: "${PERSONA_TITLE}"
    model: "${AGENT_MODEL}"
    path: "${TARGET_DIR}"
EOF
fi

echo "Provisioned agent '${AGENT_NAME}' at ${TARGET_DIR}"
EOF
chmod +x "$SCRIPTS_DIR/init-agent.sh"

# 2. Bidirectional Obsidian Git Sync Script (sync-brain.sh)
cat << 'EOF' > "$SCRIPTS_DIR/sync-brain.sh"
#!/usr/bin/env bash
set -euo pipefail
WORKSPACE_ROOT="${WORKSPACE_ROOT:-$HOME/workspace}"
VAULT_DIR="$WORKSPACE_ROOT/brain/vault"

cd "$VAULT_DIR"
if git remote get-url origin >/dev/null 2>&1; then
  git pull --rebase origin main || true
fi

if [[ -n "$(git status --porcelain)" ]]; then
  git add -A
  git commit -m "brain-sync: automated update $(date -u +"%Y-%m-%dT%H:%M:%SZ")"
  if git remote get-url origin >/dev/null 2>&1; then
    git push origin main
  fi
fi
EOF
chmod +x "$SCRIPTS_DIR/sync-brain.sh"

# 3. Media Extraction & Groq Whisper Pipeline (ingest-media.sh)
cat << 'EOF' > "$SCRIPTS_DIR/ingest-media.sh"
#!/usr/bin/env bash
set -euo pipefail
URL="${1:?Usage: ingest-media.sh <URL>}"
WORKSPACE_ROOT="${WORKSPACE_ROOT:-$HOME/workspace}"
# shellcheck disable=SC1090
[[ -f "$WORKSPACE_ROOT/.env" ]] && source "$WORKSPACE_ROOT/.env"

TRANSCRIPT_DIR="$WORKSPACE_ROOT/brain/vault/03-Transcripts"
DATE_PREFIX="$(date +%Y-%m-%d)"

if command -v yt-dlp >/dev/null 2>&1; then
  VID_ID="$(yt-dlp --print id "$URL" 2>/dev/null || echo "media_$(date +%s)")"
  OUT_FILE="$TRANSCRIPT_DIR/${DATE_PREFIX}_${VID_ID}.md"

  # Try subtitle extraction first
  yt-dlp --write-auto-sub --sub-lang en --skip-download --convert-subs srt -o "/tmp/${VID_ID}.%(ext)s" "$URL" || true

  if [[ -f "/tmp/${VID_ID}.en.srt" ]]; then
    cat <<HEADER > "$OUT_FILE"
---
title: "Transcript ${VID_ID}"
date: "${DATE_PREFIX}"
type: "transcript"
source_url: "${URL}"
---
HEADER
    sed -E '/^[0-9]+$/d; /^[0-9]{2}:[0-9]{2}:/d; /^$/d' "/tmp/${VID_ID}.en.srt" | uniq >> "$OUT_FILE"
    rm -f "/tmp/${VID_ID}.en.srt"
    echo "Saved subtitle transcript to $OUT_FILE"
    exit 0
  fi

  # Fallback to Groq Whisper API if GROQ_API_KEY is configured
  if [[ -n "${GROQ_API_KEY:-}" ]]; then
    yt-dlp -x --audio-format mp3 -o "/tmp/${VID_ID}.mp3" "$URL"
    curl -s https://api.groq.com/openai/v1/audio/transcriptions \
      -H "Authorization: Bearer $GROQ_API_KEY" \
      -F file="@/tmp/${VID_ID}.mp3" \
      -F model="whisper-large-v3-turbo" \
      -F response_format="text" > "$OUT_FILE"
    rm -f "/tmp/${VID_ID}.mp3"
    echo "Transcribed audio via Groq Whisper to $OUT_FILE"
  fi
fi
EOF
chmod +x "$SCRIPTS_DIR/ingest-media.sh"
```

---

## 7. Phase 5: Docker Stack & Lean Hermes Configuration

This script writes the token-optimized `lean-config.yaml` and the multi-container `docker-compose.yml` (Hermes Gateway + Qdrant + Honcho sovereign memory).

```bash
#!/usr/bin/env bash
# File: deploy-docker-stack.sh
# Purpose: Generate Docker Compose stack and lean Hermes configuration
set -euo pipefail

WORKSPACE_ROOT="${WORKSPACE_ROOT:-$HOME/workspace}"
# shellcheck disable=SC1090
[[ -f "$WORKSPACE_ROOT/.env" ]] && source "$WORKSPACE_ROOT/.env"
DATA_PLANE_ROOT="${DATA_PLANE_ROOT:-/srv}"

mkdir -p "$HOME/.hermes"

# 1. Seed Lean Hermes Config (Offloads auxiliary tasks & disables heavy browser/TTS schemas)
if [[ ! -f "$HOME/.hermes/config.yaml" ]]; then
  cat <<EOF > "$HOME/.hermes/config.yaml"
model:
  provider: "${PRIMARY_LLM_PROVIDER:-openrouter}"
  name: "${PRIMARY_LLM_MODEL:-anthropic/claude-3.5-sonnet}"

auxiliary:
  compression_model: "${AUXILIARY_LLM_MODEL:-groq/llama-3.3-70b-versatile}"
  title_model: "${AUXILIARY_LLM_MODEL:-groq/llama-3.3-70b-versatile}"
  approval_model: "${AUXILIARY_LLM_MODEL:-groq/llama-3.3-70b-versatile}"

toolsets:
  browser: false
  tts: false
  shell: true
  filesystem: true

telegram:
  enabled: true
  allowed_users: [${TELEGRAM_ALLOWED_USERS:-}]
EOF
fi

# 2. Generate Docker Compose Manifest
cat <<EOF > "$WORKSPACE_ROOT/control-panel/docker/docker-compose.yml"
services:
  hermes-gateway:
    image: nousresearch/hermes-agent:latest
    container_name: hermes-gateway
    restart: unless-stopped
    command: gateway run
    network_mode: host
    env_file:
      - $WORKSPACE_ROOT/.env
    environment:
      - HERMES_DASHBOARD=0
      - WORKSPACE_ROOT=$WORKSPACE_ROOT
    volumes:
      - $HOME/.hermes:/opt/data
      - $HOME/.ssh:/root/.ssh:ro
      - $WORKSPACE_ROOT:$WORKSPACE_ROOT
      - $DATA_PLANE_ROOT:$DATA_PLANE_ROOT

  qdrant:
    image: qdrant/qdrant:latest
    container_name: second-brain-qdrant
    restart: unless-stopped
    ports:
      - "127.0.0.1:6333:6333"
    volumes:
      - $DATA_PLANE_ROOT/second-brain/qdrant_storage:/qdrant/storage
    deploy:
      resources:
        limits:
          memory: 1G
EOF

cd "$WORKSPACE_ROOT/control-panel/docker"
docker compose up -d
```

---

## 8. Phase 6: Sealing as a Reusable Multi-Location VM Template

When you want to convert a configured VM into a reusable Proxmox Golden Template (so you can clone it to new locations or nodes without leaking keys or conflicting machine IDs), run this sanitization script right before shutting down the VM:

```bash
#!/usr/bin/env bash
# File: prepare-golden-template.sh
# Purpose: Sanitize secrets and machine-specific IDs before converting VM to a template
set -euo pipefail

WORKSPACE_ROOT="${WORKSPACE_ROOT:-$HOME/workspace}"

echo "=== 1. Stopping Containers ==="
cd "$WORKSPACE_ROOT/control-panel/docker" && docker compose down || true

echo "=== 2. Stripping Live Secrets (Keeping Template Structure) ==="
rm -f "$WORKSPACE_ROOT/.env"
rm -f "$HOME/.hermes/.env"

echo "=== 3. Resetting Machine ID & SSH Host Keys for Clean Cloning ==="
sudo truncate -s 0 /etc/machine-id
sudo rm -f /var/lib/dbus/machine-id
sudo ln -s /etc/machine-id /var/lib/dbus/machine-id
sudo rm -f /etc/ssh/ssh_host_*

echo "=== 4. Adding First-Boot SSH Key Regeneration Hook ==="
sudo tee /etc/rc.local >/dev/null << 'EOF'
#!/bin/sh -e
test -f /etc/ssh/ssh_host_ed25519_key || dpkg-reconfigure openssh-server
exit 0
EOF
sudo chmod +x /etc/rc.local

echo "Template sanitized. Shut down VM ('sudo poweroff') and run 'qm template <VMID>' on Proxmox."
```

---

## 9. Single-Command Master Execution Order

For an AI agent or human operator starting on a fresh Ubuntu machine, run the phases in this exact order:

```bash
# 1. Collect environment variables and secrets (prompts only for missing values)
bash preflight.sh

# 2. Install OS packages, Docker, Node, PNPM, UV, Bun, and AI CLIs
bash bootstrap-dependencies.sh

# 3. Scaffold folders and clone/init the GitHub Obsidian Vault
bash scaffold-workspace.sh

# 4. Write the Progressive Disclosure router (AGENTS.md) and reference docs
bash generate-docs-and-pointers.sh

# 5. Install control scripts and bootstrap the 3-agent team (Orchestrator, Researcher, HR-Ops)
bash install-control-scripts.sh
"$HOME/workspace/control-panel/scripts/init-agent.sh" --bootstrap-default-team

# 6. Launch the persistent Docker stack (Hermes Gateway + Qdrant)
bash deploy-docker-stack.sh
```
