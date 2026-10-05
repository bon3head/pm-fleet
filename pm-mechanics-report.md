# Functionality Inventory: Git-Native PM Layer for Agent-Driven Development

> Research date: 2026-10-05. All sources read via search index / page fetch (no live-browser verification; flagged where it matters).
> Scope: concrete functional mechanics only. Tool selection (Beads as execution layer) taken as given. No UI/dashboard.
> Confidence key: **[primary]** = the component's own repo/docs; **[eco]** = ecosystem docs/skill files (real usage, not the vendor); **[3p]** = third-party extension/analysis (not core).

## Summary

The mechanics decompose into **23 capabilities**. Verdicts: **bolt on 13** (exact OSS components named below), **custom-build 7** (all small: configs, hooks, scripts, schemas — no new platforms), **killed 3** (spec-kit as acceptance checker, gitpulse as working-tree tool, auto-increment/UUID issue IDs). The single highest-leverage bolt-on is **isitdone** (MIT, 2026-09): a zero-LLM Stop-hook gate that runs real test/typecheck/lint on the working tree and emits tamper-evident done-receipts — it is the machine-enforced half of Beads' "Land the Plane." The merge-queue decision is the only stack-depth question: Gas Town Refinery (real, batch-then-bisect, part of the GT stack) vs bors-ng (standalone OSS) vs GitHub native merge queue (zero-install). Engram overlap is heaviest in **receipts and gate/auditor patterns**; genuinely new PM-layer territory is the **working-tree truth crawler, lease/reaper policy, Dolt merge-conflict monitoring, and the bead↔receipt binding schema**.

---

## 1. VERIFIED "DONE"

### 1a. Working-tree quality gate (pre-stop enforcement)
- **What:** When an agent tries to end its turn claiming completion, run the repo's *real* test, typecheck and lint commands on the *exact* working tree and refuse the stop until they pass. Also scans the diff for weakened tests (anti-gaming: agent cannot delete/weaken tests to pass).
- **Mechanism:** Agent-harness Stop hooks (Claude Code `.claude/settings.json` Stop; Codex `.codex/hooks.json` Stop; Cursor `.cursor/hooks.json`; Gemini `.gemini/settings.json` AfterAgent; Copilot CLI `.github/hooks/isitdone.json` agentStop; + Qwen, Goose, Droid, Devin, Augment, OpenCode, Junie). Exit-code driven, zero-LLM, zero-dependency. `claim-gated` profile: lite checks every stop, full checks when the agent claims done. Warn-only post-edit hook gives mid-turn nudges.
- **Bolt-on:** **isitdone** — https://github.com/raimondasl/isitdone — MIT, npm `@aivolution/isitdone` (alias `isitdone`), created 2026-09-07 **[primary]**. Install: `npx isitdone init` per repo; `doctor` self-tests with a synthetic blocked stop.
- **Receipt produced:** `DONE receipt -> PASS (tree e5e47d9, 3 checks, 1.8s)`; `isitdone receipt --md` emits a Markdown table (check/result/duration + `Tree <tree> on <branch>@<sha> (+N uncommitted). Receipt: PASS`). Binding: git tree hash + branch + commit + dirty count + check list + durations. Semantics: further edits → **STALE**; hand-edited → **NONE** (content-derived, tamper-evident).
- **Decisions log:** `.isitdone/decisions.jsonl` — one line per stop: outcome, whether the final message claimed completion, check counts; no message text, no paths, session id hashed (gitignored by installer).
- **Engram overlap:** HIGH — this is the *evidence producer* for Engram-style verification receipts. Engram's hash-chained receipts likely cover chaining/integrity; the PM layer still needs the bead↔commit↔gate-verdict binding (see 1f).

### 1b. Gate-suite definition (what "the gates" are per repo)
- **What:** The per-repo list of commands (build/test/lint/typecheck/security scan) that constitute "done."
- **Mechanism:** Auto-detected at `isitdone init` (e.g. `npm run typecheck`, `npm run lint`, `npm test`), overridable in `.isitdone.json`.
- **Custom:** small config file per repo. Trivy (security scan) bolts on as one more check command — https://github.com/aquasecurity/trivy (well-established; seen wired as a gate step in aegis-core ADR: https://github.com/binhsu/aegis-core/blob/HEAD/docs/adr/0029-trivy-cve-scan-and-slsa-provenance.md **[eco]**).
- **Engram overlap:** PARTIAL — maps to Engram's "gate packages" concept; the package *content* is per-repo custom.

### 1c. CI-side re-verification on pull requests
- **What:** Run the same gate suite on the PR/merge ref so a green working tree can't regress through merge.
- **Mechanism options:**
  - **isitdone GitHub Action** (same repo) — same checks on PR **[primary]**.
  - **GitHub native merge queue** (repo setting, GA) — revalidates each PR against latest target + queued-ahead changes on temp branches; failing entries removed, queue rebuilds. Explainer verified against docs: https://dev.to/pyor/github-merge-queue-explained-why-green-prs-break-main-2bk1 **[eco]**.
  - **bors-ng** — OSS GitHub App, Apache 2.0: queues approved PRs, tests batches on a staging branch, bisects failures, `bors r+`/`r-`/`try` commands, `bors.toml` config. https://github.com/bors-ng/bors-ng (cited verbatim in https://news.ycombinator.com/item?id=34712345) **[eco]**.
