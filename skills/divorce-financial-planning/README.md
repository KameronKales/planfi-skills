# divorce-financial-planning (Claude Agent Skill)

Model the financial split of a divorcing household — QDRO division of 401k/pension/IRA, after-tax equalization of Roth vs traditional vs taxable awards, the equalizing cash payment, pension present-value split, home buyout vs §121-split sale, post-2019 alimony after-tax cost, MFJ→single bracket shift, and divorced-spouse Social Security (10-year rule) eligibility.

**Example — equalizing a 50/50 split:** *"We're divorcing after 12 years. He has a $400k 401(k), I have a $100k Roth, and there's a $300k brokerage (basis $100k). We agreed 50/50 and I take the splits — what's the equalizing payment?"* → the skill calls `analyze_divorce_qdro({ accounts: [...], spouseB: { otherTaxableIncome: 60000, postDivorceFilingStatus: "single" }, marriageYears: 12, targetSplitPct: 0.5 })` and leads with each spouse's after-tax award, the equalizing cash payment (who pays whom), and the divorced-spouse Social Security flag — because $1 of Roth ≠ $1 of traditional ≠ $1 of taxable once you discount for tax (fictional figures).

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

## Install

### Quickest — skills.sh CLI (recommended)

```
npx skills add holdequity/planfi-divorce-financial-planning
```

### Claude Code — copy the folder (zero prerequisites)

User-level (available in every project):

```
cp -r skills/divorce-financial-planning ~/.claude/skills/
```

Or project-level (only this repo/project):

```
mkdir -p .claude/skills && cp -r skills/divorce-financial-planning .claude/skills/
```

Restart Claude Code (or start a new session). The skill auto-loads by its description.

### Claude Code — as a plugin (shareable one-liner)

```
/plugin marketplace add holdequity/planfi-divorce-financial-planning
/plugin install divorce-financial-planning@planfi
```

### claude.ai — upload as a skill

1. Zip the folder: `cd skills && zip -r divorce-financial-planning.zip divorce-financial-planning`
2. In claude.ai go to Settings → Capabilities → Skills and upload the zip.

## Notes & honest caveats

- All decimals are fractions (24% → `0.24`); all figures are today's (real, inflation-adjusted)
  dollars; tax brackets/limits are approximate ~2026 values.
- The skill surfaces every assumption the server reports — `analyze_divorce_qdro` returns a
  structured `assumed_value[]` array (`{ field, value, reason }`) plus a `disclosures[]` string
  array (QDRO / §121 / TCJA-alimony treatment) so you can correct any silent default.
- Not financial advice. Planning estimates only.

See `SKILL.md` for the full instructions, exact tool params, and output format.
Source + issues: <https://github.com/holdequity/planfi-divorce-financial-planning>.

## License

MIT — see [LICENSE](./LICENSE).
