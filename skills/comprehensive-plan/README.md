# comprehensive-plan (Claude Agent Skill)

One comprehensive financial plan in a single deliverable — retirement/FIRE projection with Monte Carlo backtesting, 529 college funding status, estate-tax exposure, and life/disability insurance protection gaps, every number engine-computed.

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
npx skills add holdequity/planfi-comprehensive-plan
```

### Claude Code — copy the folder (zero prerequisites)

User-level (available in every project):

```
cp -r skills/comprehensive-plan ~/.claude/skills/
```

Or project-level (only this repo/project):

```
mkdir -p .claude/skills && cp -r skills/comprehensive-plan .claude/skills/
```

Restart Claude Code (or start a new session). The skill auto-loads by its description.

### Claude Code — as a plugin (shareable one-liner)

```
/plugin marketplace add holdequity/planfi-comprehensive-plan
/plugin install comprehensive-plan@planfi
```

### claude.ai — upload as a skill

1. Zip the folder: `cd skills && zip -r comprehensive-plan.zip comprehensive-plan`
2. In claude.ai go to Settings → Capabilities → Skills and upload the zip.

## Ask Claude things like…

- "Build me a financial plan — I'm 40 making $150k with $300k invested, one kid age 5."
- "Give me a complete, comprehensive plan covering retirement, college, estate, and insurance."
- "Do a full financial checkup for our household — one document, not five separate analyses."

It runs `assemble_comprehensive_plan`, which fans your household out to four engine sub-analyses and
returns one cohesive plan: retirement Monte Carlo, 529 funding status, estate-tax exposure, and
protection gaps. For a FIRE-only deep dive (when-can-I-retire, save-more-vs-spend-less trade-offs),
use the companion **financial-forecast** skill instead.

## Notes & honest caveats

- All decimals are fractions (24% → `0.24`); all figures are today's (real, inflation-adjusted)
  dollars; tax brackets/limits are approximate ~2026 values.
- The skill surfaces every assumption the server reports (`disclosures.key_assumptions`) so you can
  correct any silent default.
- Not financial advice. Planning estimates only.

See `SKILL.md` for the full instructions, exact tool params, and output format.
Source + issues: <https://github.com/holdequity/planfi-comprehensive-plan>.

## License

MIT — see [LICENSE](./LICENSE).
