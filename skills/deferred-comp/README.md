# deferred-comp (Claude Agent Skill)

Model nonqualified deferred comp (NQDC / 409A) elections for high-W2 execs — defer-now-vs-take-now, lump-vs-installment distribution, and bracket/IRMAA/NIIT/additional-Medicare smoothing into low-income FIRE bridge years, with employer unsecured-creditor risk. Thin orchestration over the planfi MCP.

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
npx skills add holdequity/planfi-deferred-comp
```

### Claude Code — copy the folder (zero prerequisites)

User-level (available in every project):

```
cp -r skills/deferred-comp ~/.claude/skills/
```

Or project-level (only this repo/project):

```
mkdir -p .claude/skills && cp -r skills/deferred-comp .claude/skills/
```

Restart Claude Code (or start a new session). The skill auto-loads by its description.

### Claude Code — as a plugin (shareable one-liner)

```
/plugin marketplace add holdequity/planfi-deferred-comp
/plugin install deferred-comp@planfi
```

### claude.ai — upload as a skill

1. Zip the folder: `cd skills && zip -r deferred-comp.zip deferred-comp`
2. In claude.ai go to Settings → Capabilities → Skills and upload the zip.

## Ask Claude things like…

- *"I'm an exec, MFJ, top bracket — should I defer $200k of my bonus into the 409A plan and take it over 10 years starting at 62?"*
- *"NQDC lump vs installment — which keeps me under the Medicare IRMAA cliffs in early retirement?"*
- *"My employer's credit rating is shaky. Is deferring $150k still worth it, or should I take it now?"*
- *"Can I defer salary past my 401(k) cap, and what does that save me this year vs taxing it now?"*

Claude routes these to `analyze_deferred_comp` (chaining `analyze_advanced_taxes`, `analyze_irmaa`,
and `analyze_roth_conversion` as needed) and leads with the real `defer_installment` /
`defer_lump` / `take_now` recommendation — not a rule of thumb.

## Notes & honest caveats

- All decimals are fractions (24% → `0.24`); all figures are today's (real, inflation-adjusted)
  dollars; tax brackets/limits are approximate ~2026 values.
- The skill surfaces every assumption the server reports (`disclosures.key_assumptions`) so you can
  correct any silent default.
- Not financial advice. Planning estimates only.

See `SKILL.md` for the full instructions, exact tool params, and output format.
Source + issues: <https://github.com/holdequity/planfi-deferred-comp>.

## License

MIT — see [LICENSE](./LICENSE).
