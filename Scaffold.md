# Holochain Scaffold

## Prerequisites

Holochain development requires Nix for a reproducible development environment. All tooling (Rust, hc CLI, holochain, lair-keystore) is managed through Holonix.

### Install Nix

```bash
# Official Nix installer (recommended)
sh <(curl -L https://nixos.org/nix/install) --no-daemon

# Or with Determinate Systems installer (more reliable, adds uninstaller)
curl --proto '=https' --tlsv1.2 -sSf -L https://install.determinate.systems/nix | sh -s -- install
```

Enable flakes (required for Holonix):
```
# Add to ~/.config/nix/nix.conf (or /etc/nix/nix.conf):
experimental-features = nix-command flakes
```

---

## Standard flake.nix (Holonix)

Pin to `main-0.6` for HDK 0.6.x stability:

```nix
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
# Now hc, cargo, rustc, and all Holochain tooling are available
```

**Why pin the branch?** Holonix `main` tracks the latest dev version. `main-0.6` pins all tooling to HDK 0.6.x compatibility. Mixing versions causes compilation failures.

---

## hc Scaffold Commands

The `hc scaffold` CLI generates boilerplate that follows Holochain conventions. Always use it before writing by hand.

### New hApp

```bash
# Create a complete new hApp project
hc scaffold happ
# Prompts for: app name, DNA name, coordinator zome name
# Generates: flake.nix, happ.yaml, dna.yaml, Cargo workspace, first zome pair
```

### New DNA (for multi-DNA hApps)

```bash
# Add a new DNA to an existing hApp
hc scaffold dna
# Prompts for: DNA name
# Generates: dna.yaml, new zome pair stubs
```

### New Zome Pair

```bash
# Add a coordinator/integrity zome pair to an existing DNA
hc scaffold zome
# Prompts for: zome name, DNA to add it to
# Generates: integrity crate + coordinator crate with Cargo.toml
```

### Entry Type

```bash
# Add an entry type to an existing zome pair
hc scaffold entry-type MyEntry
# Generates: entry struct in integrity, create/get/update/delete stubs in coordinator
# Also generates basic Tryorama test file
```

### Link Type

```bash
# Add a link type
hc scaffold link-type AgentToMyEntry
# Generates: link type variant in integrity, create/get/delete helpers in coordinator
```

### Collection

```bash
# Add a collection (global path anchor) for an entry type
hc scaffold collection
# Prompts for: entry type to index, collection type (global or by-agent)
```

---

## Verify Compilation

After any scaffold operation, always verify the project compiles:

```bash
# Generate and verify WASM compilation
hc s sandbox generate workdir/

# Or using the build alias (if package.json scripts are set up)
bun run build
```

**First build is slow** (WASM compilation + wasm-opt). Subsequent builds use the Rust cache. Expect 2-5 minutes for a fresh build.

---

## Project Structure After Scaffolding

```
my-happ/
├── flake.nix                     # Nix dev environment (Holonix)
├── Cargo.toml                    # Workspace root — pins hdk/hdi versions
├── happ.yaml                     # hApp manifest (roles, DNAs)
├── workdir/                      # Build output directory
└── dnas/
    └── my_dna/
        ├── dna.yaml              # DNA manifest (zomes, properties)
        └── zomes/
            ├── integrity/
            │   └── my_domain_integrity/
            │       ├── Cargo.toml
            │       └── src/
            │           ├── lib.rs        # Entry types, link types, validate()
            │           └── types.rs      # Entry structs
            └── coordinator/
                └── my_domain/
                    ├── Cargo.toml
                    └── src/
                        ├── lib.rs        # pub extern declarations
                        └── my_entry.rs   # CRUD implementation
```

**Tests live outside dnas:**
```
tests/
├── package.json                  # @holochain/tryorama, vitest
├── vitest.config.ts              # testTimeout: 60000
├── foundation/                   # Single-agent happy-path CRUD
│   └── my_entry.test.ts
└── integration/                  # Two-agent cross-propagation tests
    └── my_entry.test.ts
```

---

## Cargo Workspace Version Pins

Root `Cargo.toml` — always use exact pins (`=`):

```toml
[workspace]
resolver = "2"
members = [
    "dnas/my_dna/zomes/integrity/my_domain_integrity",
    "dnas/my_dna/zomes/coordinator/my_domain",
]

[workspace.dependencies]
hdi = "=0.7.0"
hdk = "=0.6.0"
serde = { version = "1", features = ["derive"] }
thiserror = "1"
```

**Why exact pins (`=`)?** Holochain zome compilation is extremely sensitive to minor version differences. Range deps (`^`) can silently pull in incompatible patch releases.

---

## Add Domain to Existing Project

When adding a new feature domain to an existing hApp:

```bash
# 1. Enter Nix dev shell if not already in it
nix develop

# 2. Scaffold a new zome pair
hc scaffold zome
# Enter: domain name (e.g., "profiles"), select existing DNA

# 3. Scaffold entry types for the domain
hc scaffold entry-type Profile
hc scaffold link-type AgentToProfile
hc scaffold link-type PathToProfile
hc scaffold link-type ProfileUpdates

# 4. Add the new crates to workspace Cargo.toml members list

# 5. Verify compilation
hc s sandbox generate workdir/
```

Proceed to `Workflows/ImplementZome.md` to fill in the implementation.

---

## Common Setup Issues

| Problem | Cause | Fix |
|---------|-------|-----|
| `nix: command not found` | Nix not installed or not in PATH | Restart shell after install; check `~/.nix-profile/bin` in PATH |
| `flakes not enabled` | Missing experimental-features config | Add `experimental-features = nix-command flakes` to `~/.config/nix/nix.conf` |
| `hc: command not found` inside nix develop | Wrong holonix branch | Check `flake.nix` ref — must be `main-0.6`, not `main` |
| `wasm32 target not found` | Rust toolchain outside Nix | Use `nix develop`; don't use system Rust for Holochain builds |
| First build hangs at `wasm-opt` | wasm-opt is slow on first run | Normal — wait 5-10 min; subsequent builds are fast |

**Reference:** [developer.holochain.org/get-started/](https://developer.holochain.org/get-started/)
