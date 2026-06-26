---
name: portfolio-rebalancer
version: 1.0.2
description: Cross-account, tax-aware rebalance trade generator. Use whenever someone has holdings across taxable / IRA / Roth / 401(k) and an IPS target allocation and wants the actual buy/sell trades to get back to target — e.g. "rebalance my accounts", "generate the trades to hit my 70/30 target", "which buy-sell trades get me back to my IPS", "my portfolio drifted, what do I trade across my IRA/Roth/401k/taxable", "rebalance my $1.2M back to target — harvest losses and don't trigger short-term gains". It detects drift beyond a rebalancing band, orders trades tax-advantaged-accounts-first, avoids short-term gains, harvests losses within the wash-sale window, and routes new contributions to underweight classes.
---

# Portfolio Rebalancer

A thin orchestration layer over the **planfi MCP** (https://ai.planfi.app/mcp/free).
All drift detection, tax-aware trade ordering, LTCG/NIIT pricing, wash-sale logic, and contribution
routing live server-side. This skill only gathers inputs and calls the tools — it does **not**
compute anything locally and bakes in no defaults of its own. Read-only.

**Related skills:** rebalancing is tax-driven — the natural follow-through is the **tax-optimizer**
skill (`analyze_tax_loss_harvesting` to harvest lot-level unrealized losses against realized gains
and up to $3,000 of ordinary income; `analyze_gain_harvesting` to see how much long-term gain you
can realize at the 0% capital-gains rate before NIIT/IRMAA bites). Use those for the loss-harvest and
0% gain-harvest follow-through after you've generated the rebalance trade list.

## Step 0 — Make sure the planfi tools are connected

This skill uses these tools (may be namespaced, e.g. `mcp__planfi__analyze_rebalancing_trades`):
`analyze_rebalancing_trades`, plus optional `analyze_tax_loss_harvesting`, `analyze_gain_harvesting`,
and `generate_financial_plan` (to mint a `plan_id` for chaining + a `share_url`). Use whichever name
your environment exposes (bare or `mcp__planfi__`-prefixed); below they are written bare.

If they're NOT available, tell the user to connect the MCP, then continue:

```
claude mcp add --transport http planfi https://ai.planfi.app/mcp/free
```

> **Try free, then add your key.** The command above adds the **free** connector — `https://ai.planfi.app/mcp/free` (no key needed). Once you create an API key, add a **new** connector with the MCP url — `https://ai.planfi.app/mcp` — and authorize it with your key.

(On claude.ai: add a custom connector pointing at https://ai.planfi.app/mcp/free.)

> **Access — free for personal use.** The planfi MCP is free to try (a small monthly allowance, no key needed). Heavy automated abuse forced us to add limits — but it stays **free for personal use**: email **kam@rateapi.dev** and we'll send you a free API key, no charge. (Companies and commercial use have paid plans.) To use a key, pass it as an `Authorization: Bearer pft_…` header in your MCP client config.


## Step 1 — (Optional) build a plan first to chain context + get a share link

If the user has (or wants) a full household model, call **`generate_financial_plan`** once and
**capture the returned `plan_id`** plus its **`share_url`** (planfi.app). `analyze_rebalancing_trades`
accepts `{ plan_id }` (plus inline overrides), so it can resolve filing status and household tax
context from the saved plan instead of you re-sending those figures — and the plan's `share_url` is
the only way to hand the user a sharable link (the rebalance tool does not emit one on its own).

This step is optional: `analyze_rebalancing_trades` runs from raw holdings + a target allocation too.

> **Engine facts to bake in:** all dollars are **today's (real) dollars**; all decimals are
> **fractions** (a 5% band → `0.05`, a 70% target → `0.70`); tax brackets/limits are approximate
> **~2026** values. The model is **federal-only** — state capital-gains tax is deferred.

## Step 2 — Route by intent

### "Rebalance my accounts / generate the trades to hit my target allocation / which buy-sell trades get me back to my IPS / my portfolio drifted, what do I trade across my IRA/Roth/401k/taxable" → `analyze_rebalancing_trades`

**Always CALL `analyze_rebalancing_trades` for these — do not answer from general knowledge or quote
rebalancing rules of thumb from memory.** When the user gives current holdings + a target allocation,
run it and lead with its real output (the actual buy/sell trade list and the detected drift).

The tool detects which asset classes have drifted beyond the rebalancing band, then generates the
ordered buy/sell trades to return to target — selling in TAX-ADVANTAGED accounts first (no gain
realized → zero tax), avoiding SHORT-term gains in taxable, harvesting LOSS lots where possible
within the wash-sale ±30-day window, and routing NEW contributions to the most-underweight classes
before any selling. It prices each taxable sell with the real federal LTCG cumulative-diff stacking
(0/15/20 bands) plus the 3.8% NIIT, and the ordinary marginal rate on short-term gains.

**REQUIRED**
- `holdings[]` — each current lot: `{ account_type, asset_class, value, cost_basis?, holding_period?, ticker? }`
  - `account_type`: `taxable` | `traditional_ira` | `roth_ira` | `401k`
  - `asset_class`: e.g. `us_equity` | `intl_equity` | `bonds` | `reit` | `cash` | `other`
  - `value`: current market value (dollars)
  - `cost_basis`: dollars (defaults to `value` → no gain — supply it so the tax-aware ordering works)
  - `holding_period`: `short` (held <1yr, ordinary rate on gains) | `long` (held ≥1yr, LTCG). Default `long`.
  - `ticker`: optional symbol for the lot
- `target_allocation` — IPS target as a record `asset_class → decimal fraction`, summing to ~1.0
  (e.g. `{ us_equity: 0.7, bonds: 0.3 }`)

**OPTIONAL**
- `rebalance_band` — drift threshold (decimal); a class is traded when `|drift|` exceeds it. Default `0.04` (±4%).
- `new_contribution` — new cash to deploy, directed to the most-underweight classes first. Default `0`.
- `ordinary_taxable_income` — ordinary taxable income after deductions; the stack base that prices LTCG and short-term gains on taxable sells. Default `0`.
- `magi` — modified AGI driving the 3.8% NIIT on net taxable gains. Default ≈ `ordinary_taxable_income` + standard deduction.
- `filing_status` — `single` | `married_joint`. Default `single`, or derived from the plan.
- `avoid_short_term_gains` — skip/deprioritize short-term-gain lots in taxable. Default `true`.
- `harvest_losses` — prefer selling loss lots in taxable to harvest the loss. Default `true`.
- `wash_sale_window_days` — wash-sale window in days (±). A loss within this window of a recent purchase is deferred. Default `30`.
- `tax_year` — brackets/limits year. Default `2026`.
- `plan_id` — saved plan handle to resolve household tax context from (`generate_financial_plan`).

```
analyze_rebalancing_trades({
  holdings: [
    { account_type: "roth_ira",        asset_class: "us_equity", value: 40000, cost_basis: 25000 },
    { account_type: "taxable",         asset_class: "us_equity", value: 30000, cost_basis: 18000, holding_period: "long" },
    { account_type: "traditional_ira", asset_class: "bonds",     value: 30000, cost_basis: 30000 }
  ],
  target_allocation: { us_equity: 0.6, bonds: 0.4 },
  rebalance_band: 0.05,
  new_contribution: 12000,
  ordinary_taxable_income: 250000,
  filing_status: "single"
})
```

## Step 3 — Surface the result honestly

- **Lead with the headline** — `needs_rebalancing` (is anything beyond band?), then the actual
  `trades[]` (each `{ action, asset_class, account, amount, rationale, realizedGain?, realizedTax?,
  term?, washSaleDeferred? }`) and the per-class `drift[]` that triggered them. Call out the
  `estimated_tax_cost`, `harvested_loss`, and any `contributions_directed[]` to underweight classes.
- **Read back `assumed_defaults[]`** — the tool returns a structured `assumed_defaults[]` array of
  `{ field, assumed_value, note }` for every input it defaulted (rebalance band, new contribution,
  short-term-gain avoidance, loss harvesting, wash-sale window, ordinary income, MAGI, filing
  status). Read these back so the user can correct any silent assumption and re-call.
- Honor `disclosures.not_advice` (a boolean flag) — present results as planning estimates, not tax
  advice — and surface `disclosures.key_assumptions` (federal-only, no state cap-gains; ±band;
  tax-advantaged-first ordering) when useful.
- **Follow `next_actions[]`** — each `{ tool, why, prefilled_args }` (carrying `{ plan_id }` when
  available). The natural chains are to the **tax-optimizer** tools: `analyze_tax_loss_harvesting`
  (lot-level loss harvest beyond the rebalance) and `analyze_gain_harvesting` (0%-bracket gain
  harvest). Use the server-suggested chain when present rather than guessing.
- **For a share link:** `analyze_rebalancing_trades` does not return a `share_url`. If the user wants
  a sharable plan, run `generate_financial_plan` (Step 1) and surface its `share_url`.

## Recommended call sequence (typical session)

1. (optional) `generate_financial_plan` → capture `plan_id` (+ `share_url`).
2. Gather `holdings[]` + `target_allocation`, then call `analyze_rebalancing_trades` (with `{ plan_id }` or raw fields).
3. Lead with `needs_rebalancing` + the `trades[]` + `drift[]`; read back `assumed_defaults[]`.
4. Follow `next_actions[]` to the tax-optimizer tools (`analyze_tax_loss_harvesting`,
   `analyze_gain_harvesting`) for the loss-harvest / 0% gain-harvest follow-through.

## Fictional examples

**1.** *"My $100k is 70% US equity / 30% bonds but my target is 60/40 — what do I trade, and don't
hit me with taxes?"*
→ `analyze_rebalancing_trades({ holdings: [{ account_type: "roth_ira", asset_class: "us_equity", value: 40000 }, { account_type: "taxable", asset_class: "us_equity", value: 30000, cost_basis: 18000 }, { account_type: "traditional_ira", asset_class: "bonds", value: 30000 }], target_allocation: { us_equity: 0.6, bonds: 0.4 }, rebalance_band: 0.05 })`.
US equity is +10% beyond the ±5% band → the tool sells $10k of equity **from the Roth IRA first**
(zero realized gain, `estimated_tax_cost` = $0) and buys $10k bonds, leaving the taxable equity lot
untouched. Lead with that trade list; read back `assumed_defaults[]` (income $0, filing status).

**2.** *"Rebalance my $1.2M across taxable/IRA/Roth/401k back to 70/30 — harvest losses and don't
trigger short-term gains. I have $12k of new cash to add."*
→ `analyze_rebalancing_trades({ holdings: [...], target_allocation: { us_equity: 0.7, bonds: 0.3 }, new_contribution: 12000, avoid_short_term_gains: true, harvest_losses: true })`.
The tool routes the $12k to the most-underweight class first (preferring contributions over selling),
harvests taxable loss lots, skips short-term-gain lots, and flags any loss inside the wash-sale ±30-day
window as deferred. Lead with `contributions_directed[]` + the trade list; chain `next_actions[]` to
`analyze_tax_loss_harvesting` for the full lot-level harvest.

*(Both examples use fictional figures — never reuse a real user's numbers in documentation.)*

## Notes

- All decimals are fractions; all dollars are today's (real) dollars; brackets/limits are ~2026.
- **Federal-only** — state capital-gains tax is deferred (deliberately not modeled in v1).
- `holdings[]` + `target_allocation` are REQUIRED — ask for them before calling. Everything else
  self-orchestrates from server defaults reported in `assumed_defaults[]`.
- Pass `{ plan_id }` to reuse a saved household model; any field you also pass is a shallow override.
- `analyze_rebalancing_trades` does **not** emit a `share_url` — chain `generate_financial_plan` for
  a sharable link.
- Not financial or tax advice. Planning estimates only.
