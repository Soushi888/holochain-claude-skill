# Testing Plan: Holochain Agent Skill v1

**Status:** Pre-release checklist
**Target:** v1.0.0 release gate

All tests are manual unless marked `[auto]`. Check each box before cutting a release.

---

## T1 — Agent Skills Open Standard Conformance

Validate that `SKILL.md` frontmatter meets the [Agent Skills Open Standard](https://agentskills.io) spec.

```bash
head -25 SKILL.md
```

| # | Test | Pass Condition |
|---|------|----------------|
| T1.1 | `name` field present | Key exists in frontmatter |
| T1.2 | `name` is lowercase | Value is `holochain` (no uppercase, no spaces) |
| T1.3 | `name` uses only alphanumeric + hyphens | Regex: `^[a-z0-9-]+$` |
| T1.4 | `description` field present | Key exists |
| T1.5 | `description` is between 1 and 1024 characters | `wc -c` on the value |
| T1.6 | `description` mentions primary use cases | Contains: "zome", "HDK", "Holochain" |
| T1.7 | `license` field is `Apache-2.0` | Value exactly matches |
| T1.8 | `compatibility` field present and non-empty | Key exists, value not blank |
| T1.9 | `metadata.author` is `soushi888` | Value matches |
| T1.10 | `metadata.version` is present | Key exists, SemVer format |
| T1.11 | `metadata.holochain-versions` references current pins | Contains `hdk=0.6.0`, `hdi=0.7.0`, `holonix ref=main-0.6` |

---

## T2 — File Integrity

Verify every file referenced in `SKILL.md` routing tables actually exists.

```bash
ls Workflows/*.md
ls *.md
```

| # | File | Exists? |
|---|------|---------|
| T2.1 | `Workflows/DesignDataModel.md` | ☐ |
| T2.2 | `Workflows/Scaffold.md` | ☐ |
| T2.3 | `Workflows/ImplementZome.md` | ☐ |
| T2.4 | `Workflows/DesignAccessControl.md` | ☐ |
| T2.5 | `Workflows/PackageAndDeploy.md` | ☐ |
| T2.6 | `Architecture.md` | ☐ |
| T2.7 | `Scaffold.md` | ☐ |
| T2.8 | `Patterns.md` | ☐ |
| T2.9 | `AccessControl.md` | ☐ |
| T2.10 | `CellCloning.md` | ☐ |
| T2.11 | `ErrorHandling.md` | ☐ |
| T2.12 | `Testing.md` | ☐ |
| T2.13 | `TypeScript.md` | ☐ |
| T2.14 | `Deployment.md` | ☐ |
| T2.15 | `LICENSE` at repo root | ☐ |
| T2.16 | `README.md` at repo root | ☐ |

---

## T3 — Routing Accuracy

For each Workflow Routing entry in `SKILL.md`, verify the trigger resolves to the correct file and the file's content matches the described purpose.

| # | Trigger phrase | Expected file | Content check |
|---|----------------|---------------|---------------|
| T3.1 | "design data model" | `Workflows/DesignDataModel.md` | Contains Step 1 (domains/zome pairs) and Step 2 (entry type definition) |
| T3.2 | "new happ" | `Workflows/Scaffold.md` | Contains Nix install and `hc scaffold happ` commands |
| T3.3 | "implement zome" | `Workflows/ImplementZome.md` | Contains `hc scaffold entry-type` and integrity/coordinator structure |
| T3.4 | "who can call" | `Workflows/DesignAccessControl.md` | Contains `CapAccess::Unrestricted`, `CapAccess::Assigned` |
| T3.5 | "package" | `Workflows/PackageAndDeploy.md` | Contains Kangaroo-Electron setup steps |

For each Context Files entry in `SKILL.md`:

| # | Load-when trigger | Expected file | Content check |
|---|-------------------|---------------|---------------|
| T3.6 | coordinator/integrity split | `Architecture.md` | Contains `hdi` and `hdk` crate explanation |
| T3.7 | Nix flake setup | `Scaffold.md` | Contains `nix develop` and `flake.nix` |
| T3.8 | entry types, CRUD | `Patterns.md` | Contains `#[hdk_entry_helper]` and `create_entry()` |
| T3.9 | cap grants | `AccessControl.md` | Contains `CapAccess::Unrestricted` and `init()` |
| T3.10 | cell cloning | `CellCloning.md` | Contains `createCloneCell` and `clone_limit` |
| T3.11 | WasmError | `ErrorHandling.md` | Contains `WasmError` and `ExternResult` |
| T3.12 | Tryorama tests | `Testing.md` | Contains `dhtSync` and two-agent scenario |
| T3.13 | holochain-client | `TypeScript.md` | Contains `callZome` and signal handling |
| T3.14 | packaging, Kangaroo | `Deployment.md` | Contains `.webhapp` and versioning guidance |

---

## T4 — Content Coverage

Verify each of the 6 skill domains has substantive (non-stub) content.

| # | Domain | Primary file | Pass condition |
|---|--------|--------------|----------------|
| T4.1 | Architecture | `Architecture.md` | > 100 lines, covers integrity/coordinator split |
| T4.2 | Design | `Workflows/DesignDataModel.md` | Has at least 4 numbered steps with examples |
| T4.3 | Scaffold | `Scaffold.md` + `Workflows/Scaffold.md` | Contains `nix develop`, `hc scaffold happ`, Nix flake template |
| T4.4 | Implement | `Patterns.md` | Contains CRUD patterns, link types, validation section |
| T4.5 | Test | `Testing.md` | Contains Tryorama setup, `dhtSync`, two-agent example |
| T4.6 | Deploy | `Deployment.md` + `Workflows/PackageAndDeploy.md` | Contains `kangaroo-electron`, `.webhapp` bundling, versioning |

---

## T5 — Code Example Accuracy

Validate specific API calls against the actual HDK 0.6 API (use the hAppenings or Nondominium codebase as reference).

### HDK / HDI API

| # | Example to validate | Expected form | File |
|---|---------------------|---------------|------|
| T5.1 | Entry type macro | `#[hdk_entry_helper]` on struct | `Patterns.md` |
| T5.2 | Entry type enum in integrity | `#[hdk_entry_types]` on enum with `#[unit_enum(UnitEntryTypes)]` | `Patterns.md` |
| T5.3 | Create entry | `create_entry(EntryTypes::MyEntry(entry))` | `Patterns.md` |
| T5.4 | Get entry | `get(hash, GetOptions::default())` or `must_get_entry(hash)` | `Patterns.md` |
| T5.5 | Delete link | `delete_link(link_hash, GetOptions::default())` (second arg required in 0.6) | `Patterns.md` |
| T5.6 | Link types enum | `#[hdk_link_types]` on enum | `Patterns.md` |
| T5.7 | Update chain tracking | `create_link(original_hash, new_hash, LinkTypes::EntryUpdates, ())` | `Patterns.md` |
| T5.8 | Validation signature | `pub fn validate(op: Op) -> ExternResult<ValidateCallbackResult>` | `Patterns.md` |
| T5.9 | `post_commit` infallible | `#[hdk_extern(infallible)]` + `pub fn post_commit(...)` | `Architecture.md` or `Patterns.md` |
| T5.10 | Remote signal cap grant | `CapAccess::Unrestricted` grant created in `init()` | `AccessControl.md` |
| T5.11 | `dhtSync` call | `dhtSync([&alice, &bob], &conductor)` or equivalent Tryorama API | `Testing.md` |
| T5.12 | Scaffold compile check | `hc s sandbox generate workdir/` | `Workflows/ImplementZome.md` |

### Version pin consistency `[auto]`

```bash
grep -rn "hdk\s*=\s*\"=" . --include="*.md" --include="*.toml" | grep -v Plans/
grep -rn "hdi\s*=\s*\"=" . --include="*.md" --include="*.toml" | grep -v Plans/
grep -rn "holonix" . --include="*.md" | grep -v Plans/
```

| # | Check | Expected value | Pass condition |
|---|-------|----------------|----------------|
| T5.13 | `hdk` pin in SKILL.md Quick Reference | `"=0.6.0"` | All occurrences match |
| T5.14 | `hdi` pin in SKILL.md Quick Reference | `"=0.7.0"` | All occurrences match |
| T5.15 | `holonix ref` in SKILL.md and Scaffold.md | `main-0.6` | All occurrences match |
| T5.16 | No file references `hdk = "0.5.*"` or older | — | Zero matches |
| T5.17 | PackageAndDeploy.md Cargo.toml example pins match current | `hdk = "=0.6.0"` | Matches T5.13 |

---

## T6 — Installation Tests

Perform each installation method in a clean environment.

### Option A — Global copy

```bash
cd /tmp
git clone https://github.com/Soushi888/holochain-agent-skill
cp -r holochain-agent-skill ~/.claude/skills/holochain
```

| # | Test | Pass condition |
|---|------|----------------|
| T6.1 | Directory created | `~/.claude/skills/holochain/` exists |
| T6.2 | `SKILL.md` present inside | `~/.claude/skills/holochain/SKILL.md` exists |
| T6.3 | `Workflows/` subdirectory present | `~/.claude/skills/holochain/Workflows/` exists with 5 files |
| T6.4 | All context files present | `Architecture.md`, `Patterns.md`, etc. all copied |

### Option B — Project-local

```bash
cd /tmp/my-test-project
mkdir -p .claude/skills
cp -r /tmp/holochain-agent-skill .claude/skills/holochain
```

| # | Test | Pass condition |
|---|------|----------------|
| T6.5 | Skill installed at project path | `.claude/skills/holochain/SKILL.md` exists |
| T6.6 | Does not affect global `~/.claude/skills/` | Global directory unchanged |

### Option C — Symlink

```bash
git clone https://github.com/Soushi888/holochain-agent-skill ~/holochain-agent-skill
ln -s ~/holochain-agent-skill ~/.claude/skills/holochain
```

| # | Test | Pass condition |
|---|------|----------------|
| T6.7 | Symlink created | `~/.claude/skills/holochain` is a symlink |
| T6.8 | Symlink resolves | `ls -la ~/.claude/skills/holochain/SKILL.md` returns file |
| T6.9 | `git pull` propagates | Pull in `~/holochain-agent-skill`, symlink sees updates immediately |

---

## T7 — Invocation Tests

Verify the skill loads and responds correctly in Claude Code.

| # | Test | Steps | Pass condition |
|---|------|-------|----------------|
| T7.1 | Explicit command invocation | Type `/holochain` in Claude Code | Skill loads, greets with Holochain context |
| T7.2 | Natural language trigger — workflow | Type "implement zome for Profile entry type" | `Workflows/ImplementZome.md` guidance appears |
| T7.3 | Natural language trigger — context file | Type "how do I set up a Tryorama test?" | `Testing.md` content cited |
| T7.4 | Natural language trigger — scaffold | Type "scaffold a new happ called my-network" | `Workflows/Scaffold.md` steps appear |
| T7.5 | Version question | Ask "what version of hdk does this skill target?" | Responds with `0.6.0` |
| T7.6 | Out-of-scope question | Ask a non-Holochain question | Skill does not answer as if it's Holochain-related |

---

## T8 — PAI Independence

Verify the skill works in a clean Claude Code environment with no PAI infrastructure.

| # | Test | Steps | Pass condition |
|---|------|-------|----------------|
| T8.1 | No `~/.claude/PAI/` required | Temporarily rename `~/.claude/PAI/` to `~/.claude/PAI_bak/`, invoke skill | Skill loads without error |
| T8.2 | No voice curl in skill files | `grep -r "localhost:8888" .` | Zero matches |
| T8.3 | No Algorithm routing references | `grep -r "ALGORITHM\|AlgorithmMode\|PAI/Algorithm" .` | Zero matches in skill files |
| T8.4 | No PROJECTS.md references | `grep -r "PROJECTS.md" .` | Zero matches in skill files |
| T8.5 | Restore PAI after test | `mv ~/.claude/PAI_bak ~/.claude/PAI` | Restore before next session |

---

## T9 — Workflow End-to-End Tests

For each workflow, walk through the steps in Claude Code with a real or simulated project and verify guidance is accurate and complete.

### T9.A — DesignDataModel

**Trigger:** "design data model for a marketplace listing"

| # | Step | Pass condition |
|---|------|----------------|
| T9.A.1 | Step 1: Identify domains | Skill asks or describes how to map business nouns to zome pairs |
| T9.A.2 | Step 2: Define entry types | Produces a Rust struct definition with field types |
| T9.A.3 | Step 3: Define link types | Produces at least AgentTo*, PathTo*, *Updates link types |
| T9.A.4 | Step 4: Discovery strategy | Explains Path anchor vs. agent-linked discovery tradeoffs |
| T9.A.5 | Step 5: Validation rules | Produces at least one validation rule per entry type |
| T9.A.6 | Output completeness | Produces a summary table or structured output usable as implementation spec |

### T9.B — Scaffold

**Trigger:** "scaffold new happ called community-app"

| # | Step | Pass condition |
|---|------|----------------|
| T9.B.1 | Nix install step | Provides `curl` Determinate Nix installer command |
| T9.B.2 | flake.nix creation | Provides template with `holonix ref=main-0.6` |
| T9.B.3 | `hc scaffold happ` command | Correct command with app name parameter |
| T9.B.4 | First DNA scaffold | `hc scaffold dna` command shown |
| T9.B.5 | First zome pair scaffold | `hc scaffold zome` for integrity + coordinator |
| T9.B.6 | Compile verification | `hc s sandbox generate workdir/` step present |

### T9.C — ImplementZome

**Trigger:** "implement zome for Profile entry type"

| # | Step | Pass condition |
|---|------|----------------|
| T9.C.1 | Scaffold step | `hc scaffold entry-type Profile` and link-type commands shown |
| T9.C.2 | Integrity crate | Produces `Profile` struct with `#[hdk_entry_helper]`, entry type enum |
| T9.C.3 | Validation function | Produces `validate()` function with `Op` pattern matching |
| T9.C.4 | Coordinator — create | Produces `create_profile()` using `create_entry()` |
| T9.C.5 | Coordinator — read | Produces `get_profile()` using `get()` with `GetOptions::default()` |
| T9.C.6 | Coordinator — update | Uses `update_entry()` and `create_link()` for update chain |
| T9.C.7 | Coordinator — delete | Uses `delete_entry()` and handles link cleanup |
| T9.C.8 | Test scaffold | Produces at minimum a two-agent Tryorama test structure |

### T9.D — DesignAccessControl

**Trigger:** "design access control for my admin zome"

| # | Step | Pass condition |
|---|------|----------------|
| T9.D.1 | Caller mapping table | Produces table of function → caller type |
| T9.D.2 | Unrestricted grant | Shows `init()` with `CapAccess::Unrestricted` for remote signals |
| T9.D.3 | Progenitor check | Shows `dna_info().provenance` check for admin-only functions |
| T9.D.4 | Assigned grant | Shows `CapAccess::Assigned` pattern with agent key |
| T9.D.5 | `recv_remote_signal` | Shows correct extern signature and cap grant pairing |

### T9.E — PackageAndDeploy

**Trigger:** "package my happ for desktop distribution"

| # | Step | Pass condition |
|---|------|----------------|
| T9.E.1 | Version compatibility check | Asks for or checks `hdk`/`hdi` versions before proceeding |
| T9.E.2 | Kangaroo-Electron setup | `git clone` command for Kangaroo repo shown |
| T9.E.3 | `.happ` bundle step | `hc app pack` or equivalent command shown |
| T9.E.4 | `.webhapp` bundle step | UI + `.happ` combined packaging step shown |
| T9.E.5 | Versioning guidance | Explains semantic version bump for DNA updates vs UI-only updates |
| T9.E.6 | CI/CD note | At minimum mentions GitHub Actions or manual release process |

---

## T10 — Cross-Tool Compatibility

### Claude Code (primary)

Covered by T7 and T9 above.

### GitHub Copilot

| # | Test | Pass condition |
|---|------|----------------|
| T10.1 | Install to `.claude/skills/holochain/` in project root | Directory exists with `SKILL.md` |
| T10.2 | Copilot agent mode recognizes skill | Skill name `holochain` appears in available skills list |
| T10.3 | Basic invocation | Copilot responds with Holochain context when asked about zomes |

### Cursor

| # | Test | Pass condition |
|---|------|----------------|
| T10.4 | Install to `.claude/skills/holochain/` in project root | Directory exists with `SKILL.md` |
| T10.5 | Cursor agent detects skill | Skill is listed or referenced in agent context |
| T10.6 | Basic invocation | Cursor responds with Holochain guidance when triggered |

### Augment Code

| # | Test | Pass condition |
|---|------|----------------|
| T10.7 | Install to `.claude/skills/holochain/` | Directory exists |
| T10.8 | Skill loaded by Augment | Skill context is included in agent workspace |

### OpenAI Codex CLI

| # | Test | Pass condition |
|---|------|----------------|
| T10.9 | Install to `.claude/skills/holochain/` | Directory exists |
| T10.10 | Codex reads SKILL.md frontmatter | Invocation triggers Holochain-domain responses |

> **Note:** T10.2–T10.10 require access to each tool. Mark as `N/A` if the tool is not installed. T10.1 and T10.4 are always testable.

---

## T11 — Repository Hygiene

| # | Test | Command | Pass condition |
|---|------|---------|----------------|
| T11.1 | `LICENSE` file is Apache-2.0 | `head -3 LICENSE` | Contains "Apache License, Version 2.0" |
| T11.2 | No `Plans/` content ships as skill | `SKILL.md` routing table has no reference to `Plans/` | Zero `Plans/` entries in routing table |
| T11.3 | No `docs/` loaded by skill | `SKILL.md` routing table has no reference to `docs/` | Zero `docs/` entries in routing table |
| T11.4 | No broken markdown links | Scan for `[text](file.md)` links in all files | All linked files exist |
| T11.5 | No TODO / STUB markers | `grep -rn "TODO\|STUB\|PLACEHOLDER" . --include="*.md"` | Zero matches in non-Plans/ files |
| T11.6 | README reflects current install path | `grep "holochain-agent-skill" README.md` | All cp/ln commands use `holochain-agent-skill` as source |
| T11.7 | CLAUDE.md license annotation | `grep "Apache" CLAUDE.md` | Matches `Apache-2.0` |

---

## Release Gate

All items below must be ✅ before tagging a release.

**Spec & Structure (non-negotiable)**
- [ ] T1 — All 11 frontmatter checks pass
- [ ] T2 — All 16 files exist
- [ ] T3 — All 14 routing entries resolve correctly
- [ ] T11 — All 7 hygiene checks pass

**Content**
- [ ] T4 — All 6 domains have substantive content
- [ ] T5.13–T5.17 — Version pins consistent across all files

**Code Accuracy** (sample — validate at least 6 of 12)
- [ ] T5.1–T5.12 — At least 6 code examples verified against real codebase

**Installation**
- [ ] T6.1–T6.4 — Option A passes
- [ ] T6.5–T6.6 — Option B passes
- [ ] T6.7–T6.9 — Option C passes

**Invocation**
- [ ] T7.1–T7.5 — Claude Code invocation tests pass (T7.6 optional)

**Workflows** (all 5 required)
- [ ] T9.A — DesignDataModel workflow complete
- [ ] T9.B — Scaffold workflow complete
- [ ] T9.C — ImplementZome workflow complete
- [ ] T9.D — DesignAccessControl workflow complete
- [ ] T9.E — PackageAndDeploy workflow complete

**Independence**
- [ ] T8.1–T8.4 — PAI independence verified

**Cross-tool** (Claude Code required; others optional for v1)
- [ ] T10.1 — Install path verified for at least one non-Claude-Code tool

---

*Once all release gate items are checked, tag `v1.0.0` and publish.*
