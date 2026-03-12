# Holochain Testing

## Three-Layer Testing Strategy

```
┌─────────────────────────────────────────────────────┐
│  Layer 3 — E2E UI (Playwright + real conductor)     │
│  "Does the UI render real data and journeys work?"  │
├─────────────────────────────────────────────────────┤
│  Layer 2 — Integration (Sweettest, cargo test)      │
│  "Do zomes, DHT sync, and validation work?"         │
├─────────────────────────────────────────────────────┤
│  Layer 1 — Unit (Vitest, stores/services/mappers)   │
│  "Do computed values and business logic work?"      │
└─────────────────────────────────────────────────────┘
```

| Layer | Tool | What it catches | What it misses |
|-------|------|----------------|---------------|
| Unit | Vitest | Store logic, mappers, computed values | Anything touching real Holochain |
| Integration | Sweettest | Zome logic, validation, DHT sync, auth | UI rendering, routing |
| E2E UI | Playwright + `@holochain/client` | Full user journeys, real data display | Isolated zome edge cases |

**Gap:** Browser-side signal handling (`recv_remote_signal`) is not well covered by any layer — it requires a running UI receiving WebSocket push events from a real conductor.

---

## Framework Overview

- **Sweettest** (`holochain::sweettest`) — Rust-native, in-process conductor. Official Holochain team recommendation. Run with `cargo test`.
- **Tryorama** (`@holochain/tryorama`) + **Vitest** — TypeScript, WebSocket-based. Scaffolded by default with `hc scaffold`. Deprecated for 0.7+ by the Holochain team.
- **Playwright** + **`@holochain/client`** — Browser automation against a real conductor. No mocks.

**Note on Tryorama for e2e:** Tryorama wraps `@holochain/client` under the hood and is designed to clean up the conductor after each scenario — the opposite of what e2e UI tests need. Use `@holochain/client` directly for e2e conductor setup.

---

## When to Use Which

| Use Case | Sweettest | Tryorama | Playwright |
|----------|-----------|---------|-----------|
| Zome logic, validation, CRUD | ✅ Preferred | Works | No |
| DHT propagation, consistency | ✅ Preferred | Works | No |
| Multi-agent scenarios | ✅ Preferred | Works | No |
| Inline zomes (no WASM compile) | ✅ Yes | No | No |
| Direct DHT database inspection | ✅ Yes | No | No |
| Full UI user journeys | No | No | ✅ Yes |
| Real data rendered in browser | No | No | ✅ Yes |
| Language | Rust | TypeScript | TypeScript |

---

## Sweettest (Rust-Native)

### Setup (Cargo.toml)

```toml
[dev-dependencies]
holochain = { version = "=0.6.0", features = ["test_utils"] }
tokio = { version = "1", features = ["full"] }
```

### Core Types

| Type | Purpose |
|------|---------|
| `SweetConductor` | Single conductor instance |
| `SweetConductorBatch` | Multiple conductors for multi-agent scenarios |
| `SweetApp` | Installed app with pre-built cells |
| `SweetCell` | Cell reference — access agent key, DNA hash, zome handles |
| `SweetZome` | `(CellId, ZomeName)` handle passed to `conductor.call()` |
| `SweetAgents` | Agent key generation utilities |
| `SweetDnaFile` | DNA construction helpers |
| `SweetInlineZomes` | Define zome functions directly in test code |

### Standard Two-Agent Test

```rust
use holochain::sweettest::*;
use std::path::Path;

#[tokio::test(flavor = "multi_thread")]
async fn two_agents_can_share_entries() {
    // 1. Create two conductors
    let mut conductors = SweetConductorBatch::from_config(
        2,
        SweetConductorConfig::standard(),
    ).await;

    // 2. Load DNA bundle
    let dna = SweetDnaFile::from_bundle(Path::new("workdir/my.dna")).await.unwrap();

    // 3. Install app on both conductors
    let apps = conductors.setup_app("my-app", &[dna]).await.unwrap();
    let ((alice_cell,), (bob_cell,)) = apps.into_tuples();

    // 4. Exchange peer info so conductors can gossip
    conductors.exchange_peer_info().await;

    // 5. Alice creates an entry
    let alice_zome = alice_cell.zome("my_coordinator");
    let hash: ActionHash = conductors[0]
        .call(&alice_zome, "create_my_entry", my_payload)
        .await;

    // 6. Wait for DHT consistency (replaces Tryorama's dhtSync)
    await_consistency(&[&alice_cell, &bob_cell]).await.unwrap();

    // 7. Bob reads the entry
    let bob_zome = bob_cell.zome("my_coordinator");
    let record: Option<Record> = conductors[1]
        .call(&bob_zome, "get_my_entry", hash)
        .await;

    assert!(record.is_some());
}
```

