# PM Fleet Protocol Directives v1.1

Canonical agent-facing rules for the fleet. Re-issued 2026-10-05 to correct
v1.0 against the evidence (corrections marked [v1.1]). Labels: VERIFIED =
exercised in verification-evidence.md (V1–V5, 2026-10-05); DOCUMENTED =
primary-source read in the audit, not exercised; DECISION = an operator
choice, not a fact; UNKNOWN = no evidence. This file is the source of truth
that the Agent Skills package and per-repo AGENTS.md sections are generated
from. When a rule changes, change it here first.

Pins: `bd` 1.3.1, `isitdone` 0.8.2. [v1.1] The `bd metrics off` pin-hygiene
note is UNVERIFIED on the evidence (U12) — run one fresh `bd` and capture
output before asserting it.

## 1. Beads are the system of record (VERIFIED, V4)

GitHub Issues is a read-mostly publication target, not a second database.
`bd github sync` is a bead-led mirror, not true bidirectional sync.

## 2. Canonical sync: dolt push / pull (VERIFIED GitLab/HTTPS; DOCUMENTED design elsewhere)

- Push bead state: `bd dolt push` (adopts the Dolt remote from git origin).
- Fresh machine or fresh clone: `bd bootstrap`. NEVER `bd init` followed by
  `bd dolt pull` — that creates a divergent history; bootstrap auto-detects
  `refs/dolt/data` on origin and clones it. (VERIFIED, V3)
- Incremental `bd dolt pull` is for already-bootstrapped machines.
- [v1.1] "HTTPS with a token works identically to SSH" is UNPROVEN:
  HTTPS+token verified on GitLab only; SSH untested on every forge; Codeberg
  untested on both transports. Treat transport choice as per-machine, not
  as an established equivalence.
- Requirement: every fleet machine needs git user.name/user.email set, or
  bd's commit of .beads/config.yaml fails (exit 128, V3).

## 3. GitHub sync is publication-only (VERIFIED semantics, V4)

Use `bd github sync` only to publish bead state to GitHub Issues — and only
in publication-safe invocations:

- `bd github sync --push-only` creates GitHub issues for beads.
- [v1.1] Plain `bd github sync` is NOT publication-only: under default
  prefer-newer, a newer GitHub-side edit is imported into the bead
  ("external is newer, importing"). Fix the invocation: `--push-only`, or
  `--prefer-local` if a full sync is ever run. Whether `--push-only`
  overwrites or skips GitHub-side edits is UNKNOWN (U7).
- Conflict flags behave as labeled: prefer-newer (default) lets the newer
  side win; --prefer-local overwrites GitHub with the bead; --prefer-github
  takes the GitHub version even when the bead is newer.
- `bd close` on a bead closes the linked GitHub issue. One-way: reopening
  the GitHub issue does NOT reopen the bead.
- Unscoped pull does NOT import foreign GitHub issues. Explicit import only:
  `bd github pull <issue-number>`.
- `bd delete` on a bead does NOT close or delete the linked GitHub issue —
  the issue is orphaned (stays open). Close beads instead of deleting them
  when they have a GitHub mirror.

## 4. Claim work with native leases (VERIFIED TTL/heartbeat/reclaim; DOCUMENTED row_lock)

- `bd update <id> --claim`, `bd heartbeat <id>` from the agent loop,
  `bd reclaim --older-than <dur>` from a supervisor timer at ~2× TTL.
- Do NOT build a custom lease, heartbeat, or reaper subsystem.
- [v1.1] TTL (~5 min), heartbeat refresh, and reclaim with grace window are
  VERIFIED (V2, one agent, one bead). row_lock serialization is DOCUMENTED
  (audit, CHANGELOG) but unexercised; concurrent-claim behavior under load
  is UNKNOWN (U8). Cross-machine claim exclusivity holds only after
  `bd dolt push` syncs the claim.

## 5. Done means gated; the plane lands with a receipt (VERIFIED gate behavior; DECISION on binding)

- `isitdone` runs as a local Stop-hook gate: an honest-mistake check, not
  adversarial evidence. HMAC uses a per-repo local key under gitignored
  `.isitdone/`; an agent that can edit settings can remove the hook.
- [v1.1] isitdone does NOT catch uncommitted files — V5's gate passed with
  14 dirty files. "Land the Plane" needs its own clean-tree step: commit
  first, gate on a clean tree (dirtyFiles: 0), then record the commit SHA
  plus `isitdone receipt --json` on the bead.
- [v1.1] A receipt on a bead is a record, not a proof: only the machine
  holding `.isitdone/key` can check the HMAC. Whether receipt.tree equals
  the committed tree of head is UNKNOWN (U10).
- [v1.1] Receipt exposure (DECISION D7, open): a receipt copied onto a bead
  travels to the Dolt remote and possibly to the GitHub issue, carrying
  branch names, check commands, and stdout tails. Default until D7 closes:
  full receipt only on unmirrored beads; on mirrored beads store the trimmed
  {version, tool, status, head, tree, dirtyFiles, hmac} and drop tails.

## 6. Log deviations as decision beads (VERIFIED, V1)

"Built X, but the spec said Y" → `bd create --type decision --validate`,
with Decision, Rationale, Alternatives Considered enforced. Link to the
originating task with discovered-from (link type DOCUMENTED, not exercised).

## 7. Never publish bead data to a public remote (DECISION D3 + VERIFIED leak mechanism)

Pushing Dolt data to a public origin publishes the whole issue database.
Dolt remotes point at private remotes only. [v1.1] The rule is about
visibility, not host: "the GitHub working copy or a private remote" is only
safe if the working copy is private — UNKNOWN (U3). DECISION (D3, operator):
no bead data on public Codeberg. Rule 7 is unenforceable until U3 (working
copy visibility), U4 (can sync.remote target a non-origin remote?), and U5
(does the GitHub→Codeberg publication path push all refs?) close.

## 8. No merge queue (DECISION D2, operator)

Agents never merge to the same main concurrently — a stated operating
constraint, not a verified fact; nothing measured it. If it stops holding,
use a host-agnostic "rebase on latest main, re-run the gate, fast-forward"
script. No merge-queue machinery.
