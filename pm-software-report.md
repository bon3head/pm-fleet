# Project Management Software for a Solo Dev Managing Dozens of Git Repos + AI Agents

Research date: 2026-10-05. 2025–2026 sources preferred. Flags: `verified live` = read on the vendor's own page during this mission; `index` = via search index (with "fetched <date>" claims recorded where the source itself fetched the vendor page).

## Summary

**No single tool satisfies all 8 requirements; the honest answer is a two-layer stack.** The centralized, git-native, agent-writable, free-for-personal-use manager board that exists *today* is **GitHub Projects (v2)** — free, cross-repo kanban/table/roadmap, full GraphQL API, official MCP server, `gh project` CLI. The agent-native execution layer that lives *in* each git repo, with verifiable quality gates and first-class deviation tracking, is **Beads** (`bd`, MIT, single binary, MCP server) — explicitly built for AI coding agents, with `bd sync` bridging to GitHub Issues. **Linear** is the best SaaS alternative (official MCP, best agent DX) but is SaaS-only, closed-source, and its free tier caps at 250 issues — a real limit under agent-generated churn. The working-tree truth layer (uncommitted/untracked/stashes across 40+ repos — requirement 3) exists in **no PM tool**; OSS dashboards (gitvantage, gitpulse, RepoRadar) cover it and must be bolted on or scripted. Requirement 4 (verifiable done-criteria, not agent self-claims) is satisfied by no product out of the box — the working pattern is CI receipts + gate checklists enforced by beads "Land the Plane" protocol and Gas Town-style verification gates.

**One recommendation:** GitHub Projects as the human system of record + Beads in every active repo as the agent execution layer. First step: create one user-level GitHub Project (Board + Roadmap views, auto-add issues from all repos), run `bd init` in your top 5 active repos, and add a Beads workflow block to your shared `AGENTS.md`.

---

## Q-A: Candidate evaluations

### KEEP — shortlist