- **Bolt-on verdict:** GitHub native merge queue for the GitHub-main setup (zero infra); bors-ng if a self-hosted/equivalent queue is needed. isitdone Action if the receipt format must match the working-tree gate.
- **Engram overlap:** LOW — merge-queue mechanics are new PM-layer territory; Engram's gates are about approval, not integrated-test-before-merge.

### 1d. Build provenance attestations (SLSA)
- **What:** Prove "artifact X was built from commit Y by workflow Z" — binds artifacts to source, not tasks to done.
- **Mechanism:** `actions/attest` (current; `actions/attest-build-provenance` v4+ is a wrapper over it). Permissions `attestations: write` + `id-token: write`; Sigstore-signed (Fulcio OIDC + Rekor); uploaded to GitHub attestations API. Verify: `gh attestation verify <artifact> --owner <org>`; `cosign verify-blob-attestation`. https://github.com/marketplace/actions/attest-build-provenance **[primary]**.
- **Artifact format:** in-toto v1 Statement: `subject` (name + sha256), `predicateType: https://slsa.dev/provenance/v1`, `predicate.buildDefinition` (buildType, workflow repo/ref, resolvedDependencies with `git+https` URI + commit sha), `runDetails.builder.id = https://github.com/actions/runner` + invocationId/timestamps.
- **Killed confusion:** SLSA provenance does NOT prove acceptance criteria were met. It is bolt-on for supply-chain, not for task-done.
- **Engram overlap:** LOW-MEDIUM — provenance is orthogonal to Engram receipts; both are signed claims but about different subjects (artifact vs task outcome).

### 1e. In-toto step link metadata
- **What:** Signed per-step records of a pipeline: command run, input file hashes (materials), output file hashes (products), byproducts (exit code, stdout/stderr).
- **Mechanism / bolt-on:** **in-toto/in-toto** (`pip install in-toto`): `in-toto-run --step-name <n> --key <functionary-key> --materials <files> --products <files> -- <cmd>` → signed `<step>.link`; `in-toto-record start/stop` for multi-part steps; `in-toto-verify` checks layout signature + link signatures + material/product chain consistency. Spec: https://github.com/in-toto/specification/blob/HEAD/in-toto-spec.md **[primary]**.
- **Engram overlap:** MEDIUM — link metadata is the generic form of what Engram's verification receipts specialize; reuse the envelope, define a PM predicate (1f).

### 1f. Done-verification receipt schema (task-level, custom predicate)
- **What:** A signed, queryable record per completed bead: bead ID, commit SHA, acceptance criteria text, per-gate verdicts, deviations.
- **Mechanism:** in-toto Statement envelope with a **custom `predicateType`** (documented pattern: `https://claude-harness.dev/attestation/control-evidence/v1` carrying control inventory + verify outputs + gate verdict; subject = repo + gitCommit; integrity hash covers the predicate). https://github.com/cwijayasundara/claude_harness_eng_v5/blob/HEAD/docs/internal/compliance-increment.md **[eco]**.
- **Custom:** the predicate schema is genuinely custom (no OSS standard for "task acceptance verdicts"). The envelope, signing (Sigstore/cosign), and evidence production (isitdone) are bolt-ons.
- **Engram overlap:** HIGH — this is where Engram's hash-chained verification receipts plug in. If Engram receipts already chain {subject, predicate, prev-hash}, the PM layer's job reduces to defining the predicate fields and emitting one receipt per closed bead.

### 1g. Beads "Land the Plane" protocol
- **What:** Scripted session-end: (1) file beads issues for remaining work; (2) run quality gates if code changed (file P0 issues if broken); (3) `bd close --reason`; (4) `git pull --rebase` → reconcile → `bd dolt push` → `git push`; (5) `git status` clean check; (6) `bd ready --json` for next session. Invariant: "the plane hasn't landed until `git push` succeeds."
- **Mechanism:** Prompt-driven convention shipped in bd's managed AGENTS.md blocks. https://github.com/pablo305/beads/blob/HEAD/AGENTS.md **[eco]**.
- **Verdict:** convention, not a component. Its *enforcement* is 1a (isitdone Stop hook) + git hooks. Nothing to build except keeping the protocol text in sync with the actual commands (note: current protocol must say `bd dolt push`, not legacy `bd sync`).
- **Engram overlap:** LOW — session-close discipline is PM-layer process.

