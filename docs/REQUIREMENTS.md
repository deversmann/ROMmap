# ROMmap requirements (authoritative)

Status: proposed baseline. This file defines product behavior and acceptance criteria. `ARCHITECTURAL_SPEC.md` records implementation decisions; `ROADMAP.md` sequences delivery. A change to a MUST below requires updating its tests and any dependent documents.

## 1. Purpose and boundaries

ROMmap inventories a collection of user-owned game files and creates reproducible directory layouts for emulation frontends. Its first release manages **local hardlink targets** on the same filesystem as a managed master pool. Hardlinks avoid extra file-data copies for these local layouts; they do not provide remote access, immutability, or portable links to another disk. A handheld SD card or remote filesystem needs a separate copy/sync export, outside the first release.

Terms:
- **Intake**: user-controlled source directory. Scanning never changes it.
- **Managed pool**: ROMmap-controlled files imported from intake; import copies or moves only according to an explicit, reviewed operation. The default is copy. Pool files are not rewritten by ROMmap after import.
- **Asset**: a physical file in the pool, with size and checksums. An asset is not necessarily a game; archives and multi-file games need distinct modeling.
- **Target**: a ROMmap-owned directory tree containing hardlinks to pool assets, on the same hardlink-capable filesystem.
- **Profile**: versioned rules mapping selected assets to relative target paths.
- **Reconcile**: compare desired layout with the managed links and apply a reviewed plan.
- **DAT**: external identification data. A DAT match is evidence of identity, not proof that a game runs.

Initial deployment context: rootless Podman on a headless RHEL 10 server, with pool and local staging target in one TrueNAS SCALE dataset exported over NFS; an NGINX reverse proxy on the same server terminates HTTPS for LAN/Tailscale browser access. This is a target configuration, not a claim of verified hardlink support. The first device workflow stages a layout for an Anbernic RG34XXSP running Knulli, then the user manually copies it to an SD card. Automated device exports remain later-scope.

Only files the user has the right to use may be ingested. ROMmap does not download or bundle copyrighted ROMs, BIOS images, DATs, or emulator cores.

## 2. Release boundaries

**MVP (M1–M3):** Linux native/container deployment, embedded database, scan/intake inventory, explicit copy import to managed pool, plain-file checksums, one documented configurable layout with a Knulli-oriented staging example validated on the intended device before claiming compatibility, dry-run and safe hardlink reconciliation, CLI and REST job/status interface. A minimal GUI for setup, inventory, plan review, and job status is M4, not required for the headless MVP.

**Later (M5+):** DAT import and matching, additional validated presets, ZIP/7z member inspection, multi-file game-aware grouping, device copy/sync exports, and optional browser emulation. These are not prerequisites for a working hardlink mapper. Archive files may be imported as opaque assets in the MVP; no claim is made about matching their contents. No automatic extraction or archive rewrite in the MVP.

## 3. Functional requirements

### Storage and intake [REQ-STO]
- **REQ-STO-01 (MUST, MVP):** Validate each pool/target pair by creating a temporary test file in the pool and hardlinking it into the target parent (then remove both). Fail before mapping if unsupported, cross-device, or unwritable. Validation must use the rootless container-visible NFS paths in the first deployment; one share/dataset is necessary but does not prove that export permissions and mount behavior support links. Existing media applications successfully using hardlinks on the NAS are encouraging operational evidence, not a substitute for this probe.
- **REQ-STO-02 (MUST, MVP):** Source scanning is read-only and resumable; it records path, size, modification time and errors. Ignore symlinks by default, never traverse outside configured roots, and report skipped files. Import into the pool requires an explicit operation with a preview of copies, existing assets, and conflicts. Never silently delete intake files or duplicates.
- **REQ-STO-03 (MUST, MVP):** Stream SHA-256, SHA-1, MD5 and CRC32 for plain files in one pass. SHA-256 plus size identifies candidate duplicates; verify before reusing a pool asset. SHA-1/MD5/CRC32 are retained for future DAT compatibility, not asserted collision-proof. Detect a source that changes during hashing/import and retry or report failure.
- **REQ-STO-04 (MUST, MVP):** Import defaults to copy via a temporary file in the pool, checksum verification and atomic placement; rerunning an import must be safe. Duplicate content may map multiple source paths to one pool asset, without altering source paths. A file that differs at the requested destination is a conflict, never overwritten implicitly.
- **REQ-STO-05 (MUST, MVP):** ROMmap never edits pool file contents through a target. Target files are created read-only where supported; the UI warns that filesystem permissions cannot guarantee immutability against privileged users or other writers. On integrity audit, report external modifications and halt affected reconciliations until resolved. Removing the pool name while links exist does not reclaim the inode; document this behavior.

