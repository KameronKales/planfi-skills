---
name: ceo
version: 1.0.0
description: >-
  Run planfi like an autonomous CEO: a meta-orchestrator that sits ABOVE the three execution skills,
  keeps a persistent journal of what's been done and how it went, and dynamically decides the
  highest-leverage next move — then delegates it to Feature Factory (grow the product), Growth Engine
  (grow audience & revenue), or Finance Explainer (grow content). Each cycle it reviews strategy +
  history + live repo signals, runs a decision workflow that weighs product/growth/content, commits an
  initiative, invokes the chosen skill, and records the outcome in ceo/journal.md. Invoke when someone
  says "run the CEO", "what should we work on next", "grow the planfi business", "decide the next
  priority and do it", or wants a hands-off strategic loop over the three skills. Runs one cycle per
  call (pair with /loop for a standing cadence).
---

# CEO — decide the next move & delegate it to grow planfi

A meta-orchestration skill. The CEO does **no execution itself** — it is the strategist and bookkeeper.
Each cycle it reads the business state, decides the single highest-leverage next initiative, hands it to
the right execution skill, and journals how it went so the next cycle is smarter. The three levers:
- **Feature Factory** — grow the **product** (research → build → ship tool/feature gaps)
- **Growth Engine** — grow **audience & revenue** (prospects + monetization)
- **Finance Explainer** — grow **content** (noob→expert guides)

Persistent memory lives in `ceo/` (see `ceo/README.md`): `strategy.md` (north star), `journal.md`
(append-only cycle log), `backlog.md` (parked ideas). This is the user's explicit opt-in to multi-agent
orchestration — run the decision workflow and delegate; don't try to strategize *and* execute solo.

## When to use vs. not
- **Use** to decide and act on the next highest-leverage move across the whole business, or to run a
  standing loop that keeps planfi progressing on its own.
- **Don't use** when you already know exactly which skill to run — invoke that skill directly
  (`/feature-factory`, `/growth-engine`, `/finance-explainer`). The CEO is for the *decision* layer.

## Inputs (all optional)
- **steer** — a theme for this cycle (e.g. "focus on revenue", "we need a flagship feature"). Weighted
  in the decision, doesn't override strategy.
- **maxPicks** — initiatives to commit this cycle (default 1; >1 only if truly independent).
- **dry-run** — decide and journal the recommendation, but **don't** invoke the execution skill.

## Step 1 — Review the state
Gather what the CEO needs to decide (do this in the main loop, in parallel where possible):
- Read `ceo/strategy.md`, the **last ~3 entries** of `ceo/journal.md`, and `ceo/backlog.md`.
- Collect **live signals**: tool count (`grep -c '^- analyze\|^- ' public/llms.txt` ≈ catalog size),
  recent commits (`git log --oneline -10`), prospect counts (`wc -l growth/prospects.jsonl`, P0 count),
  content drafts (`ls drafts/ 2>/dev/null | wc -l`), and any open follow-ups noted in the last journal
  entry.
- **Discord kickoff** (same pattern as Feature Factory): read the `discord-webhook` from memory (never
  hardcode) and post a one-line "🧭 CEO cycle starting" with the steer if given. Skip silently if no
  webhook.

## Step 2 — Decide (run the decision workflow)
```
Workflow({ name: 'ceo-prioritize', args: {
  strategy:      '<contents of ceo/strategy.md>',
  recentJournal: '<the last ~3 journal entries>',
  backlog:       '<contents of ceo/backlog.md>',
  signals:       { toolCount, prospects, p0, drafts, recentCommits: [...] },
  steer:         '<from user, or omit>',
  maxPicks:      1,
  root: '/Users/kameronkales/planfi-app'
}})
```
It fans out product/growth/content executives, then a CEO agent ranks them against the strategy and
recent history and returns `{ proposals, rationale, portfolio_note, decisions: [{ skill, initiative,
why, args, success_criteria, verify, impact, effort, expected_outcome }] }`.

Present the decision (and what was deprioritized) to the user. If **dry-run**, skip to Step 4 and
journal the recommendation without executing.

## Step 3 — Delegate (invoke the chosen skill)
For each committed decision, **one at a time**, invoke the mapped skill via the Skill tool with the
decision's `args` — and let that skill run its own full process end-to-end:
- `feature-factory` → run the Feature Factory skill (its own research→build→ship loop, resilience, and
  Discord updates). Pass `focus`/`howMany`/`exclude` from `args`.
- `growth-engine` → run the Growth Engine skill (reads its ledger, researches, persists). Pass `focus`.
- `finance-explainer` → run the Finance Explainer skill (writes + saves the guide). Pass `topic`/`audience`.

Each of those skills already persists its own artifacts and posts its own progress. The CEO's job after
is to **verify** using the decision's `verify` field (e.g. read the new `growth/runs/*` report, the new
`drafts/*.md`, or the merged PR) and judge the outcome honestly.

## Step 4 — Record (this is the point — keep the log)
1. **Append a journal entry** to `ceo/journal.md` using the template in `ceo/README.md`: the State it
   saw, the Decision + skill, the Why, the Args, the **Outcome** (what actually happened — shipped /
   saved / persisted, or partial / blocked; never inflate), and the Next steps. Use today's date.
2. **Update `strategy.md`'s "Now" line** and `backlog.md` if the picture changed (park strong runner-up
   proposals from `result.proposals` in the backlog so they aren't lost).
3. **Discord done**: post a summary — what was decided, what the skill produced, and the verified
   outcome.

## Step 5 — Loop (optional)
The CEO runs **one cycle per call**. For a standing cadence, the user can wrap it with `/loop` (e.g.
`/loop /ceo`) or schedule it; each fire reads the journal it wrote last time, so cycles compound.

## Guardrails
- **Always journal, even on dry-run / failure** — silent cycles defeat the purpose.
- **Decide, then delegate — don't duplicate** the execution skills' logic in the main loop.
- **Don't repeat freshly-done work** — the journal is read first for exactly this reason; balance the
  product/growth/content portfolio over time.
- **Honest bookkeeping** — record partials and blocks as such; only mark an initiative done when the
  underlying skill reported it done and you verified it.
- **Secrets (Discord webhook, tokens) stay in memory/config**, never written into `ceo/`.

## Notes
- Iterating on the decision logic? Edit `.claude/workflows/ceo-prioritize.js` (the executive lanes and
  the CEO synthesis prompt are at the top) and re-invoke.
- The CEO is deliberately thin: strategy + memory + a decision engine. All real work — and all
  domain-specific guardrails — live in the three execution skills it calls.
