---
name: comprehensive-plan
version: 1.0.4
description: Build ONE comprehensive financial plan in a single deliverable by orchestrating the public planfi MCP — retirement/FIRE projection with Monte Carlo backtesting, 529 college funding status, estate-tax exposure, and life/disability protection gaps, every number engine-computed. Use whenever someone asks for "a financial plan", "build me a financial plan", "give me a complete / comprehensive / full plan", "I want one full plan covering retirement, college, estate, and insurance", "do a complete financial review / checkup", or wants one cohesive document instead of several separate analyses — e.g. "build me a financial plan, I'm 40 making $150k with $300k invested" or "give me a comprehensive plan covering everything".
---

# Comprehensive Plan

A thin orchestration layer over the **planfi MCP** (https://ai.planfi.app/mcp/free).
All math + the 150-year Shiller market dataset live server-side. This skill only gathers inputs and
calls the tools — it does **not** compute anything locally, carries no business logic, math, or
defaults, and is read-only (it never changes the user's data). The server is the source of truth.

This is the **"one document" superset**: it chains four engine sub-analyses — retirement/FIRE,
529 college funding, estate-tax exposure, and insurance/protection — into a single cohesive plan
where every dollar is engine-computed. For a **FIRE-only deep dive** (savings/retirement-age/spend
trade-offs, goal-solving, scenario comparison, "what if I change X" forks), use the
**`financial-forecast`** skill instead; come here when the user wants the broad, multi-section plan.

## Step 0 — Make sure the planfi tools are connected

This skill uses these tools (may be namespaced, e.g. `mcp__planfi__assemble_comprehensive_plan`):
`assemble_comprehensive_plan` (the orchestrator), plus `generate_financial_plan`, `what_if_plan`,
`generate_financial_insights`, `generate_action_plan`, `analyze_estate_exposure`,
`analyze_529_optimization`, `analyze_insurance_needs`, `analyze_education_credits`, and
`run_backtesting` for follow-on deep dives.
Use whichever name your environment exposes (bare or `mcp__planfi__`-prefixed); below they are
written bare.

If they're NOT available, tell the user to connect the MCP, then continue:

```
claude mcp add --transport http planfi https://ai.planfi.app/mcp/free
```

> **Try free, then add your key.** The command above adds the **free** connector — `https://ai.planfi.app/mcp/free` (no key needed). Once you create an API key, add a **new** connector with the MCP url — `https://ai.planfi.app/mcp` — and authorize it with your key.

(On claude.ai: add a custom connector pointing at https://ai.planfi.app/mcp/free.)

> **Access — free for personal use.** The planfi MCP is free to try (a small monthly allowance, no key needed). Heavy automated abuse forced us to add limits — but it stays **free for personal use**: email **kam@rateapi.dev** and we'll send you a free API key, no charge. (Companies and commercial use have paid plans.) To use a key, pass it as an `Authorization: Bearer pft_…` header in your MCP client config.


## SKILL ROUTING

### "build me a financial plan" / "give me a complete / comprehensive / full plan" / "I want one plan covering retirement, college, estate, and insurance" / "do a full financial review or checkup" / "look at my whole financial picture" / "am I on track overall — retirement, kids' college, insurance, everything" → `assemble_comprehensive_plan`

**Always CALL `assemble_comprehensive_plan` for these — do not answer from general knowledge or quote
rules of thumb from memory (no "save 25x expenses", no "$X per kid for college", no "10x income in
life insurance", no estate-exemption figures from memory). When the user gives the numbers, run the
tool and lead with its real output, THEN explain.** This single call fans the household model out to
all four sub-engines and returns one cohesive deliverable: cash-flow / retirement Monte Carlo, 529
funding status, estate-tax exposure, and protection gaps — every figure engine-computed. A bare
"build me a financial plan" is exactly this tool's job; reach for it first, then enrich.

Gather inputs per Step 1 (only ages/salaries + portfolio are strictly required — everything else is
defaulted server-side and read back), then make the primary call per Step 2. Cross-link: for a
FIRE-only deep dive or scenario forks, hand off to the **`financial-forecast`** skill; to drill into
one section of the assembled plan, use the enrichment routes below (chained via `{ plan_id }`).

## Step 1 — Gather inputs (only two areas are strictly required)

**REQUIRED — ask if not volunteered:**
1. **Each earner's age + annual salary** (household, 1–4 earners).
2. **Stock / investment portfolio**: `current_value` + `monthly_contribution`.

**Plan-shaping inputs you may not have** (retirement age, desired spend, SWR, returns, inflation,
`children[]` for 529, `educationAccount`, `real_estate[]` + mortgages for estate/protection, existing
insurance coverage, filing status / state for estate tax): gather them if the conversation makes it
natural, but you don't have to chase them down. Anything omitted is defaulted server-side and
reported back so the user can correct it (see Step 4) — the server is the source of truth for what
was assumed; don't track defaults yourself.

> **Engine facts to bake in:** all dollars are **today's (real) dollars**; all decimals are
> **fractions** (7% → `0.07`, 5% → `0.05`); tax brackets/limits are approximate **~2026** values
> (noted in each tool's `disclosures`).

**Optional but recommended — mint a `plan_id` + `share_url`:** call **`generate_financial_plan`**
with the household model, **CAPTURE the returned `plan_id`** and `share_url`. Pass `{ plan_id }` to
the follow-on deep dives instead of re-sending the model. `assemble_comprehensive_plan` also accepts
the household model directly and runs cold.

## Step 2 — Assemble the comprehensive plan (PRIMARY CALL)

Call **`assemble_comprehensive_plan`** with the gathered household model (or `{ plan_id }`):

```
assemble_comprehensive_plan({
  earners: [{ age: 40, annual_salary: 150000, retirement_age: 65 }],
  stocks: { current_value: 300000, monthly_contribution: 3000 },
  desired_annual_spend: 60000,
  children: [{ current_age: 5 }],
  filing_status: "married"
})
```

It returns one envelope with four nested sections — surface each:
- **forecast** — FIRE age/year, projected net worth at retirement, Monte Carlo failure / success
  rate (this is the honest risk check; if it's still `Calculating…`/undefined, say so — never report
  a fabricated 0).
- **education** — total college cost, funded-from-529, funding shortfall, funded %.
- **estate** — projected estate, applicable exemption, federal + state estate tax, effective rate.
- **protection** — total life-coverage gap, recommended vs existing coverage, per-earner gaps, and
  any monthly disability shortfall.
- **headline** — the engine-composed one-liner across all four sections; lead with it.

## Step 3 — Enrich each section on demand (chained via `{ plan_id }`)

Only when the user wants to drill into one section — each of these is its own routed intent:

### "what if we change one thing — contributions, retirement age, salary, spend — against the saved plan" / "scale back our 401ks next year, what happens to the full plan" → `what_if_plan`

**Always CALL `what_if_plan` for a single-change scenario against the saved plan — never re-send the household and never narrate a scenario outcome the tool did not compute.** Pass `plan_id` + a small `changes` array of typed ops; unchanged fields are inherited automatically and the result includes a computed `scenario_summary` (baseline vs scenario, delta, FIRE ages) to quote verbatim. For the FULL document under the new assumptions, chain the returned scenario `plan_id` into `assemble_comprehensive_plan`.

### "what's my chance of running out of money" / "run the Monte Carlo" / "stress-test my retirement against history" / "would my portfolio have survived 1929 or the 1970s" / "how safe is a 4% withdrawal for me" → `run_backtesting`

**Always CALL `run_backtesting` for these — do not answer from general knowledge or quote a
rule-of-thumb success rate ("the 4% rule works 95% of the time") from memory.** When the user gives
the numbers, run it and lead with its real output (the Shiller 1871-present failure rate plus
worst/best portfolio longevity), THEN explain. Takes raw `portfolio_value` / `annual_spend` /
`current_age` from the plan summary (or chain `{ plan_id }`). This is the honest risk check behind
the plan's **forecast** section — if the rate is still `Calculating…`/undefined, say so; never
substitute 0. Cross-link: for goal-solving around the failure rate (save more / retire later /
spend less), hand off to the **`financial-forecast`** skill.

### "is my 529 on track" / "how much college will the 529 cover" / "should I superfund the 529" / "roll leftover 529 money into a Roth" / "what do I do with an overfunded 529" → `analyze_529_optimization`

**Always CALL `analyze_529_optimization` for these — do not answer from general knowledge or quote
529→Roth rollover caps, superfunding multiples, or per-kid college cost figures from memory.** When
the user gives the numbers (or you have a `plan_id`), run it and lead with its real output
(529→Roth rollover headroom and superfunding moves for the education section), THEN explain.
Chain `{ plan_id }` from the assembled plan. Cross-link: pairs with **`analyze_education_credits`**
(keeping expenses out of the 529 to preserve the credit).

### "how much education tax credit can I get" / "AOTC vs Lifetime Learning Credit" / "maximize my college tax credit" / "do I qualify for the American Opportunity Credit" / "should I pay tuition out of pocket or from my 529" → `analyze_education_credits`

**Always CALL `analyze_education_credits` when the user gives tuition/qualified-expense numbers and
income — do not answer from general knowledge or quote AOTC/LLC dollar limits or phase-out ranges
from memory; run it and lead with its real output** (AOTC vs Lifetime Learning per student, MAGI
phase-out, refundable split, and the $4k 529 carve-out), THEN explain. Pass the expense and income
numbers — or chain `{ plan_id }` to derive MAGI/filing status. Cross-link: pairs with
**`analyze_529_optimization`** (the carve-out coordinates with 529 distributions).

### "will my estate owe taxes" / "how much estate tax will my kids pay" / "does my state have an inheritance tax" / "am I over the estate exemption" → `analyze_estate_exposure`

**Always CALL `analyze_estate_exposure` for these — do not answer from general knowledge or quote
federal/state exemption amounts or estate-tax rates from memory (they change and are engine-tracked
at ~2026 values).** When the user gives the numbers (or you have a `plan_id`), run it and lead with
its real output (state-by-state estate / inheritance detail: projected estate, applicable exemption,
federal + state tax, effective rate), THEN explain. Chain `{ plan_id }` from the assembled plan's
**estate** section.

### "how much life insurance do I need" / "is my coverage enough" / "what happens to my family if I die or can't work" / "do I need disability insurance" → `analyze_insurance_needs`

**Always CALL `analyze_insurance_needs` for these — do not answer from general knowledge or quote
"10x income" or any coverage rule of thumb from memory.** When the user gives the numbers (or you
have a `plan_id`), run it and lead with its real output (life + disability coverage breakdown by
earner: recommended vs existing coverage, per-earner gaps, monthly disability shortfall), THEN
explain. Chain `{ plan_id }` from the assembled plan's **protection** section.

### "what should I do first" / "give me prioritized recommendations" / "what are the biggest wins in my plan" / "turn this plan into action items / next steps" → `generate_financial_insights` / `generate_action_plan`

**Always CALL `generate_financial_insights` (prioritized, dollar-quantified insights) and/or
`generate_action_plan` (time-boxed next steps) for these — do not improvise a to-do list from
general knowledge; the engine ranks moves by real dollar impact across the whole plan.** Run the
tool(s) with `{ plan_id }` and lead with their real output, THEN explain. Cross-link: each insight
maps back to one of the four plan sections; offer the matching deep-dive route above.

## Step 4 — Present the plan

- **Lead with `headline`**, then walk the four sections (retirement → education → estate →
  protection). Keep it one cohesive document, not four disconnected analyses.
- **Read back assumptions** from `assumed_defaults[]` (and/or `disclosures.key_assumptions`) so the
  user can correct any silent default and you can re-run.
- **`disclosures.not_advice`** — relay that this is a planning estimate, not financial advice.
- **`next_actions[]`** — each `{ tool, why, prefilled_args:{ plan_id } }`. Follow these
  server-suggested chains rather than guessing.
- **`share_url`** — offer the plan's planfi.app link so the user can open the full interactive plan.

## Notes

- All decimals are fractions; all dollars are today's (real) dollars; stock return is real.
- Reuse the `plan_id` across the session — never re-send the full model.
- The Monte Carlo failure rate may be `Calculating…`/undefined until the household reaches FIRE at a
  retirement row — report it honestly, never substitute 0.
- For a **FIRE-only deep dive** (goal-solving, scenario comparison, "what if" forks, savings
  sensitivity), use the **`financial-forecast`** skill; this skill is the broad multi-section
  superset.
- Not financial advice. Planning estimates only (approximate ~2026 brackets/limits where tax applies).
