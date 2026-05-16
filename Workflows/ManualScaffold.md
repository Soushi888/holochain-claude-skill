# Workflow: Manual hApp Scaffold (Without hc scaffold CLI)

Use this workflow when `hc scaffold` is not running — for example when Claude is creating project files directly in an AI coding session. The output is **identical** to what `hc scaffold happ` + `hc scaffold entry-type` would generate. Whether the user runs the CLI or Claude writes the files, the resulting project structure is the same standard hc scaffold architecture.

**Before starting:** gather these values from the user:

| Placeholder | Case | Example |
|-------------|------|---------|
| `<APP_NAME>` | kebab-case | `my-community-app` |
| `<DNA_NAME>` | snake_case | `community` |
| `<ZOME_NAME>` | snake_case | `posts` |
| `<EntryType>` | PascalCase | `Post` |
| `<entry_type>` | snake_case | `post` |

Substitute every placeholder with the user's actual values throughout all files below.

---

## Step 1 — Create Directory Tree

```bash
mkdir -p <APP_NAME>/{workdir,dnas/<DNA_NAME>/{workdir,zomes/{integrity/<ZOME_NAME>_integrity/src,coordinator/<ZOME_NAME>/src}},tests/src/<DNA_NAME>/<ZOME_NAME>}
cd <APP_NAME>
```

---

## Step 2 — Root Files

### `flake.nix`

```nix
{
  description = "Flake for Holochain app development";

  inputs = {
    holonix.url = "github:holochain/holonix?ref=main-0.6";
    nixpkgs.follows = "holonix/nixpkgs";
    flake-parts.follows = "holonix/flake-parts";
  };

  outputs = inputs@{ flake-parts, ... }: flake-parts.lib.mkFlake { inherit inputs; } {
    systems = builtins.attrNames inputs.holonix.devShells;
    perSystem = { inputs', pkgs, ... }: {
      formatter = pkgs.nixpkgs-fmt;
      devShells.default = pkgs.mkShell {
        inputsFrom = [ inputs'.holonix.devShells.default ];
        packages = (with pkgs; [
          nodejs_22
          binaryen
          bun
        ]);
        shellHook = ''
          export PS1='\[\033[1;34m\][holonix:\w]\$\[\033[0m\] '
        '';
      };
    };
  };
}
```

### `Cargo.toml`

```toml
[profile.dev]
opt-level = "z"

[profile.release]
opt-level = "z"

[workspace]
members = ["dnas/*/zomes/coordinator/*", "dnas/*/zomes/integrity/*"]
resolver = "2"

[workspace.dependencies]
hdi = "=0.7.1"
hdk = "=0.6.1"
holochain_serialized_bytes = "*"
serde = "1.0"

[workspace.dependencies.<ZOME_NAME>]
path = "dnas/<DNA_NAME>/zomes/coordinator/<ZOME_NAME>"

[workspace.dependencies.<ZOME_NAME>_integrity]
path = "dnas/<DNA_NAME>/zomes/integrity/<ZOME_NAME>_integrity"
```

### `package.json`

```json
{
  "name": "<APP_NAME>-dev",
  "private": true,
  "workspaces": ["tests"],
  "scripts": {
    "test": "bun run build:zomes && hc app pack workdir --recursive && bun run --filter tests test",
    "build:happ": "bun run build:zomes && hc app pack workdir --recursive",
    "build:zomes": "RUSTFLAGS='--cfg getrandom_backend=\"custom\"' cargo build --release --target wasm32-unknown-unknown"
  },
  "devDependencies": {
    "concurrently": "^6.5.1"
  },
  "engines": {
    "node": ">=16.0.0"
  }
}
```

### `.gitignore`

```
# editors
/.idea
/.vscode

# system files
.DS_Store

# build
/dist/
/target/
/.cargo/

# package manager
/**/node_modules/

# generated and compiled files
*.happ
*.webhapp
*.zip
*.dna

# temporary files
.hc*
.running
.hc
```

---

## Step 3 — hApp Manifest

### `workdir/happ.yaml`

```yaml
manifest_version: '0'
name: <APP_NAME>
description: null
roles:
- name: <DNA_NAME>
  provisioning:
    strategy: create
    deferred: false
  dna:
    path: ../dnas/<DNA_NAME>/workdir/<DNA_NAME>.dna
    modifiers:
      network_seed: null
      properties: null
    installed_hash: null
    clone_limit: 0
allow_deferred_memproofs: false
```

---

## Step 4 — DNA Manifest

