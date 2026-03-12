# Roadmap

## v1 — Core Spiral (current)

**Theme:** Everything needed to build, test, and deploy a Holochain hApp from scratch.

**Domains:** Architecture, Design, Scaffold, Implement, Test, Deploy

**Workflows:**
- `DesignDataModel` — DHT entry/link type design with validation rules
- `Scaffold` — New project and new domain scaffolding workflows
- `ImplementZome` — Full CRUD zome implementation
- `DesignAccessControl` — Capability grants and admin patterns
- `PackageAndDeploy` — Kangaroo-Electron packaging and CI/CD

**Target:** All Agent Skills-compatible tools (Claude Code, GitHub Copilot, Cursor, Augment, Codex)

---

## v2 — Ecosystem Expansion

**Theme:** Connect to the broader Holochain ecosystem. Cross-hApp and cross-network patterns.

**Planned additions:**

### Sub-skills
- **hREA / ValueFlows** — Scaffold and implement ValueFlows-compatible economic resource tracking; `EconomicEvent`, `EconomicResource`, `Process` entry types; REA ontology patterns
- **holochain-open-dev** — Community-standard patterns: Profiles zome, linked devices, file storage, notifications
- **ADAM (coasys)** — AD4M perspectives, expression languages, cross-hApp linking
- **Holo Hosting** — HTTP gateway setup, edge node configuration, Holo Node ISO, HolOS

### New context files
- `WindTunnel.md` — Performance testing with Wind Tunnel
- `unyt.md` — Unit-aware numeric types for resource tracking apps

### Architecture improvements
- Skill graph: parent orchestrator routing to sub-skills
- Cross-LLM portability (GLM 5, any client with skill support)

---

## v3 — GUI and Visual Tooling

**Theme:** Make Holochain accessible without deep framework knowledge. From developers to builders.

**Vision:**
- Visual DHT data model explorer — design entry/link types through a diagram interface
- No-code workflow UI — guided scaffold and deploy without terminal commands
- Architecture diagram generation — auto-generate from zome code
- Progressive disclosure — beginner mode (guided, verbose) vs. expert mode (fast, terse)
- Monitoring integration — visual DHT health, gossip status, conductor logs

**Inspiration:** Holo Node ISO's web-based Node Manager shows the direction — powerful infrastructure made accessible through UI. This skill's v3 applies the same principle to development tooling.

---

## PAI Integration (post-v1)

Once both the PAI version and vanilla version are field-tested:

1. **Audit differences** — what did each version evolve to independently?
2. **Extract shared knowledge** — create canonical knowledge files usable by both
3. **Layer PAI on top** — PAI SKILL.md wraps shared files and adds PAI-specific features (voice, project routing, Algorithm integration)
4. **Publish shared core** — vanilla skill becomes the community baseline; PAI version is a superset

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | 2026-03-12 | Initial vanilla skill — 6 domains, 5 workflows, requirements spec |
| 0.1.1 | 2026-03-12 | Agent Skills Open Standard conformance, multi-platform README, testing plan |
