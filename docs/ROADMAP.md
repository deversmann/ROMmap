# ROMmap delivery roadmap

Planning baseline: one full-time developer; estimates are elapsed working weeks **after** technical choices and environment are available. Adjust for staffing and discoveries. M0–M3 form the headless MVP (~10–14 weeks); M4 makes it approachable in a browser (~3–4 more weeks). Later items are not committed dates. Requirement IDs and acceptance criteria live in `REQUIREMENTS.md`; technical approach lives in `ARCHITECTURAL_SPEC.md`.

| Milestone | Estimate | Deliverables | Exit gate |
| --- | --- | --- | --- |
| **M0: foundation** | 1–2 weeks | Select language/framework, ADRs for import and locking; repo, CI, RHEL 10/rootless Podman dev setup, SQLite migrations, fixtures and API skeleton. Document supported OS/filesystems and threat model. | CI runs tests, service starts non-root with persistent state, migration round-trip works. |
| **M1: safe inventory/import** | 3–4 weeks | Configured roots, scan and hashing, immutable asset records, duplicate preview, explicit copy import, durable jobs, CLI/API, restore notes. `REQ-STO-02..04`, `REQ-OPS-01..03`. | Scan does not mutate intake; duplicate imports and restart/retry are safe; changed files, symlinks and disallowed paths are rejected/reported. |
| **M2: hardlink mapper** | 4–5 weeks | Hardlink capability probe, YAML v1, Knulli-oriented staging example (device compatibility to be validated), collision detection, reviewed plan/apply, owned-link tracking, state/audit. `REQ-STO-01,05`, `REQ-MAP-01..04`. | Idempotent rerun; cross-device fails before apply; unknown files survive; interrupted apply resumes; external edits cause drift/conflict. |
| **M3: headless MVP release** | 2–3 weeks | Stabilize API/OpenAPI and CLI, logical telemetry, Containerfile/Quadlet packaging, backup/restore docs, security and filesystem test matrix. `REQ-MET-01`, remaining MVP operations. | End-to-end: scan → preview/copy import → plan → apply → verify → restart → reconcile; no critical data-loss/security issues; release notes list limits. |
| **M4: web workflow** | 3–4 weeks | Setup wizard, inventory, import review, target dry-run/apply and status, job/errors dashboard, accessible responsive layout. `REQ-OPS-04`, `REQ-MET-02`. | UI performs full MVP workflow through public API; failed jobs/conflicts are visible and actionable; usability test with fresh install. |
| **M5: curated metadata/presets** | TBD after M4 | DAT parser with fixtures/provenance and manual conflict resolution, tested additional profiles, grouping model for multi-file sets. `REQ-DAT-01` and part of `02`. | Real-world fixture suites pass; multi-match and multi-file layouts do not corrupt or flatten companion files. |
| **M6: advanced formats/exports** | TBD | Bounded ZIP/7z member inspection, copy/sync device export with its own policy and storage estimate. `REQ-DAT-02`, `REQ-MAP-05`. | Archive-bomb limits, offline/removable destinations and interrupted copy/retry tested. |
| **Optional exploration** | Not scheduled | Browser emulator, advanced metadata sources, metrics integrations, additional databases/platforms. | Separate design/security/licensing review and acceptance criteria before commitment. |

## Current planning status

**M0 is open. Next step: review stack candidates and select the implementation approach in [issue #1](https://github.com/deversmann/ROMmap/issues/1), then submit ADR-0001 for human acceptance.** No language/framework has been chosen. In parallel, [issue #4](https://github.com/deversmann/ROMmap/issues/4) will validate hardlink behavior from the intended rootless Podman container against the TrueNAS NFS dataset. Existing Radarr/Sonarr hardlink success is strong prior evidence but not a ROMmap container-level probe. Issue #7 records the M0 gate; GitHub issues/PRs hold ongoing task status rather than a second status ledger.

## Tracking and decisions

Track work as GitHub issues created from `.github/ISSUE_TEMPLATE/00-task.yml`, labeled by milestone (`milestone:M0` through `milestone:M6`, or `milestone:later`). Link PRs, tests and blockers from issues. Use `docs/adr/0000-template.md` for durable decisions; the first real decision is ADR-0001. `AGENTS.md` defines ownership, review and completion rules. This roadmap is the scope/timeline authority; issue labels are the work queue, not a second roadmap.

## Decision log to close in M0

1. **Implementation stack ([#1](https://github.com/deversmann/ROMmap/issues/1)):** select language, HTTP framework, YAML library and job execution approach based on Linux portability, filesystem primitives, future REST-backed web GUI and maintenance cost. No language preference. Owner: project maintainer. Record ADR-0001.
2. **Authentication ([#2](https://github.com/deversmann/ROMmap/issues/2)):** design built-in single-admin login for LAN/Tailscale browser access via colocated NGINX HTTPS proxy; keep the backend loopback-bound. Decide bootstrap/recovery, sessions, proxy trust and an eventual OIDC boundary without implementing multi-tenancy now. Owner: project maintainer. Test unauthorized access.
3. **Pool naming and source selection:** settle content-addressed naming, tagging of platform/title for otherwise unidentified files, and copy-only import policy for v1. Owner: project maintainer. Test renames and duplicate inputs.
4. **Filesystem support matrix ([#4](https://github.com/deversmann/ROMmap/issues/4)):** prioritize a hardlink probe inside rootless Podman on RHEL 10 with pool and staging in one TrueNAS SCALE dataset over NFS; verify UID/GID, export permissions and mount behavior. Existing Radarr/Sonarr hardlink success is evidence, not this probe. Test other filesystems where available; declare only tested configurations. Owner: project maintainer. Record results.
5. **Target overwrite and retention policy:** adopt unknown-file preservation and verified-owned-link-only removal; decide how long abandoned job/plan records are retained. Owner: project maintainer. Test cleanup and restart behavior.

## Scope control

The first device workflow stages a Knulli-oriented layout for the Anbernic RG34XXSP on the NAS and the user copies it to SD manually. Do not add DAT parsing, archive extraction, automated handheld copy export or emulator execution to M0–M3 to solve a UI demo problem. Re-estimate after M2 with observed performance and filesystem behavior. Each milestone needs automated tests, a short manual verification checklist and an updated limitations section. A blocked gate delays the release rather than weakening the safety rule.
