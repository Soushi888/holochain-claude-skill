# Holochain Patterns

## Entry Types (Integrity Crate)

**What NOT to put in entry fields — already in action headers:**

Every committed action carries free metadata in its header. Never duplicate these as entry fields:

| Already in header | How to access (coordinator) |
|-------------------|-----------------------------|
| Author (agent pubkey) | `record.action().author()` |
| Timestamp | `record.action().timestamp()` |
| Entry hash | `record.action().entry_hash()` |
| Previous action hash | available on `Update`/`Delete` actions |

If you find yourself adding `created_by: AgentPubKey` or `created_at: Timestamp` to an entry struct, remove them — they're already there.

```rust
use hdi::prelude::*;

// Entry struct — always derive these
#[hdk_entry_helper]
#[derive(Clone, PartialEq)]
pub struct MyEntry {
    pub title: String,
    pub description: String,
    pub status: MyEntryStatus,
    // Use #[serde(default)] for fields added after initial deployment
    #[serde(default)]
    pub tags: Vec<String>,
    // DO NOT add: author, created_at, updated_at — those are in the action header
}

// Status enum for soft-delete pattern
#[derive(Serialize, Deserialize, Clone, PartialEq, Debug)]
pub enum MyEntryStatus {
    Active,
    Archived,
    Deleted,
}

// Register all entry types in one enum (integrity crate)
#[hdk_entry_types]
#[unit_enum(UnitEntryTypes)]
pub enum EntryTypes {
    MyEntry(MyEntry),
    AnotherEntry(AnotherEntry),
}
```

---

## Link Types (Integrity Crate)

```rust
// Register all link types in one enum (integrity crate)
#[hdk_link_types]
pub enum LinkTypes {
    // Naming convention: BaseToTarget (PascalCase)
    AgentToMyEntry,
    PathToMyEntry,
    MyEntryUpdates,       // Update chain tracking
    MyEntryToRelated,     // Bidirectional: also RelatedToMyEntry
    RelatedToMyEntry,
}
```

**Naming convention:** `{Base}To{Target}` — always PascalCase, always directional.

---

## Implicit vs. Explicit Links

Holochain has two layers of navigable relationships. Understanding the distinction prevents over-engineering and redundant data.

### Implicit — action metadata and DHT metadata (no `create_link` needed)

**1. Action metadata** — fields baked into every action header:

| Field | Type | How to access |
|-------|------|---------------|
| `author` | `AgentPubKey` | `record.action().author()` |
| `timestamp` | `Timestamp` | `record.action().timestamp()` |
| `original_action_address` | `ActionHash` | only on `Action::Update` — the original creation action |
| `deletes_address` | `ActionHash` | only on `Action::Delete` — the action being deleted |

Walking **backward** through an update chain uses this — no links needed:
```rust
// From any update action hash → find the original
match record.action().clone() {
    Action::Update(u) => current_hash = u.original_action_address, // go back one step
    Action::Create(_) => return Ok(OriginalActionHash(current_hash)), // found it
    _ => ...
}
```

**2. DHT metadata** — aggregated by the DHT automatically, returned by `get_details`:

```rust
pub struct RecordDetails {
    pub record: Record,
    pub validation_status: ValidationStatus,
    pub updates: Vec<SignedHashed<Action>>, // all Update actions on this record
    pub deletes: Vec<SignedHashed<Action>>, // all Delete actions on this record
}

pub struct EntryDetails {
    pub entry: Entry,
    pub actions: Vec<SignedHashed<Action>>, // all Create/Update actions for this entry
    pub updates: Vec<SignedHashed<Action>>,
    pub deletes: Vec<SignedHashed<Action>>,
}
```

**3. Embedded ActionHash in entry fields** — a relationship baked INTO the entry content

```rust
#[hdk_entry_helper]
#[derive(Clone, PartialEq)]
pub struct Offer {
    pub title: String,
    pub organization_hash: ActionHash, // embedded relationship — no create_link needed
}
```

**Critical tradeoff:** If `organization_hash` changes, the content changes → new entry hash → requires `update_entry`. Use embedded hashes when the reference is intrinsic to the entry's identity. Use explicit links when the relationship may change independently.

