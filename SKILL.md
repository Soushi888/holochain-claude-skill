---
name: holochain
description: >
  Holochain hApp development assistant covering coordinator/integrity zome
  architecture, Rust HDK/HDI patterns, entry/link types, CRUD, validation,
  cross-zome calls, Tryorama testing, TypeScript client integration, and
  Nix dev environments. USE WHEN writing zome code, designing DHT data models,
  scaffolding a new project, testing hApps, debugging HDK issues, implementing
  entry types or links, cap grants, access control, cell cloning, deploying
  or packaging hApps, or working on any Holochain project.
license: Apache-2.0
compatibility: >
  Requires Nix dev environment (holonix ref=main-0.6). Rust toolchain managed
  by Nix — no separate rustup install needed. Network access required for
  hc scaffold and nix flake updates.
metadata:
  author: soushi888
  version: "0.1.0"
  holochain-versions: "hdk=0.6.0, hdi=0.7.0, holonix ref=main-0.6"
---

# Holochain Development Skill

Expert assistant for Holochain hApp development. Covers the full development spiral: architecture, design, scaffolding, implementation, testing, and deployment.

## Proactive Invocation Rule

**Always invoke this skill in the PLAN phase** when the task touches a Holochain project. Do not wait to be asked explicitly.

Trigger conditions — any of these means the skill should be loaded before coding begins:
- Working directory is a Holochain project (contains `workdir/*.happ` or `dnas/*/zomes/`)
- Task involves `.rs` files inside `zomes/coordinator/` or `zomes/integrity/`
- Task involves entry types, link types, cross-DNA calls, or zome functions
- Task involves a PR on a Holochain project

When proactively invoked: load `Architecture.md` + `Patterns.md`, run the **ReviewZome** checklist against any files being modified, surface issues before implementation begins.

---

## Workflow Routing

| Workflow | Trigger | File |
|----------|---------|------|
| **ReviewZome** | review zome, audit zome, check implementation, validate patterns, before implementing, PR review, code review on zome | `Workflows/ReviewZome.md` |
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
| `Testing.md` | Four-layer strategy, Sweettest (Rust-native), E2E Playwright + AdminWebsocket, Wind-Tunnel performance |
| `WindTunnel.md` | Performance/load testing with wind-tunnel: ScenarioDefinitionBuilder, call_zome, ReportMetric, multi-agent roles, DHT sync lag measurement, InfluxDB metrics pipeline |
| `TypeScript.md` | holochain-client setup, callZome, signals, SvelteKit integration |
| `Deployment.md` | Packaging, distributing, Kangaroo-Electron, installers, desktop app, versioning |

## Quick Reference

```
Versions (current stable):  hdk = "=0.6.0"   hdi = "=0.7.0"   holonix ref=main-0.6
Dev commands:  nix develop  |  hc s sandbox generate workdir/  |  bun run test
Scaffold:      hc scaffold entry-type MyEntry  |  hc scaffold link-type AgentToMyEntry
```

## Common Pitfalls Checklist

Run this against any zome code being written or reviewed. Each item is a class of bug that has burned projects before.

### Entry Schema Evolution
- [ ] **`#[serde(default)]` on new optional fields** — Any field added to an existing entry struct after initial deployment MUST have `#[serde(default)]`. Without it, existing entries serialized before the field existed will fail to deserialize. `Option<T>` alone is NOT sufficient.
  ```rust
  #[serde(default)]          // ← REQUIRED for fields added post-deployment
  pub new_field: Option<ActionHash>,
  ```

### Cross-DNA Calls
- [ ] **`ZomeCallResponse` is exhaustive** — HDK 0.6 has 5 variants: `Ok`, `Unauthorized`, `AuthenticationFailed`, `NetworkError`, `CountersigningSession`. Wildcard `_` is safe but hides new variants. Exhaustive match is preferred.
- [ ] **Role name matches `happ.yaml`** — `CallTargetCell::OtherRole("role_name")` must exactly match the role name in `workdir/happ.yaml`. Typos fail silently at runtime.
- [ ] **Zome name matches coordinator crate name** — `ZomeName("zome_name")` must match the coordinator's `name` in `Cargo.toml`. Check both.
- [ ] **Local mirror structs for cross-DNA types** — Avoid importing the remote DNA's Cargo crate. Define a local serialization mirror struct instead.

### Validation Rules
- [ ] **No DHT reads in `validate()`** — `validate()` must be deterministic. No `get()`, `get_links()`, `agent_info()`, `sys_time()`. Only inspect the op itself.
- [ ] **Use `op.flattened::<EntryTypes, LinkTypes>()`** — Not the old `op.to_type()`. Patterns.md has the correct pattern.

### HDK 0.6 API
- [ ] **`delete_link()` requires `GetOptions`** — `delete_link(hash, GetOptions::default())` not `delete_link(hash)`.
- [ ] **`get_links()` uses `LinkQuery::try_new()`** — Not `GetLinksInputBuilder` for most cases.
- [ ] **`GetStrategy::Local` vs `Network`** — Use `Local` for own-data queries (fast, no network), `Network` for DHT queries (cross-agent data).

### Shared Utility Patterns (project-specific)
- [ ] **`agent_pub_key` and `created_at` are NOT entry fields** — They live in the action header. Remove them from entry structs.
- [ ] **If using a shared utility crate** — verify intra-DNA and cross-DNA call helpers are used consistently rather than raw `call()` inline.

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
