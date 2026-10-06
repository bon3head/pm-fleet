# PM Fleet Protocol Directives v1.3

Canonical agent-facing rules for the fleet. Re-issued 2026-10-05 with U-batch
corrections (marked [v1.2]). Labels: VERIFIED = exercised in
verification-evidence.md (V1–V5 + U-batch, 2026-10-05); DOCUMENTED =
primary-source read in the audit, not exercised; DECISION = an operator
choice, not a fact; UNKNOWN = no evidence. This file is the source of truth
that the Agent Skills package and per-repo AGENTS.md sections are generated
from. When a rule changes, change it here first.

Pins: `bd` 1.3.1, `isitdone` 0.8.2. [v1.1→v1.2] The metrics notice is VERIFIED
real (U12): printed on stderr at the first stateful run (`bd init`), not on
`--version`. First run also writes `~/.beads/machine-id` + `eventsData/`.
`bd metrics off` works and persists. Run it before fleet adoption.
[v1.2→v1.3] U5 closed with a confirmed leak: Codeberg's migrate/clone-from-GitHub
copies all refs including refs/dolt/data — rule 7 now carries the concrete
prohibition.

## 1. Beads are the system of record (VERIFIED, V4)

GitHub Issues is a read-mostly publication target, not a second database.
`bd github sync` is a bead-led mirror, not true bidirectional sync.

## 2. Canonical sync: dolt push / pull (VERIFIED GitLab/HTTPS; DOCUMENTED design elsewhere)

- Push bead state: `bd dolt push` (adopts the Dolt remote from git origin).
- Fresh machine or fresh clone: `bd bootstrap`. NEVER `bd init` followed by
  `bd dolt pull` — that creates a divergent history; bootstrap auto-detects
  `refs/dolt/data` on origin and clones it. (VERIFIED, V3)
- Incremental `bd dolt pull` is for already-bootstrapped machines.
- [v1.2] "HTTPS with a token works identically to SSH" is UNPROVEN:
  HTTPS+token verified on GitLab only; SSH untested on every forge; Codeberg
  untested on both transports. Treat transport choice as per-machine, not
  as an established equivalence.
- Requirement: every fleet machine needs git user.name/user.email set, or
  bd's commit of .beads/config.yaml fails (exit 128, V3).
- [v1.2] Non-origin Dolt remotes (for public working copies): `bd dolt remote
  add <name> <private-url>` + `bd dolt push --remote <name>` VERIFIED working;
  refs/dolt/data lands on the named remote, origin untouched. BUT
  `BD_NO_REMOTE_ADOPT=1` does NOT prevent init-time adoption — plain `bd init`
  in a repo with a git origin always adopts origin as the Dolt remote (3
  trials), contradicting the help text; it only blocks the later
  `push --yes` adoption path. Correct workflow for a public working copy:
  `bd init`, then `bd dolt remote remove origin`, then
  `bd dolt remote add <name> <private-url>` — or init before the git origin
  exists. Never rely on BD_NO_REMOTE_ADOPT alone.
- [v1.2] Conflict monitor design: on embedded storage, `bd dolt pull` never
  leaves conflicts behind — a delete-vs-edit conflict aborts the pull with
  "merge conflicts in issues require operator resolution; merge aborted and
  working set restored", and `bd conflicts` stays empty afterward. The
  "alert when `bd conflicts` is non-empty after pull" condition appears
  unreachable via the pull path. Watch the pull abort error line instead.
  Same-field concurrent edits auto-merge (last-write-wins) with a notice;
  only delete-vs-edit aborts. (U9 PARTIAL — non-empty conflicts format still
  uncaptured.)

## 3. GitHub sync is publication-only (VERIFIED semantics, V4 + U6/U7)

Use `bd github sync` only to publish bead state to GitHub Issues — and only
in publication-safe invocations:

