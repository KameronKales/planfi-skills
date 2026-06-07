# planfi skills

**Free, open-source [Claude Code](https://docs.claude.com/en/docs/claude-code) Agent Skills that turn Claude into a personal-finance analyst.** Ask in plain English — "can I retire at 50?", "should I rent or buy?", "how do I cut my taxes?", "is this Airbnb a good investment?" — and get back real numbers: net-worth projections, FIRE age, Monte Carlo success rates, tax savings, payoff timelines, and property returns.

All the math (and 150+ years of market data) runs server-side on the **public planfi MCP** (`https://ai.planfi.app/mcp`, no auth). The skills just gather your inputs and call the tools.

> **Source of truth:** these skills are generated from the `planfi-app` monorepo and mirrored here. Each skill also has its own canonical distribution repo (linked below). Don't edit here — changes flow from the monorepo via CI.

## Install

Two commands in Claude Code — add the skill catalog, then connect the MCP that powers it:

```
claude plugin marketplace add KameronKales/planfi-skills
claude mcp add --transport http planfi https://ai.planfi.app/mcp
```

Then install any skill from the catalog and just ask. Each skill below is also published as its own one-line marketplace at `holdequity/planfi-<name>`.

## Skills

| Skill | What it does | Canonical repo |
|-------|--------------|----------------|
| **str-investment-analyzer** | Analyze short-term-rental (Airbnb/VRBO) property investments by city: cash flow, cash-on-cash, cap rate, break-even occupancy, risk flag, managed-vs-self comparison, and a hold-period return (or indefinite buy-and-hold). Thin orchestration over the planfi MCP. | https://github.com/holdequity/planfi-str-analyzer |
| **financial-forecast** | Build a complete FIRE / financial forecast: net-worth projection, FIRE age, Monte Carlo success rate, insights, action plan, and what-if scenario comparisons. Thin orchestration over the planfi MCP. | https://github.com/holdequity/planfi-financial-forecast |
| **rent-vs-buy** | Rent vs buy: breakeven home-appreciation rate (pre & after tax), per-horizon BUY-vs-RENT net worth, payoff/crossover years, and invest-the-difference opportunity cost. Thin orchestration over the planfi MCP. | https://github.com/holdequity/planfi-rent-vs-buy |
| **tax-optimizer** | Cut taxes across accounts and years: broad tax optimization (asset location, tax-loss harvesting, charitable), multi-year ISO/conversion timing (AMT/NIIT/IRMAA-aware), Roth conversion ladders, mega-backdoor Roth space, and advanced surtaxes (NIIT/AMT/state). Thin orchestration over the planfi MCP. | https://github.com/holdequity/planfi-tax-optimizer |
| **equity-comp-planner** | Value and tax employer equity (RSUs, ISOs, NSOs, ESPP), find the ISO/AMT crossover, and assess single-stock concentration risk with a tax-aware diversification plan. Thin orchestration over the planfi MCP. | https://github.com/holdequity/planfi-equity-comp-planner |
| **retirement-income** | Plan retirement decumulation: tax-smart withdrawal order, Social Security claiming age, ACA healthcare bridge before Medicare, and estate-tax exposure. Thin orchestration over the planfi MCP. | https://github.com/holdequity/planfi-retirement-income |
| **debt-and-cashflow** | Pay down debt and route surplus cash optimally: avalanche/snowball debt payoff order, mortgage prepay vs invest, refinance break-even, and a next-best-dollar funding waterfall. Thin orchestration over the planfi MCP. | https://github.com/holdequity/planfi-debt-and-cashflow |
| **fire-tracker** | Track FIRE progress: are you already financially independent / can you coast, how you benchmark against peers, which net-worth milestones you've hit, and whether you can hit a target retirement age (with the easiest levers to pull). Thin orchestration over the planfi MCP. | https://github.com/holdequity/planfi-fire-tracker |

_MIT. Not financial advice — planning estimates only._
