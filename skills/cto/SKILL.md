---
name: cto
version: 1.0.0
description: >-
  Run planfi like an autonomous CTO: a technical meta-orchestrator that takes ONE top-level
  engineering goal (e.g. "make our product white-labelable", "ship the metered embed-API", "stand up
  the entitlement/billing spine"), decomposes it into an ordered set of small, independently-shippable
  CHUNKS across the right layers (UI components, REST API, MCP, data/auth/infra, packaging), and
  delegates each chunk to Feature Factory to build — in dependency order — then records the decomposition
  and outcomes in cto/journal.md. The CEO decides WHAT to pursue; the CTO decides HOW to build it and
  drives it to shipped. Invoke when someone says "run the CTO", "break this goal into build chunks and
  ship it", "enable white-labeling", "decompose <technical goal> and delegate it", or wants a hands-off
  technical-execution loop that turns one goal into shipped increments. Runs one decomposition+delegate
  cycle per call (pair with /loop for a standing cadence).
---

# CTO — decompose a technical goal into chunks & delegate them to Feature Factory

A technical meta-orchestration skill, sibling to the **CEO**. Where the CEO decides the highest-leverage
*business* move, the CTO takes ONE top-level *engineering goal* and answers **how to build it**: it
breaks the goal into small, ordered, independently-shippable **chunks**, maps each chunk to a Feature
Factory `build-gap`, and ships them in dependency order. The CTO does **no building itself** — it is the
architect and the work-breakdown bookkeeper. The one execution lever:

- **Feature Factory** (`skills/feature-factory/`, workflow `build-gap`) — builds ONE chunk end-to-end
  across engine, tests, web IA, MCP, REST, and distribution, to the Definition of Done in
  `docs/FEATURE_PLAYBOOK.md`.

Persistent memory lives in `cto/` (see `cto/README.md`): `architecture.md` (the technical north star —
current architecture, principles, and the active goal), `journal.md` (append-only decomposition log),
`backlog.md` (parked chunks/goals). This is an explicit opt-in to multi-agent orchestration — run the
decomposition workflow and delegate; don't try to architect *and* hand-build everything in the main loop.

## When to use vs. not
- **Use** to turn ONE top-level technical goal into shipped increments without you hand-planning every
  chunk — or to run a standing loop that drives a big build (white-labeling, billing spine, embed-API)
  to done over several cycles.
- **Don't use** when you already know the single gap to build — invoke `/feature-factory` (or `build-gap`)
  directly. The CTO is for the *decomposition + sequencing* layer.
- **Hand-off from the CEO:** when a CEO decision is a big *technical* build ("ship the metered embed-API
  tier"), the CEO can hand the goal to the CTO, which decomposes and ships it.

## Inputs
- **goal** (required) — the one top-level engineering goal, e.g. "enable our product to be white-labelable
  (UI components, API, and MCP) so companies can use our tools however they want".
- **steer** (optional) — a constraint/theme for this cycle (e.g. "iframe-first", "no new infra yet",
  "ship the smallest thing that unblocks Era").
- **maxChunks** (optional, default 3) — how many chunks to delegate this cycle. The rest are parked in
  `backlog.md` and picked up next cycle (so a big goal ships over a few CTO runs).
- **dry-run** (optional) — decompose + journal the plan, but **don't** invoke Feature Factory.

## Step 1 — Review the technical state
Gather what the CTO needs (main loop, in parallel where possible):
- Read `cto/architecture.md`, the **last ~3 entries** of `cto/journal.md`, and `cto/backlog.md`.
- Collect **live signals**: tool count (`grep -c '^- ' public/llms.txt`), recent commits
  (`git log --oneline -12`), open chunks/branches (`git branch --list 'ff/*' 'cto/*'`), the Feature
  Playbook DoD (`docs/FEATURE_PLAYBOOK.md`), and any open follow-ups in the last journal entry.
- **Discord kickoff** (same pattern as Feature Factory): read the `discord-webhook` from memory (never
  hardcode) and post a one-line "🛠️ CTO cycle starting — decomposing: <goal>". Skip silently if no webhook.

## Step 2 — Decompose (run the decomposition workflow)
```
Workflow({ name: 'cto-decompose', args: {
  goal:          '<the top-level engineering goal>',
  architecture:  '<contents of cto/architecture.md>',
  recentJournal: '<the last ~3 cto/journal.md entries>',
  backlog:       '<contents of cto/backlog.md>',
  signals:       { toolCount, recentCommits: [...], openBranches: [...] },
  steer:         '<from user, or omit>',
  maxChunks:     3,
  root: '/Users/kameronkales/planfi-app'
}})
```
It fans out per-layer architects (UI/components · REST API · MCP · data/auth/infra · packaging) who each
say what their layer needs for the goal and propose candidate chunks, then a CTO synthesizer dedupes and
sequences them into ONE ordered work-breakdown and returns:
`{ architecture_decision, chunks: [{ slug, name, description, layer, recommended_tool, scope, distribution,
skill_route, depends_on, build_gap_args, success_criteria, verify, effort }], sequencing, risks,
open_questions }`.

Present the work-breakdown (the ordered chunks, what's parked, and the architecture decision) to the user.
If **dry-run**, skip to Step 4 and journal the plan without delegating.

## Step 3 — Delegate (build each chunk via Feature Factory, in dependency order)
Chunks share files, so build **one at a time, in `depends_on` order**. For each committed chunk (up to
`maxChunks`):
1. Invoke **Feature Factory** via the Skill tool — or, for a single well-specified chunk, the `build-gap`
   workflow directly with the chunk's `build_gap_args` (a build-gap gap object: `{ slug, name, description,
   recommended_tool, distribution, skill_route, scope, root }`).
2. **Scope honestly:** Feature Factory's Definition of Done is tool-centric (engine + MCP + web panel).
   Many CTO chunks are INFRA/UI (a theming system, an iframe embed route, an entitlement gate) with **no
   MCP tool**. For those, pass a `scope` that names the layers in play and `recommended_tool: null` (or a
   route slug), and tell the build to follow only the relevant Playbook layers — don't force a fake tool.
3. **Verify** with the chunk's `verify` field (tests green, the build/route/page live, the gate passed),
   exactly as the CEO verifies a delegated skill. Run the full pre-ship gate before any merge; **never
   auto-deploy partner-facing infra** without surfacing it for review.

## Step 4 — Record (this is the point — keep the build log)
1. **Append a journal entry** to `cto/journal.md` (template in `cto/README.md`): the goal, the
   architecture decision, the full chunk breakdown (with which shipped this cycle vs parked), the
   per-chunk **outcome** (shipped / branch / blocked — never inflate), and the next chunk(s).
2. **Update `architecture.md`'s "Now" line** (the active goal + how far along) and `backlog.md` (park the
   un-built chunks in dependency order so the next cycle resumes exactly where this left off).
3. **Discord done**: post a summary — the goal, what shipped, what's parked, and the next chunk.

## Step 5 — Loop (optional)
The CTO runs **one decompose+delegate cycle per call**, shipping up to `maxChunks`. A big goal ships over
several cycles: each fire reads the journal + backlog it wrote last time and resumes the next chunks. For
a standing cadence wrap it with `/loop` (e.g. `/loop /cto enable white-labeling`).

## Guardrails
- **Always journal, even on dry-run / failure** — the build log is how the next cycle resumes mid-goal.
- **Decompose + delegate — don't hand-build** the chunks in the main loop. The CTO architects; Feature
  Factory builds.
- **Smallest shippable chunks, in dependency order** — each chunk must stand alone and pass the gate; an
  entitlement/auth/foundation chunk usually comes before the chunks that depend on it.
- **Honest bookkeeping** — record branch/partial/blocked as such; only mark a chunk done when the build
  reported done AND you verified it (gate green, live/reachable).
- **Respect the Definition of Done** (`docs/FEATURE_PLAYBOOK.md`) but don't force a tool-shaped DoD onto
  infra/UI chunks — scope the build to the layers that actually apply.
- **Don't auto-deploy partner-facing or irreversible infra** — build + gate + commit on a branch, surface
  for sign-off; the CEO/founder decides production cutover.
- **Secrets (Discord webhook, tokens) stay in memory/config**, never written into `cto/`.

## Notes
- Iterating on the decomposition logic? Edit `.claude/workflows/cto-decompose.js` (the per-layer architect
  lanes and the CTO synthesis prompt are at the top) and re-invoke.
- The CTO is deliberately thin: architecture + memory + a decomposition engine. All real building — and
  the build-time DoD — lives in Feature Factory (`build-gap`).
- Portable: the skill + `cto-decompose.js` + the `cto/` memory copy into any repo that has Feature Factory
  (adapt `architecture.md` + the `root` path), exactly like the CEO skill.
