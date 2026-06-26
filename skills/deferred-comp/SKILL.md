---
name: deferred-comp
version: 1.0.2
description: Model nonqualified deferred comp (NQDC / 409A) elections for high-W2 execs and profitable S/C-corp owner-operators — defer-now-vs-take-now, lump-vs-installment distribution, and bracket / Medicare IRMAA / NIIT / Additional-Medicare smoothing into low-income FIRE bridge years, with employer unsecured-creditor risk. Use whenever someone asks "should I defer my bonus", "NQDC lump vs installment", "defer salary past my 401(k) cap", "what's the creditor risk of my deferred comp", "409A election — defer now or take now?", or "smooth my NQDC payouts to avoid IRMAA". Thin orchestration over the planfi MCP.
---

# Deferred Comp (NQDC / 409A) Election Analyzer

A thin orchestration layer over the **planfi MCP** (https://ai.planfi.app/mcp/free).
All deferral economics, election-year tax, distribution-year bracket / IRMAA smoothing, and
creditor-risk math live server-side. This skill only gathers inputs and calls the tools — it does
**not** compute anything locally, carries no defaults of its own, and is read-only.

**Related skills:** for RSUs / ISOs / NSOs / ESPP valuation and single-stock concentration, see
**equity-comp-planner** (`analyze_equity_compensation`). For owner-operators stacking a 409A election
on top of a Solo 401(k) / SEP / defined-benefit plan, see **self-employed-planner**
(`analyze_self_employed_retirement`). For the broader bracket / NIIT / IRMAA and Roth-conversion
sequencing the distribution years feed into, see **tax-optimizer** (`analyze_advanced_taxes`,
`analyze_irmaa`, `analyze_roth_conversion`).

## Step 0 — Make sure the planfi tools are connected

This skill uses these tools (may be namespaced, e.g. `mcp__planfi__analyze_deferred_comp`):
`analyze_deferred_comp`, `analyze_advanced_taxes`, `analyze_irmaa`, `analyze_roth_conversion`, plus
optional `generate_financial_plan` (to mint a `plan_id` for chaining + a `share_url`). Use whichever
name your environment exposes (bare or `mcp__planfi__`-prefixed); below they are written bare.

If they're NOT available, tell the user to connect the MCP, then continue:

```
claude mcp add --transport http planfi https://ai.planfi.app/mcp/free
```

> **Try free, then add your key.** The command above adds the **free** connector — `https://ai.planfi.app/mcp/free` (no key needed). Once you create an API key, add a **new** connector with the MCP url — `https://ai.planfi.app/mcp` — and authorize it with your key.

(On claude.ai: add a custom connector pointing at https://ai.planfi.app/mcp/free.)

> **Access — free for personal use.** The planfi MCP is free to try (a small monthly allowance, no key needed). Heavy automated abuse forced us to add limits — but it stays **free for personal use**: email **kam@rateapi.dev** and we'll send you a free API key, no charge. (Companies and commercial use have paid plans.) To use a key, pass it as an `Authorization: Bearer pft_…` header in your MCP client config.


## Step 1 — (Optional) build a plan first to chain context + get a share link

If the user has (or wants) a full household model, call **`generate_financial_plan`** once and
**capture the returned `plan_id`** (+ `share_url`). `analyze_deferred_comp` accepts `{ plan_id }`
(plus inline overrides) and uses it to derive filing status and to attach the plan's `share_url` —
the specialist tools do **not** emit a share link themselves, so this is the only way to give the
user one. This step is optional: `analyze_deferred_comp` runs cold from raw inputs too.

> **Engine facts to bake in:** all dollars are **today's (real) dollars**; all decimals are
> **fractions** (37% → `0.37`, 5% growth → `0.05`); tax brackets / limits are approximate **~2026**
> values (noted in `disclosures`). NQDC deferrals sit **above** the §402(g) qualified-plan cap.

## Step 2 — Route by intent

### intent → `analyze_deferred_comp`
**"Should I defer my bonus?" · "NQDC lump vs installment" · "409A election — defer now or take now?" · "defer salary past my 401(k) cap" · "what's the creditor risk of my deferred comp?" · "smooth my NQDC payouts to avoid IRMAA" · "spread my deferred comp over 10 years in early retirement"**

**Always CALL `analyze_deferred_comp` for these — do not answer from general knowledge / quote rules
of thumb from memory.** When the user gives the numbers, run it and lead with its real output.
**Trigger condition:** if the user supplies a deferral amount **and** either a working marginal rate
(or enough income context to stack one) **or** a `plan_id`, CALL the tool. Do not hand-wave
"deferring is usually good if your bracket drops" — the answer depends on the lump-vs-installment
IRMAA tiers, the §114 state bar, and the employer's creditor-risk haircut, all of which the tool
computes. Lead with its `recommendation` (`defer_installment` / `defer_lump` / `take_now`) and the
risk-adjusted PV advantage, not a heuristic.

Models the defer-now-vs-take-now tradeoff: the election-year tax deferred (fed marginal + state +
0.9% Additional Medicare at vest under the special-timing rule), lump-vs-installment distribution and
how installments smooth federal brackets + Medicare IRMAA tiers into low-income FIRE bridge years,
and employer unsecured-creditor risk as a hazard-rate haircut on the deferred balance.

REQUIRED: `deferral_amount` (annual $ electively deferred past the §402(g) cap).
Optional: `current_marginal_rate` (working-year all-in fed marginal, e.g. `0.37`; if omitted the
server stacks the deferral on other working income), `filing_status` (`single` | `married_joint`,
default `married_joint`), `distribution_election` (`lump` | `installment`, default `installment`),
`installment_years` (default `10` — 10+ preserves the §114 former-state tax bar), `distribution_start_age`
(default `65`), `other_bridge_income` (other taxable income in the distribution / FIRE-bridge years the
slices stack on, default `0`), `employer_credit_risk` (`investment_grade` | `speculative` |
`distressed`, default `investment_grade`), `growth_rate` (REAL pre-tax in-plan growth, default `0.05`),
`tax_year` (default `2026`), `plan_id`, `overrides`.

```
analyze_deferred_comp({
  deferral_amount: 200000,
  current_marginal_rate: 0.37,
  filing_status: "married_joint",
  distribution_election: "installment",
  installment_years: 10,
  distribution_start_age: 62,
  other_bridge_income: 0,
  employer_credit_risk: "investment_grade"
})
```

Returns: `recommendation` + `recommendation_reason`; `tax_deferred_at_election`;
`grown_balance_at_distribution`; `defer_now_after_tax_pv` vs `take_now_after_tax_pv` +
`deferral_advantage` (delta PV); `bracket_smoothing_savings` (`lump_marginal_rate`,
`installment_marginal_rate`, `irmaa_tiers_crossed_lump`, `irmaa_tiers_crossed_installment`,
`effective_distribution_rate`); `creditor_risk_haircut_pct` + `risk_adjusted_defer_pv`; and a
per-year `schedule` of gross distributions.

### "Confirm the full all-in tax bite on the election or distribution year" → `analyze_advanced_taxes`
After `analyze_deferred_comp`, run the working (election) year and a distribution year through the full
federal model (brackets, NIIT, Additional Medicare, AMT) to confirm the all-in rate. This is the
server-suggested `next_actions[]` chain — follow it rather than re-deriving the rate by hand.

### "Does a lump or big installment slice trip a Medicare IRMAA surcharge?" → `analyze_irmaa`
Pass the distribution-year MAGI (`other_bridge_income` + the per-year slice) to quantify the IRMAA
Part B/D surcharge a large slice triggers, and confirm the installment path stays under the tier-1
threshold.

### "Fill the empty bridge-year brackets before the NQDC stream starts" → `analyze_roth_conversion`
In the low-income years between deferral and the distribution start age, model filling the empty
brackets with Roth conversions before the NQDC installments push income back up.

## Step 3 — Surface the result honestly

- **Lead with the headline** — the `recommendation` (`defer_installment` / `defer_lump` / `take_now`)
  and `recommendation_reason`, then the `deferral_advantage` (risk-adjusted PV delta),
  `tax_deferred_at_election`, and the lump-vs-installment IRMAA tier comparison.
- **Read back `assumed_defaults[]`** — `analyze_deferred_comp` returns a structured
  `assumed_defaults[]` of `{ field, assumed_value, note }` for every input it defaulted (filing status,
  bridge income, growth rate, credit rating, …). Read these back so the user can correct any silent
  assumption (e.g. `other_bridge_income` assumed `$0` — a higher FIRE-bridge income raises the
  distribution marginal rate and IRMAA).
- **Surface the creditor-risk caveat** — the deferred balance is an **unsecured** claim against the
  employer; it is surfaced **net of** a credit-rating hazard haircut (`creditor_risk_haircut_pct`),
  not at full liquid value. Say so explicitly when the rating is `speculative` / `distressed`.
- **Note the §114 state rule** — a qualifying 10-year-or-longer installment stream is shielded from
  the former working state's tax; shorter installments may not be. Flag it if `installment_years < 10`.
- **Honor `disclosures.not_advice`** (a boolean) — present results as planning estimates, not tax/legal advice.
- **Follow `next_actions[]`** — each `{ tool, why, prefilled_args:{ plan_id } }` chains to
  `analyze_advanced_taxes`, `analyze_irmaa`, or `analyze_roth_conversion`. Use the server-suggested
  chain rather than guessing.
- **`share_url`** — only present if you minted a `plan_id` via `generate_financial_plan`; offer it so
  the user can open the full interactive plan on planfi.app.

## Recommended call sequence (typical session)

1. (optional) `generate_financial_plan` → capture `plan_id` (+ `share_url`).
2. `analyze_deferred_comp({ deferral_amount, current_marginal_rate | plan_id, … })` → lead with the
   `recommendation`.
3. Read back the headline + `assumed_defaults[]` + the creditor-risk / §114 caveats.
4. Follow `next_actions[]` → `analyze_advanced_taxes` / `analyze_irmaa` / `analyze_roth_conversion`.

## Fictional examples

**1.** *"I'm an exec, MFJ, top bracket. Should I defer $200k of my bonus into the 409A plan and take it over 10 years starting at 62?"*
→ `analyze_deferred_comp({ deferral_amount: 200000, current_marginal_rate: 0.37, filing_status: "married_joint", distribution_election: "installment", installment_years: 10, distribution_start_age: 62 })`.
Lead with the `recommendation` (likely `defer_installment`), the `tax_deferred_at_election` (~37% fed
+ 0.9% Additional Medicare), and that the 10-year installment keeps MAGI under the IRMAA tier-1
threshold (`irmaa_tiers_crossed_installment` ≈ 0) while a lump would cross several. Read back
`assumed_defaults[]` (bridge income assumed $0, growth 0.05, investment-grade employer).

**2.** *"My employer's credit rating is shaky — is deferring $150k still worth it?"*
→ `analyze_deferred_comp({ deferral_amount: 150000, current_marginal_rate: 0.35, employer_credit_risk: "distressed" })`.
Lead with the `recommendation` — the hazard haircut (`creditor_risk_haircut_pct`) on the unsecured
balance can flip the answer to `take_now` even when the raw tax math favors deferring. Surface the
risk-adjusted PV vs the take-now PV.

*(Both examples use fictional figures — never reuse a real user's numbers in documentation.)*

## Notes

- All decimals are fractions; all dollars are today's (real) dollars; brackets/limits are ~2026.
- `analyze_deferred_comp` has ONE required field, `deferral_amount` — everything else defaults
  server-side and is reported in `assumed_defaults[]`. Pass `current_marginal_rate` or a `plan_id`
  for a meaningful answer.
- The deferred balance is an UNSECURED claim against the employer — surfaced net of a credit-rating
  hazard haircut, NOT at full liquid value, and NOT counted as a liquid asset / debt.
- 0.9% Additional Medicare applies at election (special-timing rule); no FICA or NIIT applies at
  distribution. 4 USC §114 bars the former working state from taxing a qualifying 10-year-or-longer
  installment stream.
- Only `generate_financial_plan` returns a `share_url` — chain it for a sharable link.
- Not financial, tax, or legal advice. Planning estimates only.