### `dnas/<DNA_NAME>/workdir/dna.yaml`

```yaml
manifest_version: '0'
name: <DNA_NAME>
integrity:
  network_seed: null
  properties: null
  zomes:
  - name: <ZOME_NAME>_integrity
    hash: null
    path: ../../../target/wasm32-unknown-unknown/release/<ZOME_NAME>_integrity.wasm
    dependencies: null
coordinator:
  zomes:
  - name: <ZOME_NAME>
    hash: null
    path: ../../../target/wasm32-unknown-unknown/release/<ZOME_NAME>.wasm
    dependencies:
    - name: <ZOME_NAME>_integrity
```

---

## Step 5 — Integrity Zome

### `dnas/<DNA_NAME>/zomes/integrity/<ZOME_NAME>_integrity/Cargo.toml`

```toml
[package]
name = "<ZOME_NAME>_integrity"
version = "0.0.1"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]
name = "<ZOME_NAME>_integrity"

[dependencies]
hdi = { workspace = true }
serde = { workspace = true }
holochain_serialized_bytes = { workspace = true }
```

### `dnas/<DNA_NAME>/zomes/integrity/<ZOME_NAME>_integrity/src/<entry_type>.rs`

This file holds the entry struct and per-operation validation functions.

```rust
use hdi::prelude::*;

#[derive(Clone, PartialEq)]
#[hdk_entry_helper]
pub struct <EntryType> {
    pub field1: String,
    // Add domain fields here.
    // Never add author or created_at — those live in the action header.
}

pub fn validate_create_<entry_type>(
    _action: EntryCreationAction,
    _<entry_type>: <EntryType>,
) -> ExternResult<ValidateCallbackResult> {
    // TODO: add the appropriate validation rules
    Ok(ValidateCallbackResult::Valid)
}

pub fn validate_update_<entry_type>(
    _action: Update,
    _<entry_type>: <EntryType>,
    _original_action: EntryCreationAction,
    _original_<entry_type>: <EntryType>,
) -> ExternResult<ValidateCallbackResult> {
    // TODO: add the appropriate validation rules
    Ok(ValidateCallbackResult::Valid)
}

pub fn validate_delete_<entry_type>(
    _action: Delete,
    _original_action: EntryCreationAction,
    _original_<entry_type>: <EntryType>,
) -> ExternResult<ValidateCallbackResult> {
    // TODO: add the appropriate validation rules
    Ok(ValidateCallbackResult::Valid)
}

pub fn validate_create_link_<entry_type>_updates(
    _action: CreateLink,
    base_address: AnyLinkableHash,
    target_address: AnyLinkableHash,
    _tag: LinkTag,
) -> ExternResult<ValidateCallbackResult> {
    let action_hash = base_address
        .into_action_hash()
        .ok_or(wasm_error!(WasmErrorInner::Guest(
            "No action hash associated with link".to_string()
        )))?;
    let record = must_get_valid_record(action_hash)?;
    let _<entry_type>: crate::<EntryType> = record
        .entry()
        .to_app_option()
        .map_err(|e| wasm_error!(e))?
        .ok_or(wasm_error!(WasmErrorInner::Guest(
            "Linked action must reference an entry".to_string()
        )))?;
    let action_hash = target_address
        .into_action_hash()
        .ok_or(wasm_error!(WasmErrorInner::Guest(
            "No action hash associated with link".to_string()
        )))?;
    let record = must_get_valid_record(action_hash)?;
    let _<entry_type>: crate::<EntryType> = record
        .entry()
        .to_app_option()
        .map_err(|e| wasm_error!(e))?
        .ok_or(wasm_error!(WasmErrorInner::Guest(
            "Linked action must reference an entry".to_string()
        )))?;
    // TODO: add the appropriate validation rules
    Ok(ValidateCallbackResult::Valid)
}

pub fn validate_delete_link_<entry_type>_updates(
    _action: DeleteLink,
    _original_action: CreateLink,
    _base: AnyLinkableHash,
    _target: AnyLinkableHash,
    _tag: LinkTag,
) -> ExternResult<ValidateCallbackResult> {
    Ok(ValidateCallbackResult::Invalid(
        "<EntryType>Updates links cannot be deleted".to_string(),
    ))
}
```

### `dnas/<DNA_NAME>/zomes/integrity/<ZOME_NAME>_integrity/src/lib.rs`

This file registers entry types, link types, and routes the `validate()` dispatch. For each additional entry type, add a module, an enum variant, and cases in the match arms.

