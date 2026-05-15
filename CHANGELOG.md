# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [0.2.0] — 2026-05-15

### Added

- mdBook documentation site — `book.toml`, `SUMMARY.md`, and GitHub Actions deploy to GitHub Pages on every push to `main`
- mdBook frontmatter-strip preprocessor — strips YAML frontmatter so it does not appear in rendered pages
- `Progenitor.md` — dedicated context file for the progenitor pattern (DnaProperties, check_if_progenitor, Moss enforcement, bootstrap auto-registration)

### Changed

- Version pins bumped: `hdk = "=0.6.1"`, `hdi = "=0.7.1"`, `holonix ref=main-0.6`

## [0.1.0] — 2026-04-01

Initial release of the Holochain agent skill.

### Added

- `SKILL.md` — entry point with workflow routing table and context-file index
- `Architecture.md` — coordinator/integrity split, DNA structure, Cargo workspace, Nix, private entries, multi-DNA
- `Patterns.md` — entry types, link types, CRUD patterns, update chain, validation, signals, HDK API
- `Scaffold.md` — Holonix setup, Nix flake, hc CLI commands, project scaffolding
- `AccessControl.md` — capability grants, cap claims, admin-only patterns, init() setup
- `CellCloning.md` — clone cells, partitioned data, createCloneCell, clone_limit
- `ErrorHandling.md` — thiserror enums, WasmError, ExternResult patterns
- `Testing.md` — Tryorama + Vitest setup, two-agent scenarios, dhtSync; Sweettest (Rust-native) patterns; three-layer testing strategy; E2E Playwright integration
- `TypeScript.md` — holochain-client setup, callZome, signals, SvelteKit integration
- `Deployment.md` — Kangaroo-Electron packaging, .webhapp bundling, CI/CD, versioning
- `WindTunnel.md` — performance and load testing reference using the wind-tunnel framework
- `Workflows/DesignDataModel.md` — DHT entry/link type design workflow
- `Workflows/Scaffold.md` — new project and domain scaffolding workflow
- `Workflows/ImplementZome.md` — full CRUD zome implementation workflow
- `Workflows/DesignAccessControl.md` — capability grants and admin patterns workflow
- `Workflows/PackageAndDeploy.md` — Kangaroo-Electron and CI/CD workflow
- `Workflows/ReviewZome.md` — guided code review checklist for coordinator and integrity zomes
- `docs/requirements.md` — v1 functional and non-functional requirements
- `docs/roadmap.md` — v1/v2/v3 vision and sub-skill roadmap
- Agent Skills Open Standard v1 compliance — YAML frontmatter, metadata block, and standard routing table in `SKILL.md`
- Multi-platform installation section in `README.md` covering Claude Code, GitHub Copilot, Cursor, Augment, and Codex
- DeepWiki badge in `README.md`

### Changed

- Repository renamed from `holochain-claude-skill` to `holochain-agent-skill`; all internal references updated
- All occurrences of "Claude Skill" replaced with "agent skill" for platform neutrality
- Clone URL updated to the canonical GitHub repository

[0.2.0]: https://github.com/Soushi888/holochain-agent-skill/compare/v0.1.0...v0.2.0
[0.1.0]: https://github.com/Soushi888/holochain-agent-skill/releases/tag/v0.1.0