### Explicit links — you define, create, and query them

| Link type | Purpose |
|-----------|---------|
| `PathToMyEntry` | Global discovery — browse all entries from a known path string |
| `AgentToMyEntry` | Per-agent listing — "show me this agent's entries" |
| `MyEntryUpdates` | Forward traversal — original hash → latest version |
| `MyEntryToRelated` | Cross-domain relationship navigation |

### Decision rule

| Question | Tool |
|----------|------|
| "Who created this entry? When?" | `record.action().author()` / `.timestamp()` — no links |
| "Has this record been updated or deleted?" | `get_details(action_hash)` → `.updates` / `.deletes` |
| "What is the LATEST version of this entry?" | `get_links(original_hash, UpdatesLinkType)` → max timestamp |
| "Find entries without knowing any hash" | Explicit `PathTo*` or `AgentTo*` links |
| "Navigate from entry A to related entry B" | Explicit `AToB` link |
| "Link is intrinsic to entry identity?" | Embedded `ActionHash` field in entry struct |
| "Link may change independently of entry?" | Explicit link — keeps entry hash stable |

---

## Create Pattern

```rust
pub fn create_my_entry(my_entry: MyEntry) -> ExternResult<Record> {
    let my_entry_hash = create_entry(&EntryTypes::MyEntry(my_entry.clone()))?;

    // 1. Discovery anchor (path)
    let path = Path::from("entries.active");
    create_link(
        path.path_entry_hash()?,
        my_entry_hash.clone(),
        LinkTypes::PathToMyEntry,
        (),
    )?;

    // 2. Agent index
    let agent_info = agent_info()?;
    create_link(
        agent_info.agent_initial_pubkey,
        my_entry_hash.clone(),
        LinkTypes::AgentToMyEntry,
        (),
    )?;

    // 3. Get and return the full record
    let record = get(my_entry_hash.clone(), GetOptions::default())?
        .ok_or(wasm_error!(WasmErrorInner::Guest("Entry not found after create".into())))?;

    Ok(record)
}
```

---

## Read Latest Pattern (Walking Update Chain)

```rust
pub fn get_latest_my_entry(original_action_hash: ActionHash) -> ExternResult<Option<Record>> {
    let links = get_links(
        GetLinksInputBuilder::try_new(original_action_hash.clone(), LinkTypes::MyEntryUpdates)?
            .build(),
    )?;

    let latest_link = links
        .into_iter()
        .max_by(|a, b| a.timestamp.cmp(&b.timestamp));

    let latest_hash = match latest_link {
        Some(link) => {
            link.target
                .into_action_hash()
                .ok_or(wasm_error!(WasmErrorInner::Guest("Invalid target hash".into())))?
        }
        None => original_action_hash, // No updates — original is latest
    };

    get(latest_hash, GetOptions::default())
}
```

---

## Read Collection Pattern

```rust
pub fn get_all_my_entries() -> ExternResult<Vec<Record>> {
    let path = Path::from("entries.active");
    let links = get_links(
        GetLinksInputBuilder::try_new(path.path_entry_hash()?, LinkTypes::PathToMyEntry)?.build(),
    )?;

    let get_inputs: Vec<GetInput> = links
        .into_iter()
        .filter_map(|link| link.target.into_action_hash())
        .map(|hash| GetInput::new(hash.into(), GetOptions::default()))
        .collect();

    let records = HDK.with(|hdk| hdk.borrow().get(get_inputs))?;
    Ok(records.into_iter().flatten().collect())
}
```

---

## Update Pattern

```rust
pub fn update_my_entry(
    original_action_hash: ActionHash,
    previous_action_hash: ActionHash,
    updated_entry: MyEntry,
) -> ExternResult<Record> {
    // 1. Author check
    let original_record = get(original_action_hash.clone(), GetOptions::default())?
        .ok_or(wasm_error!(WasmErrorInner::Guest("Entry not found".into())))?;
    let action = original_record.action();
    let agent = agent_info()?.agent_initial_pubkey;
    if action.author() != &agent {
        return Err(wasm_error!(WasmErrorInner::Guest("Not authorized".into())));
    }

    // 2. Update entry
    let updated_action_hash = update_entry(previous_action_hash, &EntryTypes::MyEntry(updated_entry))?;

    // 3. Track update chain with link
    create_link(
        original_action_hash,
        updated_action_hash.clone(),
        LinkTypes::MyEntryUpdates,
        (),
    )?;

    let record = get(updated_action_hash, GetOptions::default())?
        .ok_or(wasm_error!(WasmErrorInner::Guest("Updated record not found".into())))?;
    Ok(record)
}
```