### CRITICAL: await_consistency is MANDATORY Before Cross-Agent Reads

The Rust equivalent of Tryorama's `dhtSync`:

```rust
// After any write, before cross-agent reads:
await_consistency(&[&alice_cell, &bob_cell]).await.unwrap();

// Custom timeout in seconds (default is 60s):
await_consistency_s(30, &[&alice_cell, &bob_cell]).await.unwrap();

// Instant non-waiting check:
check_consistency(&[&alice_cell, &bob_cell]).await.unwrap();
```

`await_consistency` polls every 500ms, comparing all peers' DHT databases at the op level until every op is integrated across all nodes.

### Calling Zome Functions

```rust
// Standard call — panics on error, uses authorship cap automatically:
let result: MyOutputType = conductor.call(&cell.zome("my_zome"), "fn_name", payload).await;

// Fallible call — returns ConductorApiResult:
let result = conductor.call_fallible(&cell.zome("my_zome"), "fn_name", payload).await?;

// Cross-agent call — simulate another agent calling with a cap secret:
let result: MyOutputType = conductor.call_from(
    &other_agent_key,
    Some(cap_secret),
    &cell.zome("my_zome"),
    "restricted_fn",
    payload,
).await;
```

### Agent Key Generation

```rust
// Named deterministic keys (same every run — useful for debugging):
let (alice, bob) = SweetAgents::alice_and_bob();
let alice = SweetAgents::alice();

// Random keys:
let agent = SweetAgents::one(conductor.keystore()).await;
let (a, b, c) = SweetAgents::three(conductor.keystore()).await;
let agents: Vec<AgentPubKey> = SweetAgents::get(conductor.keystore(), 5).await;
```

### Inline Zomes (Quick Isolated Tests, No WASM Compile)

```rust
let mut zomes = SweetInlineZomes::new();
zomes.function("create_thing", |api, input: MyInput| {
    let hash = api.create(CreateInput::new(
        EntryDefLocation::app(0, 0),
        EntryVisibility::Public,
        Entry::app(SerializedBytes::try_from(input)?)?,
        ChainTopOrdering::default(),
    ))?;
    Ok(hash)
});
let dna = SweetDnaFile::unique_from_inline_zomes(zomes).await.unwrap();
```

### Single-Conductor Pattern (Validation and Unit Tests)

```rust
#[tokio::test(flavor = "multi_thread")]
async fn validate_entry_on_create() {
    let conductor = SweetConductor::from_config(SweetConductorConfig::standard()).await;
    let dna = SweetDnaFile::from_bundle(Path::new("workdir/my.dna")).await.unwrap();
    let app = conductor.setup_app("my-app", &[dna]).await.unwrap();
    let (cell,) = app.into_tuple();
    let zome = cell.zome("my_coordinator");

    // Test validation rejection
    let result = conductor.call_fallible(&zome, "create_my_entry", invalid_payload).await;
    assert!(result.is_err());
}
```

### Common Sweettest Failures

| Symptom | Root Cause | Fix |
|---------|-----------|-----|
| Bob can't find Alice's entry | Missing `await_consistency` | Add `await_consistency(&[&alice_cell, &bob_cell]).await.unwrap()` |
| Compilation error on `call()` | Missing feature flag | Add `features = ["test_utils"]` to holochain dev-dep |
| Timeout in `await_consistency` | Conductors not networked | Call `conductors.exchange_peer_info().await` after `setup_app` |
| Wrong type on `call()` | Type annotation missing | Add explicit type: `let result: MyType = conductor.call(...)` |
| `into_tuple()` fails | Wrong number of cells destructured | Match tuple arity to number of DNA roles |

---

## Tryorama (TypeScript)

### Setup (package.json)

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

---

## E2E UI Testing (Playwright + Real Conductor)

For full end-to-end tests that drive the UI against a real Holochain backend — no mocks.

**Do not use Tryorama for this.** Tryorama tears down the conductor after each scenario, which is the opposite of what Playwright needs (conductor must stay alive while the browser runs). Use `@holochain/client` directly.

### Setup (package.json)

```json
{
  "devDependencies": {
    "@playwright/test": "^1.40.0",
    "@holochain/client": "^0.18.0"
  }
}
```

