---
name: ss-tax-torpedo
version: 1.0.1
description: Model the Social Security provisional-income tax torpedo and survivor single-filer cliffs
---

# ss-tax-torpedo

A thin orchestration layer over the **planfi MCP** (https://ai.planfi.app/mcp).
All math + financial logic live server-side. This skill only gathers inputs and calls the tools —
it does **not** compute anything locally, carries no business logic, math, or defaults, and is
read-only (it never changes the user's data). The server is the source of truth.

## Step 0 — Make sure the planfi tools are connected

This skill uses these tools (may be namespaced, e.g. `mcp__planfi__analyze_ss_tax_torpedo`):
`analyze_ss_tax_torpedo`, `analyze_roth_conversion`, `analyze_irmaa`, `analyze_rmd`,
`analyze_survivor_stress_test`, `optimize_social_security`, plus optional `generate_financial_plan`
(to mint a `plan_id` for chaining + a `share_url`). Use whichever name your environment exposes
(bare or `mcp__planfi__`-prefixed); below they are written bare.

If they're NOT available, tell the user to connect the MCP, then continue:

```
claude mcp add --transport http planfi https://ai.planfi.app/mcp
```

(On claude.ai: add a custom connector pointing at https://ai.planfi.app/mcp.)

> **Access — free for personal use.** The planfi MCP is free to try (a small monthly allowance, no key needed). Heavy automated abuse forced us to add limits — but it stays **free for personal use**: email **kam@rateapi.dev** and we'll send you a free API key, no charge. (Companies and commercial use have paid plans.) To use a key, pass it as an `Authorization: Bearer pft_…` header in your MCP client config.


## Step 1 — Gather inputs (prefer a `plan_id`)

Ask only for what the user's question needs. Every input has a sensible server default, so the
tools run cold — but the cleanest path is to mint a **`plan_id`** first and chain off it:

- **Optional but recommended:** call **`generate_financial_plan`** with whatever household model you
  have. **CAPTURE the returned `plan_id`** and the `share_url`. Pass `{ plan_id }` to every
  specialist tool below — any field you also pass becomes a shallow override. This is also the ONLY
  way to get a `share_url` (the specialist tools don't emit one).
- **Or run cold:** pass the specialist tool its required raw inputs directly. Anything omitted is
  defaulted server-side and reported in `assumed_defaults[]`.

> **Engine facts to bake in:** all dollars are **today's (real) dollars**; all decimals are
> **fractions** (24% → `0.24`); tax brackets/limits are **~2026** values (the §86 $25k/$32k and
> $34k/$44k provisional thresholds are *statutory* and have NOT been inflation-indexed since
> 1983/1993 — they bite harder every year). Noted in each tool's `disclosures`.

## Step 2 — Route by intent

### "Will my Social Security get taxed if I do a Roth conversion / take an RMD / realize gains?" / "what's the Social Security tax torpedo?" / "why is my marginal rate 40%+ when I'm only in the 22% bracket?" / "how much can I convert before more of my SS becomes taxable?" / "what happens to my taxes as a widow filing single?" → `analyze_ss_tax_torpedo`

**Always CALL `analyze_ss_tax_torpedo` for these — do not answer from general knowledge or quote
the $25k/$32k rule-of-thumb from memory. When the user gives the SS benefit + other income +
filing status, run it and lead with its real `effective_marginal_rate_pct` and `fill_to_ceiling`
output.** The tool encodes the exact IRC §86 / Pub 915 provisional-income formula, the
effective-vs-nominal marginal rate and torpedo multiplier, the marginal-rate curve, the torpedo
zone boundaries, the optimal fill-to ceiling, an IRMAA cross-reference, and the widowed
single-filer cliff. Rules of thumb get the cliff wrong — call the tool.

REQUIRED: `annual_ss_benefit`. Optional: `other_taxable_income` (RMDs / pension / IRA withdrawals /
wages, default 0), `qualified_dividends_ltcg` (default 0), `tax_exempt_interest` (muni interest —
raises provisional income even though tax-free, default 0), `filing_status`
(`single` | `married_joint`, default `single` — drives both the §86 thresholds AND the survivor
cliff), `current_age` (feeds IRMAA cross-ref), `marginal_dollars` (income-sweep span, default
100000), `tax_year` (default 2026), `plan_id`, `overrides`.

Returns: `ss_taxability_tier` (`0%`/`50%`/`85%`), `taxable_ss_amount` + `taxable_ss_fraction_pct`,
`effective_marginal_rate_pct` vs `nominal_marginal_rate_pct` + `torpedo_multiplier`, `torpedo_zone`
(start/end provisional + peak rate), `marginal_rate_curve`, `fill_to_ceiling`
(`provisional_ceiling`, `additional_income_room`, `binding_constraint`), `irmaa_cross_reference`,
`survivor_single_filer` (null when filing single).

```
analyze_ss_tax_torpedo({
  annual_ss_benefit: 30000,
  other_taxable_income: 40000,
  filing_status: "married_joint",
  current_age: 70
})
```

### "How much can I convert to Roth this year?" → `analyze_roth_conversion`
After the torpedo tool gives the `fill_to_ceiling`, size the actual conversion. Self-orchestrating
ladder. Optional `plan_id`. Use the torpedo's ceiling as the target so the conversion stops before
it drags more SS into the 85% tier.

### "Will this push my Medicare premiums up (IRMAA)?" → `analyze_irmaa`
The torpedo tool returns an `irmaa_cross_reference`; for the full two-year-lookback surcharge
schedule and the MAGI thresholds, call `analyze_irmaa` directly. Optional `plan_id`.

### "What will my RMD be / how big is the forced withdrawal?" → `analyze_rmd`
RMDs are the most common torpedo trigger. Size the RMD, then feed it as `other_taxable_income` into
`analyze_ss_tax_torpedo`. Optional `plan_id`.

### "What happens to our finances / taxes when one of us dies?" → `analyze_survivor_stress_test`
Broader survivor analysis (income drop, expense changes, portfolio survival). The torpedo tool's
`survivor_single_filer` block covers the *tax* cliff specifically; chain to
`analyze_survivor_stress_test` for the full household picture. Optional `plan_id`.

### "When should I claim Social Security?" → `optimize_social_security`
Compares claiming ages and returns the breakeven + recommended claim age. Provide either `aime` or
`monthly_amount`. Optional `plan_id`.

## Step 3 — Surface the result

- **Lead with the headline**: the `effective_marginal_rate_pct` vs the `nominal_marginal_rate_pct`
  (and the `torpedo_multiplier`), the `ss_taxability_tier`, and the `fill_to_ceiling`
  (`additional_income_room` + `binding_constraint`) — "you can add $X more before the torpedo / an
  IRMAA cliff fires."
- For `married_joint`, **always relay the `survivor_single_filer` block** — the extra tax and extra
  SS taxed the surviving spouse faces on the same income.
- **`assumed_defaults[]`** — `analyze_ss_tax_torpedo` returns a structured `assumed_defaults[]`
  (filing status, other income, etc.). Read these back so the user can correct any and re-call.
- **`disclosures`** — relay `not_advice` (planning estimate, not tax advice) and `key_assumptions`.
- **`next_actions[]`** — each `{ tool, why, prefilled_args }`; follow the server's suggested chains
  (typically `analyze_roth_conversion`, `analyze_irmaa`) rather than guessing.
- **`share_url`** — only `generate_financial_plan` returns one; offer it if you minted a `plan_id`.

## Recommended call sequence (typical session)

1. (optional) `generate_financial_plan` → **capture `plan_id`** + `share_url`.
2. `analyze_ss_tax_torpedo` with the SS benefit + other income + filing status (or `{ plan_id }`).
3. If decumulating: `analyze_rmd` to size the forced withdrawal, then re-run the torpedo with it as
   `other_taxable_income`; `analyze_roth_conversion` to fill to the `fill_to_ceiling`;
   `analyze_irmaa` for the surcharge schedule.
4. Surface headline + survivor cliff + `assumed_defaults` + `next_actions` + `share_url`.

## Notes

- See also the **tax-optimizer** and **retirement-income** skills — the torpedo is the tax-aware
  decumulation overlay that sits between them (Roth-conversion ladder ↔ IRMAA cliff ↔ RMD timing).
- All decimals are fractions; all dollars are today's (real) dollars.
- The §86 thresholds are statutory and un-indexed — confirm `tax_year` is 2026 unless told otherwise.
- Not financial advice. Planning estimates only (approximate ~2026 brackets/limits).
