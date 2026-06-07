# PlanFi Skills — AI personal-finance & FIRE planning for Claude Code

[![Claude Code Agent Skills](https://img.shields.io/badge/Claude%20Code-Agent%20Skills-d97757)](https://docs.claude.com/en/docs/claude-code) [![Powered by MCP](https://img.shields.io/badge/MCP-ai.planfi.app-3b82f6)](https://ai.planfi.app/mcp) [![License: MIT](https://img.shields.io/badge/License-MIT-22c55e.svg)](./LICENSE) ![Skills](https://img.shields.io/badge/skills-8-f59e0b)

**Free, open-source [Claude Code](https://docs.claude.com/en/docs/claude-code) Agent Skills that turn Claude into a personal-finance & FIRE planning analyst.** Ask in plain English and get real numbers back — net-worth projections, your FIRE (financial-independence / retire-early) age, Monte Carlo success rates, tax savings, debt-payoff timelines, rent-vs-buy break-evens, and rental-property returns.

No spreadsheets, no signup, no API key. The math — and 150+ years of historical market data — runs on the public **planfi MCP** server; these skills just turn your questions into the right tool calls.

## Ask Claude things like…

- "Can I retire at 50 with $1.2M saved and $4k/mo spending?"
- "Should I rent or buy a $500k house — what appreciation rate makes buying win?"
- "How do I cut my taxes — Roth conversions, mega-backdoor Roth, tax-loss harvesting?"
- "When should I claim Social Security, and how do I bridge healthcare before Medicare?"
- "Avalanche or snowball — and should I prepay my mortgage or invest the cash?"
- "Are my RSUs too concentrated, and when do my ISOs trigger AMT?"
- "Is this Airbnb a good investment — cash-on-cash, cap rate, break-even occupancy?"
- "Am I already financially independent, or how many years until I am?"

## Quick start

Two commands in [Claude Code](https://docs.claude.com/en/docs/claude-code) — add the skills catalog, then connect the MCP that powers it:

```
claude plugin marketplace add KameronKales/planfi-skills
claude mcp add --transport http planfi https://ai.planfi.app/mcp
```

Install any skill from the catalog and just ask. Want only one? Each skill is also its own one-line marketplace: `claude plugin marketplace add holdequity/planfi-<name>`.

## Skills

| Skill | What it does | Install solo |
|-------|--------------|--------------|
| **str-investment-analyzer** | Analyze short-term-rental (Airbnb/VRBO) property investments by city: cash flow, cash-on-cash, cap rate, break-even occupancy, risk flag, managed-vs-self comparison, and a hold-period return (or indefinite buy-and-hold). | [`holdequity/planfi-str-analyzer`](https://github.com/holdequity/planfi-str-analyzer) |
| **financial-forecast** | Build a complete FIRE / financial forecast: net-worth projection, FIRE age, Monte Carlo success rate, insights, action plan, and what-if scenario comparisons. | [`holdequity/planfi-financial-forecast`](https://github.com/holdequity/planfi-financial-forecast) |
| **rent-vs-buy** | Rent vs buy: breakeven home-appreciation rate (pre & after tax), per-horizon BUY-vs-RENT net worth, payoff/crossover years, and invest-the-difference opportunity cost. | [`holdequity/planfi-rent-vs-buy`](https://github.com/holdequity/planfi-rent-vs-buy) |
| **tax-optimizer** | Cut taxes across accounts and years: broad tax optimization (asset location, tax-loss harvesting, charitable), multi-year ISO/conversion timing (AMT/NIIT/IRMAA-aware), Roth conversion ladders, mega-backdoor Roth space, and advanced surtaxes (NIIT/AMT/state). | [`holdequity/planfi-tax-optimizer`](https://github.com/holdequity/planfi-tax-optimizer) |
| **equity-comp-planner** | Value and tax employer equity (RSUs, ISOs, NSOs, ESPP), find the ISO/AMT crossover, and assess single-stock concentration risk with a tax-aware diversification plan. | [`holdequity/planfi-equity-comp-planner`](https://github.com/holdequity/planfi-equity-comp-planner) |
| **retirement-income** | Plan retirement decumulation: tax-smart withdrawal order, Social Security claiming age, ACA healthcare bridge before Medicare, and estate-tax exposure. | [`holdequity/planfi-retirement-income`](https://github.com/holdequity/planfi-retirement-income) |
| **debt-and-cashflow** | Pay down debt and route surplus cash optimally: avalanche/snowball debt payoff order, mortgage prepay vs invest, refinance break-even, and a next-best-dollar funding waterfall. | [`holdequity/planfi-debt-and-cashflow`](https://github.com/holdequity/planfi-debt-and-cashflow) |
| **fire-tracker** | Track FIRE progress: are you already financially independent / can you coast, how you benchmark against peers, which net-worth milestones you've hit, and whether you can hit a target retirement age (with the easiest levers to pull). | [`holdequity/planfi-fire-tracker`](https://github.com/holdequity/planfi-fire-tracker) |

## How it works

These skills are **thin orchestration** — they contain no financial logic of their own. Every calculation, every tax rule, and 150+ years of historical market data live server-side on the public **planfi MCP** (`https://ai.planfi.app/mcp` — no auth, no API key). A skill gathers your inputs, calls the right tool(s), and shows the result. Any MCP-capable agent gets the same answers; the skills just make it one-click inside Claude Code.

## Why PlanFi Skills

- **Real models, not vibes** — Monte Carlo backtesting on real historical returns, tax-aware (AMT / NIIT / IRMAA), inflation-adjusted.
- **Free & open source (MIT)** — no account, no paywall.
- **Composable** — start with a forecast, then drill into taxes, Social Security, or real estate; every result points to the next best step.

## FAQ

**Is it free?** Yes — MIT-licensed skills plus a public, no-auth MCP.

**Do I need an account or API key?** No.

**Is this financial advice?** No — these are planning estimates to inform your own decisions.

**What is FIRE?** Financial Independence, Retire Early: the point where your investments can cover your spending indefinitely.

**Can I use the tools without Claude Code?** Yes — connect the MCP (`https://ai.planfi.app/mcp`) from any MCP-capable client. The skills just make it turnkey in Claude Code.

---

_Generated from the [`planfi-app`](https://github.com/holdequity/planfi-app) monorepo (source of truth) and mirrored here — don't edit directly; changes flow via CI. MIT licensed. Not financial advice; planning estimates only._
