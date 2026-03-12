# Holochain Testing

## Framework Overview

- **Tryorama** (`@holochain/tryorama`) — Holochain test orchestration
- **Vitest** — Test runner (ESM-native, works with Bun)
- Tests run against real compiled `.happ` bundles, not mocks

---

## Setup (package.json)

```json
{
  "scripts": {
    "test": "vitest run",
    "test:foundation": "vitest run tests/foundation",
    "test:integration": "vitest run tests/integration"
  },
  "devDependencies": {
    "@holochain/tryorama": "^0.17.0",
    "vitest": "^1.0.0"
  }
}
```

## vitest.config.ts

```typescript
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    testTimeout: 60000,   // Holochain tests are slow — 60s minimum
    hookTimeout: 60000,
  },
});
```

---

## Two-Agent Scenario (the Standard Pattern)

Most Holochain tests require two agents to validate DHT propagation:

```typescript
import { runScenarioWithTwoAgents } from "@holochain/tryorama";
import { assert } from "vitest";
import { it } from "vitest";

it("agent B can read entry created by agent A", async () => {
  await runScenarioWithTwoAgents(async (t, alice, bob) => {
    // Get the cell for a specific DNA role
    const aliceCell = alice.namedCells.get("my_dna_role_name");
    const bobCell = bob.namedCells.get("my_dna_role_name");

    // Alice creates an entry
    const record = await aliceCell.callZome({
      zome_name: "my_zome",
      fn_name: "create_my_entry",
      payload: {
        title: "Test Entry",
        description: "Created by Alice",
        status: "Active",
      },
    });

    assert.ok(record);
    const originalHash = record.signed_action.hashed.hash;

    // MANDATORY: dhtSync before any cross-agent read
    await dhtSync([alice, bob], alice.cells[0].cell_id[0]);

    // Bob reads Alice's entry
    const fetched = await bobCell.callZome({
      zome_name: "my_zome",
      fn_name: "get_my_entry",
      payload: originalHash,
    });

    assert.ok(fetched);
    assert.equal(fetched.entry.Present.entry.title, "Test Entry");
  });
});
```

---

## CRITICAL: dhtSync is MANDATORY Before Cross-Agent Reads

**The #1 cause of flaky tests in Holochain: missing `dhtSync`.**

```typescript
import { dhtSync } from "@holochain/tryorama";

// After Alice creates/updates/deletes, BEFORE Bob reads:
await dhtSync([alice, bob], alice.cells[0].cell_id[0]);
```

**Why it's needed:** DHT propagation is asynchronous. Without `dhtSync`, Bob may try to `get()` an entry before the gossip has propagated to his node. The test passes 70% of the time (when fast) and fails 30% (when slow). `dhtSync` waits for full propagation.

**Rule:** Any test where Agent B reads data created by Agent A requires `dhtSync` between the write and the read.

---

## Update Test Pattern

The update pattern requires fetching the `previous_action_hash` from the latest record:

```typescript
it("update entry requires previous action hash", async () => {
  await runScenarioWithTwoAgents(async (t, alice, bob) => {
    const aliceCell = alice.namedCells.get("my_dna");

    // Create
    const created = await aliceCell.callZome({
      zome_name: "my_zome",
      fn_name: "create_my_entry",
      payload: { title: "Original", status: "Active" },
    });
    const originalHash = created.signed_action.hashed.hash;

    // SYNC before update (even same-agent, sometimes needed for path links)
    await dhtSync([alice, bob], alice.cells[0].cell_id[0]);

    // Fetch latest to get previous_action_hash
    const latestRecord = await aliceCell.callZome({
      zome_name: "my_zome",
      fn_name: "get_latest_my_entry",
      payload: originalHash,
    });
    const previousHash = latestRecord.signed_action.hashed.hash;

    // Update using BOTH original and previous hashes
    const updated = await aliceCell.callZome({
      zome_name: "my_zome",
      fn_name: "update_my_entry",
      payload: {
        original_action_hash: originalHash,
        previous_action_hash: previousHash,
        updated_entry: { title: "Updated", status: "Active" },
      },
    });

    assert.ok(updated);
    assert.equal(updated.entry.Present.entry.title, "Updated");
  });
});
```

---

## Sample Data Helpers

Use spread pattern for flexible test data:

```typescript
// tests/helpers.ts
export function sampleMyEntry(overrides: Partial<MyEntry> = {}): MyEntry {
  return {
    title: "Test Entry",
    description: "A test entry for automated testing",
    status: "Active",
    tags: [],
    ...overrides,  // Caller overrides specific fields
  };
}

// Usage in test:
const record = await aliceCell.callZome({
  zome_name: "my_zome",
  fn_name: "create_my_entry",
  payload: sampleMyEntry({ title: "Custom Title" }),
});
```

---

## Test Organization Layers

```
tests/
├── foundation/           # Single-agent, happy-path CRUD
│   └── my_entry.test.ts  # Create, read, update, delete by same agent
├── integration/          # Two-agent, cross-zome, collection reads
│   └── my_entry.test.ts  # DHT propagation, agent index, path queries
└── scenario/             # Full user journeys
    └── marketplace.test.ts  # Multi-step business logic flows
```

**Foundation first:** Write and pass foundation tests before integration tests. Foundation tests catch basic API issues without the complexity of multi-agent sync.

---

## Cell Access

```typescript
// Access by DNA role name (defined in happ.yaml)
const cell = agent.namedCells.get("my_dna_role");

// Or by index (less readable)
const cell = agent.cells[0];

// Cell ID for dhtSync
const dnaHash = alice.cells[0].cell_id[0];
await dhtSync([alice, bob], dnaHash);
```

---

## Test Commands

```bash
bun run test                       # Run all tests
bun run test:foundation            # Foundation layer only
bun run test:integration           # Integration layer only
vitest run --reporter verbose      # Verbose output
vitest run tests/my_entry.test.ts  # Single file
```

---

## Common Test Failures

| Symptom | Root Cause | Fix |
|---------|-----------|-----|
| Bob can't find Alice's entry | Missing `dhtSync` | Add `await dhtSync([alice, bob], dnaHash)` |
| Update fails with "wrong hash" | Using `original_hash` as `previous_hash` | Fetch latest record first, use `signed_action.hashed.hash` |
| Test times out at 30s | Default timeout too short | Set `testTimeout: 60000` in vitest config |
| `namedCells.get()` returns undefined | Wrong role name | Check `roles[].name` in `happ.yaml` |
| Validation error on create | Entry struct mismatch | Check serde field names match Rust struct |