- `bd github sync --push-only` creates GitHub issues for beads.
- Plain `bd github sync` is NOT publication-only: under default prefer-newer,
  a newer GitHub-side edit is imported into the bead ("external is newer,
  importing"). Fix the invocation: `--push-only`, or `--prefer-local` if a
  full sync is ever run.
- Conflict flags behave as labeled: prefer-newer (default) lets the newer
  side win; --prefer-local overwrites GitHub with the bead; --prefer-github
  takes the GitHub version even when the bead is newer.
- [v1.2] --push-only vs GitHub-side edits (U7): when the bead is unchanged,
  push-only is a no-op and the GitHub edit is preserved; when the bead
  changed, push-only overwrites GitHub unconditionally (no conflict prompt).
- `bd close` on a bead closes the linked GitHub issue. One-way: reopening
  the GitHub issue does NOT reopen the bead.
- Unscoped pull does NOT import foreign GitHub issues. Explicit import only:
  `bd github pull <issue-number>`.
- `bd delete` on a bead does NOT close or delete the linked GitHub issue —
  the issue is orphaned (stays open). Close beads instead of deleting them
  when they have a GitHub mirror.
- [v1.2] Published surface (U6): title + description + labels only. Notes
  and acceptance criteria do NOT publish (issue body = description verbatim).
  Implication: fields that must not reach GitHub go in notes, never in the
  description.

## 4. Claim work with native leases (VERIFIED TTL/heartbeat/reclaim + claim race)

- `bd update <id> --claim`, `bd heartbeat <id>` from the agent loop,
  `bd reclaim --older-than <dur>` from a supervisor timer at ~2× TTL.
- Do NOT build a custom lease, heartbeat, or reaper subsystem.
- TTL (~5 min), heartbeat refresh, and reclaim with grace window VERIFIED
  (V2, one agent, one bead). [v1.2] Claim race VERIFIED (U8): two concurrent
  `--claim` invocations — the winner takes the lease (rc=0), the loser fails
  cleanly with "issue already claimed by <winner>" (rc=1); no steal, no
  silent double-win. Same-actor re-claim is idempotent.
- row_lock serialization is DOCUMENTED (audit, CHANGELOG) but unexercised
  beyond the two-actor race. Cross-machine claim exclusivity holds only
  after `bd dolt push` syncs the claim.

## 5. Done means gated; the plane lands with a receipt (VERIFIED gate behavior; DECISION on binding)

- `isitdone` runs as a local Stop-hook gate: an honest-mistake check, not
  adversarial evidence. HMAC uses a per-repo local key under gitignored
  `.isitdone/`; an agent that can edit settings can remove the hook.
  (D5 resolved by operator 2026-10-05: local gate only, no CI re-verification.)
- isitdone does NOT catch uncommitted files — V5's gate passed with 14 dirty
  files. "Land the Plane" needs its own clean-tree step: commit first, gate
  on a clean tree (dirtyFiles: 0), then record the commit SHA plus
  `isitdone receipt --json` on the bead.
- [v1.2] Receipt binding VERIFIED (U10): on a clean tree, receipt.tree ==
  `git rev-parse HEAD^{tree}` and receipt.head == HEAD with dirtyFiles: 0.
  The SHA+receipt pair binds committed state. Whether receipt.tree equals the
  committed tree on a dirty tree is UNKNOWN.
- A receipt on a bead is a record, not a proof: only the machine holding
  `.isitdone/key` can check the HMAC.
- Receipt exposure (D7, resolved by operator 2026-10-05): since notes do
  not publish to GitHub, the receipt goes in the bead's NOTES field, never
  in the description. Trimmed
  {version, tool, status, head, tree, dirtyFiles, hmac} only if a receipt
  must live in a published field.

## 6. Log deviations as decision beads (VERIFIED, V1)

"Built X, but the spec said Y" → `bd create --type decision --validate`,
with Decision, Rationale, Alternatives Considered enforced. Link to the
originating task with discovered-from (link type DOCUMENTED, not exercised).

## 7. Never publish bead data to a public remote (DECISION D3 + VERIFIED leak mechanism)

Pushing Dolt data to a public origin publishes the whole issue database.
Dolt remotes point at private remotes only. The rule is about visibility,
not host. DECISION (D3, operator): no bead data on public Codeberg.
[v1.2] For public working copies: `bd init`, then
`bd dolt remote remove origin`, then `bd dolt remote add <name> <private-url>`
(see rule 2 — BD_NO_REMOTE_ADOPT=1 is insufficient).
[v1.3] Codeberg's "Migrate repository" / clone-from-GitHub option copies ALL
refs, VERIFIED (U5): a migrated repo carries refs/dolt/data and
refs/heads/__dolt_remote_info__ to Codeberg — invisible on the branches page,
present in ls-remote. Never use it on a repo with Dolt data. If it was used,
delete the refs from the Codeberg repo afterward:
`git push <codeberg-remote> :refs/dolt/data :refs/heads/__dolt_remote_info__`.
Rule 7 is fully enforceable with this check in place.

## 8. No merge queue (DECISION D2, operator)

Agents never merge to the same main concurrently — a stated operating
constraint, not a verified fact; nothing measured it. If it stops holding,
use a host-agnostic "rebase on latest main, re-run the gate, fast-forward"
script. No merge-queue machinery.