---

## Delete Pattern

```rust
pub fn delete_my_entry(original_action_hash: ActionHash) -> ExternResult<ActionHash> {
    let path = Path::from("entries.active");
    let path_links = get_links(
        GetLinksInputBuilder::try_new(path.path_entry_hash()?, LinkTypes::PathToMyEntry)?.build(),
    )?;
    for link in path_links {
        if let Some(hash) = link.target.into_action_hash() {
            if hash == original_action_hash {
                delete_link(link.create_link_hash)?;
            }
        }
    }
    delete_entry(original_action_hash)
}
```

---

## Status Transition (Soft Delete)

Prefer updating status over deleting for data that other agents may reference:

```rust
pub fn archive_my_entry(original_action_hash: ActionHash, previous_action_hash: ActionHash)
    -> ExternResult<Record> {
    let mut record = get_latest_my_entry(original_action_hash.clone())?
        .ok_or(wasm_error!(WasmErrorInner::Guest("Entry not found".into())))?;

    let mut entry: MyEntry = record.entry().to_app_option()?.ok_or(
        wasm_error!(WasmErrorInner::Guest("Expected MyEntry".into()))
    )?;

    if entry.status == MyEntryStatus::Deleted {
        return Err(wasm_error!(WasmErrorInner::Guest("Cannot archive deleted entry".into())));
    }

    entry.status = MyEntryStatus::Archived;
    update_my_entry(original_action_hash, previous_action_hash, entry)
}
```

---

## Cross-Zome Calls

```rust
// In utils/src/cross_zome.rs
pub fn external_local_call<I, T>(zome_name: &str, fn_name: &str, input: I) -> ExternResult<T>
where
    I: serde::Serialize + std::fmt::Debug,
    T: serde::de::DeserializeOwned + std::fmt::Debug,
{
    let zome_call_response = call(
        CallTargetCell::Local,
        zome_name.into(),
        fn_name.into(),
        None,
        input,
    )?;

    match zome_call_response {
        ZomeCallResponse::Ok(result) => {
            let typed: T = result.decode().map_err(|e| {
                wasm_error!(WasmErrorInner::Guest(format!("Decode error: {:?}", e)))
            })?;
            Ok(typed)
        }
        ZomeCallResponse::Error(e) => {
            Err(wasm_error!(WasmErrorInner::Guest(format!("Zome call error: {:?}", e))))
        }
        _ => Err(wasm_error!(WasmErrorInner::Guest("Unexpected call response".into()))),
    }
}

// Usage:
let result: MyOtherEntry = external_local_call("other_zome", "get_entry", hash)?;
```

---

## Signals (post_commit)

```rust
#[derive(Serialize, Deserialize, Debug)]
#[serde(tag = "type")]
pub enum Signal {
    LinkCreated { action: SignedActionHashed, link_type: LinkTypes },
    LinkDeleted { action: SignedActionHashed, link_type: LinkTypes },
    EntryCreated { action: SignedActionHashed, app_entry: EntryTypes },
    EntryUpdated { action: SignedActionHashed, app_entry: EntryTypes, original_app_entry: EntryTypes },
    EntryDeleted { action: SignedActionHashed, original_app_entry: EntryTypes },
}

// NOTE: post_commit is infallible — use #[hdk_extern(infallible)] and log errors
#[hdk_extern(infallible)]
pub fn post_commit(committed_actions: Vec<SignedActionHashed>) {
    for action in committed_actions {
        if let Err(err) = signal_action(action) {
            error!("Error signaling new action: {:?}", err);
        }
    }
}
```

**Remote signals** — send signals to other agents:

