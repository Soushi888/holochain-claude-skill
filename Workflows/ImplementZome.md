# Workflow: Implement a Zome Pair

Use this workflow when implementing a new zome pair (integrity + coordinator) for a Holochain domain. Prerequisites: data model designed (see `Workflows/DesignDataModel.md`).

---

## Step 1: Scaffold — Generate Boilerplate

Start from scaffold output to avoid blank-page overhead:

```bash
# Navigate to your DNA directory
cd dnas/my_dna

# Scaffold entry type (generates integrity + coordinator stubs)
hc scaffold entry-type MyEntry

# Scaffold link types
hc scaffold link-type AgentToMyEntry
hc scaffold link-type PathToMyEntry
hc scaffold link-type MyEntryUpdates

# Verify compilation after scaffolding
cd ../../
hc s sandbox generate workdir/
```

**What scaffolding generates:**
- Integrity crate: entry type variant + link type variants + stub `validate()`
- Coordinator crate: stub `create_*`, `get_*`, `update_*`, `delete_*` functions
- Updated `happ.yaml` and `dna.yaml` (verify these are correct)

After scaffolding: READ the generated files before editing. Understand what's there.

---

## Step 2: Integrity Crate — Define Types and Validation

**File: `zomes/integrity/{domain}_integrity/src/lib.rs`**

```rust
use hdi::prelude::*;

// 1. Entry struct (from DesignDataModel output)
#[hdk_entry_helper]
#[derive(Clone, PartialEq)]
pub struct MyEntry {
    pub title: String,
    pub description: String,
    pub status: MyEntryStatus,
}

// 2. Status enum (if soft-delete pattern needed)
#[derive(Serialize, Deserialize, Clone, PartialEq, Debug)]
pub enum MyEntryStatus {
    Active,
    Archived,
    Deleted,
}

// 3. Entry types enum (register all entry types)
#[hdk_entry_types]
#[unit_enum(UnitEntryTypes)]
pub enum EntryTypes {
    MyEntry(MyEntry),
}

// 4. Link types enum (register all link types)
#[hdk_link_types]
pub enum LinkTypes {
    AgentToMyEntry,
    PathToMyEntry,
    MyEntryUpdates,
}

// 5. Validation callback — MUST use op.flattened() (NOT op.to_type())
#[hdk_extern]
pub fn validate(op: Op) -> ExternResult<ValidateCallbackResult> {
    match op.flattened::<EntryTypes, LinkTypes>()? {
        FlatOp::StoreEntry(store_entry) => match store_entry {
            OpEntry::CreateEntry { app_entry, .. } => match app_entry {
                EntryTypes::MyEntry(entry) => validate_create_my_entry(entry),
            },
            OpEntry::UpdateEntry { app_entry, .. } => match app_entry {
                EntryTypes::MyEntry(entry) => validate_update_my_entry(entry),
            },
            _ => Ok(ValidateCallbackResult::Valid),
        },
        _ => Ok(ValidateCallbackResult::Valid),
    }
}

fn validate_create_my_entry(entry: MyEntry) -> ExternResult<ValidateCallbackResult> {
    if entry.title.trim().is_empty() {
        return Ok(ValidateCallbackResult::Invalid(
            "MyEntry title cannot be empty".into(),
        ));
    }
    Ok(ValidateCallbackResult::Valid)
}

fn validate_update_my_entry(entry: MyEntry) -> ExternResult<ValidateCallbackResult> {
    validate_create_my_entry(entry)
}
```

---

## Step 3: Coordinator Crate — Implement CRUD

**File: `zomes/coordinator/{domain}/src/my_entry.rs`**

Implement in this order: create → get_latest → get_all → update → delete

