---
name: growth-engine
version: 1.0.0
description: >-
  Grow and monetize planfi by fanning out a workflow of research agents that find people and
  organizations who could find value in planfi — companies & B2B partners, influencers & creators,
  communities, media/press, and distribution/integration partners — and propose how each could
  become revenue (planfi is currently free with no monetization). It is a REPEATABLE loop: it reads
  the prospects already compiled in growth/prospects.jsonl so each run finds NET-NEW leads instead of
  repeating, then persists the new ones plus a dated run report and any new monetization angles.
  Invoke when someone says "find people/companies who'd want planfi", "who could we reach out to",
  "compile a list of influencers/partners for planfi", "how do we grow planfi", "ways to monetize
  planfi", "run growth engine", or "research growth & monetization ideas". Runs BROAD (whole-funnel
  sweep) or FOCUSED on one segment/angle. Produces a saved run report and an updated ledger.
---

# Growth Engine — find who'd value planfi & how to monetize it

A growth-research skill. It fans out a **Workflow** of agents that web-research real, named prospects
who could find value in planfi, score them, and attach a monetization model to each — then synthesizes
the batch into concrete revenue plays. planfi is currently free with **no revenue**, so every run pairs
*reach* (who could amplify or use planfi) with *money* (how that turns into sustainable revenue without
breaking planfi's free, privacy-first promise).

It is **repeatable and compounding**: the persistent memory in `growth/` (see `growth/README.md`) lets
each run skip what's already been found. The workflow does the research; this skill does the
ledger-reading and persistence around it.

It runs in **two modes**, auto-selected by whether you pass a `focus`:
- **BROAD** (no `focus`) — a sweep across all five segments (companies, creators, communities, media,
  distribution). The default discovery run.
- **FOCUSED** (`focus` set) — drill into one segment/angle (e.g. "FIRE YouTubers", "RIAs", "MCP
  directories"). A scoping agent derives sub-segments; the SAME pipeline researches just that slice.

This skill is the user's explicit opt-in to multi-agent orchestration — run the workflow; don't try to
research the whole list solo in the main loop.

## When to use vs. not
- **Use** to discover and grow a pipeline of prospects + monetization angles for planfi, or to refresh
  it with net-new leads in a segment.
- **Don't use** to write a single LinkedIn post (that's the `linkedin-finance-marketer` agent), to
  brainstorm monetization in the abstract with no prospecting (that's the
  `financial-monetization-strategist` agent), or to analyze a user's own financial plan
  (`comprehensive-plan` / `financial-forecast`).

## Progress updates (Discord)
Keep the user updated in Discord, the same way Feature Factory does. Read the webhook URL from your
memory/config (the `discord-webhook` reference) — **never hardcode it**. The workflow runs in the
background, so post at the checkpoints the main loop controls, via
`curl -H 'Content-Type: application/json' -d '{"content":"..."}' <webhook>`:
- **Kickoff** — when you launch the workflow (mode + focus, e.g. "🚀 Growth Engine: broad sweep started").
- **Done** — after Step 2 persists: net-new count, the P0 prospect names, the top revenue play, and
  the saved report path.
If no webhook is in memory, skip silently (don't block the run).

## Step 0 — Read the ledger (de-dupe input)
Before running, read `growth/prospects.jsonl` and collect the `name` of every existing prospect into a
`seen` array. (If the file is empty, `seen` is `[]`.) This is what makes runs compound instead of
repeat. A quick way:

```
Bash: test -s growth/prospects.jsonl && cut -d'"' -f4 growth/prospects.jsonl || true
```

(or Read the file and pull each `name`). Pass the resulting list as `seen`.

Optional inputs — all have defaults, so you can run cold:
- **focus** — set for a FOCUSED sweep on one segment/angle (e.g. `"FIRE newsletter authors"`). Omit for BROAD.
- **perSegment** — target net-new prospects per segment (default 8).

## Step 1 — Run the workflow (PRIMARY ACTION)
Invoke the saved workflow, passing the `seen` list and whatever the user specified:

```
// Broad sweep:
Workflow({ name: 'growth-engine', args: {
  seen: [ ...names from growth/prospects.jsonl... ],
  root: '/Users/kameronkales/planfi-app'
}})

// Focused on one segment/angle:
Workflow({ name: 'growth-engine', args: {
  focus: 'FIRE YouTubers',
  seen: [ ...names... ],
  root: '/Users/kameronkales/planfi-app'
}})
```

What it does (one well-scoped fan-out):
0. **Scope** *(focused only)* — derive sub-segments for the requested focus.
1. **Prospect** — one agent per segment WebSearches for real, named prospects (excluding `seen`), each
   with a fit rationale, reach estimate, outreach angle, and monetization model.
2. **Qualify** — score each prospect's fit/reach/effort, assign a P0–P2 priority, sharpen the angle.
3. **Monetize** — a strategist synthesizes the batch into concrete revenue plays + new angles.
4. **Compile** — assemble the dated run report markdown.

It returns `{ focus, mode, segments, counts, prospects, monetization, new_angles, markdown }`.

## Step 2 — Persist (this is the point — retain the work in the repo)
1. **Append the new prospects to the ledger.** For each item in `result.prospects`, append one JSON
   line to `growth/prospects.jsonl` in the record shape documented in `growth/README.md` — map the
   workflow fields and stamp `status: "new"`, `first_seen: "<today's date>"`, `source: "<run slug>"`.
   Append only; never rewrite existing lines.
2. **Save the run report.** Write `result.markdown` to `growth/runs/<YYYY-MM-DD>-<slug>.md` where
   `slug` is `broad` or a kebab of the focus (e.g. `2026-06-12-fire-youtubers.md`).
3. **Update the playbook.** Append `result.new_angles` as dated bullets under "## Discovered angles"
   in `growth/monetization-playbook.md` (newest first). Skip angles already listed there.

Use today's date from context for all stamps. Create `growth/runs/` if missing.

## Step 3 — Present
Summarize for the user: mode, `result.counts` (sourced / net-new / qualified), the **P0 prospects** by
name with their angle + monetization model, and the **revenue plays** from `result.monetization`. Point
to the saved report path; optionally `SendUserFile` it. Offer follow-ups: a FOCUSED run on the most
promising segment, drafting outreach for the P0 list (hand to `linkedin-finance-marketer` /
`fintech-pitch-architect`), or deepening one revenue play with `financial-monetization-strategist`.

## Guardrails (bake into any prompt you add)
- **Real, verifiable prospects only** — named, lookup-able, with a working URL. No invented companies
  or people; reach figures are labelled estimates, never fabricated facts.
- **Don't re-propose `seen` prospects** — the whole point of the ledger is net-new each run.
- **Monetization must respect planfi's promise** — free core, privacy-first, no selling user data, no
  ad-tracking wall. Revenue comes from partners/distribution/affiliate/premium layers.
- **This is prospecting & strategy, not outreach** — nothing here contacts anyone. The user decides
  what to act on; `status` in the ledger is theirs to advance.

## Notes
- Iterating on the workflow? Edit `.claude/workflows/growth-engine.js` and re-invoke
  `Workflow({ name: 'growth-engine', ... })`; the segments are defined at the top of that file —
  add/retune them as the strategy evolves.
- The persistent memory lives in `growth/` (`prospects.jsonl`, `monetization-playbook.md`, `runs/`);
  `growth/README.md` documents the record shape and status lifecycle.