### Conductor Setup Pattern (globalSetup)

The critical pattern: use `AdminWebsocket` to install the app and get a proper auth token, then keep the conductor alive for all Playwright tests.

```typescript
// tests/e2e/setup/global-setup.ts
import { AdminWebsocket, AppWebsocket } from '@holochain/client';
import { execSync, spawn } from 'child_process';

export default async function globalSetup() {
  // 1. Start conductor via hc sandbox
  const conductor = spawn('hc', ['sandbox', 'run', '--root', './test-workdir'], {
    stdio: ['ignore', 'pipe', 'pipe']
  });

  // 2. Wait for conductor ready signal in stdout (not polling)
  await new Promise<void>((resolve, reject) => {
    conductor.stdout?.on('data', (data: Buffer) => {
      if (data.toString().includes('Conductor ready')) resolve();
    });
    setTimeout(() => reject(new Error('Conductor startup timeout')), 30000);
  });

  // 3. Connect admin client (use admin port, not app port)
  const admin = await AdminWebsocket.connect({
    url: new URL('ws://localhost:8888')
  });

  // 4. Install and enable the happ properly
  const agentKey = await admin.generateAgentPubKey();
  await admin.installApp({
    installed_app_id: 'my_happ',
    agent_key: agentKey,
    path: './workdir/my_happ.happ',
  });
  await admin.enableApp({ installed_app_id: 'my_happ' });

  // 5. Open app interface on a free port
  const { port } = await admin.attachAppInterface({ port: 0 });

  // 6. Issue auth token
  const { token } = await admin.issueAppAuthenticationToken({
    installed_app_id: 'my_happ'
  });

  // 7. Connect app client and seed test data
  const client = await AppWebsocket.connect({
    url: new URL(`ws://localhost:${port}`),
    token,
  });

  await seedTestData(client);

  // 8. Store conductor process for teardown
  process.env.E2E_CONDUCTOR_PID = String(conductor.pid);
  process.env.E2E_APP_PORT = String(port);
}
```

### Data Seeding

Seed data directly via `AppWebsocket.callZome` before Playwright opens the browser:

```typescript
async function seedTestData(client: AppWebsocket) {
  // Seed in dependency order
  await client.callZome({
    role_name: 'my_dna',
    zome_name: 'my_coordinator',
    fn_name: 'create_service_type',
    payload: { name: 'Web Development', description: '...' },
  });

  await client.callZome({
    role_name: 'my_dna',
    zome_name: 'my_coordinator',
    fn_name: 'create_offer',
    payload: { title: 'Seed Offer', description: '...' },
  });
}
```

### Playwright Test Pattern

```typescript
// tests/e2e/specs/offers.spec.ts
import { test, expect } from '@playwright/test';

test('user sees seeded offers on load', async ({ page }) => {
  await page.goto('/offers');

  // Wait for Holochain connection (not a mock — real loading time)
  await expect(page.locator('[data-testid="offer-card"]'))
    .toHaveCount(1, { timeout: 15000 });

  await expect(page.locator('text=Seed Offer')).toBeVisible();
});
```

### playwright.config.ts Key Settings

```typescript
export default defineConfig({
  globalSetup: './tests/e2e/setup/global-setup.ts',
  globalTeardown: './tests/e2e/setup/global-teardown.ts',
  workers: 1,           // Single worker — one conductor, no conflicts
  fullyParallel: false, // Holochain state is shared across tests
  timeout: 60000,       // Holochain operations are slow
  use: {
    baseURL: 'http://localhost:5173',
  },
  webServer: {
    command: 'bun run dev',
    url: 'http://localhost:5173',
    reuseExistingServer: true,
  },
});
```

### Common E2E Failures

| Symptom | Root Cause | Fix |
|---------|-----------|-----|
| Conductor never ready | Polling instead of stdout | Listen for `"Conductor ready"` in stdout |
| `callZome` rejected | Using `AppWebsocket` on admin port | Use `AdminWebsocket` on admin port (8888), `AppWebsocket` on app port |
| Auth error on connect | Missing token | Call `admin.issueAppAuthenticationToken()` and pass token to `AppWebsocket.connect` |
| Tests interfere with each other | Shared conductor state | Run with `workers: 1`, reset data in `beforeEach` if needed |
| UI shows no data | Race — browser loads before seeding | Seed in `globalSetup` (runs before browser opens), not in `beforeAll` |
