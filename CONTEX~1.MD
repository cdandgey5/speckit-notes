# Context Engineering for AI Agents — Team Guide

Based on Anthropic's [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) (Sep 2025), plus practical application notes.

---

## The One Idea

**Context is a finite resource with diminishing returns. Find the smallest set of high-signal tokens that gets the outcome you want.**

Not "the model has 200k tokens, use them." More context makes output *worse* past a point.

---

## Why: The Attention Budget

LLMs have an attention budget, and every token spends it.

- Transformer attention is **n² over tokens** — n tokens means n² pairwise relationships. As n grows, attention gets stretched thin.
- Models are **trained mostly on short sequences**. A 200k context window doesn't mean the model saw many 200k-token examples during training. Long-context ability is partly extended after the fact via position-encoding interpolation, not natively learned.
- Result: **context rot** — recall and long-range reasoning degrade as context fills. A gradient, not a cliff.

**Practical implication:** important instructions at the start, immediate task at the end. Middle-of-context is where recall is weakest. Keep facts that need to be related to each other close together.

---

## Prompt Engineering vs. Context Engineering

| | Prompt engineering | Context engineering |
|---|---|---|
| Scope | The instructions you write | The entire token state: system prompt, tools, examples, message history, tool results, retrieved data |
| Timing | One-time authoring | Re-decided every single turn |
| Question | "What words work best?" | "What configuration of context produces the behavior we want?" |

Context engineering is the superset. Prompt engineering is one component of it.

---

## The Four Levers

### 1. System prompts — hit the "right altitude"

Two failure modes, one target between them.

**Too brittle** (hardcoded if-else logic in English):

```
If the user asks about billing, respond with template A.
If they mention "refund" but not "cancel", use flow B.
If they mention both, check whether the account is >30 days old.
If the message contains "angry" or "frustrated", escalate.
```

You're writing a program in prose. Every new case adds a branch. Breaks on anything you didn't enumerate. Unmaintainable.

**Too vague:**

```
You are a helpful customer support agent. Be professional and resolve issues.
```

No concrete signal. The model guesses your policies.

**Right altitude** — constraints and heuristics, model reasons within them:

```xml
<role>
You handle billing support for [product]. You can issue credits up to $50
without approval; anything higher goes to a human.
</role>

<principles>
- Resolve in one turn when you have enough information. Ask at most one
  clarifying question.
- Prefer refunds over credits when the charge was our error.
- Escalate rather than guess on anything involving account deletion.
</principles>

<output>
Plain response to the user. If escalating, end with ESCALATE: <reason>.
</output>
```

**Method:** Start with a minimal prompt on the **strongest available model**. Run it. Add one instruction per *observed* failure. Never add rules speculatively.

> Why strongest model first: on a weak model you can't tell if a failure is the prompt's fault or the model's. You end up padding the prompt to compensate for capability — and that bloat is permanent.

Minimal ≠ short. It means no token that isn't earning its place.

### 2. Tools — minimal and distinct, not many-with-a-routing-guide

**The test:** if a human engineer can't say definitively which tool applies in a given situation, the agent can't either.

A routing paragraph doesn't fix ambiguity — it relocates the confusion into the prompt and charges you tokens on every turn.

Rules:

- Every tool schema sits in context on **every turn**. 30 tools is a permanent tax.
- Merge overlapping tools. `search_users` + `find_user_by_email` + `lookup_account` → one `find_user`.
- If two tools are always called together, make them one tool.
- Return values must be token-efficient. A tool returning 5k tokens of JSON when 200 would do is a context leak that fires repeatedly.
- Disambiguate in the **tool description**, not a separate guide: *"Use for X. Do NOT use for Y — use `other_tool` instead."*

In Claude Code: disable MCP servers you aren't using for the current task.

### 3. Examples — diverse and canonical, not exhaustive

Few-shot prompting works. Stuffing every edge case does not — that's the if-else problem in example form.

Aim for **3–5 examples that span different behaviors**, not 20 variations of one behavior.

```xml
<examples>
<example>
<input>Our API returns 500s intermittently on POST /orders.</input>
<output>
Problem: Intermittent 500s on POST /orders.
Likely cause: unhandled DB constraint violation under concurrent writes.
Next step: check application logs for stack traces on the failing requests.
</output>
</example>

<example>
<input>How do I center a div?</input>
<output>
Not a debugging request — answering directly.
`display: flex; justify-content: center; align-items: center;` on the parent.
</output>
</example>
</examples>
```

The second example demonstrates a **different mode** (short-circuit for non-debugging questions), not another debugging variant. That's what diversity means here.

### 4. Retrieval — just-in-time beats pre-loading

Don't dump data into context up front. Keep **lightweight identifiers** (file paths, saved queries, URLs) and load content at runtime via tools.

This mirrors how people work — you don't memorize a filesystem, you `grep` it.

Free signal comes from metadata: folder hierarchy, naming conventions, file sizes, timestamps. `tests/test_utils.py` means something different from `src/core_logic/test_utils.py`.

**The hybrid pattern** (what Claude Code does): a small amount of orientation up front + on-demand retrieval tools.

`CLAUDE.md` is the **map, not the territory** — where things live and how to find them, never the things themselves:

