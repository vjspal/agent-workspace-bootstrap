# agent-workspace-bootstrap
structured agent repository documentation for full agentic environment deployment, portability, and flexible reusable approach. 

What the Standalone Template Includes
Phase 0 Pre-Flight Discovery (preflight.sh): Auto-detects $USER and $HOME, checks $WORKSPACE_ROOT/.env, and prompts only for missing secrets (PRIMARY_API_KEY, GROQ_API_KEY, TELEGRAM_BOT_TOKEN) and optional Git URLs (CENTRAL_TEMPLATE_REPO, OBSIDIAN_VAULT_REPO).

Idempotent Toolchain Installer (bootstrap-dependencies.sh): Installs Docker, Compose v2, Python 3, uv, Node 22 LTS, npm, pnpm, bun, wrangler, cloudflared, claude-code, codex, and agent-reach with systemd user lingering enabled.

Pull-First, Generate-Fallback Scaffolding (scaffold-workspace.sh): Pulls shared configs from your central Git repository if provided, or generates all directory trees (control-panel/, agents/pool/, brain/vault/, /srv/agent-bus/) and clones/initializes your GitHub Obsidian vault locally.

Progressive Disclosure Index (generate-docs-and-pointers.sh): Writes a compact root AGENTS.md (<1.5 KB) that routes agents conditionally to five modular reference files (docs/task-bus-protocol.md, docs/second-brain-taxonomy.md, docs/media-and-file-ingestion.md, and docs/infrastructure-ops.md).

Dual-Mode Agent Provisioner (init-agent.sh): Supports both an interactive personality questionnaire and non-interactive CLI flags (--bootstrap-default-team) to generate soul.md, inventory.md, and runbook.md for your orchestrator, researcher, and hr-ops team with progressive disclosure pointers wired in.

Golden Template Sanitizer (prepare-golden-template.sh): Strips live .env secrets, resets /etc/machine-id, and configures first-boot SSH host key regeneration before you run qm template on Proxmox for multi-location cloning.

You can copy this file directly into your central GitHub repository as its root BOOTSTRAP.md (or README.md) and tell any CLI agent on a new VM: "Read agent_workspace_bootstrap_template.md and execute Phases 0 through 5."
