# ROMmap: project overview

**Map your games. Save your storage.** ROMmap is a self-hosted inventory and layout tool for user-owned emulation files. It catalogs files in a managed pool and projects local, frontend-oriented directory trees using hardlinks. A file can have multiple names in different local layouts without duplicating its data blocks for each placement.

Hardlinks have boundaries: pool and target must support hardlinking within the same mounted filesystem; a link is not a portable pointer to a NAS from a handheld. All links share file contents, so edits through a target can change the pool. ROMmap verifies capabilities, tracks only links it creates, and does not silently delete source files or unknown target files. Copy/sync exports to removable or remote devices are a later, distinct feature.

## First usable release

1. Configure an intake directory, a managed pool and a local target root.
2. Scan intake without changing it; review file/hash inventory and a copy-import preview.
3. Import selected files into the managed pool explicitly.
4. Select a versioned layout profile and review a dry-run with collisions/conflicts.
5. Apply the plan, inspect job results, and audit/reconcile the target after changes.

The headless CLI and REST API come first; a web setup/dashboard follows. Linux and containers with persistent database and correct mount configuration are initial deployment targets. SQLite is the initial database. DAT recognition, multi-file grouping, archive-member inspection, handheld exports and browser emulation are later work, not implied by the first release.

## Design principles

- **Safety before automation:** read-only source scans, explicit import, preview before apply, verified-owned-link-only cleanup.
- **Reproducibility:** versioned profiles and deterministic plans with durable job outcomes.
- **Honest accounting:** display logical pool size, target placements and avoided-copy estimates—not invented exact disk savings.
- **One interface contract:** CLI and web UI use the same core operations/API.
- **Progressive disclosure:** a guided workflow for new users; profile YAML and documented API for advanced users.

## Read next

- `REQUIREMENTS.md` — authoritative behavior, scope, acceptance gates and requirement IDs.
- `ARCHITECTURAL_SPEC.md` — implementation decisions, data model, workflows and API baseline.
- `ROADMAP.md` — milestone sequence, estimated timeline, gates and outstanding decisions.
- `WIREFRAME_SETUP.md` and `DASHBOARD.md` — M4 GUI proposals, subordinate to requirements.
