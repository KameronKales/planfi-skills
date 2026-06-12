---
name: finance-explainer
version: 1.0.0
description: >-
  Turn the depth of the planfi financial toolset into ONE long-form "noob → seasoned expert" blog
  post by fanning out a workflow of agents that research, distill, architect a table of contents, and
  write. Runs in two modes: a BROAD whole-of-personal-finance guide, or a DEEP single-topic deep dive
  (e.g. "529 college investing", "Roth conversions", "rent vs buy"). Invoke when someone says "write a
  beginner-to-expert personal finance guide", "distill all the planfi tools into a blog post", "write
  a deep-dive post on <topic>", "explain <topic> from scratch to mastery", "run finance explainer on
  529 investing", or wants a single cohesive long-read that takes a complete beginner to confident.
  Produces a finished markdown article.
---

# Finance Explainer — distill the toolset into a noob→expert guide

A content-generation skill. It fans out a **Workflow** of agents over the planfi tool catalog to
write **one long-form blog post** that walks a complete beginner all the way to near-expert. The
planfi engine/MCP is the source of truth for the *depth*; the agents translate that depth into
plain-English teaching prose. The output is a finished markdown article — it does **not** change any
user data and invents no figures (illustrative examples only; the real math stays in planfi).

It runs in **two modes**, auto-selected by whether you pass a `topic`:
- **BROAD** (no `topic`) — one post spanning the whole personal-finance journey across all ~11 domain
  clusters (the showcase guide).
- **DEEP** (`topic` set) — a focused deep dive on ONE topic (e.g. "529 college investing"). A scoping
  agent reads the catalog, picks the tools relevant to that topic, and derives the sub-angle clusters;
  the SAME distill → architect → write → edit pipeline then produces a tightly-scoped article. This is
  the reusable mode for per-topic content flows.

This skill is the user's explicit opt-in to multi-agent orchestration — run the workflow; don't try
to write the whole guide solo in the main loop.

## When to use vs. not
- **Use (broad)** for a single cohesive long-read that teaches personal finance start-to-mastery, or
  to showcase the breadth of planfi's ~94 tools as an educational narrative.
- **Use (deep)** when the user names a topic — "write a post on 529 investing", "deep dive on Roth
  conversions", "explain rent vs buy from scratch to mastery". Pass it as `topic`.
- **Don't use** for analyzing a specific person's plan (that's `comprehensive-plan` /
  `financial-forecast`) or for a short LinkedIn post (that's the `linkedin-finance-marketer` agent).

## Step 0 — Confirm scope (ask only if the user left it open)
Optional inputs — all have defaults, so you can run cold:
- **topic** — set this for a DEEP single-topic deep dive (e.g. `"529 college investing"`). Omit for a
  BROAD whole-of-personal-finance guide. If the user named a subject, pass it here.
- **title** — the post title (defaults sensibly per mode).
- **audience** — who it's for (default: "a smart, motivated beginner who knows almost nothing").
- **focus** — narrow the angle further (default: the whole journey, or "master `<topic>`" in deep mode).
- **out** — where to save the finished markdown (default: `drafts/finance-explainer-guide.md`).

If the user named a topic/title/audience/angle, pass it through; otherwise take the defaults and proceed.

## Progress updates (Discord)
Keep the user updated in Discord, the same way Feature Factory does. Read the webhook URL from your
memory/config (the `discord-webhook` reference) — **never hardcode it**. The workflow runs in the
background, so post at the checkpoints the main loop controls, via
`curl -H 'Content-Type: application/json' -d '{"content":"..."}' <webhook>`:
- **Kickoff** — when you launch the workflow (mode + topic/title, e.g. "🚀 Finance Explainer: deep dive on Roth conversions started").
- **Done** — after Step 2 saves the article: title, mode, chapter count, and the saved file path.
If no webhook is in memory, skip silently (don't block the run).

## Step 1 — Run the workflow (PRIMARY ACTION)
Invoke the saved workflow, passing whatever the user specified:

```
// Broad guide:
Workflow({ name: 'finance-explainer', args: { root: '/Users/kameronkales/planfi-app' }})

// Deep single-topic dive:
Workflow({ name: 'finance-explainer', args: {
  topic: '529 college investing',
  audience: '<from user, or omit>',
  root: '/Users/kameronkales/planfi-app'
}})
```

What it does (one well-scoped fan-out):
0. **Scope** *(deep mode only)* — a scoping agent reads `public/llms-full.txt`, selects the tools
   relevant to the `topic`, and derives 4–8 ordered sub-angle clusters (basics → advanced levers →
   pitfalls). In broad mode this phase is skipped and the fixed 11 domain clusters are used.
1. **Distill** — one agent per cluster reads the catalog (encoding the real engine depth) and produces
   beginner-true notes: core concepts, the advanced "levers," jargon-in-plain-English, common pitfalls.
2. **Architect** — one editor agent synthesizes all distillations into a progressive table of
   contents ordered beginner → advanced, each chapter with a learning goal.
3. **Write** — one agent per chapter drafts the prose from the TOC + only its relevant distillations.
4. **Edit** — one agent stitches the chapters into a single cohesive post (intro, linked TOC,
   smoothed transitions, glossary, planfi CTA, not-advice disclaimer).

The workflow returns `{ title, topic, mode, audience, chapters, toc, markdown }`.

## Step 2 — Save and present
1. **Write** `result.markdown` to the `out` path. Default to `drafts/finance-explainer-guide.md` for
   broad runs, or `drafts/<topic-slug>-guide.md` for deep runs (slugify `result.topic`). Create the
   `drafts/` dir if needed.
2. **Summarize** for the user: the title, `result.mode`, chapter count, and the table of contents
   (`result.toc`), then offer the saved file path. Optionally send it with `SendUserFile`.
3. **Offer follow-ups**: re-run with a different `focus`/`audience`, expand a chapter, or hand the
   draft to the `retirement-copywriter` agent for a copy polish / the `llm-seo-optimizer` agent for
   search visibility.

## Guardrails (bake into any prompt you add)
- **No invented numbers or rules of thumb.** Agents describe *what the planfi tools let someone
  decide and why it matters*; concrete figures must be clearly labelled as illustrative examples,
  never guarantees or personalized advice. The engine is the source of truth for real math.
- **Plain language.** Every term is defined on first use; the whole point is noob → expert.
- **One cohesive read**, not stitched fragments — the Edit phase enforces this; preserve it.
- **Not financial advice.** The post ends with that disclaimer; keep it.

## Notes
- Iterating on the workflow? Edit `.claude/workflows/finance-explainer.js` and re-invoke
  `Workflow({ name: 'finance-explainer', ... })`; clusters are defined at the top of that file —
  add/retune them as the toolset grows.
- The catalog (`public/llms.txt` / `public/llms-full.txt`) is regenerated as tools ship, so the
  guide stays current with the engine's real depth on each run.