```rust
use hdk::prelude::*;
use {domain}_integrity::*;

// CREATE
#[hdk_extern]
pub fn create_my_entry(my_entry: MyEntry) -> ExternResult<Record> {
    let hash = create_entry(&EntryTypes::MyEntry(my_entry.clone()))?;

    // Path anchor
    let path = Path::from("entries.active");
    create_link(path.path_entry_hash()?, hash.clone(), LinkTypes::PathToMyEntry, ())?;

    // Agent index
    create_link(
        agent_info()?.agent_initial_pubkey,
        hash.clone(),
        LinkTypes::AgentToMyEntry,
        (),
    )?;

    get(hash, GetOptions::default())?
        .ok_or(wasm_error!(WasmErrorInner::Guest("Record not found after create".into())))
}

// GET LATEST (walks update chain)
#[hdk_extern]
pub fn get_latest_my_entry(original_action_hash: ActionHash) -> ExternResult<Option<Record>> {
    let links = get_links(
        GetLinksInputBuilder::try_new(original_action_hash.clone(), LinkTypes::MyEntryUpdates)?
            .build(),
    )?;
    let latest_hash = links
        .into_iter()
        .max_by(|a, b| a.timestamp.cmp(&b.timestamp))
        .and_then(|l| l.target.into_action_hash())
        .unwrap_or(original_action_hash);
    get(latest_hash, GetOptions::default())
}

// GET ALL (from path anchor)
#[hdk_extern]
pub fn get_all_my_entries(_: ()) -> ExternResult<Vec<Record>> {
    let path = Path::from("entries.active");
    let links = get_links(
        GetLinksInputBuilder::try_new(path.path_entry_hash()?, LinkTypes::PathToMyEntry)?.build(),
    )?;
    let inputs: Vec<GetInput> = links
        .into_iter()
        .filter_map(|l| l.target.into_action_hash())
        .map(|h| GetInput::new(h.into(), GetOptions::default()))
        .collect();
    let records = HDK.with(|hdk| hdk.borrow().get(inputs))?;
    Ok(records.into_iter().flatten().collect())
}

// UPDATE
#[hdk_extern]
pub fn update_my_entry(input: UpdateMyEntryInput) -> ExternResult<Record> {
    let original = get(input.original_action_hash.clone(), GetOptions::default())?
        .ok_or(wasm_error!(WasmErrorInner::Guest("Original not found".into())))?;
    if original.action().author() != &agent_info()?.agent_initial_pubkey {
        return Err(wasm_error!(WasmErrorInner::Guest("Not authorized".into())));
    }
    let updated = update_entry(input.previous_action_hash, &EntryTypes::MyEntry(input.updated_entry))?;
    create_link(input.original_action_hash, updated.clone(), LinkTypes::MyEntryUpdates, ())?;
    get(updated, GetOptions::default())?
        .ok_or(wasm_error!(WasmErrorInner::Guest("Record not found after update".into())))
}

// DELETE
#[hdk_extern]
pub fn delete_my_entry(original_action_hash: ActionHash) -> ExternResult<ActionHash> {
    // Clean path links
    let path = Path::from("entries.active");
    for link in get_links(
        GetLinksInputBuilder::try_new(path.path_entry_hash()?, LinkTypes::PathToMyEntry)?.build(),
    )? {
        if link.target.into_action_hash() == Some(original_action_hash.clone()) {
            delete_link(link.create_link_hash)?;
        }
    }
    delete_entry(original_action_hash)
}

// Input type for update (needed since update takes 3 params)
#[derive(Serialize, Deserialize, Debug)]
pub struct UpdateMyEntryInput {
    pub original_action_hash: ActionHash,
    pub previous_action_hash: ActionHash,
    pub updated_entry: MyEntry,
}
```

**lib.rs — register all functions:**
```rust
pub mod my_entry;
use my_entry::*;

#[hdk_extern]
pub fn create_my_entry(entry: MyEntry) -> ExternResult<Record> { my_entry::create_my_entry(entry) }
// ... repeat for all pub functions
```

---

## Step 4: Utils Crate (if cross-zome calls needed)

Add to `utils/src/errors.rs`:
```rust
// (see ErrorHandling.md for full pattern)
```

Add to `utils/src/cross_zome.rs`:
```rust
// (see Patterns.md for external_local_call helper)
```

Update workspace `Cargo.toml` to include utils crate.

---

## Step 5: Tests

Write tests in this order. Use **Sweettest** (Rust, `cargo test`) or **Tryorama** (TypeScript, `bun run test`) — see `Testing.md` for full patterns for both.

**Foundation (single-agent):**
```
1. Create an entry — assert record returned
2. Get latest — assert matches created entry
3. Get all — assert list contains created entry
4. Update — assert updated fields reflected
5. Delete — assert entry gone from list
```

**Integration (two agents):**
```
1. Alice creates → await_consistency / dhtSync → Bob reads — assert cross-agent read works
2. Alice creates → await_consistency / dhtSync → Bob gets all — assert entry in collection
3. Alice creates → updates → await_consistency / dhtSync → Bob gets latest — assert latest version
```

**Sweettest (Rust) commands:**
```bash
cargo test --package my_dna_tests
cargo test --package my_dna_tests two_agents  # single test
```

**Tryorama (TypeScript) commands:**
```bash
bun run test:foundation
bun run test:integration
```

See `Testing.md` for full code patterns including `await_consistency` (Sweettest) and `dhtSync` (Tryorama) placement.

---

## Step 6: Build and Verify

```bash
# Full build — verify no compile errors
hc s sandbox generate workdir/

# If build succeeds, run tests
bun run test:foundation

# After foundation passes, run integration
bun run test:integration
```

**Common build errors:**

| Error | Cause | Fix |
|-------|-------|-----|
| `cannot find type EntryTypes` | Missing import | Add `use {domain}_integrity::*;` |
| `op.to_type()` deprecated | Old API | Replace with `op.flattened()` |
| `expected ExternResult, found ValidateCallbackResult` | Wrong return | Use `Ok(ValidateCallbackResult::Valid)` |
| Link type not found | Unregistered link | Add to `#[hdk_link_types]` enum in integrity |
| `wasm-opt` timeout | Build too slow | Normal for first build; subsequent builds cache |
