# Holochain Claude Skill

A comprehensive Claude Code skill for Holochain hApp development. Covers the full development spiral from architecture and design through scaffolding, implementation, testing, and deployment.

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

### Option A: Global (all projects)

```bash
# Copy the skill folder to your global Claude Code skills directory
cp -r holochain-claude-skill ~/.claude/skills/holochain
```

### Option B: Project-local

```bash
# Copy into your project's .claude directory
mkdir -p your-project/.claude/skills
cp -r holochain-claude-skill your-project/.claude/skills/holochain
```

### Option C: Symlink (recommended for development)

```bash
# Clone and symlink for easy updates
git clone https://github.com/Soushi888/holochain-claude-skill ~/holochain-claude-skill
ln -s ~/holochain-claude-skill ~/.claude/skills/holochain
```

Once installed, Claude Code will load `SKILL.md` when you use the `/holochain` command or when Claude detects Holochain-related work.

> **Upgrading from v0.1.0?** The install directory name changed from `Holochain` to `holochain` (lowercase) to conform to the Agent Skills Open Standard. Rename your existing install: `mv ~/.claude/skills/Holochain ~/.claude/skills/holochain`

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