```rust
pub mod <entry_type>;
use hdi::prelude::*;
pub use <entry_type>::*;

#[derive(Serialize, Deserialize)]
#[serde(tag = "type")]
#[hdk_entry_types]
#[unit_enum(UnitEntryTypes)]
pub enum EntryTypes {
    <EntryType>(<EntryType>),
}

#[derive(Serialize, Deserialize)]
#[hdk_link_types]
pub enum LinkTypes {
    <EntryType>Updates,
}

#[hdk_extern]
pub fn genesis_self_check(_data: GenesisSelfCheckData) -> ExternResult<ValidateCallbackResult> {
    Ok(ValidateCallbackResult::Valid)
}

pub fn validate_agent_joining(
    _agent_pub_key: AgentPubKey,
    _membrane_proof: &Option<MembraneProof>,
) -> ExternResult<ValidateCallbackResult> {
    Ok(ValidateCallbackResult::Valid)
}

#[hdk_extern]
pub fn validate(op: Op) -> ExternResult<ValidateCallbackResult> {
    match op.flattened::<EntryTypes, LinkTypes>()? {
        FlatOp::StoreEntry(store_entry) => match store_entry {
            OpEntry::CreateEntry { app_entry, action } => match app_entry {
                EntryTypes::<EntryType>(<entry_type>) => {
                    validate_create_<entry_type>(EntryCreationAction::Create(action), <entry_type>)
                }
            },
            OpEntry::UpdateEntry { app_entry, action, .. } => match app_entry {
                EntryTypes::<EntryType>(<entry_type>) => {
                    validate_create_<entry_type>(EntryCreationAction::Update(action), <entry_type>)
                }
            },
            _ => Ok(ValidateCallbackResult::Valid),
        },
        FlatOp::RegisterUpdate(update_entry) => match update_entry {
            OpUpdate::Entry { app_entry, action } => {
                let original_action = must_get_action(action.clone().original_action_address)?
                    .action()
                    .to_owned();
                let original_create_action = match EntryCreationAction::try_from(original_action) {
                    Ok(action) => action,
                    Err(e) => {
                        return Ok(ValidateCallbackResult::Invalid(format!(
                            "Expected to get EntryCreationAction from Action: {e:?}"
                        )));
                    }
                };
                match app_entry {
                    EntryTypes::<EntryType>(<entry_type>) => {
                        let original_app_entry =
                            must_get_valid_record(action.clone().original_action_address)?;
                        let original_<entry_type> = match <EntryType>::try_from(original_app_entry) {
                            Ok(entry) => entry,
                            Err(e) => {
                                return Ok(ValidateCallbackResult::Invalid(format!(
                                    "Expected to get <EntryType> from Record: {e:?}"
                                )));
                            }
                        };
                        validate_update_<entry_type>(
                            action,
                            <entry_type>,
                            original_create_action,
                            original_<entry_type>,
                        )
                    }
                }
            }
            _ => Ok(ValidateCallbackResult::Valid),
        },
        FlatOp::RegisterDelete(delete_entry) => {
            let original_action_hash = delete_entry.clone().action.deletes_address;
            let original_record = must_get_valid_record(original_action_hash)?;
            let original_record_action = original_record.action().clone();
            let original_action = match EntryCreationAction::try_from(original_record_action) {
                Ok(action) => action,
                Err(e) => {
                    return Ok(ValidateCallbackResult::Invalid(format!(
                        "Expected to get EntryCreationAction from Action: {e:?}"
                    )));
                }
            };
            let app_entry_type = match original_action.entry_type() {
                EntryType::App(app_entry_type) => app_entry_type,
                _ => return Ok(ValidateCallbackResult::Valid),
            };
            let entry = match original_record.entry().as_option() {
                Some(entry) => entry,
                None => {
                    return Ok(ValidateCallbackResult::Invalid(
                        "Original record for a delete must contain an entry".to_string(),
                    ));
                }
            };
            let original_app_entry = match EntryTypes::deserialize_from_type(
                app_entry_type.zome_index,
                app_entry_type.entry_index,
                entry,
            )? {
                Some(app_entry) => app_entry,
                None => {
                    return Ok(ValidateCallbackResult::Invalid(
                        "Original app entry must be one of the defined entry types for this zome"
                            .to_string(),
                    ));
                }
            };
            match original_app_entry {
                EntryTypes::<EntryType>(original_<entry_type>) => validate_delete_<entry_type>(
                    delete_entry.clone().action,
                    original_action,
                    original_<entry_type>,
                ),
            }
        }
        FlatOp::RegisterCreateLink {
            link_type,
            base_address,
            target_address,
            tag,
            action,
        } => match link_type {
            LinkTypes::<EntryType>Updates => {
                validate_create_link_<entry_type>_updates(action, base_address, target_address, tag)
            }
        },
        FlatOp::RegisterDeleteLink {
            link_type,
            base_address,
            target_address,
            tag,
            original_action,
            action,
        } => match link_type {
            LinkTypes::<EntryType>Updates => validate_delete_link_<entry_type>_updates(
                action,
                original_action,
                base_address,
                target_address,
                tag,
            ),
        },
        FlatOp::StoreRecord(store_record) => match store_record {
            OpRecord::CreateEntry { app_entry, action } => match app_entry {
                EntryTypes::<EntryType>(<entry_type>) => {
                    validate_create_<entry_type>(EntryCreationAction::Create(action), <entry_type>)
                }
            },
            OpRecord::UpdateEntry { original_action_hash, app_entry, action, .. } => {
                let original_record = must_get_valid_record(original_action_hash)?;
                let original_action = match original_record.action().clone() {
                    Action::Create(create) => EntryCreationAction::Create(create),
                    Action::Update(update) => EntryCreationAction::Update(update),
                    _ => {
                        return Ok(ValidateCallbackResult::Invalid(
                            "Original action for an update must be a Create or Update action"
                                .to_string(),
                        ));
                    }
                };
                match app_entry {
                    EntryTypes::<EntryType>(<entry_type>) => {
                        let result = validate_create_<entry_type>(
                            EntryCreationAction::Update(action.clone()),
                            <entry_type>.clone(),
                        )?;
                        if let ValidateCallbackResult::Valid = result {
                            let original_<entry_type>: Option<<EntryType>> = original_record
                                .entry()
                                .to_app_option()
                                .map_err(|e| wasm_error!(e))?;
                            let original_<entry_type> = match original_<entry_type> {
                                Some(e) => e,
                                None => {
                                    return Ok(ValidateCallbackResult::Invalid(
                                        "The updated entry type must be the same as the original entry type"
                                            .to_string(),
                                    ));
                                }
                            };
                            validate_update_<entry_type>(
                                action,
                                <entry_type>,
                                original_action,
                                original_<entry_type>,
                            )
                        } else {
                            Ok(result)
                        }
                    }
                }
            }
            OpRecord::DeleteEntry { original_action_hash, action, .. } => {
                let original_record = must_get_valid_record(original_action_hash)?;
                let original_action = match original_record.action().clone() {
                    Action::Create(create) => EntryCreationAction::Create(create),
                    Action::Update(update) => EntryCreationAction::Update(update),
                    _ => {
                        return Ok(ValidateCallbackResult::Invalid(
                            "Original action for a delete must be a Create or Update action"
                                .to_string(),
                        ));
                    }
                };
                let app_entry_type = match original_action.entry_type() {
                    EntryType::App(app_entry_type) => app_entry_type,
                    _ => return Ok(ValidateCallbackResult::Valid),
                };
                let entry = match original_record.entry().as_option() {
                    Some(entry) => entry,
                    None => {
                        return Ok(ValidateCallbackResult::Invalid(
                            "Original record for a delete must contain an entry".to_string(),
                        ));
                    }
                };
                let original_app_entry = match EntryTypes::deserialize_from_type(
                    app_entry_type.zome_index,
                    app_entry_type.entry_index,
                    entry,
                )? {
                    Some(app_entry) => app_entry,
                    None => {
                        return Ok(ValidateCallbackResult::Invalid(
                            "Original app entry must be one of the defined entry types for this zome"
                                .to_string(),
                        ));
                    }
                };
                match original_app_entry {
                    EntryTypes::<EntryType>(original_<entry_type>) => {
                        validate_delete_<entry_type>(action, original_action, original_<entry_type>)
                    }
                }
            }
            OpRecord::CreateLink { base_address, target_address, tag, link_type, action } => {
                match link_type {
                    LinkTypes::<EntryType>Updates => validate_create_link_<entry_type>_updates(
                        action,
                        base_address,
                        target_address,
                        tag,
                    ),
                }
            }
            OpRecord::DeleteLink { original_action_hash, base_address, action } => {
                let record = must_get_valid_record(original_action_hash)?;
                let create_link = match record.action() {
                    Action::CreateLink(create_link) => create_link.clone(),
                    _ => {
                        return Ok(ValidateCallbackResult::Invalid(
                            "The action that a DeleteLink deletes must be a CreateLink".to_string(),
                        ));
                    }
                };
                let link_type =
                    match LinkTypes::from_type(create_link.zome_index, create_link.link_type)? {
                        Some(lt) => lt,
                        None => return Ok(ValidateCallbackResult::Valid),
                    };
                match link_type {
                    LinkTypes::<EntryType>Updates => validate_delete_link_<entry_type>_updates(
                        action,
                        create_link.clone(),
                        base_address,
                        create_link.target_address,
                        create_link.tag,
                    ),
                }
            }
            OpRecord::CreatePrivateEntry { .. } => Ok(ValidateCallbackResult::Valid),
            OpRecord::UpdatePrivateEntry { .. } => Ok(ValidateCallbackResult::Valid),
            OpRecord::CreateCapClaim { .. } => Ok(ValidateCallbackResult::Valid),
            OpRecord::CreateCapGrant { .. } => Ok(ValidateCallbackResult::Valid),
            OpRecord::UpdateCapClaim { .. } => Ok(ValidateCallbackResult::Valid),
            OpRecord::UpdateCapGrant { .. } => Ok(ValidateCallbackResult::Valid),
            OpRecord::Dna { .. } => Ok(ValidateCallbackResult::Valid),
            OpRecord::OpenChain { .. } => Ok(ValidateCallbackResult::Valid),
            OpRecord::CloseChain { .. } => Ok(ValidateCallbackResult::Valid),
            OpRecord::InitZomesComplete { .. } => Ok(ValidateCallbackResult::Valid),
            _ => Ok(ValidateCallbackResult::Valid),
        },
        FlatOp::RegisterAgentActivity(agent_activity) => match agent_activity {
            OpActivity::CreateAgent { agent, action } => {
                let previous_action = must_get_action(action.prev_action)?;
                match previous_action.action() {
                    Action::AgentValidationPkg(AgentValidationPkg { membrane_proof, .. }) => {
                        validate_agent_joining(agent, membrane_proof)
                    }
                    _ => Ok(ValidateCallbackResult::Invalid(
                        "The previous action for a `CreateAgent` action must be an `AgentValidationPkg`"
                            .to_string(),
                    )),
                }
            }
            _ => Ok(ValidateCallbackResult::Valid),
        },
    }
}
```