```rust
// Sender:
send_remote_signal(recipient_pubkey, SerializedBytes::try_from(MySignal::Ping)?)?;

// Receiver callback:
#[hdk_extern]
pub fn recv_remote_signal(signal: SerializedBytes) -> ExternResult<()> {
    let sig: MySignal = signal.try_into()?;
    emit_signal(sig)?;
    Ok(())
}

// REQUIRED: cap grant in init() so any agent can call recv_remote_signal:
#[hdk_extern]
pub fn init(_: ()) -> ExternResult<InitCallbackResult> {
    let mut functions = HashSet::new();
    functions.insert((zome_info()?.name, "recv_remote_signal".into()));
    create_cap_grant(ZomeCallCapGrant {
        tag: "remote_signals".into(),
        access: CapAccess::Unrestricted,
        functions: GrantedFunctions::Listed(functions),
    })?;
    Ok(InitCallbackResult::Pass)
}
```

Note: `send_remote_signal` is fire-and-forget — it does not wait for confirmation and does not queue messages for offline agents.

---

## HDK 0.6 API Changes (Breaking)

### `delete_link()` — now requires GetOptions

```rust
// WRONG (pre-0.6):
delete_link(link.create_link_hash)?;

// CORRECT (0.6+):
delete_link(link.create_link_hash, GetOptions::default())?;
```

### `LinkQuery::new()` + `GetStrategy`

```rust
let links = get_links(
    LinkQuery::new(original_action_hash.clone(), LinkTypes::MyEntryUpdates),
    GetStrategy::Local,
)?;
```

**`GetStrategy` decision rule:**

| Strategy | When to use |
|----------|-------------|
| `GetStrategy::Local` | Source chain only — use for `get_my_*` (own authored data, fast, no network) |
| `GetStrategy::Network` | DHT — use for `get_all_*` (data authored by others, default behavior) |

**Additional LinkQuery features:**

```rust
// Tag prefix filter:
let query = LinkQuery::new(base, LinkTypes::MyLink)
    .tag_prefix(tag_bytes);

// Count without fetching records:
let count = count_links(query)?;

// Include deleted links:
let details = get_links_details(query)?;
```

### `HDK.with()` Batch Gets

More efficient than N individual `get()` calls:

```rust
let get_inputs: Vec<GetInput> = links
    .into_iter()
    .filter_map(|link| link.target.into_action_hash())
    .map(|hash| GetInput::new(hash.into(), GetOptions::default()))
    .collect();
let records = HDK.with(|hdk| hdk.borrow().get(get_inputs))?;
let records: Vec<Record> = records.into_iter().flatten().collect();
```

---

## `must_get_*` Family (Fail-Fast Gets)

Unlike `get()` which returns `Option`, these return an error immediately if the record is not found.

```rust
// In coordinator — authorship check before update:
let original_record = must_get_valid_record(input.original_action_hash.clone().into())?;
let author = original_record.action().author().clone();

// In integrity validation — authorship check:
let original_action_record = must_get_action(original_action_hash.clone())?;
if action.action().author() != original_action_record.action().author() {
    return Ok(ValidateCallbackResult::Invalid(
        "Only the original author can update this entry.".to_string(),
    ));
}
```

Full family:
- `must_get_valid_record(action_hash)` — record that passed validation
- `must_get_action(action_hash)` — raw action (use in validation)
- `must_get_entry(entry_hash)` — entry content
- `must_get_agent_activity(agent, filter)` — agent's source chain slice

---

## Validation (Integrity Crate)

```rust
// CORRECT: use op.flattened() — NOT the old op.to_type()
#[hdk_extern]
pub fn validate(op: Op) -> ExternResult<ValidateCallbackResult> {
    match op.flattened::<EntryTypes, LinkTypes>()? {
        FlatOp::StoreEntry(store_entry) => match store_entry {
            OpEntry::CreateEntry { app_entry, .. } => match app_entry {
                EntryTypes::MyEntry(entry) => validate_create_my_entry(entry),
                EntryTypes::AnotherEntry(entry) => validate_create_another_entry(entry),
            },
            OpEntry::UpdateEntry { app_entry, .. } => match app_entry {
                EntryTypes::MyEntry(entry) => validate_update_my_entry(entry),
                _ => Ok(ValidateCallbackResult::Valid),
            },
            _ => Ok(ValidateCallbackResult::Valid),
        },
        _ => Ok(ValidateCallbackResult::Valid),
    }
}

fn validate_create_my_entry(entry: MyEntry) -> ExternResult<ValidateCallbackResult> {
    if entry.title.is_empty() {
        return Ok(ValidateCallbackResult::Invalid("Title cannot be empty".into()));
    }
    Ok(ValidateCallbackResult::Valid)
}
```

