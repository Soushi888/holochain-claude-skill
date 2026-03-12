# Holochain Access Control

## Why Capability Grants Exist

Holochain zome functions are not open by default. When agent A wants to call a zome function on agent B's cell (a "remote call"), B's cell must have an explicit **capability grant** authorizing that call. Without a grant, the call is rejected.

This applies to:
- `call_remote()` — calling a zome function on another agent's cell
- `send_remote_signal` → `recv_remote_signal` — the receiver needs a grant so the signal handler can be invoked

Calls from the **same hApp's UI** (same agent, same cell) do not need grants.

---

## Three `CapAccess` Tiers

### 1. `CapAccess::Unrestricted` — Any agent may call

```rust
use std::collections::HashSet;

#[hdk_extern]
pub fn init(_: ()) -> ExternResult<InitCallbackResult> {
    let mut functions = HashSet::new();
    functions.insert((zome_info()?.name, "recv_remote_signal".into()));

    create_cap_grant(ZomeCallCapGrant {
        tag: "open_to_all".into(),
        access: CapAccess::Unrestricted,
        functions: GrantedFunctions::Listed(functions),
    })?;

    Ok(InitCallbackResult::Pass)
}
```

Use when: the function should be callable by any agent (e.g., `recv_remote_signal`).

### 2. `CapAccess::Transferable { secret }` — Any agent with the secret may call

```rust
let secret = generate_cap_secret()?;

create_cap_grant(ZomeCallCapGrant {
    tag: "transferable_grant".into(),
    access: CapAccess::Transferable { secret },
    functions: GrantedFunctions::Listed(functions),
})?;

// Share `secret` with the grantee out-of-band (e.g., via a private entry or direct message)
```

Use when: you want to delegate access to anyone who holds the secret — like a token.

### 3. `CapAccess::Assigned { secret, assignees }` — Only specific agents with the secret

```rust
let secret = generate_cap_secret()?;
let mut assignees = BTreeSet::new();
assignees.insert(grantee_pubkey.clone());

create_cap_grant(ZomeCallCapGrant {
    tag: "assigned_grant".into(),
    access: CapAccess::Assigned { secret, assignees },
    functions: GrantedFunctions::Listed(functions),
})?;
```

Use when: access is explicitly scoped to one or more named agents.

---

## Grant Lifecycle

```
Grantor side:                              Grantee side:
─────────────────────────────────          ──────────────────────────────────
1. generate_cap_secret()?              →   (receive secret out-of-band)
2. create_cap_grant(ZomeCallCapGrant   →   3. create_cap_claim(CapClaim {
      { tag, access, functions })              tag, grantor, secret,
                                            })?
                                       →   4. call_remote(
                                               grantor_pubkey,
                                               zome_name,
                                               fn_name,
                                               Some(secret),
                                               payload,
                                           )?
```

**Step-by-step:**
1. Grantor generates a secret: `let secret = generate_cap_secret()?;`
2. Grantor creates grant on their source chain (stored locally, not DHT)
3. Grantee receives the secret via private entry, signal, or other channel
4. Grantee stores it as a cap claim: `create_cap_claim(CapClaim { tag, grantor, secret })?;`
5. Grantee calls with secret: `call_remote(grantor, zome, fn_name, Some(secret), payload)?;`

---

## Decision Table

| Scenario | Pattern |
|----------|---------|
| `recv_remote_signal` open to all agents | `CapAccess::Unrestricted` in `init()` |
| Delegate a specific function to one agent | `CapAccess::Assigned` + share secret via private entry |
| UI calling own zome (same agent, same cell) | No grant needed |
| Admin-only zome function | Progenitor check in coordinator (see `Architecture.md` § DNA Properties) |
| Public API any agent can call | `CapAccess::Unrestricted` in `init()` for that function |

---

## Notes

- Cap grants are stored on the **grantor's source chain** — they are private, not shared to the DHT
- Cap claims are stored on the **grantee's source chain**
- Revoking a grant: use `delete_cap_grant(grant_action_hash)?;`
- `GrantedFunctions::All` grants access to ALL functions in the zome — use with extreme caution

**Reference:** [developer.holochain.org/build/capabilities/](https://developer.holochain.org/build/capabilities/)
