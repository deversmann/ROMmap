# Setup and first-run workflow (M4 UI proposal)

This is a UI proposal implementing `REQ-OPS-04`, `REQ-STO-01..04` and `REQ-MAP-01..03`. `REQUIREMENTS.md` wins if behavior here conflicts with it. All actions use the public API in `ARCHITECTURAL_SPEC.md`; no GUI-only filesystem access. The wizard can be resumed after a restart. No step implies an intake file was moved or deleted.

## Step 1 — Configure and validate storage

```text
ROMmap / Setup                         1 of 4: Storage
Intake (read-only source):             [ intake1        ]
Managed pool:                          [ pool1          ]
Local target root:                     [ targets1       ]
[ Check permissions and hardlink support ]
Result: pool1 → targets1: supported (link probe passed)
        intake1: readable; pool and targets writable
[ Continue ]
```

These are **server-configured roots**, not arbitrary browser-entered host paths. If roots are not configured, show container mount examples and a link to admin setup, not a host filesystem browser. A failed probe explains cross-mount, permissions or read-only errors and blocks target creation. The UI does not claim that two paths on the same physical disk must work.

## Step 2 — Scan and review import

```text
ROMmap / Setup                         2 of 4: Inventory
[ Scan intake1 ]   Job: running (files 820/1,400; errors 2)
Found: 790 plain files; 8 opaque archives; 22 candidate duplicates
Skipped/changed: 2 [Inspect errors]
[ Review copy import ]
Copies: 768 | Reuse existing pool assets: 22 | Conflicts: 0
Estimated additional logical pool bytes: 12.4 GB
[ Import selected files ]  [ Skip for now ]
```

Show a durable job ID and per-file errors; counts are examples, not guaranteed outcomes. Explain that copying is the default, sources remain untouched, and archives are opaque in the MVP. Import apply uses a reviewed plan and can be retried after interruption.

## Step 3 — Create a local target and review plan

```text
ROMmap / Setup                         3 of 4: Local layout
Profile: [ simple-local-layout v1 ]   Target: [ targets1/console ]
[ Validate and preview ]
Create 610  Retain 0  Remove 0  Conflicts 2  Skip 12
Example: snes/Example Game (Unknown).sfc → hardlink to pool asset
Conflicts: [View paths and resolution guidance]
[ Apply safe operations ]  [ Back ]
```

Only choose a validated target under a configured root. Show the path examples and why each skip/conflict occurred. Unknown files are preserved; conflicts cannot be resolved by silently overwriting them. The UI must not advertise handheld-device deployment as hardlink-compatible.

## Step 4 — Verify and hand off

```text
ROMmap / Setup                         4 of 4: Status
Apply job: succeeded; 610 verified hardlinks; 12 skipped
Target state: synced at profile revision 1, verified just now
[ Open dashboard ]  [ View audit ]
```

If the job is partial or failed, show the last completed action, errors and a safe retry path instead of a success screen. Incomplete scan/import is also visible on the dashboard. DAT upload, device copy/sync and browser game previews are later workflows, not onboarding dependencies.