---

## Step 6 — Coordinator Zome

### `dnas/<DNA_NAME>/zomes/coordinator/<ZOME_NAME>/Cargo.toml`

```toml
[package]
name = "<ZOME_NAME>"
version = "0.0.1"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]
name = "<ZOME_NAME>"

[dependencies]
hdk = { workspace = true }
serde = { workspace = true }
holochain_serialized_bytes = { workspace = true }
<ZOME_NAME>_integrity = { workspace = true }
```

### `dnas/<DNA_NAME>/zomes/coordinator/<ZOME_NAME>/src/lib.rs`

This file provides `init`, the `Signal` enum (required for UI reactivity), and `post_commit` / `signal_action`. Do not modify the `Signal` enum or the `for` loop in `post_commit` — the scaffold tool adds signals per entry type by extending these.

```rust
pub mod <entry_type>;
use hdk::prelude::*;
use <ZOME_NAME>_integrity::*;

#[hdk_extern]
pub fn init() -> ExternResult<InitCallbackResult> {
    Ok(InitCallbackResult::Pass)
}

#[derive(Serialize, Deserialize, Debug)]
#[serde(tag = "type")]
pub enum Signal {
    LinkCreated { action: SignedActionHashed, link_type: LinkTypes },
    LinkDeleted {
        action: SignedActionHashed,
        create_link_action: SignedActionHashed,
        link_type: LinkTypes,
    },
    EntryCreated { action: SignedActionHashed, app_entry: EntryTypes },
    EntryUpdated {
        action: SignedActionHashed,
        app_entry: EntryTypes,
        original_app_entry: EntryTypes,
    },
    EntryDeleted { action: SignedActionHashed, original_app_entry: EntryTypes },
}

#[hdk_extern(infallible)]
pub fn post_commit(committed_actions: Vec<SignedActionHashed>) {
    for action in committed_actions {
        if let Err(err) = signal_action(action) {
            error!("Error signaling new action: {:?}", err);
        }
    }
}

fn signal_action(action: SignedActionHashed) -> ExternResult<()> {
    match action.hashed.content.clone() {
        Action::CreateLink(create_link) => {
            if let Ok(Some(link_type)) =
                LinkTypes::from_type(create_link.zome_index, create_link.link_type)
            {
                emit_signal(Signal::LinkCreated { action, link_type })?;
            }
            Ok(())
        }
        Action::DeleteLink(delete_link) => {
            let record = get(delete_link.link_add_address.clone(), GetOptions::default())?.ok_or(
                wasm_error!(WasmErrorInner::Guest(
                    "Failed to fetch CreateLink action".to_string()
                )),
            )?;
            match record.action() {
                Action::CreateLink(create_link) => {
                    if let Ok(Some(link_type)) =
                        LinkTypes::from_type(create_link.zome_index, create_link.link_type)
                    {
                        emit_signal(Signal::LinkDeleted {
                            action,
                            link_type,
                            create_link_action: record.signed_action.clone(),
                        })?;
                    }
                    Ok(())
                }
                _ => Err(wasm_error!(WasmErrorInner::Guest(
                    "Create Link should exist".to_string()
                ))),
            }
        }
        Action::Create(_) => {
            if let Ok(Some(app_entry)) = get_entry_for_action(&action.hashed.hash) {
                emit_signal(Signal::EntryCreated { action, app_entry })?;
            }
            Ok(())
        }
        Action::Update(update) => {
            if let Ok(Some(app_entry)) = get_entry_for_action(&action.hashed.hash) {
                if let Ok(Some(original_app_entry)) =
                    get_entry_for_action(&update.original_action_address)
                {
                    emit_signal(Signal::EntryUpdated {
                        action,
                        app_entry,
                        original_app_entry,
                    })?;
                }
            }
            Ok(())
        }
        Action::Delete(delete) => {
            if let Ok(Some(original_app_entry)) = get_entry_for_action(&delete.deletes_address) {
                emit_signal(Signal::EntryDeleted { action, original_app_entry })?;
            }
            Ok(())
        }
        _ => Ok(()),
    }
}

fn get_entry_for_action(action_hash: &ActionHash) -> ExternResult<Option<EntryTypes>> {
    let record = match get_details(action_hash.clone(), GetOptions::default())? {
        Some(Details::Record(record_details)) => record_details.record,
        _ => return Ok(None),
    };
    let entry = match record.entry().as_option() {
        Some(entry) => entry,
        None => return Ok(None),
    };
    let (zome_index, entry_index) = match record.action().entry_type() {
        Some(EntryType::App(AppEntryDef { zome_index, entry_index, .. })) => {
            (zome_index, entry_index)
        }
        _ => return Ok(None),
    };
    EntryTypes::deserialize_from_type(*zome_index, *entry_index, entry)
}
```

