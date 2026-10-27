# Agent Workspace Bootstrap

> Structured repository documentation, automation, and progressive disclosure architecture for deploying portable, self-healing, multi-agent AI environments across any Ubuntu VM, physical server, or homelab.

---

## Overview

**Agent Workspace Bootstrap** is a repeatable, location-agnostic template for running production AI agents (such as [Hermes Agent](https://github.com/nousresearch/hermes-agent), [Claude Code](https://docs.anthropic.com/en/docs/agents-and-tools/claude-code/overview), or [Codex](https://github.com/openai/codex)). It decouples configuration, code, and identities from underlying host hardware, storage pools, or usernames.

Whether deploying on a local hypervisor (Proxmox, KVM, VMware), a cloud VPS (Hetzner, DigitalOcean, AWS), or a bare-metal machine, this repository establishes a standard three-zone environment:
1. **Control Panel**: Centralized governance, maintenance tooling, service discovery, and Docker orchestration.
2. **Agent Area**: Modular agent pool with explicit identity files (`soul.md`), tool permissions (`inventory.md`), and standard operating procedures (`runbook.md`).
3. **Functioning Brain**: Bidirectional Git-synced [Obsidian](https://obsidian.md) vault with Cortex taxonomy, paired with a local [Qdrant](https://qdrant.tech) vector database for semantic search and an automated media ingestion pipeline.

---

## Architectural Pillars

* **Zero Hardcoded Assumptions**: No usernames (`root`, `v`), home paths (`/home/v`), hostnames (`rcmprox`), or storage pools (`local-zfs`) are hardcoded. All scripts resolve paths dynamically via `$USER`, `$HOME`, and a local `.env` manifest.
* **Progressive Disclosure (`AGENTS.md` < 2 KB)**: Adheres to token-budgeting best practices. The root instruction file stays under ~350 tokens, containing conditional pointers (`WHEN [task] -> READ docs/[file].md`) so agents only load reference syntax into active context when performing that specific task.
* **Separation of Control vs. Data Plane**: All configuration files, prompts, and runbooks live in the Git repository (`workspace/`). All runtime databases (SQLite, Qdrant, PostgreSQL, Redis) and the atomic file-based task bus live in the persistent data plane (`/srv/`).
* **Sovereign Long-Term Memory**: Integrates local behavioral modeling ([Honcho](https://github.com/elkimek/honcho-self-hosted)), vector retrieval, and Markdown knowledge vaults without sending personal identity data to third-party memory clouds.

---

## Repository Structure

```
agent-workspace-bootstrap/
├── README.md                          # Repository documentation and human guide
├── AGENTS.md                          # Master Progressive Disclosure routing index (<2 KB)
├── BOOTSTRAP.md                       # Standalone single-file executable specification
│
├── control-panel/                     # Zone 1: Governance, scripts, and runtime
│   ├── config/
│   │   ├── service-registry.yaml      # Internal IP/port endpoints (n8n, Proxmox, local DNS)
│   │   └── lean-hermes.yaml           # Token-pruned Hermes runtime configuration
│   ├── docker/
│   │   └── docker-compose.yml         # Container stack (Hermes Gateway + Qdrant)
│   └── scripts/
│       ├── preflight.sh               # Phase 0: Environment discovery & secret prompt
│       ├── bootstrap-dependencies.sh  # Phase 1: OS packages, runtimes, and AI CLIs
│       ├── scaffold-workspace.sh      # Phase 2: Folder trees & Obsidian vault clone
│       ├── init-agent.sh              # Phase 4: Guided agent provisioning wizard
│       ├── sync-brain.sh              # Bidirectional Git sync for Obsidian vault
│       ├── ingest-media.sh            # YouTube/TikTok extraction via Agent-Reach & Whisper
│       ├── sys-update.sh              # Single-command system maintenance
│       └── prepare-golden-template.sh # Phase 6: Machine-id & secret sanitization
│
├── agents/                            # Zone 2: Dedicated agent definitions
│   ├── registry.yaml                  # Manifest of registered agents, models, and states
│   └── pool/                          # Directory containing each active agent
│       ├── orchestrator/              # Front-door router & task dispatcher
│       │   ├── soul.md
│       │   ├── inventory.md
│       │   └── runbook.md
│       ├── researcher/                # Deep technical research & transcript synthesizer
│       └── hr-ops/                    # Agent health monitor, prompt auditor, and archivist
│
├── brain/                             # Zone 3: Knowledge vault & media staging
│   ├── vault/                         # Git-synced Obsidian vault (People, Companies, Ideas, Research)
│   ├── raw-ingest/                    # Drop zone for legacy PDFs, text dumps, and audio
│   └── vector-store/                  # Persistent Qdrant database volume
│
├── docs/                              # On-demand reference documentation
│   ├── task-bus-protocol.md           # Asynchronous JSON task queue schemas
│   ├── second-brain-taxonomy.md       # Cortex note structure and frontmatter standards
│   ├── media-and-file-ingestion.md    # Audio, video, and web extraction procedures
│   ├── agent-provisioning-guide.md    # How to design personas and tool inventories
│   └── infrastructure-ops.md          # Proxmox CLI, Tailscale, and Docker procedures
│
└── .gitignore                         # Strict rules blocking .env, *.db, and keys
```

---

## Quickstart

### Option A: Autonomous AI Agent Execution

If you have an AI agent with command-line execution (`claude`, `codex`, or `hermes`) running on a fresh Ubuntu machine:

1. Clone this repository:
   ```bash
   git clone https://github.com/YOUR_USERNAME/agent-workspace-bootstrap.git ~/workspace
   cd ~/workspace
   ```
2. Command your agent:
   > *"Read `BOOTSTRAP.md` and execute Phases 0 through 5 sequentially. Prompt me only for missing API keys or repository URLs."*

---

### Option B: Step-by-Step Human Deployment

#### Step 1: Run Pre-Flight Discovery
Collect environment variables and verify paths:
```bash
bash control-panel/scripts/preflight.sh
```
*(The script checks `$WORKSPACE_ROOT/.env` and only prompts for keys that are missing or blank).*

#### Step 2: Install Core Dependencies
Install OS packages, Docker, Node 22 LTS, PNPM, Astral UV, Bun, Cloudflare tooling, and AI agent CLIs:
```bash
bash control-panel/scripts/bootstrap-dependencies.sh
```

#### Step 3: Scaffold Workspace & Connect Brain
Create directory trees and clone your existing Obsidian vault:
```bash
bash control-panel/scripts/scaffold-workspace.sh
```

#### Step 4: Bootstrap the Default Three-Man Agent Team
Generate `soul.md`, `inventory.md`, and `runbook.md` for the core team:
```bash
bash control-panel/scripts/init-agent.sh --bootstrap-default-team
```

#### Step 5: Start the Container Stack
Launch the persistent Hermes Gateway and Qdrant vector database:
```bash
cd control-panel/docker && docker compose up -d
```

---

## Default Three-Man Agent Team

The bootstrap script provisions a balanced, functional team following the separation of concerns:

| Agent Identifier | Operational Archetype | Primary Role | Assigned Model | Permitted Capabilities |
| :--- | :--- | :--- | :--- | :--- |
| **`orchestrator`** | Direct Technical Operator | Front-door routing, task bus dispatching, operator liaison via Telegram. | `anthropic/claude-3.5-sonnet` | Task creation, Telegram gateway, session management. |
| **`researcher`** | Analytical Synthesis Specialist | Deep knowledge synthesis, YouTube/TikTok transcription, and cross-linking. | `openrouter/deepseek-chat` | `agent-reach`, Groq Whisper, Obsidian vault write. |
| **`hr-ops`** | Governance & Systems Architect | Prompt auditing, token-budget enforcement, health checks, and skill upgrades. | `openai/gpt-4o-mini` | `init-agent.sh`, `sys-update.sh`, task log review. |

To provision an additional specialized agent interactively, run:
```bash
bash control-panel/scripts/init-agent.sh
```

---

## Functioning Brain & Obsidian Integration

The brain template treats your Obsidian vault as the primary Markdown source of truth, synchronized bidirectionally with your private GitHub repository.

### Cortex Taxonomy Structure
All notes deposited by agents or human operators adhere to standard Obsidian WikiLinks (`[[Note Title]]`) and YAML frontmatter:
* `00-Meta/`: Templates, tag indexes, and system logs.
* `01-Profile/`: `core_context.md` (user background, voice business, homelab setup) and `telos.md` (active goals).
* `02-Knowledge/`: Typed entity archives (`People/`, `Companies/`, `Ideas/`, `Research/`, `Work/`).
* `03-Transcripts/`: Verbatim audio and video transcripts named `YYYY-MM-DD_<id>.md`.
* `04-Insights/`: Executive summaries, mental models, and action items linking back to source transcripts.

### Automated Bidirectional Vault Sync
The script `control-panel/scripts/sync-brain.sh` checks for remote edits made from your mobile or desktop Obsidian app, pulls them with rebase, commits any new notes generated by agents, and pushes back to GitHub:
```bash
# Add to crontab to run every 15 minutes:
crontab -l | { cat; echo "*/15 * * * * $HOME/workspace/control-panel/scripts/sync-brain.sh >/dev/null 2>&1"; } | crontab -
```

---

## Reusable Proxmox Golden Template (`qm template`)

To turn a configured workspace VM into a reusable Golden Template for multi-node or multi-location deployment (e.g., spinning up new instances on remote Proxmox hosts without secret leakage):

1. **Sanitize the VM**:
   Run the sanitization script inside the VM to wipe live `.env` secrets, reset `/etc/machine-id`, and queue first-boot SSH host key regeneration:
   ```bash
   bash control-panel/scripts/prepare-golden-template.sh
   sudo poweroff
   ```
2. **Convert to Template on Proxmox**:
   On your Proxmox hypervisor, lock the VM as a template:
   ```bash
   qm template 200
   ```
3. **Instantly Clone New Workspaces**:
   ```bash
   qm clone 200 205 --name ai-workspace-branch --full --storage local-zfs
   qm start 205
   ```

---

## Security & Credential Hygiene

* **No Secrets in Git**: The `.gitignore` file strictly excludes `.env`, `*.key`, `*.pem`, `*.db`, and credentials.
* **Restricted File Permissions**: Persistent data directories (`/srv/agent-bus/` and `/srv/second-brain/`) are assigned `chmod 2775` with group ownership to prevent permission collisions between container users and host accounts.
* **Public Template Safe**: This repository can be hosted publicly as a template repository; zero private API keys, Telegram bot tokens, or private notes exist in its tracked files.

---

## License

This project is licensed under the [MIT License](https://www.google.com/search?q=LICENSE).
