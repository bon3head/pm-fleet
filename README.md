# pm-fleet

Git-native, host-agnostic project-management layer for a fleet of AI-built repos.

One developer, dozens of repositories, AI coding agents doing the work. This repo
holds the working material for the management layer that keeps the human in the
loop: verified completion, working-tree truth, and first-class deviation tracking.

Status: research phase. No specs yet. Functionality first; dashboards later.

## Contents

- `proposal-brief.md` — consolidated build / buy / extend proposal
- `pm-software-report.md` — tool landscape research (what to buy)
- `pm-mechanics-report.md` — functionality mechanics research (what to build or bolt on)

## Direction

- Host-agnostic: GitHub is the main working space, Codeberg holds published repos,
  GitLab possible later. Nothing here may assume GitHub remotes everywhere.
- Doctrine: DVD (Data, Verification, Determinism). Score what you see. Kill weak
  candidates. Never present a claim as verified when it is not.
- Agent execution layer: Beads (`bd`), issues live in git.
- Machine-enforced done gates: isitdone (pinned version).