### `dnas/<DNA_NAME>/zomes/coordinator/<ZOME_NAME>/src/<entry_type>.rs`

Full CRUD implementation including update-chain tracking via `<EntryType>Updates` links:

```rust
use hdk::prelude::*;
use <ZOME_NAME>_integrity::*;

#[hdk_extern]
pub fn create_<entry_type>(<entry_type>: <EntryType>) -> ExternResult<Record> {
    let <entry_type>_hash = create_entry(&EntryTypes::<EntryType>(<entry_type>.clone()))?;
    let record = get(<entry_type>_hash.clone(), GetOptions::default())?.ok_or(wasm_error!(
        WasmErrorInner::Guest("Could not find the newly created <EntryType>".to_string())
    ))?;
    Ok(record)
}

#[hdk_extern]
pub fn get_latest_<entry_type>(original_<entry_type>_hash: ActionHash) -> ExternResult<Option<Record>> {
    let links = get_links(
        LinkQuery::try_new(original_<entry_type>_hash.clone(), LinkTypes::<EntryType>Updates)?,
        GetStrategy::default(),
    )?;
    let latest_link = links
        .into_iter()
        .max_by(|link_a, link_b| link_a.timestamp.cmp(&link_b.timestamp));
    let latest_<entry_type>_hash = match latest_link {
        Some(link) => link
            .target
            .clone()
            .into_action_hash()
            .ok_or(wasm_error!(WasmErrorInner::Guest(
                "No action hash associated with link".to_string()
            )))?,
        None => original_<entry_type>_hash.clone(),
    };
    get(latest_<entry_type>_hash, GetOptions::default())
}

#[hdk_extern]
pub fn get_original_<entry_type>(original_<entry_type>_hash: ActionHash) -> ExternResult<Option<Record>> {
    let Some(details) = get_details(original_<entry_type>_hash, GetOptions::default())? else {
        return Ok(None);
    };
    match details {
        Details::Record(details) => Ok(Some(details.record)),
        _ => Err(wasm_error!(WasmErrorInner::Guest(
            "Malformed get details response".to_string()
        ))),
    }
}

#[hdk_extern]
pub fn get_all_revisions_for_<entry_type>(
    original_<entry_type>_hash: ActionHash,
) -> ExternResult<Vec<Record>> {
    let Some(original_record) = get_original_<entry_type>(original_<entry_type>_hash.clone())? else {
        return Ok(vec![]);
    };
    let links = get_links(
        LinkQuery::try_new(original_<entry_type>_hash.clone(), LinkTypes::<EntryType>Updates)?,
        GetStrategy::default(),
    )?;
    let get_input: Vec<GetInput> = links
        .into_iter()
        .map(|link| {
            Ok(GetInput::new(
                link.target
                    .into_action_hash()
                    .ok_or(wasm_error!(WasmErrorInner::Guest(
                        "No action hash associated with link".to_string()
                    )))?
                    .into(),
                GetOptions::default(),
            ))
        })
        .collect::<ExternResult<Vec<GetInput>>>()?;
    let records = HDK.with(|hdk| hdk.borrow().get(get_input))?;
    let mut records: Vec<Record> = records.into_iter().flatten().collect();
    records.insert(0, original_record);
    Ok(records)
}

#[derive(Serialize, Deserialize, Debug)]
pub struct Update<EntryType>Input {
    pub original_<entry_type>_hash: ActionHash,
    pub previous_<entry_type>_hash: ActionHash,
    pub updated_<entry_type>: <EntryType>,
}

#[hdk_extern]
pub fn update_<entry_type>(input: Update<EntryType>Input) -> ExternResult<Record> {
    let updated_<entry_type>_hash =
        update_entry(input.previous_<entry_type>_hash.clone(), &input.updated_<entry_type>)?;
    create_link(
        input.original_<entry_type>_hash.clone(),
        updated_<entry_type>_hash.clone(),
        LinkTypes::<EntryType>Updates,
        (),
    )?;
    let record =
        get(updated_<entry_type>_hash.clone(), GetOptions::default())?.ok_or(wasm_error!(
            WasmErrorInner::Guest("Could not find the newly updated <EntryType>".to_string())
        ))?;
    Ok(record)
}

#[hdk_extern]
pub fn delete_<entry_type>(original_<entry_type>_hash: ActionHash) -> ExternResult<ActionHash> {
    delete_entry(original_<entry_type>_hash)
}

#[hdk_extern]
pub fn get_all_deletes_for_<entry_type>(
    original_<entry_type>_hash: ActionHash,
) -> ExternResult<Option<Vec<SignedActionHashed>>> {
    let Some(details) = get_details(original_<entry_type>_hash, GetOptions::default())? else {
        return Ok(None);
    };
    match details {
        Details::Entry(_) => Err(wasm_error!(WasmErrorInner::Guest(
            "Malformed details".into()
        ))),
        Details::Record(record_details) => Ok(Some(record_details.deletes)),
    }
}

#[hdk_extern]
pub fn get_oldest_delete_for_<entry_type>(
    original_<entry_type>_hash: ActionHash,
) -> ExternResult<Option<SignedActionHashed>> {
    let Some(mut deletes) = get_all_deletes_for_<entry_type>(original_<entry_type>_hash)? else {
        return Ok(None);
    };
    deletes.sort_by(|delete_a, delete_b| {
        delete_a
            .action()
            .timestamp()
            .cmp(&delete_b.action().timestamp())
    });
    Ok(deletes.first().cloned())
}
```

