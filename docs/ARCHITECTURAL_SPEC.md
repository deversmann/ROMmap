# ROMmap architecture (implementation decisions)

`REQUIREMENTS.md` is authoritative for behavior and scope. This document proposes the implementation to satisfy it. If these documents disagree, change this document or explicitly revise the requirement. Milestones are in `ROADMAP.md`.

## 1. Runtime and storage

Start with a single Linux service/process hosting a job worker and REST API, plus a CLI that calls the same application services (locally or over the API). Use SQLite with WAL, foreign keys and versioned migrations; one writer coordinates imports/reconciles, and a per-target job lock prevents competing applies. Do not require PostgreSQL, gRPC or a scheduler for MVP. Choose language/framework during M0 and record the choice in an ADR; the behavior below is independent of that choice.

Container mounts: read-only intake, writable managed pool, writable target root(s), persistent application state/DB, and scratch space on the pool filesystem for atomic pool placement. Pool and target must be reachable via the same hardlink-capable mounted filesystem from inside the container. `st_dev` comparison can reject obvious incompatibility, but only an actual temporary `link()` test confirms support; repeat before each apply. Mount points and bind mounts can behave differently from physical disks. Map a non-root UID/GID and document ownership/umask. Keep application state outside ROM trees.

Configured allowed roots are loaded at startup, not from API request bodies. Canonicalize roots, reject nesting and unsafe overlap, use directory-relative no-follow operations where supported and revalidate containment at use time. A browser path picker, if offered, lists only configured roots. Service defaults to loopback; require authenticated sessions or API tokens before listening on other interfaces, and redact tokens from logs.

## 2. Data model (MVP)

Use integer IDs, explicit timestamps and schema migrations. The following are conceptual tables, not executable migration SQL:

| Table | Key fields | Purpose |
| --- | --- | --- |
| `sources` | id, configured_root_id, relative_path, size, mtime, status, last_seen | Source observations; multiple paths can refer to the same content. |
| `assets` | id, pool_relative_path (unique), original_extension, size, sha256, sha1, md5, crc32, status, verified_at | Physical pool files; index `(sha256, size)` for candidate duplicates. Retain a chosen original extension separately from pool naming; do not make DAT hashes unique identifiers. |
| `source_assets` | source_id, asset_id, import_job_id | Source-to-pool lineage; preserve every source name and extension. |
| `asset_tags` | asset_id, platform, title, region, provenance | Explicit user-supplied tags for MVP layouts; unset is distinct from a verified DAT match. |
| `profiles` | id, schema_version, revision, YAML, validated_at | Rules plus version; save an immutable revision per change. |
| `targets` | id, configured_root_id, relative_base, profile_id, applied_revision, state | One managed namespace per target. Reject overlapping target namespaces. |
| `mapped_links` | target_id, relative_path, asset_id, recorded_dev, recorded_ino, state | ROMmap-owned links only; unique `(target_id, relative_path)`; dev/ino used for drift checks, not as permanent asset identity. |
| `jobs` / `job_events` | id, kind, state, progress, error_code, timestamps | Durable execution, retry and diagnostics. |

Store relative paths under configured roots rather than accepting arbitrary persisted absolute paths as authority. SHA-256 plus size is a deduplication candidate, confirmed by full content comparison or trusted verified hashes from a stable copy. Do not make `game_metadata.sha1` unique: later DAT data is many-to-many. Future tables: `dat_sources`, `dat_entries`, `asset_matches`, `archive_members`, `game_groups`, `group_members`. Archive-member hash is not the archive's hash. A game group can include multiple assets and require a target profile to preserve relative companion paths.

## 3. Workflows

**Scan:** traverse an allowed intake root without following symlinks; record observations and per-file errors. Stream hashes, checking size/mtime before and after; an altered file is `changed`, not imported. A subsequent scan updates observations without deleting or modifying source files. No assumption that filename implies system or game identity: user-supplied platform tags or explicit mapping can supply missing metadata in MVP.

**Import:** return a preview first. Copy a selected source into a uniquely named temp file on the pool filesystem while hashing; verify a stable source and final temp contents, deduplicate against pool candidates, then atomically rename into a content-addressed pool path (e.g. sha256-prefix/hash with disambiguation). Insert/update DB records transactionally. Crash leftovers are quarantined/cleaned on restart; never remove source data. Importing an archive at MVP stores its original bytes as one opaque file. A future policy for archive inspection must not claim a ZIP's internal checksum matches its outer file.

**Plan:** load an immutable profile revision and inventory snapshot. Expand only allowlisted tokens; define deterministic fallback (`unknown` or skip) per field. Normalize/sanitize individual components, retain original extension, reject unsafe paths and collisions. Compare desired paths with owned links and filesystem observations, producing create/retain/remove/conflict/skip plus warnings. Store plan ID, revisions and expiry; show estimated changes before apply. A changed inventory/profile or filesystem check invalidates the plan.