### Mapping [REQ-MAP]
- **REQ-MAP-01 (MUST, MVP):** Provide a versioned YAML profile schema for selecting assets and generating relative paths, including `{platform}`, `{title}`, `{region}`, `{checksum_short}` and original extension. Missing metadata must have documented defaults or exclude the asset with a reason. At least one example layout is shipped and tested against sample input; do not claim general device compatibility from a folder name alone.
- **REQ-MAP-02 (MUST, MVP):** Profile evaluation produces a deterministic dry-run plan (create, retain, remove, conflict, skip) with counts and affected paths. Reject absolute paths, traversal, separators inside tokens, reserved/invalid names, excessive path lengths and case-fold collisions as appropriate for the destination. Never allow the pool, target or intake roots to overlap unsafely.
- **REQ-MAP-03 (MUST, MVP):** Apply is idempotent and recoverable after interruption. Only remove/replace links recorded as ROMmap-owned **and** verified to still reference the expected asset; never delete unknown files. Unexpected contents at a desired path are conflicts. Record progress, failures and the last successful profile/inventory revision. Concurrent jobs touching the same target are prohibited.
- **REQ-MAP-04 (MUST, MVP):** Distinguish `synced`, `pending`, `drifted`, `conflict` and `unknown` based on desired-plan revision and a filesystem check; a database row alone does not prove a link still exists. No unverified claim of checksum-perfect sync.
- **REQ-MAP-05 (MUST, later):** A copy/sync device export is a distinct target type with explicit space estimates, overwrite policy and destination validation. Do not call such exports hardlink or zero-copy targets.

### Metadata and formats [REQ-DAT]
- **REQ-DAT-01 (MUST, later):** Import versioned DAT entries with source/provider, system and hashes. Preserve conflicting or multiple matches, provenance and unmatched entries; do not collapse them to a single asset-to-game foreign key. Specify supported DAT dialects and fixtures before claiming No-Intro, Redump or TOSEC compatibility.
- **REQ-DAT-02 (MUST, later):** Distinguish physical files, archive members, releases/games, and groups such as cue/bin and multi-disc sets. ZIP/7z checksums refer to uncompressed members only when members were actually streamed; archive-byte hashes are separate. Bound extraction time, member count, sizes and expansion ratio. Never claim inspection from headers alone.

### API, operations and security [REQ-OPS]
- **REQ-OPS-01 (MUST, MVP):** Core functions run without the GUI on Linux and in a container. CLI and versioned REST API expose configured-root scan, import preview/apply, inventory, profile validation, target plan/apply, job status and health. Long operations return a job ID; document errors, cancellation/retry behavior and API request/response schemas.
- **REQ-OPS-02 (MUST, MVP):** The service binds to loopback by default (including when proxied by colocated NGINX). If network exposure is enabled, require authentication for all mutating and file-revealing endpoints; do not assume a home LAN or Tailscale alone is authorization. The M4 browser workflow requires a single-admin login, even when HTTPS terminates at NGINX; exact authentication design is an M0 ADR decision. Configure allowed intake/pool/target roots at startup; API clients cannot supply arbitrary host paths. Recheck canonical containment when accessing paths, avoid symlink escapes/races and run with least privilege.
- **REQ-OPS-03 (MUST, MVP):** Persist job outcomes and structured errors; provide logs with no secrets, `/health` and `/ready` endpoints, database migration/version handling and backup/restore instructions. Database, temporary files and configured mounts survive container restarts.
- **REQ-OPS-04 (MUST, M4):** GUI supports setup validation, scan/import review, inventory, target plan preview, apply and job/error inspection through the same API. No GUI-only operation.
- **REQ-OPS-05 (MAY, later):** Browser emulation is an optional isolated feature with separately defined core licensing, supported formats/BIOS, resource limits, access control and ROM transfer policy. A playable preview is not DAT validation.

### Telemetry [REQ-MET]
- **REQ-MET-01 (MUST, MVP):** Show pool logical bytes (sum of unique managed asset sizes), managed target logical bytes (sum of link sizes, counting each target placement) and **avoided copy bytes** (managed target logical bytes, relative to independently copying every mapped file). Do not subtract the entire pool from target bytes. Label all three as logical estimates, not measured physical blocks; compression, sparse files, snapshots and filesystem overhead can change physical consumption. Avoided-copy accounting applies only to verified hardlinks.
- **REQ-MET-02 (MUST, M4):** UI labels use the same definitions as the API and display unavailable/partial values when a verification or scan is incomplete.

## 4. Acceptance gates

A milestone is complete only when the associated requirements have automated tests and documented manual verification steps. At minimum, test: same-filesystem and cross-mount hardlink attempts; duplicate and changed-during-import sources; identical reruns; restart halfway through reconciliation; externally deleted/modified links; unknown files in targets; path traversal and symlink escape attempts; collisions after sanitization and case folding; read-only intake; non-admin container UID; empty collection; and backup/restore of a populated database. Use fixture files created by tests, not distributed ROMs.

Open design choices and their owners are tracked in `ROADMAP.md`. Any later-scope requirement needs its own acceptance tests before it is promoted to MVP.
