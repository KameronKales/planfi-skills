# portfolio-rebalancer (Claude Agent Skill)

Cross-account rebalance trade generator — tax-aware buy/sell lists to return to your IPS target allocation

It's a **thin orchestration layer** over the public **planfi MCP** (`https://ai.planfi.app/mcp/free`,
public, no auth) — all the math and financial logic live server-side. The skill itself bundles no
engine; it just gathers inputs and calls the tools.

### MCP bootstrap (one time)

If the planfi tools aren't connected yet, run:

```
claude mcp add --transport http planfi https://ai.planfi.app/mcp/free
```

> **Try free, then add your key.** The command above adds the **free** connector — `https://ai.planfi.app/mcp/free` (no key needed). Once you create an API key, add a **new** connector with the MCP url — `https://ai.planfi.app/mcp` — and authorize it with your key.

On **claude.ai**: Settings → Connectors → add a custom connector pointing at
`https://ai.planfi.app/mcp/free` (no auth). The skill also reminds you to do this if the tools are
missing when you invoke it.

## What it does

Give it your current holdings across taxable / traditional IRA / Roth / 401(k) and your IPS target
allocation; it detects which classes have drifted beyond the rebalancing band and returns the actual
**buy/sell trade list** to get back to target — tax-aware: it rebalances tax-advantaged accounts
first (no gain realized → zero tax), avoids realizing short-term gains in taxable, harvests loss lots
within the wash-sale ±30-day window, and routes new contributions to underweight classes before
selling. The single tool is **`analyze_rebalancing_trades`**.

### Fictional example

*"My $100k is 70% US equity / 30% bonds (the equity split between a Roth IRA and a taxable account)
but my target is 60/40 — what do I trade without a tax hit?"* → the skill calls
`analyze_rebalancing_trades` with your holdings + `target_allocation: { us_equity: 0.6, bonds: 0.4 }`.
US equity is +10% beyond the ±5% band, so it sells $10k of equity **from the Roth IRA first** (zero
realized gain, $0 estimated tax) and buys $10k of bonds — leaving the taxable equity lot untouched.
*(Fictional figures — never reuse a real user's numbers.)*

### Related skills

Rebalancing is tax-driven — pair it with the **tax-optimizer** skill for the follow-through:
`analyze_tax_loss_harvesting` (harvest lot-level unrealized losses against realized gains + up to
$3,000 of ordinary income) and `analyze_gain_harvesting` (realize long-term gains at the 0%
capital-gains rate before NIIT/IRMAA bites).

## Install

### Quickest — skills.sh CLI (recommended)

```
npx skills add holdequity/planfi-portfolio-rebalancer
```

### Claude Code — copy the folder (zero prerequisites)

User-level (available in every project):

```
cp -r skills/portfolio-rebalancer ~/.claude/skills/
```

Or project-level (only this repo/project):

```
mkdir -p .claude/skills && cp -r skills/portfolio-rebalancer .claude/skills/
```

Restart Claude Code (or start a new session). The skill auto-loads by its description.

### Claude Code — as a plugin (shareable one-liner)

```
/plugin marketplace add holdequity/planfi-portfolio-rebalancer
/plugin install portfolio-rebalancer@planfi
```

### claude.ai — upload as a skill

1. Zip the folder: `cd skills && zip -r portfolio-rebalancer.zip portfolio-rebalancer`
2. In claude.ai go to Settings → Capabilities → Skills and upload the zip.

## Notes & honest caveats

- All decimals are fractions (24% → `0.24`); all figures are today's (real, inflation-adjusted)
  dollars; tax brackets/limits are approximate ~2026 values.
- The skill surfaces every assumption the server reports (`disclosures.key_assumptions`) so you can
  correct any silent default.
- Not financial advice. Planning estimates only.

See `SKILL.md` for the full instructions, exact tool params, and output format.
Source + issues: <https://github.com/holdequity/planfi-portfolio-rebalancer>.

## License

MIT — see [LICENSE](./LICENSE).
