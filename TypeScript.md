# Holochain TypeScript Client

## Package Versions

```
@holochain/client   ^0.20.x   (compatible with hdk 0.6.x / hdi 0.7.x)
```

---

## Connection Setup

```typescript
import { AppWebsocket, AppAgentWebsocket } from "@holochain/client";

// Basic connection (for simple apps)
const appWs = await AppWebsocket.connect(
  new URL(`ws://localhost:${process.env.HC_PORT}`),
  30000  // timeout ms
);

// Agent-aware connection (recommended — wraps calls with cell context)
const client = await AppAgentWebsocket.connect(
  new URL(`ws://localhost:${process.env.HC_PORT}`),
  "my-app-id"  // Installed app ID
);
```

---

## callZome Pattern

```typescript
// Direct AppWebsocket (requires explicit cell_id)
const record = await appWs.callZome({
  cell_id: [dnaHash, agentPubKey],
  zome_name: "my_zome",
  fn_name: "create_my_entry",
  payload: {
    title: "New Entry",
    description: "Created from TypeScript",
    status: "Active",
  },
  cap_secret: null,
  provenance: agentPubKey,
});

// AppAgentWebsocket (cleaner — cell resolved by role name)
const record = await client.callZome({
  role_name: "my_dna",
  zome_name: "my_zome",
  fn_name: "create_my_entry",
  payload: { title: "New Entry", status: "Active" },
});
```

---

## Signal Subscription

```typescript
// Subscribe to all signals from the app
appWs.on("signal", (signal) => {
  if (signal.type !== "App") return;  // Filter system signals

  const { zome_name, payload } = signal.data.payload;

  // Discriminate by zome
  if (zome_name === "my_zome") {
    handleMyZomeSignal(payload);
  }
});

// Signal payload matches Rust enum (serde tag = "type")
type MySignal =
  | { type: "EntryCreated"; action: SignedActionHashed }
  | { type: "EntryUpdated"; action: SignedActionHashed; original_action_hash: HoloHash }
  | { type: "EntryDeleted"; action: SignedActionHashed; original_action_hash: HoloHash };

function handleMyZomeSignal(payload: MySignal) {
  switch (payload.type) {
    case "EntryCreated":
      // Refresh entry list
      break;
    case "EntryUpdated":
      // Update specific entry in store
      break;
  }
}
```

---

## Effect Library Pattern

The Effect library provides typed error handling and timeouts for zome calls:

```typescript
import * as E from "effect";
import { Effect, pipe } from "effect";

// Typed error
class ZomeCallError {
  readonly _tag = "ZomeCallError";
  constructor(readonly message: string, readonly cause?: unknown) {}
}

// Wrapped zome call with timeout and error handling
function callZomeEffect<T>(params: CallZomeRequest) {
  return pipe(
    E.tryPromise({
      try: () => client.callZome(params) as Promise<T>,
      catch: (cause) => new ZomeCallError(`Zome call failed: ${params.fn_name}`, cause),
    }),
    E.timeout("10 seconds"),
    E.mapError((e) =>
      e._tag === "TimeoutException"
        ? new ZomeCallError(`Zome call timed out: ${params.fn_name}`)
        : e
    )
  );
}

// Usage
const result = await E.runPromise(
  callZomeEffect<MyEntry>({
    role_name: "my_dna",
    zome_name: "my_zome",
    fn_name: "get_my_entry",
    payload: actionHash,
  })
);
```

---

## Svelte 5 Reactive Store Integration

```typescript
// stores/myEntry.svelte.ts
import { AppAgentWebsocket } from "@holochain/client";

export class MyEntryStore {
  entries = $state<MyEntry[]>([]);
  loading = $state(false);
  error = $state<string | null>(null);

  private client: AppAgentWebsocket;

  constructor(client: AppAgentWebsocket) {
    this.client = client;

    // Subscribe to signals for real-time updates
    client.on("signal", (signal) => {
      if (signal.type !== "App") return;
      const { zome_name, payload } = signal.data.payload;
      if (zome_name === "my_zome") this.handleSignal(payload);
    });
  }

  async loadAll() {
    this.loading = true;
    try {
      const records = await this.client.callZome({
        role_name: "my_dna",
        zome_name: "my_zome",
        fn_name: "get_all_my_entries",
        payload: null,
      });
      this.entries = records.map(decodeEntry);
    } catch (e) {
      this.error = String(e);
    } finally {
      this.loading = false;
    }
  }

  private handleSignal(signal: MySignal) {
    switch (signal.type) {
      case "EntryCreated":
        this.loadAll();
        break;
      case "EntryDeleted":
        this.entries = this.entries.filter(
          (e) => e.originalHash !== signal.original_action_hash
        );
        break;
    }
  }
}
```

---

## Type Utilities

```typescript
import { decodeHashFromBase64, encodeHashToBase64, HoloHash } from "@holochain/client";

// Hash serialization (for URLs, localStorage)
const hashString = encodeHashToBase64(actionHash);
const hashBack = decodeHashFromBase64(hashString);

// Decode entry from record
function decodeEntry<T>(record: Record): T {
  if (!("Present" in record.entry)) {
    throw new Error("Expected Present entry");
  }
  return decode(record.entry.Present.entry) as T;
}

// Extract action hash from record
function getActionHash(record: Record): HoloHash {
  return record.signed_action.hashed.hash;
}
```

---

## Connection Context (SvelteKit)

```typescript
// src/lib/holochainClient.ts
import { AppAgentWebsocket } from "@holochain/client";
import { getContext, setContext } from "svelte";

const CLIENT_KEY = Symbol("holochain-client");

export function setHolochainClient(client: AppAgentWebsocket) {
  setContext(CLIENT_KEY, client);
}

export function getHolochainClient(): AppAgentWebsocket {
  const client = getContext<AppAgentWebsocket>(CLIENT_KEY);
  if (!client) throw new Error("Holochain client not initialized");
  return client;
}

// In +layout.svelte:
// const client = await AppAgentWebsocket.connect(...);
// setHolochainClient(client);
```

---

## Environment Variables

```
HC_PORT=8888           # Holochain conductor WebSocket port
HC_ADMIN_PORT=9000     # Admin port (for conductor management)
VITE_HC_PORT=8888      # Vite prefix for browser access
```
