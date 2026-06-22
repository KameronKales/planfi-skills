# ss-tax-torpedo (Claude Agent Skill)

Model the Social Security provisional-income tax torpedo and survivor single-filer cliffs

It's a **thin orchestration layer** over the public **planfi MCP** (`https://ai.planfi.app/mcp`,
public, no auth) — all the math and financial logic live server-side. The skill itself bundles no
engine; it just gathers inputs and calls the tools.

### MCP bootstrap (one time)

If the planfi tools aren't connected yet, run:

```
claude mcp add --transport http planfi https://ai.planfi.app/mcp
```

On **claude.ai**: Settings → Connectors → add a custom connector pointing at
`https://ai.planfi.app/mcp` (no auth). The skill also reminds you to do this if the tools are
missing when you invoke it.

## Install

### Quickest — skills.sh CLI (recommended)

```
npx skills add holdequity/planfi-ss-tax-torpedo
```

### Claude Code — copy the folder (zero prerequisites)

User-level (available in every project):

```
cp -r skills/ss-tax-torpedo ~/.claude/skills/
```

Or project-level (only this repo/project):

```
mkdir -p .claude/skills && cp -r skills/ss-tax-torpedo .claude/skills/
```

Restart Claude Code (or start a new session). The skill auto-loads by its description.

### Claude Code — as a plugin (shareable one-liner)

```
/plugin marketplace add holdequity/planfi-ss-tax-torpedo
/plugin install ss-tax-torpedo@planfi
```

### claude.ai — upload as a skill

1. Zip the folder: `cd skills && zip -r ss-tax-torpedo.zip ss-tax-torpedo`
2. In claude.ai go to Settings → Capabilities → Skills and upload the zip.

## What it does

Answers the "tax torpedo" question: as retirement income (an RMD, a Roth conversion, a realized
gain) rises, the IRC §86 provisional-income formula pulls first 50% then 85% of your Social
Security benefit into taxable income — so an extra dollar can be taxed at **40-50%+** even when
your stated bracket is only 22%. The tool returns your `0%`/`50%`/`85%` SS-taxability tier, the
**effective vs nominal marginal rate** and **torpedo multiplier** (e.g. 1.85× in the 85% phase),
the marginal-rate curve and torpedo-zone boundaries, the optimal **fill-to ceiling** (how much more
income before the torpedo or an IRMAA cliff fires), an IRMAA cross-reference, and the **widowed
single-filer cliff** (the same household income hitting the narrower single brackets $25k/$34k vs
$32k/$44k). The §86 thresholds are statutory and have not been inflation-indexed since 1983/1993.

## Example (fictional figures)

> *"I'm 70, married filing jointly, $30k of Social Security, and I'm about to take a $40k RMD —
> how much of my SS gets taxed and what's my real marginal rate?"*

```
analyze_ss_tax_torpedo({
  annual_ss_benefit: 30000,
  other_taxable_income: 40000,
  filing_status: "married_joint",
  current_age: 70
})
```

The skill leads with the `effective_marginal_rate_pct`, the SS-taxability tier, the `fill_to_ceiling`
room before the next cliff, and — because it's joint — the `survivor_single_filer` extra tax the
surviving spouse would face. Then it chains to `analyze_roth_conversion` (fill to the ceiling) and
`analyze_irmaa` (surcharge schedule).

## Related skills

- **tax-optimizer** — Roth conversions, multi-year tax optimization, gain harvesting.
- **retirement-income** — withdrawal sequencing, Social Security claiming, healthcare bridge.

The torpedo is the tax-aware overlay that sits between those two during decumulation.

## Notes & honest caveats

- All decimals are fractions (24% → `0.24`); all figures are today's (real, inflation-adjusted)
  dollars; tax brackets/limits are approximate ~2026 values.
- The skill surfaces every assumption the server reports (`disclosures.key_assumptions`) so you can
  correct any silent default.
- Not financial advice. Planning estimates only.

See `SKILL.md` for the full instructions, exact tool params, and output format.
Source + issues: <https://github.com/holdequity/planfi-ss-tax-torpedo>.

## License

MIT — see [LICENSE](./LICENSE).