---

## Step 7 — Tests (Sweettest — primary)

**Sweettest (Rust) is the primary testing layer.** See `Testing.md` for full two-agent patterns.

Create the test crate:

```bash
mkdir -p dnas/<DNA_NAME>/tests/src
```

`dnas/<DNA_NAME>/tests/Cargo.toml`:
```toml
[package]
name = "<DNA_NAME>_tests"
version = "0.1.0"
edition = "2021"

[dev-dependencies]
holochain = { version = "=0.6.1", features = ["test_utils"] }
tokio     = { version = "1", features = ["full"] }
```

Add the test crate explicitly to the root `Cargo.toml` members (the glob does not pick it up):
```toml
[workspace]
members = [
    "dnas/*/zomes/coordinator/*",
    "dnas/*/zomes/integrity/*",
    "dnas/<DNA_NAME>/tests",
]
```

---

> **Tryorama (TypeScript) — deprecated.** `hc scaffold happ` generates a TypeScript/Tryorama test scaffold under `tests/` as a convenience. The files are provided below for completeness but Tryorama is not the recommended path for new test work. Use Sweettest.

<details>
<summary>Tryorama scaffold files (for reference only)</summary>

### `tests/package.json`

```json
{
  "name": "tests",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "test": "vitest run"
  },
  "dependencies": {
    "@msgpack/msgpack": "^2.8.0",
    "@holochain/client": "^0.20.0",
    "@holochain/tryorama": "^0.19.0",
    "typescript": "^5.6.3",
    "vitest": "^3.1.3"
  },
  "type": "module"
}
```