**Apply:** acquire target lock; revalidate root, link support, plan inputs and each destination. Create parents within the owned target root, create hardlink at a temporary name in the destination directory, then rename into place only if destination is still absent/owned as expected. Never overwrite an unknown destination. Before removing an obsolete owned path, compare actual `(dev, ino)` with the current pool asset and recorded link; report conflict on mismatch. Persist progress per operation, resume safely after restart, and audit at end. Atomic rename is per entry, not an atomic whole-tree swap; partial runs are reported as such. A process crash must leave recoverable records rather than a falsely `synced` target. Permissions help prevent accidental writes but cannot stop a process with sufficient rights from modifying any link.

**Audit/state:** `pending` means the desired revision differs from applied; `drifted` means expected links are absent/changed; `conflict` means a managed path has unexpected contents or cannot be safely reconciled; `unknown` means not checked or checking failed; `synced` requires successful filesystem verification at a known revision. Full checksum audits are separate from routine inode/path checks; show last audit time. If a pool asset fails checksum verification, block operations involving it until repaired or explicitly replaced.

## 4. Profile contract (schema v1)

```yaml
schema_version: 1
name: simple-local-layout
selection:
  platforms: [snes, gba] # explicit inventory tags in MVP
path: "{platform}/{title} ({region}){extension}"
missing:
  title: original_stem
  region: Unknown
  platform: skip
```

A real parser validates fields, supported tokens and YAML size/depth; no arbitrary code execution, filesystem includes or user-defined expressions. `{checksum_short}` uses the first eight hex characters of SHA-256 (not SHA-1). `{developer}` can be introduced after metadata support. Presets are versioned profiles with fixture-based integration tests and documented compatibility assumptions; RetroArch and Pegasus may also need non-directory metadata, and stock OS naming differs by device/version. Do not promise a preset until its contract is tested.

## 5. REST/CLI contract (MVP baseline)

JSON under `/api/v1`; publish an OpenAPI document and use consistent errors `{code, message, details}`. Mutations validate an idempotency key or request revision where retries could duplicate work. Long-running requests return `202 {job_id, status_url}`; clients poll `GET /api/v1/jobs/{id}`. Jobs have queued/running/succeeded/failed/cancelled states; cancellation is best-effort at safe operation boundaries.

| Endpoint | Function |
| --- | --- |
| `GET /health`, `GET /ready` | Liveness and dependency readiness. |
| `GET /api/v1/roots` | Configured roots, capabilities and validation state; no arbitrary path entry. |
| `POST /api/v1/scans`, `GET /api/v1/assets`, `PATCH /api/v1/assets/{id}/tags` | Scan a configured intake root; paginated inventory and explicit metadata tags. |
| `POST /api/v1/import-plans`, `POST /api/v1/import-plans/{id}/apply` | Preview and execute explicit copy import. |
| `GET /api/v1/profiles`, `POST /api/v1/profiles/validate` | Versioned profiles and validation. |
| `GET /api/v1/targets`, `POST /api/v1/targets/{id}/plans`, `POST /api/v1/targets/{id}/plans/{plan_id}/apply` | Target state, reviewed reconciliation and apply. |
| `GET /api/v1/jobs/{id}`, `GET /api/v1/telemetry/storage` | Job outcome and logical accounting. |

CLI examples: `rommap scan --root intake1`, `rommap import plan --root intake1`, `rommap target plan TARGET_ID`, `rommap target apply TARGET_ID PLAN_ID`, `rommap job show JOB_ID`. Define token provision for remote CLI and local CLI configuration without embedding credentials in command histories. API must never accept a client-selected `/mnt/...` path to scan. The GUI uses these same endpoints.

Telemetry fields: `pool_logical_bytes`, `managed_target_logical_bytes`, `avoided_copy_bytes`, `verified_link_count`, `verification_state`, `measured_at`. Avoided-copy bytes count only verified target hardlinks and express a comparison to copying each placement independently, not disk blocks saved in reality. No division-based multiplier without a clearly labeled denominator.

## 6. Operational verification

Unit tests cover profile parsing, fallback and collision behavior; integration tests use temporary filesystems/directories for hardlink creation, restart recovery and database migrations. A separate Linux/container test job covers mount boundaries, unprivileged execution, permissions, case-fold simulation and symlink races. Document `EXDEV`, `EACCES`, `EMLINK` and read-only filesystem failures. Back up DB and pool consistently (pause writes or snapshot); restoring the DB alone does not restore assets or links. Provide a rebuild/audit path after restore. See acceptance gates in `REQUIREMENTS.md`.
