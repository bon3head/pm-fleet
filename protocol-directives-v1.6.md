# PM Fleet Protocol Directives v1.6

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
[v1.3→v1.4] Codeberg publication standard added to rule 7: the checkable
definition of a leak-free Codeberg repo.
[v1.5→v1.6] Transport decision (operator 2026-10-06): SSH-only mandate
replaced by a property-based rule — no credential in the URL or the repo,
auth from a machine-local secret store; default is HTTPS + credential helper
(DECISION: HTTPS is the only verified Dolt transport, V3). SSH stays as the
pending alternative (U1/U2 verify before mandating). E2E-1 (bd-layer
end-to-end, local): U3 workflow v2 — `bd config set sync.remote
<private-url>` so fresh clones bootstrap from it; plain push/pull honor
sync.remote with no origin re-adoption; plain bootstrap on a fresh clone
without sync.remote creates a divergent empty db (footgun, closed by the
sync.remote step). Grade's mechanical fixes: Codeberg checks 4–5 dropped
(covered by 1–3), anonymous ls-remote cross-check labeled UNKNOWN,
leak-audit scope noted. OPEN, not in v1.6: rule 6 vs description-is-public
(decision-bead publication call belongs to the operator), failed-pull
recovery path, cross-machine claim gap, Codeberg containment-vs-cleanup
wording.

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
  `push --yes` adoption path. U3 WORKFLOW v2 (E2E-1, 2026-10-06): `bd init`,
  then `bd dolt remote remove origin`, then
  `bd dolt remote add <name> <private-url>`, then
  `bd config set sync.remote <private-url>`, then commit and push the working
  copy. Fresh machine: `git clone` + `bd bootstrap` — bootstrap's first
  auto-detect branch clones the database from sync.remote. Ongoing: plain
  `bd dolt push` / `bd dolt pull` honor sync.remote; no --remote flag needed.
  Without the sync.remote step, a fresh clone's plain bootstrap creates a
  DIVERGENT EMPTY database (E2E-1 footgun). With sync.remote unset and no
  Dolt remote named "origin", plain push FAILS SAFE ("remote 'origin' not
  found") — no leak, but no push either. Never rely on BD_NO_REMOTE_ADOPT
  alone. NEVER `bd dolt push --yes` where the git origin is public: the
  adoption path commits `sync.remote` into `.beads/config.yaml` (E2E-1).
- [v1.6] Git auto-commit behavior (E2E-1): `bd init` and
  `bd dolt remote remove origin` auto-commit to git; `bd config set` does
  NOT — commit it manually before pushing the working copy.
- [v1.2] Conflict monitor design: on embedded storage, `bd dolt pull` never
  leaves conflicts behind — a delete-vs-edit conflict aborts the pull with
  "merge conflicts in issues require operator resolution; merge aborted and
  working set restored", and `bd conflicts` stays empty afterward. The
  "alert when `bd conflicts` is non-empty after pull" condition appears
  unreachable via the pull path. Watch the pull abort error line instead.
  Same-field concurrent edits are SILENT LOST UPDATES (last-write-wins) with
  a notice and no conflict reported — one value disappears (U9 PARTIAL;
  non-empty conflicts format still uncaptured). The protocol's answer is
  claim-before-mutate (rule 4), not conflict detection.
- [v1.5] `bd init` FREEZE: run no new `bd init` until the Agent Skills
  package ships. Every `bd init` adopts a public origin by default (rule 2,
  U4), and agents do not read these directives — the safe workflow currently
  exists only on paper. The Skills package pre-flight (rule 9) is the
  enforcement point.

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
  [v1.5] This is INTENDED behavior, not a bug: beads are the system of
  record (rule 1). GitHub-side edits are conveniences that the next bead
  change discards. Do not build workflows that depend on GitHub-side state
  surviving.
- [v1.5] U6/U7 were exercised on bd 1.3.1 only. Re-test both whenever the
  bd pin changes — the published surface and push-only semantics are
  version-dependent.
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
- [v1.5] Description is PUBLIC under U3+U6: in a public working copy every
  bead description and label publishes to GitHub. Decision-bead rationale
  (rule 6) sitting in a description is published rationale. D7 covers
  receipts (notes field); it does not cover descriptions. Keep sensitive
  reasoning in notes.

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
- [v1.5] CLAIM-BEFORE-MUTATE: in any multi-writer repo, `bd claim` the bead
  before mutating it. Same-field concurrent edits are silent lost updates
  (rule 2) — the claim lease is the protocol's only coordination primitive.
  An edit to an unclaimed bead in a shared repo is a protocol violation.

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
- [v1.5] A failed `bd dolt pull` or `bd dolt push` means NOT LANDED. A
  delete-vs-edit abort (rule 2) leaves the machine unable to sync; the
  bead's state must not be treated as shared until the push succeeds.
  Surface the failure to the operator — never retry silently into a
  divergent history.
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
[v1.5] WIDENED: no public remote EVER carries refs/dolt/* or
refs/heads/__dolt_remote_info__, on ANY copy path — migrate UI, --mirror,
pull mirrors, or any future mechanism. The U5 migrate ban was one instance
of this rule.
[v1.6] PROPERTY-BASED TRANSPORT RULE (DECISION, operator 2026-10-06): no
Dolt remote URL carries a credential — no `https://user:token@host` ever.
Auth comes from a machine-local secret store (credential helper, SSH key,
0600 netrc), never from the URL, never from the repo. Default transport is
HTTPS + credential helper: HTTPS is the only verified Dolt transport (V3,
GitLab), and HTTPS keeps rule 9's anonymous ls-remote cross-check possible.
SSH is the pending alternative — unverified for Dolt on every forge
(U1/U2); verify before mandating. (Replaces the v1.5 SSH-only mandate,
which was an error: it mandated the one transport with zero verification.)
The URL lives in gitignored `.beads/embeddeddolt/` (verified bd 1.3.1),
BUT the push-time adoption path commits `sync.remote` into the tracked
`.beads/config.yaml` (E2E-1) — so the no-credential-in-URL rule is
load-bearing even with the gitignore. `__dolt_remote_info__` is written by
`bd dolt push` to the Dolt remote — safe while that remote is private, which
is exactly what this rule enforces.
[v1.2] For public working copies: `bd init`, then
`bd dolt remote remove origin`, then `bd dolt remote add <name> <private-url>`
(see rule 2 — BD_NO_REMOTE_ADOPT=1 is insufficient).
[v1.3] Codeberg's "Migrate repository" / clone-from-GitHub option copies ALL
refs, VERIFIED (U5): a migrated repo carries refs/dolt/data and
refs/heads/__dolt_remote_info__ to Codeberg — invisible on the branches page,
present in ls-remote. Never use it on a repo with Dolt data. If it was used,
delete the refs from the Codeberg repo afterward:
`git push <codeberg-remote> :refs/dolt/data :refs/heads/__dolt_remote_info__`.
[v1.6] SUPERSEDED by the property-based transport rule above: the v1.5
SSH-only mandate was an error (it mandated the one transport with zero
verification). The `__dolt_remote_info__` note stands: written by
`bd dolt push` to the Dolt remote — safe while that remote is private, which
is exactly what this rule enforces.
[v1.5] Integration secrets (github.token, linear.api_key, etc.) go in ENV
VARS, never `bd config set`. `bd config set` writes them into the committed
`.beads/config.yaml` (the file's own comments warn of git exposure). The
crawler scans the committed config.yaml for secret keys on every run.
[v1.6] Leak audit is SCHEDULED, not one-off. First run 2026-10-05 covered
one repo: github.com/bon3head/pm-fleet clean (no refs/dolt/*, no
__dolt_remote_info__, only HEAD and main) — a single-repo result, not a
fleet result. Every public remote in the visibility manifest (rule 9) gets
checked on the schedule.

[v1.4] Codeberg publication standard. A Codeberg repo counts as published
only when every check below passes. Verify with `git ls-remote` — never the
Codeberg UI branches page, which hides non-branch refs (U5 proved the leak is
invisible there). Re-run the check after every publish.
1. `git ls-remote <codeberg-url>` shows no `refs/dolt/*` lines.
2. No `refs/heads/__dolt_remote_info__` line.
3. Only intended refs exist: `refs/heads/<branches>`, `refs/tags/<tags>`, HEAD.
   (Prose until the manifest lists allowed refs per repo — rule 9.)
4. Ongoing publishing uses an explicit refspec only, e.g.
   `+refs/heads/main:refs/heads/main +refs/tags/*:refs/tags/*`.
[v1.6] Checks 4–5 from v1.4 (migrate-history, --mirror-history) dropped:
they are about history and checks 1–3 already verify current state; the
migrate remediation stays in the [v1.3] paragraph above. A Codeberg repo
failing any check is not published — fix the refs first.

## 8. No merge queue (DECISION D2, operator)

Agents never merge to the same main concurrently — a stated operating
constraint, not a verified fact; nothing measured it. If it stops holding,
use a host-agnostic "rebase on latest main, re-run the gate, fast-forward"
script. No merge-queue machinery.

## 9. Remote visibility manifest (DECISION, operator 2026-10-05)

Git has no concept of remote visibility, and rule 7 depends on knowing it.
So visibility is DECLARED, in a versioned manifest owned by the operator,
and every remote operation is checked against it. The manifest lives in the
fleet repo (fleet-remotes.yaml). Format:

```yaml
remotes:
  - repo: bon3head/pm-fleet
    host: github
    name: origin
    url: https://github.com/bon3head/pm-fleet.git
    visibility: public      # public | private
    purpose: working-copy   # working-copy | bead-store | publication
    dolt: false             # true if this remote carries refs/dolt/*
```

Enforcement is mechanical, three layers:

1. Crawler, every run: list actual git remotes + Dolt remotes per repo.
   Every actual remote must have a manifest entry. Violations: a remote
   with no entry; a public remote carrying refs/dolt/* or
   `__dolt_remote_info__`.
   [v1.6] Anonymous ls-remote cross-check is UNKNOWN (untested): the probe
   needs empty HOME, `credential.helper=`, `GIT_TERMINAL_PROMPT=0`, HTTPS
   URLs only — SSH remotes cannot be probed, and credential helpers / netrc
   / GCM / ssh-agent can authenticate silently. A failed probe does not
   prove the remote is private. Do not rely on it until exercised.
2. Skills pre-flight: before `bd init`, `bd dolt push`, `bd dolt pull`,
   or any publish step, the agent reads the manifest and confirms the
   target remote's visibility and purpose. No manifest entry = stop, do
   not proceed, surface to the operator.
3. Scheduled leak audit (rule 7) walks every manifest entry.

Rules for the manifest itself: one entry per (repo, remote name); `dolt:
true` requires `visibility: private`; `purpose: bead-store` requires
`dolt: true`; changing a remote's visibility is a manifest commit reviewed
like code. A remote not in the manifest does not exist as far as the fleet
is concerned.