### `tests/vitest.config.ts`

```typescript
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    testTimeout: 60 * 1000 * 4,
    poolOptions: { forks: { singleFork: true } },
  },
});
```

### `tests/tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2017",
    "module": "ESNext",
    "moduleResolution": "node",
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true
  }
}
```

### `tests/src/<DNA_NAME>/<ZOME_NAME>/common.ts`

```typescript
import { Record } from "@holochain/client";
import { CallableCell } from "@holochain/tryorama";

export async function sample<EntryType>(cell: CallableCell, partial<EntryType> = {}) {
  return {
    ...{
      field1: "Lorem ipsum dolor sit amet, consectetur adipiscing elit.",
    },
    ...partial<EntryType>,
  };
}

export async function create<EntryType>(
  cell: CallableCell,
  <entry_type> = undefined,
): Promise<Record> {
  return cell.callZome({
    zome_name: "<ZOME_NAME>",
    fn_name: "create_<entry_type>",
    payload: <entry_type> || (await sample<EntryType>(cell)),
  });
}
```

### `tests/src/<DNA_NAME>/<ZOME_NAME>/<entry_type>.test.ts`

```typescript
import { assert, test } from "vitest";
import { Record, AppBundleSource } from "@holochain/client";
import { dhtSync, runScenario } from "@holochain/tryorama";
import { decode } from "@msgpack/msgpack";
import { create<EntryType>, sample<EntryType> } from "./common.js";

