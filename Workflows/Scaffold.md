# Workflow: Scaffold a Holochain Project

Use this workflow to set up a new Holochain project from scratch, or to add a new domain to an existing hApp.

**Reference:** `../Scaffold.md` for full details on any step.

---

## Path A: New hApp From Scratch

### Step 1 — Install Nix and Holonix

```bash
# Install Nix (Determinate Systems installer — recommended)
curl --proto '=https' --tlsv1.2 -sSf -L https://install.determinate.systems/nix | sh -s -- install

# Enable flakes (add to ~/.config/nix/nix.conf)
echo "experimental-features = nix-command flakes" >> ~/.config/nix/nix.conf
```

Restart your shell after installation. Verify: `nix --version`

**Checkpoint:** `nix --version` returns a version number.

---

### Step 2 — Create the flake.nix

In your new project directory:

```nix
# flake.nix
{
  inputs = {
    holonix.url = "github:holochain/holonix?ref=main-0.6";
    nixpkgs.follows = "holonix/nixpkgs";
    flake-parts.follows = "holonix/flake-parts";
  };

  outputs = inputs: inputs.flake-parts.lib.mkFlake { inherit inputs; } {
    systems = builtins.attrNames inputs.holonix.devShells;
    perSystem = { inputs', ... }: {
      devShells.default = inputs'.holonix.devShells.default;
    };
  };
}
```

Enter the dev shell:
```bash
nix develop
```

**Checkpoint:** `hc --version` returns a version number inside `nix develop`.

---

### Step 3 — Scaffold the hApp

```bash
# Inside nix develop:
hc scaffold happ
```

The CLI will prompt for:
- **App name** — e.g., `my-community-app` (kebab-case)
- **DNA name** — e.g., `community` (the first domain)
- **Coordinator zome name** — e.g., `posts` (first feature)

This generates the complete project structure.

**Checkpoint:** `ls` shows `happ.yaml`, `Cargo.toml`, `flake.nix`, and `dnas/` directory.

---

### Step 4 — Verify Cargo Workspace

Check `Cargo.toml` at the root uses exact version pins:

```toml
[workspace.dependencies]
hdi = "=0.7.0"
hdk = "=0.6.0"
serde = { version = "1", features = ["derive"] }
```

If the scaffold generated range versions (`^`), replace them with exact pins (`=`).

**Why:** Holochain is sensitive to minor version differences. Range deps can silently break compilation.

---

### Step 5 — Add Entry Types

For each data type in your domain:

```bash
# Inside nix develop, from project root
hc scaffold entry-type MyEntry

# Then add required link types
hc scaffold link-type AgentToMyEntry
hc scaffold link-type PathToMyEntry
hc scaffold link-type MyEntryUpdates
```

---

### Step 6 — Verify Compilation

```bash
hc s sandbox generate workdir/
```

**Expected:** Build succeeds (may take 5-10 minutes on first run due to WASM compilation).

**Common issues:**
- `wasm32 target not found` — you're outside `nix develop`; run `nix develop` first
- Slow first build — normal; wait for `wasm-opt` to complete

---

### Step 7 — Set Up Tests

Create the test directory structure:

```bash
mkdir -p tests/{foundation,integration}

# Initialize package.json
cd tests
bun init  # or npm init

# Install test dependencies
bun add -d @holochain/tryorama vitest
```

Create `tests/vitest.config.ts`:
```typescript
import { defineConfig } from "vitest/config";
export default defineConfig({
  test: { testTimeout: 60000, hookTimeout: 60000 },
});
```

Add test scripts to `tests/package.json`:
```json
{
  "scripts": {
    "test": "vitest run",
    "test:foundation": "vitest run foundation",
    "test:integration": "vitest run integration"
  }
}
```

**Checkpoint:** `bun run test` runs without errors (may have no tests yet — that's fine).

---

### Step 8 — Initial Commit

```bash
git init
git add .
git commit -m "feat: scaffold initial happ structure"
```

Proceed to `Workflows/DesignDataModel.md` to design your first domain's data model, then `Workflows/ImplementZome.md` to implement.

---

## Path B: Add Domain to Existing hApp

Use this path when your hApp already exists and you need to add a new feature domain.

### Step 1 — Enter Dev Shell

```bash
nix develop
```

### Step 2 — Scaffold New Zome Pair

```bash
hc scaffold zome
# Enter: domain name (e.g., "profiles")
# Select: existing DNA to add it to
```

### Step 3 — Scaffold Entry Types

```bash
hc scaffold entry-type Profile
hc scaffold link-type AgentToProfile
hc scaffold link-type PathToProfile
hc scaffold link-type ProfileUpdates
```

### Step 4 — Register in Cargo Workspace

Add new crates to root `Cargo.toml` members:

```toml
[workspace]
members = [
    # ... existing members ...
    "dnas/my_dna/zomes/integrity/profiles_integrity",
    "dnas/my_dna/zomes/coordinator/profiles",
]
```

### Step 5 — Verify Compilation

```bash
hc s sandbox generate workdir/
```

### Step 6 — Commit

```bash
git add .
git commit -m "feat(profiles): scaffold profiles zome pair"
```

Proceed to `Workflows/ImplementZome.md` to implement the domain.

---

## Quick Reference

```bash
# Enter dev environment
nix develop

# New project
hc scaffold happ

# New domain
hc scaffold zome
hc scaffold entry-type MyEntry
hc scaffold link-type AgentToMyEntry

# Verify build
hc s sandbox generate workdir/

# Run tests
bun run test:foundation
bun run test:integration
```

**Reference:** `../Scaffold.md` for full setup details, troubleshooting, and workspace structure.