### 1h. Bors-style merge queue (Gas Town Refinery)
- **What:** Completed agent branches merge to main only through a batching + bisecting queue; agents never push to main or open PRs themselves.
- **Mechanism [primary]:** https://github.com/gastownhall/gastown/blob/HEAD/docs/design/architecture.md — "batch-then-bisect merge queue (Bors-style)… a core capability, not a pluggable strategy." Batch: rebase MRs A..D as a stack on main → test the tip → PASS: fast-forward all → FAIL: binary bisect on the midpoint to isolate the culprit. Gates (test/lint commands) pluggable per rig. Shipped v0.9.0 (2026-03-01): https://github.com/gastownhall/gastown/blob/HEAD/CHANGELOG.md.
- **Worker contract:** `gt done` → push branch → submit MR (`bd update --mr-ready`) → teardown worktree → idle. "You NEVER push directly to main. Do NOT create GitHub PRs either." https://github.com/gastownhall/gastown/blob/HEAD/internal/templates/polecat-CLAUDE.md.
- **Refinery patrol cycle:** inbox-check → queue-scan → process-branch (fetch+rebase) → run-tests → handle-failures (bisect, isolate, notify) → merge-push (`git push origin temp:main`) → notify (MERGED mail) → cleanup (close MR bead, delete remote branch). https://github.com/gastownhall/gastown/blob/HEAD/.claude/commands/patrol.md.
- **Bolt-on verdict:** real OSS (described as MIT in secondary comparison https://github.com/laurence-nz/superset-pwsh/blob/HEAD/apps/marketing/content/compare/superset-vs-gastown.mdx — license NOT primary-verified). Caveat: Refinery is a *piece of the Gas Town orchestration stack* (agent-driven patrols, mail, Dogs); extracting it standalone is unverified. For a solo dev on GitHub: **GitHub native merge queue** is the proportionate bolt-on; **bors-ng** the standalone OSS equivalent.
- **Engram overlap:** LOW — merge mechanics are new; Engram's auditor pattern could plug into the conflict-resolution step (Refinery spawns a fresh agent to re-implement on conflict).

### 1i. spec-kit acceptance-criteria checking — KILLED as a verification mechanism
- **What spec-kit actually is:** GitHub's spec-driven toolkit (https://github.com/github/spec-kit **[primary]**): `specify` CLI, templates (`spec.md`, `plan.md`, `tasks.md` under `specs/NNN-*/`), slash commands (`/speckit.constitution|specify|clarify|plan|analyze|tasks|taskstoissues|implement`), `/speckit.taskstoissues` converts tasks to GitHub issues.
- **Acceptance checking mechanic:** the spec template's "Review & Acceptance Checklist" is **Markdown verified by the agent reading it** — there is no automated acceptance-criteria evaluator in spec-kit. Do not bolt it on for verification.
- **Real bolt-ons for the underlying need:** `bd create --acceptance "..."` field + `bd lint` (presence check) + isitdone (executes the checks).

---

## 2. DEVIATION/DRIFT LOGGING ("built X but spec said Y" as queryable data)

### 2a. Decision record as a queryable issue
- **What:** First-class record of a deviation/decision with rationale, rejected alternatives, and links to affected work — SQL-queryable, not chat history.
- **Mechanism / bolt-on:** bd **`decision` issue type** (aliases `dec`, `adr`) **[primary schema]**. Conventions **[eco]**: structured description (`## Decision` / `## Rationale` / `## Alternatives Considered` / `## Affects`); `bd create --validate` rejects missing required sections; `bd lint` flags incomplete decision beads; supersede-don't-edit workflow; link work via `bd dep add <decision> <affected> --type related|relates-to|validates`; namespace IDs with `bd create --type decision --id adr-N --force`. Query: `bd list --type decision`. Sources: https://github.com/nguyensitrung/conductor-beads/blob/HEAD/.claude/skills/beads/commands/decision.md, https://github.com/srobroek/omp-plugins/blob/HEAD/beads/skills/adr/SKILL.md, https://github.com/voxpelli/claude-beads/blob/HEAD/skills/retrospective/SKILL.md.
- **Engram overlap:** LOW-MEDIUM — decision beads are PM-layer data; Engram's deterministic tracking may already log *that* a decision happened, but the structured ADR content + graph links are new.

### 2b. Deviation entry schema (what makes it useful later)
- **What:** The fields a "built X, spec said Y" entry needs for later queries ("show me all deviations from spec S", "why does module M differ from its plan").
- **Mechanism:** bead fields, no new component: `issue_type=decision`, `acceptance_criteria` (what done meant), `design` (what was actually built + why it diverged), `close_reason`, `labels` (e.g. `deviation`), `metadata` JSON (machine fields), `dependencies`: **`discovered-from`** edge to the originating task (exists exactly for "found while doing X"), `related` to the spec bead. All timestamped RFC3339 + `created_by` actor.
- **Custom:** the *convention doc* (which fields are mandatory for a deviation entry) + `--validate` enforcement. ~1 page of SKILL.md.
- **Engram overlap:** MEDIUM — if Engram's DVD doctrine already mandates structured records, this schema should be aligned with it rather than invented separately.

### 2c. ADR file mirror
- **What:** Render `docs/adr/*.md` from decision beads so non-beads readers (PR reviewers) can read decisions.
- **Mechanism:** MADR format (https://adr.github.io/madr/; template https://github.com/adr/madr; `adr-tools` by npryce/adr-tools) **[primary]** + a **custom pre-commit hook** that renders markdown from `bd list --type decision --json` (pattern documented in omp-plugins adr skill; hook exits 0 when bd absent).
- **Custom:** the render hook (~50 lines). The format is a bolt-on.
- **Engram overlap:** NONE new — file rendering is PM-layer glue.

### 2d. Signed deviation attestation
- **What:** Tamper-evident proof that a deviation was recorded and accepted (for audit).
- **Mechanism:** same custom in-toto predicate as 1f, with a `deviations[]` array. No extra component.
- **Engram overlap:** HIGH — identical shape to Engram verification receipts; likely the same machinery.

---

## 3. WORKING-TREE TRUTH (fleet-wide dirty/stash/worktree state)

### 3a. Per-repo state crawl
- **What:** For each of 40+ repos (and every linked worktree): staged/modified/untracked files, ahead/behind vs upstream, stash entries, worktree list with per-worktree dirtiness, branch↔bead attribution.
- **Mechanism — bolt-on = git itself:**
  - `git status --porcelain=v2 --branch` — machine-parseable; header lines `# branch.ab +<ahead> -<behind>`, `# branch.upstream`; per-path XY codes. The stable parse target.
  - `git stash list` — `stash@{n}` lines; count = parked work.
  - `git worktree list --porcelain` — `worktree <path>`, `HEAD`, `branch`/`detached`, `locked`/`prunable`. **Critical:** this shows worktrees but NOT their uncommitted files — the sweep must run `git status` *inside each worktree path*.
  - `git branch -vv` / `git for-each-ref` for ref state (needs prior `git fetch` for remote truth).
- **Crawler verdict: CUSTOM.** The pattern is proven by **mgitstatus** (https://github.com/fboender/multi-git-status — MIT, single-file shell: scans `.git` dirs to DEPTH, reports uncommitted/untracked/needs-push/needs-upstream/needs-pull/stash count, does NOT fetch by default) **[primary]**, but the author accepts no contributions and it's a human-readable poller — **copy the pattern, don't depend on the tool**. **gita** (https://github.com/nosarthur/gita — Python multi-repo CLI) **[eco]** is a candidate bolt-on for the sweep; its worktree/stash coverage is unverified — needs a live check before adopting. Honest build: ~100-line parallel crawler (`xargs -P`), emitting JSONL.
- **Engram overlap:** NONE — working-tree ground truth is genuinely new PM-layer territory.

### 3b. Poll scheduler + alert rules
- **What:** Periodic sweep + rules (unlanded work aging, stale worktrees, upstream advances, snooze).
- **Mechanism:** cron/systemd-timer sweep of 3a + a small rule engine. **gitvantage** (https://github.com/rocketraman/gitvantage **[primary]**, desktop app) is the reference for the *rule shapes* (per-repo/per-worktree aging, unlanded-work, reminder thresholds, snooze, upstream-advance notifications) — but it is a local desktop poller with no fleet API; use it as a spec, not a component.
- **Custom:** the scheduler + rules. Small.
- **Engram overlap:** NONE.

### 3c. Push-based state reporting
- **What:** Near-real-time updates instead of polling gaps (agents create and destroy worktrees between sweeps).
- **Mechanism:** (i) fail-open git hooks (`post-commit`, `post-checkout`, `post-merge`) emitting a porcelain summary to a collector; (ii) agent self-report — branch name + worktree path written to the bead via `bd update` at claim/close (Gas Town polecat template mandates persisting to the bead early/often because sessions can die: https://github.com/gastownhall/gastown/blob/HEAD/internal/templates/polecat-CLAUDE.md).
- **Tradeoffs:** POLL = no per-repo install, works on any checkout; stale between sweeps, `git fetch` per repo is slow/rate-limited (documented by mgitstatus), misses transient states, per-machine only. PUSH = near-real-time, catches short-lived states; hooks must be installed per repo *and* per linked worktree, bypassable (`--no-verify`), needs a receiver. **Recommended hybrid:** agents push coarse state to beads (authoritative "who is doing what") + poller sweeps porcelain truth (catches dead agents).
- **Custom:** hooks + receiver are custom. No OSS component found for agent-pushed working-tree truth.
- **Engram overlap:** NONE.

### 3d. gitpulse — KILLED
- No working-tree-truth tooling under this name exists. Search surfaced only GitHub activity/analytics dashboards (mobile app, Laravel app, portfolio trackers). Do not pursue.

---

## 4. AGENT TASK PROTOCOL (claim / lease / advance without collision)

### 4a. Ready selection
- **What:** Each agent finds unblocked, prioritized work without reasoning over the whole graph.
- **Mechanism / bolt-on:** `bd ready --json` — topological sort over the dependency graph (only `blocks` edges affect readiness), priority-ordered; `bd ready --unassigned` for self-serve; `bd ready --claim` claims in one shot **[primary]**.
- **Engram overlap:** NONE new (selection is bd's).

### 4b. Atomic claim (what prevents double-claim)
- **What:** Two agents must not grab the same task.
- **Mechanism / bolt-on:** `bd update <id> --claim` — atomically sets assignee + status=in_progress; first-writer-wins; idempotent for the current holder **[primary]** (https://github.com/steveyegge/beads). The atomicity holds at the **single serialized writer** (bd daemon / Dolt server). Supporting guards: `revision` optimistic-concurrency token in `bd show --json`; `--if-assignee`/`--if-status` CAS flags (documented in omp-plugins **[3p]**).
- **Dolt wrinkle:** cell-level merge means two writers touching *different* columns of the same row auto-merge without conflict; the same cell → conflict in system tables. A `row_lock` BIGINT column (every mutation touches it) converts concurrent writes into serialization conflicts (yojana analysis **[3p]**: https://github.com/ninthhousestudios/yojana/blob/HEAD/docs/beads-comparison.md). Cross-machine: a claim is only exclusive after `bd dolt push`; an offline claim is invisible until pushed — races resolve at merge time (last-writer-wins per cell, conflicts surfaced).
- **Engram overlap:** LOW — Engram's determinism wants a single-writer discipline here; the PM layer must guarantee claims funnel through one bd writer per repo.

### 4c. Lease liveness + dead-agent reaping
- **What:** Detect agents that claimed work and died; return stranded beads to `open`.
- **Mechanism:** schema already has `lease_expires_at`, `heartbeat_at` (bd 1.2.2) **[primary schema]**. Enforcement is policy: heartbeat writer + reaper scan. **Third-party reference implementation** (srobroek/omp-plugins **[3p]**, not core bd): `lease_host`/`lease_pid` anchors, `kill -0 <pid>` liveness proof before takeover, guarded release (`--if-assignee`), `bd reclaim` reaper, `bd-lease-gate`. https://github.com/srobroek/omp-plugins/blob/HEAD/beads/rules/beads-core.md.
- **Custom:** the heartbeat + reaper policy is genuinely custom (copy the omp-plugins pattern). Small daemon or cron.
- **Engram overlap:** MEDIUM — liveness/heartbeat is watchdog territory; if Engram has agent-health machinery, the lease policy should reuse its liveness signal rather than a second `kill -0` scheme.

### 4d. Protocol distribution (skill files)
- **What:** Every agent (Cursor, Claude Code, Codex, Gemini CLI) follows the same claim/advance/land procedure.
- **Mechanism / bolt-on:** **Agent Skills open standard** — https://agentskills.io/specification (spec repo https://github.com/agentskills/agentskills; donated to the Linux Foundation's Agentic AI Foundation) **[primary]**. `SKILL.md` + YAML frontmatter (required `name`, `description`; optional `license`, `compatibility`, `metadata`, `allowed-tools`), progressive disclosure (~50–100 tokens/skill at discovery), ships in `<project>/.agents/skills/` and `~/.agents/skills/`; one SKILL.md runs on 26+ platforms. Existing beads skills demonstrate the `allowed-tools: "Read,Bash(bd:*)"` scoping pattern.
- **Custom:** the *content* of the PM protocol skill (claim → work → note → gate → close → land). The format is a bolt-on.
- **Engram overlap:** NONE (distribution format), though Engram doctrine could be one more skill.

### 4e. MCP task mutation — DO NOT BUILD
- **What:** Agents mutate tasks via MCP instead of shell.
- **Mechanism / bolt-on:** **beads-mcp** (in-repo: https://github.com/gastownhall/beads/blob/main/integrations/beads-mcp/README.md **[primary]**; `pip install beads-mcp` / `uv tool install beads-mcp`). Tools 1:1 with CLI: init/create/list/ready/show/update/close/dep/comment/comments/note/blocked/stats/reopen/set_context; resource `beads://quickstart`; all take `workspace_root`. Notably `update(status="closed")` routes through close/reopen to respect approval workflows.
- **Anti-pattern to avoid** (getspur/spur RCA 2026-04-18 **[eco]**: https://github.com/getspur/spur/blob/HEAD/docs/rca/2026-04-18-brain-worker-beads-mcp-journey.md): MCP tool descriptions promising semantics the backend doesn't implement; "PM writes must be coherent under failure"; "observation must be at least as strong as mutation." If a custom mutation API is ever needed, copy vibesync's ETag/`If-Match` + `Idempotency-Key` + 409-on-mismatch pattern (https://github.com/oculairmedia/vibesync **[3p]**) — the REST equivalent of bd's `revision` token.
- **Engram overlap:** NONE.

---

## 5. DISTRIBUTED ISSUE IDENTITY

### 5a. ID scheme — hash IDs (bolt-on, with known failure modes)
- **What:** Merge-safe IDs for issues created concurrently on different branches/machines.
- **Mechanism / bolt-on:** bd adaptive hash IDs **[primary]**: https://github.com/gastownhall/beads/blob/main/engdocs/COLLISION_MATH.md. Alphabet `[a-z0-9]` (36 chars); birthday-paradox P ≈ 1−e^(−n²/2N); auto length-scaling at 25% collision probability (configurable `max_collision_prob`): 4 chars to ~500 issues (7.17% at 500), 5 to ~1.5k, 6 to ~15k. Collision → nonce retry: 10 attempts at base length, 10 at +1, 10 at +2 (30 total).
- **Failure modes:** (i) *Same content, two machines* → same hash → same ID → Dolt merge treats them as the **same row** (cell-level union; two "different" issues silently become one). Mitigation: IDs should incorporate randomness (nonce), not pure content hash — verify bd does this before relying on it. (ii) *Birthday collision across machines* → same ID, different issues → merge conflict on the row (surfaced, resolvable). (iii) *ID width drift* — length grows with DB size; tooling must not assume fixed width.
- **Killed alternatives:** auto-increment (needs a central allocator; offline/concurrent creation impossible — this is why GitHub issue numbers can't work offline); UUIDv4 (collision-safe but untypable, unmemorable, no ordering — acceptable only as `metadata` correlation IDs, not primary IDs).
- **Engram overlap:** LOW — identity is bd's; Engram should reference bead IDs opaquely, not mint its own.

### 5b. Versioned-SQL store — Dolt (bolt-on)
- **What:** SQL-queryable, branchable, mergeable issue store inside the repo.
- **Mechanism / bolt-on:** **Dolt** (dolthub/dolt), embedded by bd at `.beads/dolt/`. Merge = **cell-level three-way merge** **[primary]**: https://dolthub.com/blog/2020-07-15-three-way-merge/. Same PK + disjoint columns → auto-merge, no conflict. Same cell → different values → conflict (stored in system tables for resolution). Same change both sides → no conflict. Delete-vs-modify → conflict.
- **Failure modes:** (i) concurrent same-cell writes → conflict rows requiring resolution (needs a monitor/reaper — custom, see 4c); (ii) Dolt transactions are repeatable-read — read-then-write on the same cell can lost-update (third-party analysis: https://github.com/moekyawaungvivov30pro-design/beads/blob/HEAD/docs/design/dolt-concurrency.md **[3p]**); (iii) commit-graph ops (DOLT_COMMIT/MERGE) serialize on a global lock — hundreds of ops/sec, fine for PM workloads, not for high-frequency machine writes.
- **Engram overlap:** MEDIUM — if Engram's deterministic tracking needs its own store, Dolt is the candidate substrate rather than a second database; otherwise Engram stays a consumer of bd's Dolt log.

### 5c. Sync/merge protocol — `bd dolt push/pull` (bolt-on)
- **What:** Cross-machine convergence of issue state.
- **Mechanism / bolt-on [primary]:** https://github.com/gastownhall/beads/blob/main/docs/core-concepts/sync-concepts.md. Dolt remote protocol over the **same git remote**, history under **`refs/dolt/data`** (separate from `refs/heads/*`). `bd dolt push` / `bd dolt pull`; `bd bootstrap` clones Dolt history on fresh machines; `bd init` auto-wires `origin` as the Dolt remote.
- **Failure modes (verified in the wild):** (i) `git push` does NOT carry issues — a session that closes without `bd dolt push` strands issues on one machine (salish-sea postmortem: https://github.com/salish-sea/salishsea-io/blob/HEAD/docs/agents/issue-tracker.md **[eco]**); (ii) `.beads/issues.jsonl` is **export-only** — import is upsert-only and cannot represent deletions; using it as the sync channel causes silent staleness (same source documents a 3-week silent auto-export outage); (iii) legacy `bd sync` (JSONL/git-branch era) is superseded — protocol text must say `bd dolt push`.
- **Engram overlap:** LOW — sync is bd's; Engram consumes converged state.

### 5d. GitHub Issues mirror — `bd github sync` (bolt-on, partially verified)
- **What:** Bidirectional sync; bd is source of truth, GitHub Issues are the mirror (`bd github sync [--pull-only|--push-only]`).
- **Status:** observed in current ecosystem AGENTS.md docs (https://github.com/andrzejchm/fdb/blob/HEAD/AGENTS.md **[eco]**), **not verified against primary bd docs** — treat exact conflict semantics as unknown until checked live.
- **Related:** `/speckit.taskstoissues` converts spec-kit tasks to GitHub issues (https://github.com/github/spec-kit) — a second path that must not fight the bd mirror.
- **Engram overlap:** NONE.

---

## 6. BUILD vs BOLT-ON — consolidated verdicts

| # | Capability | Verdict | Exact component or why custom |
|---|---|---|---|
| 1a | Working-tree quality gate | **BOLT ON** | isitdone — https://github.com/raimondasl/isitdone — MIT, npm `@aivolution/isitdone` |
| 1b | Gate-suite definition | **CUSTOM** | `.isitdone.json` per repo (+ trivy as one more check cmd) |
| 1c | CI re-verification / merge queue | **BOLT ON** | GitHub native merge queue (zero-install) · or bors-ng https://github.com/bors-ng/bors-ng (Apache 2.0) · or isitdone GitHub Action |
| 1d | Build provenance | **BOLT ON** | `actions/attest` + `gh attestation verify` + cosign/Sigstore |
| 1e | Step link metadata | **BOLT ON** | in-toto/in-toto (`pip install in-toto`) |
| 1f | Task done-receipt schema | **CUSTOM** (schema) | in-toto envelope + Sigstore are bolt-ons; the predicate fields are custom |
| 1g | Land the Plane | **CONVENTION** | prompt text in AGENTS.md; enforcement = 1a + git hooks |
| 1h | Bors-style merge queue | **BOLT ON** | Gas Town Refinery (in GT stack) · standalone: bors-ng · lightest: GitHub MQ |
| 1i | spec-kit acceptance checking | **KILLED** | no automated checker exists; use `bd --acceptance` + `bd lint` + 1a |
| 2a | Decision records | **BOLT ON** | bd `decision` type + `--validate`/`bd lint` + dep edges |
| 2b | Deviation schema | **CUSTOM** | convention doc (~1 page SKILL.md); fields already exist on the bead |
| 2c | ADR file mirror | **BOLT ON** format, **CUSTOM** hook | MADR https://adr.github.io/madr + pre-commit render hook (~50 lines) |
| 2d | Signed deviation attestation | **CUSTOM** (schema) | same envelope as 1f |
| 3a | State crawl | **CUSTOM** | ~100-line parallel crawler on git porcelain (pattern: mgitstatus; candidate CLI: gita unverified) |
| 3b | Poll scheduler + rules | **CUSTOM** | cron + rule engine (gitvantage as spec reference only) |
| 3c | Push reporting | **CUSTOM** | fail-open git hooks + agent self-report on bead |
| 3d | gitpulse | **KILLED** | no such working-tree tooling exists |
| 4a | Ready selection | **BOLT ON** | `bd ready --json` |
| 4b | Atomic claim | **BOLT ON** | `bd update <id> --claim` (+ `revision` CAS token) |
| 4c | Lease liveness + reaper | **CUSTOM** | policy daemon; copy omp-plugins pattern (lease_host/lease_pid, kill -0 proof, `bd reclaim`) |
| 4d | Protocol distribution | **BOLT ON** (format) | Agent Skills https://agentskills.io/specification; content custom |
| 4e | MCP mutation | **BOLT ON — do not build** | beads-mcp (in-repo) |
| 5a | Issue IDs | **BOLT ON** | bd adaptive hash IDs (COLLISION_MATH.md); auto-increment & UUID killed |
| 5b | Versioned-SQL store | **BOLT ON** | Dolt via bd (cell-level merge) |
| 5c | Sync protocol | **BOLT ON** | `bd dolt push/pull` over `refs/dolt/data` |
| 5d | GitHub Issues mirror | **BOLT ON** (unverified) | `bd github sync` — confirm semantics live |

---

## 7. ENGRAM OVERLAP MAP (signals for the operator's final judgment)

| Mechanic | Engram machinery | Overlap signal |
|---|---|---|
| 1a isitdone gate + receipts | hash-chained verification receipts | **HIGH** — isitdone *produces* the evidence; Engram receipts likely *chain/anchor* it. PM work: bind receipt → bead ID → commit. |
| 1b gate packages | gate packages | **PARTIAL** — Engram's package concept may map 1:1 to per-repo/per-MR gate suites; execution substrate (hooks, runners) is new. |
| 1d/1e SLSA + in-toto | verification receipts | **LOW-MEDIUM** — different subjects (artifact vs task); reuse envelope + signing, not content. |
| 1f/2d custom verification/deviation predicate | hash-chained receipts | **HIGH** — this is the probable integration seam: Engram receipt = chain layer, PM predicate = claim content. |
| 1h merge queue + auditor conflict resolution | independent-auditor approval rounds | **PARTIAL** — auditor *pattern* exists in Engram; wiring auditors to MR conflict-resolution / bead transitions is new. |
| 2a/2b decision/deviation records | deterministic tracking, DVD doctrine | **MEDIUM** — align the deviation schema with DVD rather than inventing a parallel record format. |
| 3a–3c working-tree truth | — | **NONE** — genuinely new PM-layer territory. |
| 4b/4c claim atomicity + lease/reaper | agent health / liveness | **LOW-MEDIUM** — single-writer discipline is new; liveness signal could reuse Engram health machinery instead of a second `kill -0` scheme. |
| 5a–5c identity, Dolt, sync | deterministic tracking | **MEDIUM** — Dolt's versioned log *is* a deterministic record; Engram should consume it, not duplicate it. |

**Net:** Engram likely already covers *receipt chaining, gate packaging, and auditor patterns*. The PM layer's genuinely new builds are: the working-tree truth subsystem (3a–3c), the lease/reaper policy (4c), Dolt merge-conflict monitoring (5b/5c failure modes), the bead↔receipt binding schema (1f/2d), and the protocol content itself (skills, Land-the-Plane text, deviation conventions).

---

## Could not verify (explicit unknowns)

1. **Gas Town license + Refinery extractability.** MIT license claimed only by a secondary comparison (superset-vs-gastown.mdx); whether Refinery can run standalone outside the GT stack (daemon, mail, Dogs) is unverified.
2. **`bd github sync` exact semantics.** Seen in current ecosystem AGENTS.md docs; not confirmed in primary bd docs. Conflict resolution, label mapping, and rate-limit behavior unknown.
3. **gita's suitability** (worktree/stash coverage, machine-readable output) — listed as candidate only; needs a live check.
4. **`bd --claim` atomicity under concurrent load** on one Dolt server — documented as atomic in the README; the Dolt-layer caveats (repeatable-read lost updates, global commit-graph lock) come from third-party analysis, not from load testing here.
5. **isitdone maturity risk.** Project created 2026-09-07 (~4 weeks old at research time); receipt format has no stated schema-versioning; fast-moving (v0.8.x era 2026-09-27). Pin the version.
6. **omp-plugins lease/gate extensions vs upstream bd roadmap** — third-party; may be superseded by bd-native features at any time. The `lease_expires_at`/`heartbeat_at`/`revision` *fields* are core schema (bd 1.2.2); the *enforcement* is not.
7. **`bd lint` full rule set** — only the acceptance_criteria presence check is confirmed.
8. **Whether bd hash IDs incorporate per-creation randomness** (vs pure content hash) — determines whether failure mode 5a(i) (same content, two machines → same row) is real or theoretical. COLLISION_MATH.md documents the birthday analysis, not the preimage construction.

---

## Sources

All read 2026-10-05 via search index / page fetch.

**Primary (component's own repo/docs)**
- https://github.com/gastownhall/beads/blob/main/docs/core-concepts/sync-concepts.md — Dolt sync architecture
- https://github.com/gastownhall/beads/blob/main/engdocs/COLLISION_MATH.md — hash ID math
- https://github.com/gastownhall/beads/blob/main/integrations/beads-mcp/README.md — beads-mcp tools
- https://github.com/steveyegge/beads — bd README (`--claim` atomicity)
- https://github.com/gastownhall/gastown/blob/HEAD/docs/design/architecture.md — Refinery batch-then-bisect
- https://github.com/gastownhall/gastown/blob/HEAD/CHANGELOG.md — v0.9.0 Refinery shipment
- https://github.com/gastownhall/gastown/blob/HEAD/.claude/commands/patrol.md — Refinery patrol cycle
- https://github.com/gastownhall/gastown/blob/HEAD/internal/templates/polecat-CLAUDE.md — `gt done` contract, persist-to-bead
- https://github.com/raimondasl/isitdone — verified-done Stop hook (full README read)
- https://github.com/marketplace/actions/attest-build-provenance — SLSA attestation action
- https://github.com/in-toto/specification/blob/HEAD/in-toto-spec.md — link metadata spec
- https://dolthub.com/blog/2020-07-15-three-way-merge/ — Dolt cell-level merge
- https://github.com/fboender/multi-git-status — mgitstatus crawler mechanics
- https://github.com/rocketraman/gitvantage — working-tree state reference
- https://github.com/github/spec-kit/blob/main/docs/installation.md — spec-kit official
- https://agentskills.io/specification (via https://github.com/agentskills/agentskills) — Agent Skills standard
- https://adr.github.io/madr/ — MADR format

**Ecosystem (real usage, secondary)**
- https://github.com/salish-sea/salishsea-io/blob/HEAD/docs/agents/issue-tracker.md — `bd dolt push` stranding postmortem
- https://github.com/bunsdev/beads/blob/HEAD/docs/architecture/index.md — issue schema
- https://github.com/cloud-spe/livepeer-open-clearinghouse/blob/HEAD/.agents/skills/beads/references/02-core-concepts.md — schema incl. `revision`, lease/gate fields
- https://github.com/pablo305/beads/blob/HEAD/AGENTS.md — Land the Plane steps
- https://github.com/nguyensitrung/conductor-beads/blob/HEAD/.claude/skills/beads/commands/decision.md — decision record convention
- https://github.com/voxpelli/claude-beads/blob/HEAD/skills/retrospective/SKILL.md — decision supersede workflow
- https://github.com/cwijayasundara/claude_harness_eng_v5/blob/HEAD/docs/internal/compliance-increment.md — custom in-toto predicate pattern
- https://github.com/andrzejchm/fdb/blob/HEAD/AGENTS.md — `bd github sync`
- https://github.com/getspur/spur/blob/HEAD/docs/rca/2026-04-18-brain-worker-beads-mcp-journey.md — MCP contract-drift anti-patterns
- https://github.com/affromero/gitpane — multi-repo tool comparison table (gita)
- https://dev.to/pyor/github-merge-queue-explained-why-green-prs-break-main-2bk1 — GitHub merge queue semantics
- https://news.ycombinator.com/item?id=34712345 — bors-ng reference
- https://github.com/laurence-nz/superset-pwsh/blob/HEAD/apps/marketing/content/compare/superset-vs-gastown.mdx — Gas Town overview (license claim secondary)
- https://github.com/binhsu/aegis-core/blob/HEAD/docs/adr/0029-trivy-cve-scan-and-slsa-provenance.md — trivy+SLSA wiring
- https://dev.to/kamal_namdeo/slsa-sigstore-and-provenance-complete-practical-guide-1bl5 — SLSA/in-toto/Sigstore explainer
- https://github.com/anthropics/claude-code/issues/24327 — hook exit-2 vs user-denial failure mode
- https://github.com/Cyrax321/CONTINUUM/pull/224 — cross-harness hook surface (Claude/Gemini/Codex)
- https://github.com/holomush/holomush/issues/3917 — ADR capture at spec finalize → `bd create -t decision`

**Third-party extensions/analysis (not core bd — flagged inline)**
- https://github.com/srobroek/omp-plugins/blob/HEAD/beads/rules/beads-core.md — lease anchors, CAS guards
- https://github.com/srobroek/omp-plugins/blob/HEAD/beads/rules/beads-lifecycle.md — gate types, `bd gate check`
- https://github.com/srobroek/omp-plugins/blob/HEAD/beads/skills/adr/SKILL.md — adr-N namespacing, docs/adr render hook
- https://github.com/ninthhousestudios/yojana/blob/HEAD/docs/beads-comparison.md — leases table, row_lock analysis
- https://github.com/moekyawaungvivov30pro-design/beads/blob/HEAD/docs/design/dolt-concurrency.md — Dolt two-layer concurrency
- https://github.com/oculairmedia/vibesync — ETag/Idempotency-Key mutation pattern

## Integrity note

No page encountered attempted to redirect the mission or issued instructions affecting this research; all page content was treated as source material only. Nothing was downloaded, installed, or executed beyond reading.
