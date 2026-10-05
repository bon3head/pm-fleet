# PM Layer Proposal Brief — 2026-10-05

Consolidated build / buy / extend proposal for the git-native, host-agnostic
project-management layer. Status: research phase, no specs yet.
Functionality first; dashboards and display come only after the mechanics
are settled.

## Context

- Solo developer, 40+ git repos, AI coding agents do the work, human is the manager.
- Host-agnostic: GitHub is the main working space, Codeberg holds published repos,
  GitLab likely later. No layer may assume GitHub remotes everywhere.
- Doctrine: DVD (Data, Verification, Determinism). Score what you see, kill weak
  candidates, no padded lists. Never present a claim as verified when it is not.
- Overlap: the operator's Engram project (agent-memory / fleet-memory system with
  hash-chained verification receipts, gate packages, auditor approval rounds)
  covers adjacent territory. Receipt chaining: HIGH overlap. Gates/auditor:
  PARTIAL. Dolt log / deviation schema: MEDIUM. Working-tree truth: NONE
  (entirely new PM-layer territory).

## BUY (adopt as-is, $0)

- Beads (`bd`) in every active repo. Git-native issues, `decision` issue type for
  drift, `bd ready --json` topo-sorted work selection, atomic claim. MIT, single
  Go binary, zero infra.
- isitdone. Zero-LLM Stop hook for 12 agent harnesses. Runs the repo's real
  test/typecheck/lint on the exact working tree, blocks false "done" claims,
  detects weakened tests, emits tamper-evident receipts (PASS with tree hash,
  STALE on further edits, NONE on hand-edit). MIT. Verified live 2026-10-05:
  repo exists, created 2026-09-07. Caveat: 1 star, 61 commits, ~1 month old.
  Pin the version.
- GitHub native merge queue for integration re-verification on GitHub-hosted
  repos; bors-ng (Apache 2.0, standalone) as the host-agnostic equivalent.
- in-toto for signed link metadata; SLSA via `actions/attest` + `gh attestation
  verify` for artifact-to-commit binding. Note: SLSA proves artifact binding,
  NOT task-done. That confusion is killed.
- Agent Skills open standard (agentskills.io, Linux Foundation) to distribute one
  protocol across harnesses.
- gitvantage for working-tree visibility UI if wanted without building the crawler.

## EXTEND (OSS base, operator adds the missing piece)

- Beads lease enforcement. The schema carries lease fields; enforcement is custom.
  Copy the lease_host/lease_pid + `kill -0` proof + `bd reclaim` reaper pattern.
- Beads "Land the Plane". Prompt-only convention today; isitdone becomes the
  machine-enforced half.
- Custom in-toto predicate schema for task-level done receipts. This is the
  Engram seam: Engram's hash-chained receipts as the chain layer, the PM
  predicate as the claim content.
- `bd github sync` for the human board mirror. Exact semantics unverified against
  primary docs. Verify before relying on it.
- Gas Town Refinery only if standalone extractability and license resolve;
  otherwise skip (proportionate choice is the native merge queue / bors-ng).

## BUILD (genuinely custom, small)

- Parallel porcelain crawler: `git status --porcelain=v2`, `git stash list`,
  `git worktree list --porcelain` across every repo and linked worktree,
  JSONL snapshot. ~200 lines. Note: worktree dirtiness requires per-path status.
- Poll rules and alerts: aging unlanded work, stale worktrees, upstream movement.
- Heartbeat plus guarded lease reaper.
- Bead-to-commit-to-gate-verdict predicate linkage, aligned to Engram receipts.
- Dolt conflict monitor (same-cell concurrent writes conflict; cross-machine claim
  races resolve at `bd dolt push/pull`).
- Deviation and decision conventions plus ADR render hook.
- Fleet aggregator: `bd list --json` across repos plus crawler snapshot into one
  status page.

## Honest cost

20-40 focused hours for the glue: about one weekend each for the fleet aggregator
and the receipt-gate harness, an afternoon for deviation conventions. A polished
product is 2-3 months plus a permanent small maintenance tax.

## Explicit unknowns

1. Whether Beads hash IDs include per-creation randomness (determines if the
   same-content collision mode is real).
2. Gas Town Refinery standalone extractability and license.
3. `bd github sync` exact semantics.
4. isitdone is ~1 month old: pin the version; receipt format has no schema
   versioning.
5. Plane MCP server on AGPL Community Edition (from earlier pass).
6. GitHub remote MCP Projects v2 tools (from earlier pass).
