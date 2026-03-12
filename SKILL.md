---
name: Holochain
description: Holochain hApp development assistant covering coordinator/integrity zome architecture, Rust HDK/HDI patterns, entry/link types, CRUD, validation, cross-zome calls, Tryorama testing, TypeScript client integration, Nix dev environments, and deployment. USE WHEN writing zome code, designing DHT data models, scaffolding a new project, testing hApps, debugging HDK issues, implementing entry types or links, cap grants, access control, cell cloning, deploying or packaging hApps, or working on any Holochain project.
---

# Holochain Development Skill

Expert assistant for Holochain hApp development. Covers the full development spiral: architecture, design, scaffolding, implementation, testing, and deployment.

## Workflow Routing

| Workflow | Trigger | File |
|----------|---------|------|
| **DesignDataModel** | design data model, model entries, what entries, what links, DHT schema | `Workflows/DesignDataModel.md` |
| **Scaffold** | scaffold, new happ, new project, setup environment, init project, Holonix, nix develop, hc scaffold | `Workflows/Scaffold.md` |
| **ImplementZome** | implement zome, create zome, scaffold zome, write zome | `Workflows/ImplementZome.md` |
| **DesignAccessControl** | design access control, who can call, cap grant design | `Workflows/DesignAccessControl.md` |
| **PackageAndDeploy** | deploy, package, distribute, kangaroo, installer, desktop app, webhapp | `Workflows/PackageAndDeploy.md` |

## Context Files

Load on demand based on task:

| File | Load When |
|------|-----------|
| `Architecture.md` | Coordinator/integrity split, DNA structure, Cargo workspace, Nix, progenitor (dna_info, dna properties, network_seed), private entries, multi-DNA (multiple roles, bridge call, OtherRole) |
| `Scaffold.md` | New project setup, Holonix installation, Nix flake, hc CLI, `hc scaffold` commands, adding a new domain to existing project |
| `Patterns.md` | Entry types, link types, CRUD, cross-zome calls, validation, HDK 0.6 API (GetStrategy, LinkQuery, Local vs Network), must_get, signals (remote signal, init cap grant) |
| `AccessControl.md` | Cap grants, capability system, cap claim, recv_remote_signal setup, admin-only access |
| `CellCloning.md` | Cell cloning, partitioned data, clone roles, createCloneCell, clone_limit |
| `ErrorHandling.md` | Error types, WasmError, ExternResult patterns, thiserror |
| `Testing.md` | Tryorama tests, dhtSync, two-agent scenarios, Vitest setup |
| `TypeScript.md` | holochain-client setup, callZome, signals, SvelteKit integration |
| `Deployment.md` | Packaging, distributing, Kangaroo-Electron, installers, desktop app, versioning |

## Quick Reference

```
Versions (current stable):  hdk = "=0.6.0"   hdi = "=0.7.0"   holonix ref=main-0.6
Dev commands:  nix develop  |  hc s sandbox generate workdir/  |  bun run test
Scaffold:      hc scaffold entry-type MyEntry  |  hc scaffold link-type AgentToMyEntry
```

## Examples

**Example 1: Design a new entry type for a marketplace listing**
```
User: "I need to model a Listing entry with status transitions"
→ Loads Patterns.md (entry types, status enum, link types)
→ Designs ListingStatus enum (Active/Archived/Deleted)
→ Defines link types (AgentToListing, PathToListing, ListingUpdates)
→ Implements soft-delete via status field update, not entry deletion
```

**Example 2: Debug a cross-agent test that fails intermittently**
```
User: "My Tryorama test passes alone but fails when another agent reads the entry"
→ Loads Testing.md
→ Identifies missing dhtSync call before cross-agent read
→ Adds dhtSync([alice, bob], t) after Alice's create, before Bob's get
→ Test passes reliably
```

**Example 3: Scaffold a new hApp from scratch**
```
User: "Start a new Holochain project for a community coordination app"
→ Loads Scaffold.md + Workflows/Scaffold.md
→ Guides: nix flake setup → hc scaffold happ → first DNA → first zome pair
→ Verifies compilation with hc s sandbox generate workdir/
```

**Example 4: Implement CRUD for a new zome**
```
User: "Implement a full resource zome with create, read, update, delete"
→ Loads Architecture.md + Patterns.md
→ Invokes Workflows/ImplementZome.md
→ Creates integrity crate (entry struct, link enum, validation)
→ Creates coordinator crate (create/read/update/delete functions)
→ Writes Tryorama tests at foundation + integration layers
```