const testAppPath = process.cwd() + "/../workdir/<APP_NAME>.happ";
const appSource = { appBundleSource: { type: "path", value: testAppPath } as AppBundleSource };

test("create <EntryType>", async () => {
  await runScenario(async scenario => {
    const [alice, bob] = await scenario.addPlayersWithApps([appSource, appSource]);
    await scenario.shareAllAgents();
    const record: Record = await create<EntryType>(alice.cells[0]);
    assert.ok(record);
  });
});

test("create and read <EntryType>", async () => {
  await runScenario(async scenario => {
    const [alice, bob] = await scenario.addPlayersWithApps([appSource, appSource]);
    await scenario.shareAllAgents();
    const sample = await sample<EntryType>(alice.cells[0]);
    const record: Record = await create<EntryType>(alice.cells[0], sample);
    assert.ok(record);
    await dhtSync([alice, bob], alice.cells[0].cell_id[0]);
    const readOutput: Record = await bob.cells[0].callZome({
      zome_name: "<ZOME_NAME>",
      fn_name: "get_original_<entry_type>",
      payload: record.signed_action.hashed.hash,
    });
    assert.deepEqual(sample, decode((readOutput.entry as any).Present.entry) as any);
  });
});
```

</details>

---

## Step 8 — Install Test Dependencies and Verify

```bash
# Inside nix develop (or after entering the dev shell):
cd tests && bun install && cd ..

# Verify zome compilation
RUSTFLAGS='--cfg getrandom_backend="custom"' cargo build --release --target wasm32-unknown-unknown

# Pack and run tests
hc app pack workdir --recursive
cd tests && bun run test
```

**First build is slow** (5-10 min for WASM + wasm-opt). Subsequent builds use the Rust cache.

---

## Adding More Entry Types

For each additional entry type, repeat Steps 5-6 entry-type files and:

1. Add `pub mod <new_entry_type>;` and `pub use <new_entry_type>::*;` to integrity `lib.rs`
2. Add `<NewEntryType>(<NewEntryType>)` to the `EntryTypes` enum
3. Add `<NewEntryType>Updates` to the `LinkTypes` enum
4. Add the entry type's match arms in the `validate()` function
5. Add `pub mod <new_entry_type>;` to coordinator `lib.rs`
6. Create the coordinator `<new_entry_type>.rs` with CRUD functions
7. Add a test file to `tests/src/<DNA_NAME>/<ZOME_NAME>/`

Proceed to `Workflows/ImplementZome.md` to fill in validation rules and domain logic.
