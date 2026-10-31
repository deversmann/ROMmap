# Agent operating rules for ROMmap

These instructions apply to contributors and coding agents working in this repository. Do not treat a conversation, issue comment, or generated summary as more authoritative than the committed project documents.

## Sources of truth

1. `docs/REQUIREMENTS.md` defines behavior, release scope, requirement IDs, and acceptance criteria. It is authoritative if documents conflict.
2. `docs/ARCHITECTURAL_SPEC.md` describes the current implementation design. Accepted decisions live in `docs/adr/`; update the architecture when an accepted ADR changes it.
3. `docs/ROADMAP.md` defines milestone deliverables and exit gates.
4. GitHub issues are the work queue and source of task status, assignment, discussion, and evidence. Pull requests are the review record. No parallel `docs/tasks/` files or hand-maintained status ledger.

If instructions conflict, stop and flag the discrepancy before implementing. Never quietly weaken a MUST or an acceptance gate to make a task pass.

## Resuming a session

- Check `git status`, current branch and remote tracking before changing files. Inspect open GitHub PRs and the M0 milestone-gate issue (#7); an unmerged PR is proposed work, not the state of `main`.
- Read the project status/next action in `README.md` and `docs/ROADMAP.md`, then the linked GitHub issue and any accepted ADRs. Check issue comments and PRs for newer decisions; if the docs are stale, propose a reviewed update instead of guessing.
- As of the M0 planning handoff, the next decision is stack selection in issue #1 (ADR-0001); PR #8 proposes the latest deployment-context documentation and must be reviewed before relying on it as merged guidance. After that PR merges, use the merged roadmap and issue statuses rather than treating this historical note as a permanent task assignment.
- If GitHub is unreachable, report which PR/issue state could not be checked and avoid claiming a task was completed or a decision accepted.

## Before starting work

- Read the linked issue, its requirement IDs, relevant architecture sections, accepted ADRs, and related code/tests.
- An issue is ready only if it states an outcome, scope and exclusions, acceptance checks, requirement IDs, dependencies, and a `milestone:M0`–`milestone:M6` label (or `milestone:later`). If not, clarify it first. Use `.github/ISSUE_TEMPLATE/00-task.yml` for new tasks. One issue should normally fit in one reviewable PR.
- Comment briefly on the issue with the intended approach, tests, and assumptions. Claim the issue or confirm ownership before changing shared files. Do not start dependent work while its decision is merely proposed.
- Work in a branch named `issue-<number>-<short-topic>`; do not commit directly to the protected default branch. Check the working tree before editing and do not discard changes you did not make.

## During implementation

- Keep changes within issue scope. Add automated tests for acceptance checks, including failure modes; use synthetic fixtures, not distributed ROMs or BIOS images.
- For filesystem operations prioritize non-destructive behavior: read-only intake scan, reviewed import, safe target dry-run, verified owned-link removal only. Stop for review on any change to deletion, overwrite, path authorization, authentication, or data migration semantics.
- If a significant, difficult-to-reverse choice is needed, propose a new numbered ADR from `docs/adr/0000-template.md`; request human acceptance before implementing dependent work. Do not edit an accepted ADR to erase its rationale; supersede it with a new ADR. An ADR is not a substitute for updating the current architecture.
- If new findings change product behavior, propose edits to `docs/REQUIREMENTS.md` explicitly and obtain review. Keep implementation docs and tests aligned with accepted changes.
- Run relevant tests and record commands/results. If tests cannot run, say why and leave the issue open. Do not claim a platform or filesystem is supported without evidence.

## Handoff and review

- Open a PR linked with `Closes #<issue>` (or `Refs #<issue>` for partial work). PR description must name requirement IDs, summarize design and safety impact, list test commands/results, and note limitations, migrations, and manual verification. Do not close an issue for partial work.
- Reviewers check the diff against the issue, requirements, acceptance tests, safety boundaries, and ADRs. CI and human review are required before merge; no self-approval by the implementing agent. The project owner reviews the GitHub PR, requests changes if necessary, and merges when satisfied. The agent must not merge its own PR unless explicitly authorized for that specific PR.
- On blocked/interrupted work, leave an issue comment with current state, attempted checks, remaining risks, and the next concrete action; use label `status:blocked` as appropriate. On completion, record evidence in the PR/issue and remove transient status labels. Closing the issue follows merge and verified acceptance, not merely code generation.
- Milestone completion requires the exit gate in `docs/ROADMAP.md` to pass end-to-end. Record the gate outcome in a dedicated milestone-gate GitHub issue, linking tests and remaining blockers. A list of closed tasks alone does not complete a milestone.

## GitHub conventions

- Use one `milestone:M0` … `milestone:M6` or `milestone:later` label per issue; the milestone labels are the scheduling index. `docs/ROADMAP.md` remains the authoritative scope/timeline. Do not also maintain a separate GitHub Milestone schedule unless the team explicitly chooses it.
- Suggested other labels: `type:task`, `type:decision`, `type:bug`, `status:blocked`, `area:ingest`, `area:mapping`, `area:api`, `area:ui`, `area:ops`. Labels are filters, not substitutes for issue contents.
- An ADR PR should link its `type:decision` issue. Once accepted, link that ADR from dependent issues. GitHub discussion is useful context; the accepted, versioned decision must be in git.
- If GitHub or its CLI is unavailable, do not invent issue numbers or claim an issue was created/closed. Draft the proposed issue text locally or in the response and ask for access; keep the git history usable independently of GitHub.
