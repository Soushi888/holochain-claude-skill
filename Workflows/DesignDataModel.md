# Workflow: Design DHT Data Model

Use this workflow when designing the data model for a new domain or feature in a Holochain hApp. Produces: entry type definitions, link type definitions, discovery strategy, and validation rules ready for implementation.

---

## Step 1: Identify Domains and Zome Pairs

Map the business domain to Holochain's zome architecture:

```
For each distinct business domain:
  → 1 integrity crate:   {domain}_integrity
  → 1 coordinator crate: {domain}
```

**Questions to answer:**
- What are the distinct nouns in this feature? (e.g., Request, Offer, Person, Resource)
- Which nouns belong together conceptually? (e.g., all marketplace data in one zome pair)
- Which nouns need to be queried independently at scale? (separate zome pairs)

**Output:** List of zome pairs with their domain responsibilities.

---

## Step 2: Define Entry Types Per Domain

For each entry type, define:

```
Entry: {EntryName}
Fields:
  - field_name: type   (required)
  - field_name: type   (required)
  - status: StatusEnum (if soft-delete needed)
  - optional_field: type   #[serde(default)]  (if backward-compatible addition)

Visibility: Public | Private
  Public: stored on DHT, visible to all agents
  Private: stored locally only, not shared

State enum (if applicable):
  enum {Entry}Status { Active, Archived, Deleted }
```

**Decision criteria:**
- Is this data meaningful to other agents? → Public
- Is this personal/sensitive? → Private
- Does this entry transition through states? → Add `status` field with enum
- Can this entry be "updated in place" or should old versions be preserved? → Update chain (links) vs overwrite

---

## Step 3: Design Link Types

For every relationship between entries, define a directional link:

```
Link: {Base}To{Target}
  Base: {what you start from}
  Target: {what you navigate to}
  Tag: bytes | () | typed data for filtering

Required links per entry:
  ┌─ PathTo{Entry}           Discovery from global path anchor
  ├─ AgentTo{Entry}          Discovery from agent's pubkey
  └─ {Entry}Updates          Update chain tracking (for get-latest)

Optional:
  ├─ {Entry}To{Related}      Bidirectional relationship
  └─ {Related}To{Entry}      Reverse direction (add both)
```

**Bidirectional rule:** If you need to navigate A → B and B → A, create two link types. Never navigate backwards through a forward link.

**Update chain rule:** Every entry that supports `update` needs a `{Entry}Updates` link type that records the chain from `original_action_hash` → `updated_action_hash`.

---

## Step 4: Choose Discovery Strategy

How will agents find entries?

| Pattern | Link | Use When |
|---------|------|----------|
| Global path anchor | `Path::from("entries.active")` → Entry | All agents browse all entries |
| Status-scoped path | `Path::from("entries.active")` vs `"entries.archived"` | Browse by status |
| Agent-centric | `AgentPubKey` → Entry | Each agent manages their own entries |
| Both | Path + Agent links | Global browse AND per-agent listing |
| Hierarchical path | `Path::from("category.{id}.entries")` | Category/tag based grouping |

**Decision:** Almost always use Both (path + agent) unless the domain is strictly personal.

---

## Step 5: Write Validation Rules

For each entry type, define what makes it INVALID:

```
Validation rules for {EntryName}:
  Field constraints:
    - title: non-empty, max 200 chars
    - description: max 2000 chars
    - status: must be valid enum variant

  Business rules (that can be checked deterministically):
    - Cannot create entry with status = Deleted
    - Cannot have duplicate fields X and Y both empty
    - Tags: max 10 items, each max 50 chars

  FORBIDDEN in validation (causes non-determinism):
    - No DHT reads (get, get_links)
    - No agent_info()
    - No sys_time() comparisons to current time
    - No randomness
```

**Key rule:** Validation runs in integrity. It must be pure and deterministic — same input always produces same result, regardless of when or where it runs.

---

## Step 6: Review — Apply the Splitting Test

Before finalizing, run each design decision through the splitting test:

**Entry field review:**
- Is every field necessary? (Remove if unused by UI or other zomes)
- Are there fields that could be derived? (Remove if computable)
- Are there fields that change independently? (May belong in a separate entry)

**Link review:**
- Does every link have a clear query use case?
- Are bidirectional links actually needed in both directions?
- Are `{Entry}Updates` links present for every updatable entry?

**Validation review:**
- Is every validation rule actually deterministic?
- Are validation error messages user-readable?
- Are there business rules that need to be enforced elsewhere (coordinator) because they require DHT reads?

---

## Output Artifacts

After completing this workflow, you have:

1. **Zome pair list** — domain to crate name mapping
2. **Entry structs** (Rust) — ready to paste into integrity crate
3. **Link type enum** — ready to paste into integrity crate
4. **Discovery strategy** — path vs agent vs both, with path strings
5. **Validation checklist** — rules ready for `validate()` callback
6. **Summary table:**

```
| Entry | Links out | Update chain? | Discovery | Status enum? |
|-------|-----------|---------------|-----------|-------------|
| MyEntry | AgentToMyEntry, PathToMyEntry | Yes (MyEntryUpdates) | Path + Agent | Yes |
```

Proceed to `Workflows/ImplementZome.md` to implement.
