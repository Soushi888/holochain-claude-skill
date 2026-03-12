# Holochain Claude Skill

A comprehensive [Agent Skills Open Standard](https://agentskills.io) skill for Holochain hApp development, compatible with Claude Code, GitHub Copilot, Cursor, Augment, and any other tool supporting the standard. Covers the full development spiral from architecture and design through scaffolding, implementation, testing, and deployment.

## What It Covers

| Domain | Description |
|--------|-------------|
| **Architecture** | Coordinator/integrity zome split, DNA structure, Cargo workspace, Nix dev environment, progenitor pattern, multi-DNA, private entries |
| **Design** | DHT data modeling, entry/link type design, discovery strategy, validation rules |
| **Scaffold** | Holonix setup, Nix flake, `hc` CLI, `hc scaffold` commands, new project and new domain workflows |
| **Implement** | Entry types, link types, CRUD patterns, cross-zome calls, signals, validation, HDK 0.6 API |
| **Test** | Tryorama + Vitest setup, two-agent scenarios, `dhtSync`, update/delete patterns, test organization |
| **Deploy** | Kangaroo-Electron packaging, `.webhapp` bundling, CI/CD, versioning semantics, auto-update |

**Current version pins:** `hdk = "=0.6.0"` | `hdi = "=0.7.0"` | `holonix ref=main-0.6`

## Installation

This skill conforms to the [Agent Skills Open Standard](https://agentskills.io). The universal install path `.claude/skills/holochain/` is recognized by all compatible tools.

### Compatible Tools

| Tool | Supported | Invocation |
|------|-----------|------------|
| [Claude Code](https://claude.ai/code) | ✅ | `/holochain` |
| [GitHub Copilot](https://github.com/features/copilot) | ✅ | via agent skills |
| [Cursor](https://cursor.com) | ✅ | via agent skills |
| [Augment Code](https://augmentcode.com) | ✅ | via agent skills |
| [OpenAI Codex CLI](https://openai.com/codex) | ✅ | via agent skills |

---

### Claude Code

**Option A: Global** — available in all projects

```bash
cp -r holochain-claude-skill ~/.claude/skills/holochain
```

**Option B: Project-local** — scoped to one project

```bash
mkdir -p your-project/.claude/skills
cp -r holochain-claude-skill your-project/.claude/skills/holochain
```

**Option C: Symlink** (recommended — auto-updates with `git pull`)

```bash
git clone https://github.com/Soushi888/holochain-claude-skill ~/holochain-claude-skill
ln -s ~/holochain-claude-skill ~/.claude/skills/holochain
```

Once installed, invoke with `/holochain` or let Claude detect Holochain-related work automatically.

---

### Cursor, GitHub Copilot, Augment, and others

Install to your **project root's** `.claude/skills/` directory — all Agent Skills-compatible tools search this path:

```bash
mkdir -p .claude/skills
cp -r holochain-claude-skill .claude/skills/holochain
```

The tool discovers the skill automatically on next launch. For global install paths specific to each tool, refer to the tool's own documentation.

---

> **Upgrading from an older install?** If you previously installed to `.claude/skills/Holochain/` (uppercase), rename the directory:
> ```bash
> mv ~/.claude/skills/Holochain ~/.claude/skills/holochain
> ```

## Quick Start

```
# Design a new data model
/holochain design data model for a marketplace listing with status transitions

# Scaffold a new hApp from scratch
/holochain scaffold new happ called my-network

# Implement a full CRUD zome
/holochain implement zome for Profile entry type

# Debug a flaky test
/holochain my Tryorama test passes alone but fails when Bob reads Alice's entry

# Package for distribution
/holochain deploy package my happ for desktop distribution
```

## Workflow Triggers

| Say... | Triggers |
|--------|---------|
| "design data model", "model entries", "what entries" | DesignDataModel workflow |
| "scaffold", "new happ", "new project", "setup environment" | Scaffold workflow |
| "implement zome", "create zome", "write zome" | ImplementZome workflow |
| "design access control", "cap grant", "who can call" | DesignAccessControl workflow |
| "deploy", "package", "webhapp", "kangaroo" | PackageAndDeploy workflow |

## Ecosystem Roadmap

This skill follows a spiral from core to periphery:

**v1 (current):** Full development cycle — architecture, design, scaffold, implement, test, deploy

**v2 (planned):** Ecosystem expansion
- hREA / ValueFlows sub-skill
- holochain-open-dev patterns
- ADAM (coasys) integration
- Wind Tunnel performance testing
- unyt integration
- Cross-LLM portability

**v3 (vision):** GUI and visual tooling
- No-code workflow interface
- Visual DHT data model explorer
- Diagram generation for architecture
- Progressive disclosure (junior to senior)

## Contributing

Contributions welcome. The skill follows this structure:

```
SKILL.md              Entry point — routing table and quick reference
Architecture.md       Core concepts: zome split, DNA, Nix, progenitor
Patterns.md           Implementation patterns: entry types, links, CRUD, signals
Scaffold.md           Dev environment and project scaffolding
AccessControl.md      Capability grants system
CellCloning.md        Partitioned data via clone cells
ErrorHandling.md      thiserror + WasmError patterns
Testing.md            Tryorama + Vitest patterns
TypeScript.md         holochain-client, signals, Svelte integration
Deployment.md         Kangaroo-Electron packaging and distribution
Workflows/            Step-by-step guided workflows
docs/                 Requirements, roadmap, and design decisions
```

When updating for new Holochain versions, update the version pins in `SKILL.md` Quick Reference and in any code examples across all files.

## License

Apache-2.0