```markdown
## Layout
- `src/api/`    — HTTP handlers. One file per resource.
- `src/core/`   — business logic. No framework imports here.
- `migrations/` — timestamped, never edit an applied one.

## Finding things
- DB schema: `migrations/` (latest wins) or run `npm run schema:dump`
- Env vars: `.env.example` is authoritative
- Test one file: `npm test -- <path>`

## Conventions
- Errors bubble to `src/api/errorHandler.ts`. Don't catch-and-log locally.
```

~150 tokens that unlocks 50k tokens of material on demand — instead of pasting 50k tokens and hoping the right 2k are in there.

**Keep `CLAUDE.md` short.** It loads every session. A 3,000-token `CLAUDE.md` has reintroduced the exact problem you're solving.

Tradeoff to be honest about: runtime exploration is slower than precomputed retrieval, and a poorly-guided agent can waste context chasing dead ends.

---

## Long-Horizon Tasks: Three Techniques

When work exceeds the context window.

### Compaction

Summarize the conversation so far, restart the context window with that summary. **Same continuing session** — mechanically a fresh window, seamless from your side.

Keep: architectural decisions, unresolved bugs, implementation details, recently-touched files.
Discard: redundant tool outputs, superseded messages, stale results.

In Claude Code: automatic near the limit, or run `/compact`. Steerable — `/compact focus on the auth refactor decisions and the failing test`.

- Compact **before** you're forced to. Auto-compaction at 95% full has less to work with.
- Compact at natural boundaries — subtask finished, switching files. Never mid-debug.
- Tuning a compaction prompt: maximize recall first, then trim for precision.

**`/clear` vs `/compact`:** related work → compact. Genuinely new work → clear. Carrying a summary of irrelevant work is pure cost.

**Best for:** tasks with extensive back-and-forth in one continuous line of work.

### Structured note-taking (agentic memory)

The agent writes notes to a file outside the context window and reads them back later. Three distinct purposes:

1. **Working memory during the task** — a checklist keeps the agent on-goal across 40 tool calls in the *same* session. This is most of the value.
2. **Surviving context resets** — after compaction, the agent re-reads its own notes.
3. **Cross-session state** — resume tomorrow where you stopped.

Not "write the final output as notes." It's scratch state maintained *while working*: what's done, what's blocked, what was decided and why.

Prompt to start a long task:

```
Before starting, create NOTES.md with the plan as a checklist.
Update it after each step: mark done, record decisions and any surprises.
If we lose context, that file is how you resume.
```

**Decisions and dead ends matter more than status.** "Tried X, failed because Y" stops the agent from retrying X after a reset.

Reference point: Claude playing Pokémon maintained maps, objectives, and combat strategy notes across thousands of steps and multiple context resets.

**Best for:** iterative development with clear milestones.

### Sub-agent architectures

A lead agent holds the high-level plan. Specialized sub-agents do deep work in **clean, isolated context windows** and return only a distilled summary.

A sub-agent may burn 30k+ tokens exploring and return 1,000–2,000 tokens. The search context stays isolated; the lead agent's context stays clean.

**Don't use when subtasks depend on each other** — sub-agents can't see each other's work.

**Best for:** parallel research and analysis where independent exploration pays off.

---

## Context Rot: Detection and Prevention

**Definition:** as context grows, the model's ability to accurately recall and use information *already in that context* degrades. The information is present; attention over it is diluted.

### Symptoms

- Re-reads a file it already read
- Reverts a decision made earlier in the conversation
- Reintroduces a bug you already fixed together
- Answers get generic — pattern-matched instead of grounded in your specifics
- "Forgets" a constraint stated once, 60k tokens ago
- Asks something you already answered

### Prevention

- Watch the context indicator. Treat **~50% as "start planning a boundary."**
- **Restate critical constraints near the current task.** They decay with distance. Repetition is cheap and effective.
- One task per session. Long catch-all sessions are the main cause.
- Prefer new sessions over long ones — cheaper to re-establish context than to fight rot.
- **Two symptoms = stop arguing with the model.** Compact or clear. Correcting inside a rotted context adds tokens and rarely works.

---

## Team Checklist

Do these now:

- [ ] **Audit `CLAUDE.md`.** Over ~100 lines? Cut to layout map + commands + conventions. Delete anything the agent could find with grep.
- [ ] **Audit enabled tools/MCP servers.** Turn off what isn't needed for current work. Merge overlapping tools.
- [ ] **Review system prompts** for hardcoded if-else logic. Replace branches with constraints + heuristics.
- [ ] **Trim example sets** to 3–5 canonical, behaviorally-diverse cases.
- [ ] **Adopt `NOTES.md`** on any task longer than ~30 minutes.
- [ ] **Build the `/clear`-on-task-switch habit** instead of one long-running session.
- [ ] **Set a team norm:** compact at 50–60% context, not at the wall.

---

## Quick Reference

| Situation | Do this |
|---|---|
| Long continuous task, lots of back-and-forth | Compaction |
| Iterative work with milestones | Structured note-taking |
| Parallel, independent research | Sub-agents |
| Switching to unrelated work | `/clear`, not `/compact` |
| Prompt isn't working | Add one rule per observed failure, on the strongest model |
| Agent picks the wrong tool | Merge or remove tools — don't add routing prose |
| Need project context | Map in `CLAUDE.md` + retrieval tools, not pasted content |
| Model seems to be "forgetting" | Context rot — compact or clear, don't argue |

---

*Source: [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — Anthropic Applied AI team, Sep 29, 2025.*
