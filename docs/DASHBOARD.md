# Dashboard and inventory (M4 UI proposal)

Implements `REQ-OPS-04`, `REQ-MET-01..02`, `REQ-MAP-04`. See `WIREFRAME_SETUP.md` for onboarding. Values below are illustrative; the requirements and API contract define behavior. No browser emulator is included in M4.

```text
ROMmap       Dashboard | Inventory | Targets | Jobs | Settings

COLLECTION AND LAYOUTS (logical estimates, not measured disk blocks)
Managed pool assets                 242.5 GB
Managed target placements           966.7 GB across 4 local layouts
Avoided copy bytes                  966.7 GB vs copying each placement
Verified hardlinks                  14,022   checked 5 minutes ago
[ How these numbers are calculated ]

TARGETS
Knulli-like local layout   pending: 2 new assets        [ Plan changes ]
  targets1/knulli          last verified 5 days ago
Pegasus example layout    conflict: unknown file       [ Inspect conflict ]
  targets1/pegasus         last checked just now

JOBS & ALERTS
Import job #42            succeeded                    [ Details ]
Apply job #43             failed: permission denied    [ Retry plan ]
Pool audit                not run                       [ Start audit ]
```

Use responsive navigation and accessible text in addition to colors/icons. Show `unknown` or `partial` instead of a number when verification is incomplete; old timestamps remain visible. Avoid a `synced` badge based solely on the DB. A target marked `pending` has a changed desired revision; `drifted` means a link is missing/changed; `conflict` means safe reconciliation is blocked. Clicking a target opens the create/retain/remove/conflict/skip plan, with explicit review before apply.

## Inventory

```text
Search assets [________]  Platform [all]  Import status [all]
Filename                 Platform  SHA-256 prefix  Match/status
Example Game.sfc         snes      a2c1b4d0       imported, unverified metadata
Disc 1.bin              unknown   fe41a201       missing platform tag
Collection.zip          unknown   88bc1299       opaque archive
[ Open details ]  [ Tag platform ]  [ Scan intake ]
```

Inventory distinguishes a physical asset from a recognized game. Unknown metadata is visible and editable through documented API operations; imported bytes are not modified by metadata changes. DAT matching may be added in M5, with provenance and multiple matches displayed instead of assuming every file has one canonical title.

## API integration

The dashboard reads `GET /api/v1/telemetry/storage`, `GET /api/v1/targets`, `GET /api/v1/assets` and `GET /api/v1/jobs/{id}`; it creates target plans and applies reviewed plans via the target endpoints in `ARCHITECTURAL_SPEC.md`. Browser previews and ROM-byte delivery are out of scope; see `REQ-OPS-05` for a separately gated future investigation.
