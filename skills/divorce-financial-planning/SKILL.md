---
name: divorce-financial-planning
version: 1.0.2
description: Model the financial split of a divorcing household — QDRO division of 401k/pension/IRA, after-tax equalization of Roth vs traditional vs taxable awards, the equalizing cash payment, pension present-value split, home buyout vs §121-split sale, post-2019 alimony after-tax cost, MFJ→single bracket shift, and divorced-spouse Social Security (10-year rule) eligibility. Use whenever someone is dividing assets in a divorce — e.g. "how do we split our 401k in a divorce", "QDRO division of my pension", "who keeps the house, buyout vs sell", "what's the equalizing payment for a 50/50 split", "can I claim my ex's Social Security".
---

# Divorce Financial Planning

A thin orchestration layer over the **planfi MCP** (https://ai.planfi.app/mcp/free).
All math + financial logic live server-side. This skill only gathers inputs and calls the tools —
it does **not** compute anything locally, carries no business logic, math, or defaults, and is
read-only (it never changes the user's data). The server is the source of truth.

**Related skills:** for claiming/timing Social Security after the split (including the
divorced-spouse benefit in depth), see **retirement-income** (`optimize_social_security`). For the
full multi-year ordinary/LTCG/NIIT/AMT picture of a settlement year, see **tax-optimizer**
(`analyze_advanced_taxes`, `optimize_multi_year_tax`). For the whole-household forecast that the
settlement feeds into (net worth, FIRE %, Monte-Carlo backtesting), see **financial-forecast**
(`generate_financial_plan`).

## Step 0 — Make sure the planfi tools are connected

This skill uses these tools (may be namespaced, e.g. `mcp__planfi__analyze_divorce_qdro`):
`analyze_divorce_qdro`, `optimize_social_security`, `analyze_advanced_taxes`, `analyze_estate_exposure`,
plus optional `generate_financial_plan` (to mint a `plan_id` for chaining + a `share_url`).
Use whichever name your environment exposes (bare or `mcp__planfi__`-prefixed); below they are written bare.

If they're NOT available, tell the user to connect the MCP, then continue:

```
claude mcp add --transport http planfi https://ai.planfi.app/mcp/free
```

> **Try free, then add your key.** The command above adds the **free** connector — `https://ai.planfi.app/mcp/free` (no key needed). Once you create an API key, add a **new** connector with the MCP url — `https://ai.planfi.app/mcp` — and authorize it with your key.

(On claude.ai: add a custom connector pointing at https://ai.planfi.app/mcp/free.)

> **Access — free for personal use.** The planfi MCP is free to try (a small monthly allowance, no key needed). Heavy automated abuse forced us to add limits — but it stays **free for personal use**: email **kam@rateapi.dev** and we'll send you a free API key, no charge. (Companies and commercial use have paid plans.) To use a key, pass it as an `Authorization: Bearer pft_…` header in your MCP client config.


## Step 1 — (Optional) build a plan first to chain context + get a share link

If the user has (or wants) a full household model, call **`generate_financial_plan`** once and
**capture the returned `plan_id`**. The specialist tools accept `{ plan_id }` (plus inline
overrides), so they can resolve age, income, filing status, and balances from the saved plan instead
of you re-sending every figure. `generate_financial_plan` also returns the **`share_url`**
(planfi.app) — `analyze_divorce_qdro` itself emits a `share_url` only when a plan is in scope, so the
plan path is the reliable way to give the user a sharable artifact.

This step is optional: every tool runs from raw inputs too. Prefer the plan path when the session
already has a model or the user wants a sharable link.

> **Engine facts to bake in:** all dollars are **today's (real) dollars**; all decimals are
> **fractions** (a 50/50 split → `0.5`, 22% marginal → `0.22`); tax brackets/limits are approximate
> **~2026** values (noted in each tool's `disclosures`).

## Step 2 — Route by intent

### "Split our 401k / IRA / pension in a divorce" / "QDRO division of my retirement accounts" / "who keeps the house — buyout vs sell" / "what's the equalizing cash payment for a 50/50 split" / "is my Roth worth more than my traditional in the settlement" / "after-tax cost of alimony post-2019" / "can I claim my ex's Social Security (10-year marriage)" → `analyze_divorce_qdro`

**Always CALL `analyze_divorce_qdro` for these — do not answer from general knowledge or quote
QDRO / §121 / TCJA rules of thumb from memory. When the user gives the account balances and the
agreed split, run it and lead with its real output (each spouse's after-tax award, the equalizing
cash payment, and the SS divorced-spouse flag).** A $1 of Roth ≠ $1 of traditional ≠ $1 of taxable —
the whole point of this tool is to discount each account to its REAL after-tax value (traditional by
the recipient's marginal rate, taxable by the embedded built-in gain at the LTCG rate) so the split
is truly equal, not just equal on paper. Never eyeball that by hand.

Models the financial split of a divorcing household, deterministically:
- **QDRO division** of each `traditional` / `roth` / `taxable` / `pension` account — penalty-free
  transfer incident to divorce — split by an explicit `splitPct`, a `coverture` fraction, or the
  `targetSplitPct`.
- **After-tax equalization** — discounts each award to its real value and returns the
  **equalizing cash payment** (who pays whom, and how much) for a truly 50/50 (or agreed) settlement.
- **Pension PV split** — present value of the marital portion, divided per the agreed split.
- **Home** — buyout (one spouse keeps, pays the other their equity share) vs §121-split sale (each
  spouse excludes up to $250k of gain) — compares after-tax proceeds and recommends a path.
- **Post-2019 alimony** — non-deductible to payer / non-taxable to payee (TCJA) after-tax cost.
- **Filing-status shift** — MFJ → single/HoH bracket delta.
- **Divorced-spouse Social Security** — flags the 10-year-marriage-rule eligibility (and estimates
  the benefit if the ex's PIA is supplied).

REQUIRED to be useful: `accounts[]` — each `{ label, type, balance, owner, recipient }` where `type`
is `traditional` | `roth` | `taxable` | `pension`. Optional per-account: `basis` (taxable only),
`splitPct`, `coverture { numerator, denominator }`. Optional top-level blocks: `pension`, `home`,
`alimony`, plus `spouseA` / `spouseB` (`{ otherTaxableIncome, postDivorceFilingStatus }`),
`marriageYears`, `remarriedA` / `remarriedB`, `exPiaA` / `exPiaB`, `targetSplitPct`, `taxYear`,
`plan_id`. Anything omitted is defaulted server-side and reported back (see Step 3).

```
analyze_divorce_qdro({
  accounts: [
    { label: "His 401k",   type: "traditional", balance: 400000, owner: "A", recipient: "B" },
    { label: "Her Roth",   type: "roth",        balance: 100000, owner: "B", recipient: "B" },
    { label: "Brokerage",  type: "taxable",     balance: 300000, basis: 100000, owner: "A", recipient: "B" }
  ],
  spouseB: { otherTaxableIncome: 60000, postDivorceFilingStatus: "single" },
  marriageYears: 12,
  targetSplitPct: 0.5
})
```

### "When should I / my ex claim Social Security?" / divorced-spouse benefit timing → `optimize_social_security`
When the user wants to dig past the eligibility flag into claim-age timing for the divorced-spouse
benefit, chain to `optimize_social_security`. REQUIRED: the earnings/PIA inputs that tool documents.
Pass `{ plan_id }` when you have one.

### "What's the full tax picture of the settlement year?" → `analyze_advanced_taxes`
For the complete ordinary / LTCG / NIIT / AMT bite of the split (e.g. a taxable-account sale, a large
capital gain on a home sale), feed the figures into `analyze_advanced_taxes`. Pass `{ plan_id }` when
available.

### "Does the divorce change my estate exposure?" → `analyze_estate_exposure`
Splitting a joint estate halves each spouse's exemption picture. Use `analyze_estate_exposure` when
the user asks how the divorce affects estate-tax exposure or beneficiary planning. Pass `{ plan_id }`.

## Step 3 — Surface the result

For `analyze_divorce_qdro`:
- **Lead with the headline** — each spouse's **gross vs after-tax award**, the **equalizing cash
  payment** (`from` / `amount`), the **pension PV split**, the **home buyout-vs-split-sale**
  recommendation, the **alimony after-tax cost**, and the **divorced-spouse SS eligibility flag**.
- **Read back `assumed_value[]`** — a structured array of `{ field, value, reason }` for every
  default the engine applied (e.g. `sellingCostPct`, `discountRate`, `targetSplitPct`,
  `leaverSharePct`, `ssClaimAge`, and the HoH→single mapping flag). Surface these so the user can
  correct any silent assumption and re-call with overrides.
- **Read back `disclosures[]`** — the QDRO / transfer-incident-to-divorce treatment, the §121 dual
  $250k exclusion, the post-2019 TCJA alimony change, the HoH→single federal-tax approximation, and
  the real-dollar PV convention. Present everything as a planning estimate, not tax or legal advice.
- **`next_actions[]` / chaining** — follow server-suggested chains; otherwise suggest
  `optimize_social_security` (divorced-spouse claim timing), `analyze_advanced_taxes` (full
  settlement-year tax bite), or `generate_financial_plan` for a `share_url`.
- **`share_url`** — `analyze_divorce_qdro` emits one only when a plan is in scope. For a reliable
  sharable link, mint a `plan_id` via `generate_financial_plan` (Step 1) and surface its `share_url`.

## Recommended call sequence (typical session)

1. (optional) `generate_financial_plan` → **capture `plan_id`** + `share_url`.
2. `analyze_divorce_qdro` with the accounts + agreed split (and `{ plan_id }` if minted).
3. Surface headline + `assumed_value[]` + `disclosures[]`.
4. Chain on demand → `optimize_social_security`, `analyze_advanced_taxes`, `analyze_estate_exposure`.

## Fictional examples

**1.** *"We're divorcing after 12 years. He has a $400k 401(k), I have a $100k Roth, and there's a
$300k brokerage (basis $100k). We agreed 50/50 and I take the splits — what's the equalizing payment?"*
→ `analyze_divorce_qdro({ accounts: [{ label:"His 401k", type:"traditional", balance:400000, owner:"A", recipient:"B" }, { label:"Her Roth", type:"roth", balance:100000, owner:"B", recipient:"B" }, { label:"Brokerage", type:"taxable", balance:300000, basis:100000, owner:"A", recipient:"B" }], spouseB:{ otherTaxableIncome:60000, postDivorceFilingStatus:"single" }, marriageYears:12, targetSplitPct:0.5 })`.
Lead with each spouse's after-tax award and the `equalizingPayment` (who pays whom), and flag that
the 12-year marriage clears the 10-year divorced-spouse Social Security rule. Read back
`assumed_value[]` (discount rate, target split) and `disclosures[]` (QDRO transfer-incident, TCJA).

**2.** *"Should I keep the house (FMV $1M, mortgage $300k) or should we sell it?"*
→ `analyze_divorce_qdro({ accounts: [], home: { fairMarketValue: 1000000, mortgageBalance: 300000, costBasis: 500000, keeper: "A" } })`.
Lead with the `homeComparison` — buyout cash-to-leaver vs the §121-split-sale after-tax per-spouse
proceeds (with the $440k gain fully inside the dual $500k exclusion → $0 LTCG) — and the
`recommended` path. Read back the assumed `sellingCostPct` and `leaverSharePct`.

**3.** *"Was I married long enough to claim my ex's Social Security?"*
→ `analyze_divorce_qdro({ accounts: [], marriageYears: 9 })` returns `tenYearRuleMet: false` →
not eligible. With `marriageYears: 12` (and not remarried) it flags eligible; add `exPiaB` to
estimate the benefit. For claim-age timing, chain to `optimize_social_security`.

*(All examples use fictional figures — never reuse a real user's numbers in documentation.)*

## Notes

- All decimals are fractions; all dollars are today's (real) dollars; brackets/limits are ~2026.
- Pass `{ plan_id }` to reuse a saved household model; any field you also pass is a shallow override.
- `analyze_divorce_qdro` returns a structured `assumed_value[]` (`{ field, value, reason }`) and a
  `disclosures[]` string array — read both back so the user can correct silent assumptions.
- Head-of-household maps to `single` for the federal-tax helper (a documented approximation surfaced
  in `disclosures[]`); supply `mfjBaselineIncome` to also get the MFJ→single bracket-shift delta.
- This is a high-stakes deterministic valuation, not legal or tax advice. Planning estimates only;
  confirm the actual split with a QDRO attorney and tax professional.