**GitHub Projects (v2).** Cross-repo boards/tables/roadmaps; custom fields (status, priority, iteration/sprints, dates); iterations; milestones; automations (auto-add on issue creation, auto-archive on merge/close); insights charts (velocity, cumulative flow); WIP limits (visual only). Git ingestion is native: issues, PRs, Actions runs, branch links feed the board in real time. `index` (https://github.com/open-source-accessibility/open-source-assistive-technology-hackathon/blob/HEAD/learning-room/docs/appendix-i-github-projects.md; https://github.com/jamesburton/ghkanban). Public trail matches the operator's hiring narrative.

**Beads (steveyegge/beads → gastownhall org).** Agent-first distributed git-backed issue tracker: `.beads/` Dolt version-controlled SQL DB in-repo, hash IDs (`bd-a1b2`) merge-safe, issue types task/bug/feature/epic/**decision**/message, 5 dependency types, priorities, `bd ready --json` (topo-sorted unblocked work), `bd prime` context injection, `bd compact` memory decay, `bd sync` over git remote, syncs with **Linear/Jira/GitHub**, "Land the Plane" protocol (run quality gates → file P0s on failure → close/update → verify clean git state → push). Docs describe **gates/molecules** workflow constructs; PyPI **`beads-mcp`** MCP server exists. MIT, Go single binary, ~19.5–23K stars by mid-2026, v1.0.4 (2026-05-09). `index` (https://github.com/mikeschinkel/endless/blob/HEAD/docs/research-2026-04-14-beads-analysis.md; https://github.com/cameronsjo/spec-compare/blob/HEAD/docs/beads.md; https://github.com/pmackay/sdlc-wiki/blob/HEAD/raw/beads/2026-08-31-beads.md; https://github.com/b0001/teach/blob/HEAD/AGENTS.md). No built-in UI (community TUIs/dashboards exist); per-repo by design — no native multi-repo fleet view.

**Linear.** Official remote MCP server at `https://mcp.linear.app/mcp` (Streamable HTTP, read-only variant `/mcp/readonly`; OAuth per-user; tools for issues/projects/comments/workflow states; announced 2025-05-01). Best-in-class GraphQL API + TS SDK + webhooks. Cycles (sprints), initiatives/projects, milestones, roadmaps, issue SLAs, strong GitHub integration (branch naming, auto-close on merge). Pricing `verified live` feature-structure (https://linear.app/pricing, read 2026-10-05) + `index` vendor-page fetch (2026-09-07, via https://github.com/ahmedyehya92/saas-platform-teardown-kit): **Free $0** (unlimited members, **2 teams, 250 issues**, 10MB uploads, includes Agent platform + MCP access); **Basic $10/user/mo** (yearly; 5 teams, unlimited issues); **Business $16/user/mo** (yearly; unlimited teams, private teams, guests, Triage Intelligence, Loops, Insights, SLAs); **Enterprise custom** (annual-only; SAML/SCIM, audit log, HIPAA). AI credits are a separate prepaid wallet (Coding Sessions $0.25/20-min sandbox + tokens; Loops $0.07–0.20/run). SaaS-only, closed source. The 250-issue free cap is a genuine risk under agent issue churn.

### Conditional / complementary (not the core)

- **Plane** — AGPL-3.0 Community Edition self-hostable (Docker); official MIT MCP server; REST + webhooks in CE; AI BYOK incl. Ollama. Pricing `index` via official plane.so blog (2026) + https://plane.so/pricing read 2026-10-05: Free Community Edition self-host; Cloud Free up to 12 users; **Pro $6/seat/mo**, **Business $13/seat/mo** (same for commercial self-host). **Kill-as-primary mechanisms:** epics/initiatives and custom work-item types are Pro-gated; enforced state transitions ("Workflows") are Business-gated — so the free CE lacks the epic/verification layer the mission needs; CE vs Commercial are different codebases and the Community seat limit is documented four inconsistent ways (open issue https://github.com/makeplane/plane/issues/9086 since 2026-05-16, unresolved); MCP-on-Community unverified; 60 req/min/token rate limit is tight for agents; ~12 containers, 2vCPU/4GB minimum; "documented history of rough upgrades; thin funding" (per takomo design doc). Best "self-hosted Linear-like," but for a *solo free* operator it delivers less than GitHub Projects $0.
- **Obsidian + dotpm** (https://github.com/dotpm/obsidian-pm) — PM inside Obsidian over plain Markdown (table/Gantt/kanban, subtasks, dependencies, custom fields), **local HTTP + MCP server + CLI so agents edit the same notes the views do**, project folders diffable in git. Excellent human notebook; but no git-activity ingestion (commits don't feed the board), per-project not fleet-wide, no verification beyond notes. Keep as manager's notebook, not the system.
- **spec-kit (github/spec-kit)** — spec-driven development CLI: `/speckit.constitution` (governing principles incl. testing standards), spec → plan → tasks pipeline, agent slash-commands for Copilot/Codex/Claude. `index` (https://github.com/dspeig/spec-kit). The spec layer for requirement 6: deviations get recorded against an explicit spec. Not a PM tool itself.
- **gitvantage** (https://github.com/rocketraman/gitvantage) — desktop dashboard, every local repo on one screen: staged/modified/untracked, stashes, submodules, **worktrees incl. uncommitted work in other checkouts**, ahead/behind/diverged, branch hygiene, side-by-side diffs, reminders/notifications. The closest existing answer to requirement 3.
- **gitpulse / RepoRadar / check-projects** — TUI/tray/CLI fleet status (uncommitted, untracked, unpushed, stashes, end-of-day alerts; gitpulse has bulk fetch/pull/push and activity digests). Lighter complements.
- **Wakapi / Wakana / Hakatime** (self-hosted WakaTime-compatible) — heartbeat IDE telemetry with Prometheus export; WakaTime Premium is $9/mo with a generous free tier. `index` (https://github.com/muety/muetsch.io; https://github.com/jemiluv8/wakana). Caveat: measures *human keyboard time* — blind to agent sessions.
- **YouTrack** — free up to 10 users (Cloud + Server) with unlimited functionality; ~$3.67–4.40/user/mo beyond; REST API + webhooks + workflows + GitHub/GitLab VCS integration; agile boards, Gantt. `index` (slashdot.org comparisons, crawled Sep 2026). No MCP server found; closed source; no verification primitives beyond workflows. Honorable mention only.
- **GitLab** — self-managed CE free; Duo + MCP on Self-Managed since 18.2. Epics/roadmaps need **Ultimate** (~$99–149/user/mo); heavy install; issue tracking is a side-feature. Killed on cost/complexity for a GitHub-based solo dev.

### KILL — with mechanism

| Candidate | Verdict | Kill mechanism |
|---|---|---|
| Jira | KILL | SaaS-only for new buys; per-seat ($5+/user/mo, 10-user free cap); enterprise bloat; not git-native; fails req 8 spirit |
| Asana | KILL | Cloud-only; Starter $10.99/mo with 2-seat minimum (bad for solo); portfolios/goals gated at $24.99; no git ingestion; fails req 2/8 |
| Notion | KILL | Cloud-only; no git-activity ingestion, no working-tree visibility, no roadmap verification; official MCP exists but scoped to Work Graph; fails req 2/3/4 |
| Shortcut | KILL | SaaS-only; ~$10/user/mo; no self-host; no MCP evidence; no git-native ingestion |
| OpenProject | KILL | GPLv3 but MCP server is Enterprise add-on only (v17.2, Mar 2026); baselines paid; classic Gantt PM orientation; heavy for solo; fails req 7 on CE |
| Taiga | KILL | Maintenance-only since 2024 (successor Tenzu immature); no agent surface at all; fails req 7 |
| Focalboard | KILL | **Unmaintained** — mattermost-community/focalboard README carries a "currently not maintained" warning |
| Nextcloud Deck | KILL | Own README: "not yet ready for intensive usage" (13 boards × 100 cards × 5 attachments → 6500 DB queries); no git integration, no agent API, no roadmap views |
| Leantime | KILL | Custom fields = $39 plugin, MCP = $29 plugin and cloud-side only; JSON-RPC only, no webhooks; fails req 7/8 |
| Huly | KILL | Cloud EOL July 2026 + blockchain pivot; 10+ containers, 8–16GB RAM; **no public REST API or webhooks**; no agent API |
| Redmine | KILL | GPLv2, zero paywall, but 2006 UX; REST partly alpha; no webhooks/GraphQL; only a community MCP plugin; weak roadmap visuals; fails req 5/7 |
| git-bug | KILL | v0.11 (2026-09-22) added a full web UI + GraphQL + bridges; but per-repo, no kanban/roadmap/epics, no MCP, slower cadence — Beads dominates it on req 7 |
| ticgit-ng / git-issue & successors | KILL | No evidence of active development in 2025–2026; superseded by beads/git-bug |
| Jellyfish / LinearB-style metrics | KILL | Enterprise team products; no solo-dev pricing; measure team throughput, not solo fleet state |

## Q-B: Serious-candidate deep dives

### GitHub Projects (v2)
- **Git ingestion:** native and best-in-class — issues, PRs, branch links, Actions/CI runs, milestones all feed project views automatically; automations move cards on merge/close. Cannot see uncommitted/untracked state (push-event-driven only).
- **Uncommitted state:** no. Requires the external crawler/dashboard layer (req 3 answer: gitvantage/gitpulse or a script).
- **API/MCP:** full GraphQL API + webhooks; official GitHub MCP server (remote `https://api.githubcopilot.com/mcp`, local Docker `ghcr.io/github/github-mcp-server`, ~100+ tools incl. a `projects` toolset); `gh project` CLI commands — agents can script without MCP. Caveat `index`: a 2026-01-03 evaluation found the *remote* MCP variant shipped zero Projects v2 tools (local had them); treat remote Projects support as possibly still limited — the local server / `gh` CLI are the reliable agent paths.
- **Roadmap verification:** roadmaps, iterations, milestones, custom fields, insights (velocity/cumulative flow) exist, but stage advancement is claim-based — no done-criteria enforcement.
- **Self-host/pricing:** SaaS only; **free for personal use** (Projects included on free plan — `index`, corroborated by multiple 2026 sources); public projects give the operator the public trail for hiring. Data portable via API/CSV export; moderate lock-in.

### Beads (`bd`)
- **Git ingestion:** the tracker *is* git data — `.beads/` Dolt DB versioned in-repo, pushed/pulled over git remotes; `bd sync` bridges to GitHub Issues/Linear/Jira. No CI/PR ingestion of its own.
- **Uncommitted state:** no (tracks issues, not working tree).
- **API/MCP:** CLI is the API (`--json` everywhere); PyPI `beads-mcp` MCP server; deep skill-file ecosystem (Cursor/Claude Code/Codex) with "Land the Plane" protocol.
- **Roadmap verification:** closest existing match — gates/molecules workflow constructs; Gas Town's Refinery runs verification gates (build/test/lint) in a Bors-style merge queue; session protocol requires running quality gates and filing P0s on failure before closing issues. Still convention + tooling, not a product guarantee — receipts (CI run links, test reports) get attached as issue content.
- **Self-host/pricing:** MIT, single Go binary, zero infra, offline-first. $0 forever. Most local-first option in the field.

### Linear
- **Git ingestion:** strong GitHub/GitLab integration (branch naming, PR auto-link, auto-close on merge, diffs) but push-event-driven; no working-tree visibility.
- **API/MCP:** official hosted MCP (`https://mcp.linear.app/mcp` + read-only) with OAuth; GraphQL + TS SDK + webhooks; 2,500 req/hr/key.
- **Roadmap verification:** cycles, initiatives (5 levels), milestones, SLAs, Insights dashboards — but advancement is agent-claimable; no done-criteria enforcement; no enforced state transitions by design.
- **Pricing:** verified above. Free tier's **250-issue cap** is the load-bearing limit for agent-driven workflows.

### Plane (self-host path)
- **Git ingestion:** GitHub/GitLab integrations are paid-tier; CE has REST + webhooks to wire your own.
- **API/MCP:** REST + webhooks in CE; official MIT MCP server (Community availability unverified).
- **Roadmap verification:** epics/initiatives = Pro; enforced workflows = Business; CE has none of the verification layer.
- **Self-host/pricing:** AGPL CE free; commercial self-host = cloud prices ($6/$13). ~12 containers, 2vCPU/4GB.

## Q-C: Build-vs-buy — the "fit my own" path

**Don't build a PM tool. Buy/adapt the two-layer stack (GitHub Projects + Beads + a fleet script) and build only the thin aggregation/verification glue that no product provides.**

Reference architecture (all local-first):

1. **Repo crawler (custom, small):** a scheduled script (Python + `pygit2` or parallel `git` CLI) walks the workspace roots; per repo emits: branch, ahead/behind vs upstream (`git rev-list --count`), `git status --porcelain=v2` (staged/unstaged/untracked counts), `git stash list`, worktree list. This is ~200 lines; gitvantage/gitpulse already do it if you prefer a UI. Output: JSON snapshot + a daily "dirty repos" digest issue on the GitHub Project.
2. **Working-tree diff engine (reuse):** `git status --porcelain=v2 --branch`, `git stash`, `git worktree list --porcelain`. No custom VCS parsing needed — git *is* the engine.
3. **Spec/stage store with verification receipts (custom convention + small harness):** specs authored with spec-kit (`/speckit.specify` → spec.md with acceptance criteria); each stage maps to a Beads epic with **done-criteria as a checklist**; a gate script (or Gas Town-style Refinery) runs `build → test → lint`, and stage advancement requires attaching **receipts**: CI run URL, coverage/test report, and a human or reviewer sign-off. Conceded deviations (req 6) are filed as Beads **`decision`** issues (or GitHub issues labeled `spec-drift`) linked `discovered-from` the spec — first-class, never silent.
4. **Dashboard UI (reuse + thin custom):** GitHub Projects roadmap/board for the human; one generated Markdown/HTML fleet page (nightly script: beads `bd list --json` × 40 repos + crawler snapshot → single status page, or the dotpm/Obsidian notebook as the human-readable layer). Optional telemetry: Wakapi for IDE time.

**Reuse vs custom:** reuse beads (issues), git CLI (crawler engine), GitHub Projects (UI/roadmap), spec-kit (spec scaffolding), gitvantage/gitpulse (working-tree UI), Wakapi (telemetry). Custom: the fleet aggregator (~1 weekend), the gate/receipt convention + verifier script (~1 weekend), the deviation-log convention (an afternoon in AGENTS.md).

**Honest cost:** MVP in 20–40 hours of focused part-time work; a polished version 2–3 months; plus a permanent small maintenance tax (beads schema upgrades, GitHub API changes). The buy/adapt path is $0 and works this week — build only the glue.

## Q-D: Ranked shortlist

1. **GitHub Projects + Beads (stack) — KEEP, recommended.** Covers req 1 (cross-repo board+roadmap), req 2 (native git ingestion + git-backed issues), req 5 (roadmap/insights), req 7 (GraphQL/MCP/`gh` + beads CLI/MCP), req 8 (both free for personal use; beads MIT). Req 3 via gitvantage/script; req 4 via gates+receipts convention; req 6 via `decision`-type deviation beads. Matches operator's existing GitHub presence and public-trail goal.
2. **Linear — KEEP as fallback.** Best agent DX and official MCP; killed as primary by SaaS-only + closed source + 250-issue free cap + no verification layer. Choose if you want zero setup and accept lock-in ($10/mo when you outgrow free).
3. **Plane (self-hosted Pro, $6/mo) — WEAK KEEP.** Only if self-hosting the PM database is non-negotiable *and* you'll pay $6/mo; the free CE lacks epics/custom fields/enforced workflows, which guts req 4/5.
4. **Obsidian + dotpm — notebook only.** Killed as the system; fine as the human journal layer.
5. **Everything else — KILL** per the table in Q-A (mechanisms listed there, no padding).

### Top-3 × 8 requirements (compact)

| Req | GitHub Projects | Linear | Beads |
|---|---|---|---|
| 1. Multi-repo board/epics/roadmap | ✅ cross-repo board, roadmap, iterations, milestones | ✅ projects/initiatives, cycles, roadmaps | ⚠️ per-repo epics/deps; no fleet view |
| 2. Git as data source | ✅ native (issues/PRs/Actions) | ✅ GitHub/GitLab sync | ✅ issues live in git |
| 3. Working-tree truth | ❌ (none do) → gitvantage/script | ❌ | ❌ |
| 4. Verified done-criteria | ❌ claim-based | ❌ claim-based | ⚠️ gates + Land-the-Plane receipts (convention) |
| 5. Roadmap/burndown/deps visuals | ✅ roadmap, insights, WIP | ✅ cycles, insights, SLAs | ❌ (community dashboards only) |
| 6. Spec-drift as first-class entry | ⚠️ convention (labels) | ⚠️ convention | ✅ `decision` issue type + `discovered-from` links |
| 7. Agent-writable (API/MCP) | ✅ GraphQL, webhooks, MCP, `gh` CLI (remote-MCP Projects caveat) | ✅ official MCP, GraphQL, SDK | ✅ CLI `--json`, beads-mcp, skills |
| 8. Self-host/portable, free-cheap | ⚠️ SaaS, free tier; API-portable | ⚠️ SaaS, free 250 issues; lock-in | ✅ MIT, local, $0 |

### The single recommendation

**Adopt GitHub Projects as the manager's system of record, with Beads as the agent execution layer in every repo.** It's $0, works this week, keeps all planning on the public GitHub trail, and gives agents a git-native issue protocol with gates and deviation tracking.

**Concrete first step (this week):** (1) Create one user-level GitHub Project with a Board view (Backlog/Next/In Progress/Review/Done) and a Roadmap view; enable the built-in workflow to auto-add new issues from all your repos and auto-archive on merge. (2) `bd init` in your 5 most active repos and add the Beads workflow block (`bd ready --json`, claim, Land-the-Plane protocol) to your shared `AGENTS.md`/skill files. (3) File conceded deviations as `decision` beads linked to their spec from day one — that's the habit that makes req 6 real.

## Q-E: Could not verify

- Whether Plane's official MCP server ships in the AGPL Community Edition (Plane docs don't connect the claims; `index` sources conflict).
- Whether GitHub's *remote* MCP server now includes Projects v2 tools (a 2026-01-03 eval found zero; may have changed — local server and `gh` CLI are confirmed paths).
- Exact current GitHub free-plan Projects entitlements on the vendor pricing page (corroborated by multiple 2026 `index` sources; not re-read on github.com/pricing this mission).
- `git-issue` and other niche git-native tracker successors: no 2025–2026 activity found; treated as dead.
- Focalboard: confirmed unmaintained via its own README warning.

## Sources

- GitHub Projects v2 deep dive — https://github.com/open-source-accessibility/open-source-assistive-technology-hackathon/blob/HEAD/learning-room/docs/appendix-i-github-projects.md (`index`, crawled 2026-10-01)
- ghkanban (Kanban over GitHub Issues + AI agent triggers) — https://github.com/jamesburton/ghkanban (`index`)
- Multi-repo GitHub Projects guide — https://github.com/cr-nattress/prompting/blob/HEAD/github/GITHUB-PROJECTS.md (`index`)
- Linear pricing, official page — https://linear.app/pricing (`verified live`, 2026-10-05; feature structure) + https://github.com/ahmedyehya92/saas-platform-teardown-kit/blob/HEAD/examples/linear-teardown/08-pricing-revenue-sources.md (`index`; fetched official page 2026-09-07) + https://procurementvms.com/vendors/linear-pricing-guide.html (`index`; verified Sep 19, 2026)
- Linear MCP server — https://theagenttimes.com/articles/linear-mcp-server-gives-ai-agents-direct-access-to-issue-tra-3126339e (`index`, 2026-09-06)
- Linear review 2026 (pricing/features) — https://nubiapage.com/linear-review-2026-app-ai-login-company/ (`index`, updated 2026-10-04)
- Hosted-trackers design verdicts (Linear/Plane/Tegon) — https://github.com/christiankohlberg/takomo/blob/HEAD/docs/design/03-hosted-trackers.md (`index`, updated 2026-09-30)
- Self-hosted trackers review 2026 (GitLab/Plane/OpenProject/Huly/Taiga) — https://github.com/williamweatherholtz/sysmlv2-ai-toolkit/blob/HEAD/docs/reviews/self-hosted-trackers-2026-09-07.md (`index`, 2026-09-07)
- Plane pricing, official page — https://plane.so/pricing (`verified live`, 2026-10-05) + https://plane.so/blog/plane-vs-asana-which-should-you-choose-in-2026 (official blog, `index`, pricing Pro $6 / Business $13)
- Plane pricing teardown — https://www.getbeton.ai/blog/plane-pricing-teardown/ (`index`)
- Plane Community seat-limit issue — https://github.com/makeplane/plane/issues/9086 (`index`, open since 2026-05-16)
- OSS PM landscape 2026 (Plane/OpenProject/Huly/Taiga/Leantime/Redmine) — https://github.com/radd-hq/radd/blob/HEAD/research/landscape.md (`index`, 2026-09)
- GitHub MCP server (official toolsets, remote/local) — https://github.com/forcewake/forge/blob/HEAD/docs/research/2026-09-14-mcp-surface.md (`index`, 2026-09-14)
- GitHub MCP remote Projects-v2 gap — https://github.com/oviney/economist-agents/blob/HEAD/docs/GITHUB_MCP_SERVER_EVALUATION.md (`index`, 2026-01-03)
- Beads analysis — https://github.com/mikeschinkel/endless/blob/HEAD/docs/research-2026-04-14-beads-analysis.md (`index`, 2026-04-14)
- Beads spec sheet — https://github.com/cameronsjo/spec-compare/blob/HEAD/docs/beads.md (`index`)
- Beads repo capture (gates/molecules, beads-mcp) — https://github.com/pmackay/sdlc-wiki/blob/HEAD/raw/beads/2026-08-31-beads.md (`index`, 2026-08-31)
- Beads skill/protocol — https://github.com/b0001/teach/blob/HEAD/AGENTS.md (`index`)
- git-bug 0.11 — https://dev.to/jamilxt/this-bug-tracker-lives-inside-your-git-repo-a-hands-on-guide-to-git-bug-3bjp (`index`, 2026-09-25) + https://linuxiac.com/git-bug-0-11-adds-a-full-web-ui-for-issues-and-code-browsing/ (`index`, 2026-09-23)
- gitvantage — https://github.com/rocketraman/gitvantage (`index`, updated 2026-09-15)
- gitpulse — https://github.com/lebiraja/gitpulse (`index`)
- RepoRadar — https://github.com/behrad87/reporadar (`index`, updated 2026-09-26)
- check-projects — https://dev.to/uralys/monitor-all-your-git-projects-at-once-1a33 (`index`, 2025-11-25)
- WakaTime vs Wakapi/Hakatime — https://github.com/muety/muetsch.io (`index`, 2026-09-30); Wakana — https://github.com/jemiluv8/wakana (`index`); WakaTime Premium $9/mo — https://pandev-metrics.com/docs/blog/best-wakatime-alternative-2026 (`index`, 2026)
- dotpm (Obsidian PM + MCP) — https://github.com/dotpm/obsidian-pm (`index`, updated 2026-09-30)
- spec-kit — https://github.com/dspeig/spec-kit (`index`; upstream github/spec-kit)
- Nextcloud Deck — https://github.com/nextcloud/deck/blob/HEAD/README.md (`index`; performance-limitations section)
- Focalboard unmaintained — https://github.com/mattermost-community/focalboard (`index`)
- Notion pricing 2026 — https://lifestack.ai/blog/notion-pricing (`index`, 2026-09-29)
- Asana pricing — https://www.saasworthy.com/product/asana/pricing (`index`, 2026-10-04)
- YouTrack pricing — https://slashdot.org/software/comparison/Shortcut-vs-YouTrack/ (`index`, crawled Sep 2026)
- GitLab pricing — https://appstackinsider.com/insights/gitlab-pricing-explained/ (`index`, 2026-09-23); GitLab Duo Agent Platform free-tier credits — https://itbrief.co.nz/story/gitlab-widens-ai-access-sets-flat-review-pricing (`index`, 2026-03)
- OSS PM roundup 2026 — https://www.thedigitalprojectmanager.com/tools/best-open-source-project-management-software/ (`index`, 2026-09-28)
