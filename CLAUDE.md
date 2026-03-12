# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

A vanilla agent skill for Holochain hApp development. It is a documentation-only repository — no Rust, no TypeScript, no build system. All files are Markdown. The skill is loaded by Claude Code when a user invokes `/holochain` or when Holochain-related work is detected.

**PAI-independence constraint:** This skill must work with zero PAI infrastructure (`~/.claude/PAI/` not required). No voice notification curls, no Algorithm routing, no PROJECTS.md references. All content must be self-contained.

**License:** Apache-2.0

## File Roles

```
SKILL.md              Entry point — routing table, context file index, quick reference
Architecture.md       Coordinator/integrity split, DNA structure, Cargo workspace, Nix, progenitor, multi-DNA, private entries
Patterns.md           Entry types, link types, CRUD patterns, update chain, validation, signals, HDK 0.6 API
Scaffold.md           Holonix setup, Nix flake, hc CLI commands, project scaffolding
AccessControl.md      Capability grants, cap claims, admin-only patterns, init() setup
CellCloning.md        Clone cells, partitioned data, createCloneCell, clone_limit
ErrorHandling.md      thiserror enums, WasmError, ExternResult patterns
Testing.md            Tryorama + Vitest setup, two-agent scenarios, dhtSync, test organization
TypeScript.md         holochain-client setup, callZome, signals, SvelteKit integration
Deployment.md         Kangaroo-Electron packaging, .webhapp bundling, CI/CD, versioning
Workflows/            Step-by-step guided workflows (called from SKILL.md routing table)
docs/                 Requirements spec and roadmap (not loaded by the skill itself)
README.md             Installation instructions and quick start for humans
```

## Routing Architecture

`SKILL.md` is the entry point. It contains:
1. A **Workflow Routing** table mapping natural-language triggers to `Workflows/*.md` files
2. A **Context Files** table specifying which `*.md` file to load per topic
3. **Quick Reference** — version pins and common commands

Context files are loaded **on demand**, not all at once. When editing `SKILL.md`, maintain this lazy-loading discipline.

## Version Pins (update all occurrences when bumping)

```
hdk = "=0.6.0"
hdi = "=0.7.0"
holonix ref=main-0.6
```

Exact pins (`=`) are required — Holochain is sensitive to minor version changes. When updating version pins, grep all files for the old version string and update every occurrence including `SKILL.md` Quick Reference, `Architecture.md` workspace examples, and any code blocks in context files.

## Maintaining the Skill

- **Code examples must compile** — validate against the hAppenings or Nondominium codebase before committing new examples
- **HDK 0.6 breaking changes** — `delete_link()` now requires `GetOptions::default()` as second argument; `GetLinksInputBuilder` replaced the old `LinkQuery::new()` in some contexts (see `Patterns.md` for the authoritative API)
- **No duplication across files** — each pattern lives in one canonical file; `SKILL.md` routes to it
- **Workflow files** in `Workflows/` are step-by-step sequences; context files (`Architecture.md`, `Patterns.md`, etc.) are reference material. Keep these roles distinct
- **docs/ is not part of the skill** — `docs/requirements.md` and `docs/roadmap.md` are project tracking, not loaded by skill routing

## Key Architectural Concepts (for editing context files accurately)

- Every Holochain domain = one integrity crate (`hdi`) + one coordinator crate (`hdk`)
- Integrity code is locked to the DNA hash; coordinator code can be hot-swapped post-deployment
- Validation in integrity must be **pure/deterministic**: no `get()`, no `agent_info()`, no time comparisons
- Update chain tracking: `create_link(original_hash, updated_action_hash, LinkTypes::MyEntryUpdates)` — always link from original, never chain links
- `post_commit` is infallible — must use `#[hdk_extern(infallible)]`
- `send_remote_signal` is fire-and-forget; `recv_remote_signal` requires an unrestricted cap grant in `init()`
