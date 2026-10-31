# ROMmap

**Map your games. Save your storage.**

ROMmap is a planned self-hosted tool for inventorying user-owned emulation files and presenting them in different directory layouts. It will keep a managed pool of files and create local, frontend-oriented trees using filesystem hardlinks, so multiple layouts can reference the same file data without copying it for each placement.

> **Project status:** M0 planning; there is no runnable application yet. **Next step:** review stack options in [issue #1](https://github.com/deversmann/ROMmap/issues/1) and select ADR-0001. The scope and timeline below are proposals, not released features.

## How it is intended to work

1. Scan an intake directory without changing its contents.
2. Review and explicitly copy selected files into a managed pool.
3. Choose a versioned layout profile and preview the paths and any conflicts.
4. Apply the plan to a local target, then audit and reconcile it as the collection changes.
5. For the first device use case, manually copy the staged Knulli-oriented layout to an Anbernic RG34XXSP SD card.

The first deployment is planned for a headless RHEL 10 host running rootless Podman, with pool and staging directories in one TrueNAS SCALE NFS dataset and SQLite/application state on local persistent storage. A colocated NGINX proxy provides HTTPS for the later web UI; a single-admin login is planned. The first usable release is a headless CLI and REST service, followed by the web setup flow and dashboard. See the [overview](docs/OVERVIEW.md) for the product direction.

**Important limits:** Hardlinks require a compatible pool and target on the same mounted filesystem; they do not create portable links to a handheld SD card or remote filesystem. Links share file contents, so editing a target can alter the pool file. ROMmap is designed to preserve intake files and unknown files in targets, and to show changes before applying them. Copy/sync exports for devices are a separate, later feature.

## Project documents

| Document | Purpose |
| --- | --- |
| [Overview](docs/OVERVIEW.md) | Vision, first-release workflow, and design principles. |
| [Requirements](docs/REQUIREMENTS.md) | **Authoritative** behavior, scope, and acceptance criteria. |
| [Architecture](docs/ARCHITECTURAL_SPEC.md) | Proposed implementation decisions, data model, workflows, and API. |
| [Roadmap](docs/ROADMAP.md) | Milestones, estimated timeline, exit gates, and open decisions. |
| [Agent workflow](AGENTS.md) | GitHub issue/PR workflow, decision records, evidence, and review rules. |
| [Setup wireframe](docs/WIREFRAME_SETUP.md) | Proposed first-run web workflow. |
| [Dashboard wireframe](docs/DASHBOARD.md) | Proposed dashboard and inventory views. |

The initial milestone is to settle the implementation stack and deployment/security decisions. The [roadmap](docs/ROADMAP.md) estimates roughly 10–14 weeks for a headless MVP and 3–4 additional weeks for the web workflow, assuming one full-time developer; these are planning estimates, not commitments.

ROMmap does not provide ROMs, BIOS files, DATs, or emulator cores. Users are responsible for the rights to the files they manage.