**Determinism rules for validation:**
- No `get()`, `get_links()`, or any DHT reads
- No `agent_info()` (can vary by context)
- No `sys_time()` comparisons against current time
- Only inspect the op itself and its embedded data

---

## Path Anchors

```rust
// Global discovery anchor
let path = Path::from("entries.active");
let path_hash = path.path_entry_hash()?;

// Hierarchical paths
let category_path = Path::from(format!("entries.{}.active", category));

// Ensure path exists (creates the path entry if not present)
path.ensure()?;
```

---

## `get_details()` + `Details::Record` Deserialization

```rust
pub fn get_original_record(hash: ActionHash) -> ExternResult<Option<Record>> {
    let Some(details) = get_details(hash, GetOptions::default())? else {
        return Ok(None);
    };
    match details {
        Details::Record(d) => Ok(Some(d.record)),
        _ => Err(wasm_error!(WasmErrorInner::Guest("Expected record".into()))),
    }
}
```

**In `post_commit` — extracting app entry type from a committed action:**

```rust
let (zome_index, entry_index) = match record.action().entry_type() {
    Some(EntryType::App(AppEntryDef { zome_index, entry_index, .. })) => (zome_index, entry_index),
    _ => return Ok(None),
};
EntryTypes::deserialize_from_type(*zome_index, *entry_index, entry)
```

---

## Update Chain Utilities

### `find_original_action_hash()` — traverse backward to the Create action

Given any action hash in an update chain, loop back to the original Create:

```rust
pub fn find_original_action_hash(action_hash: ActionHash) -> ExternResult<OriginalActionHash> {
    let mut current_hash = action_hash;
    loop {
        let record = get(current_hash.clone(), GetOptions::default())?
            .ok_or(wasm_error!(WasmErrorInner::Guest("Record not found".into())))?;
        match record.action().clone() {
            Action::Create(_) => return Ok(OriginalActionHash(current_hash)),
            Action::Update(u) => { current_hash = u.original_action_address; }
            _ => return Err(wasm_error!(WasmErrorInner::Guest("Unexpected action type".into()))),
        }
    }
}
```

### `get_all_revisions_for_entry()` — original + all updates chronologically

Use `LinkQuery::new()` + `GetStrategy::Local` over the `{Entry}Updates` link type, prepend the original record. Returns all versions in order from oldest to newest.

---

## Path Status Hierarchies

For status-filtered global collections, use hierarchical path strings rather than a single path + runtime filtering:

```rust
const PENDING_PATH: &str = "entries.status.pending";
const APPROVED_PATH: &str = "entries.status.approved";
const REJECTED_PATH: &str = "entries.status.rejected";

// On creation — add link to pending path:
let pending_hash = Path::from(PENDING_PATH).path_entry_hash()?;
create_link(pending_hash, entry_hash.clone(), LinkTypes::AllEntries, ())?;

// On approval — move from pending to approved:
let approved_hash = Path::from(APPROVED_PATH).path_entry_hash()?;
create_link(approved_hash, entry_hash, LinkTypes::AllEntries, ())?;
// (delete the pending link separately)
```

Enables `get_links` filtered by status without fetching all entries — queries only the relevant path.

---

## Type-Safe Hash Wrappers

Prevent passing wrong hash type to functions:

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OriginalActionHash(pub ActionHash);

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PreviousActionHash(pub ActionHash);

// Function signature is self-documenting and compile-time safe
pub fn update_my_entry(
    original: OriginalActionHash,
    previous: PreviousActionHash,
    entry: MyEntry,
) -> ExternResult<Record> { ... }
```
