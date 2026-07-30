# Building Effective Agents — Team Summary

Source: Anthropic Engineering — *Building Effective Agents*
https://www.anthropic.com/engineering/building-effective-agents

---

## TL;DR

**Don't build an agent unless you need one.** Most problems labeled "agent" are solved better, cheaper, and more reliably by a **workflow** with a fixed control flow.

Start with a single LLM call plus retrieval and tools. Add complexity only when it measurably improves results.

---

## The core distinction

| | Workflow | Agent |
|---|---|---|
| Who decides the steps | You (hardcoded) | The LLM (at runtime) |
| Predictability | High | Low |
| Cost | Known upfront | Scales with autonomy |
| Testable | Yes | Harder |
| Use when | You know the steps | You can't know the steps |

---

## The 5 workflow patterns

### 1. Prompt chaining
Fixed sequential steps. Each LLM output feeds the next input. Add programmatic **gates** between steps to kill the chain early on bad output.

**Example — bug report to fix:**
```
Bug report
  → LLM: write reproduction steps
  → [GATE: valid steps? if no, stop]
  → LLM: locate the file
  → LLM: write the patch
```

**Use when:** you already know the steps.
**Note:** GitHub Spec Kit (`/specify → /plan → /tasks → /implement`) is this pattern.

---

### 2. Routing
Classify the input first, then send it to a specialist prompt and toolset.

**Example — support inbox:**
```
Incoming message
  → Classifier LLM: refund / bug / sales / spam
      refund → refund prompt + refund tools (cheap model)
      bug    → engineering prompt + log-search tools
      sales  → sales prompt + CRM tools
      spam   → drop, no LLM call
```

**Use when:** a single mega-prompt makes each category worse because instructions conflict.
**Bonus:** this is the main cost lever — route easy traffic to a cheaper model.

---

### 3. Parallelization
Two distinct flavors.

**Sectioning** — split into independent subtasks, run simultaneously.
> PR review: fire 4 calls at once — security, performance, style, test coverage. Merge results. 4x faster, and each prompt stays focused.

**Voting** — run the same task N times, aggregate.
> "Does this code contain SQL injection?" Run 5x, flag if 2 of 5 say yes. Deliberately low bar because false negatives cost more than false positives.

**Use when:** sectioning for speed and focus; voting for confidence.

---

### 4. Orchestrator–workers
A lead LLM breaks the task into subtasks **at runtime** and delegates to workers.

**Example — "add rate limiting to our API":**
```
Orchestrator LLM: reads codebase, decides
  → "4 files need changes: middleware.ts, config.ts,
     routes/auth.ts, + new Redis client"
  → spawns 1 worker per file, in parallel
Workers: each edits its file
  → Orchestrator: merges, resolves conflicts, verifies
```

**Difference from prompt chaining:** in a chain, *you* decided there are 4 phases. Here *the LLM* decided there are 4 files — could have been 1 or 12.

**Use when:** the subtask list depends on input you can't see at design time.

---

### 5. Evaluator–optimizer
One LLM generates, another critiques, loop until it passes.

**Example — marketing page translation:**
```
Generator: produces Japanese translation
  → Evaluator: "too literal, line 3 loses the pun, tone too formal"
  → Generator: revises
  → loop until pass, or max 3 attempts
```

**Use when:** you can articulate what "bad" looks like in words, AND a second pass measurably improves output.
**Skip it when:** your evaluator can only say "looks fine" — you're paying double for nothing.

---

## When to use a true agent

Open-ended problems where the number of steps can't be predicted and the path can't be hardcoded.

Requirements before you build one:

- **Good tools** — well-documented, hard to misuse
- **Ground-truth feedback** — the agent must be able to check its own work against the real environment (tests, compiler, API response)
- **Stopping conditions** — max iterations, budget caps

Cost scales with autonomy. It has to be worth it.

---

## The three closing principles

1. **Keep the design simple.** Fewest moving parts that works.
2. **Be transparent.** Show the planning steps explicitly. Hidden reasoning is undebuggable.
3. **Invest in the agent–computer interface (ACI).** Write tool descriptions and definitions with the same care you'd write a prompt for a new human developer. **Most agent failures are tool-design failures, not model failures.**

---

## On frameworks

The article deliberately avoids starting with frameworks (LangGraph, etc.). They add abstraction layers that hide the prompts and make debugging harder.

**Recommendation:** use raw API calls first. Understand exactly what's being sent. Add a framework later if you still want one.

---

## Patterns compose

Real systems nest these. Example upgrade path for a Spec Kit-style pipeline:

| Stage | Pattern |
|---|---|
| `/specify` → `/plan` | Prompt chaining |
| `/tasks` → execution | Orchestrator–workers (1 worker per task, parallel) |
| Inside each worker | Evaluator–optimizer against the test suite |

Three patterns, one pipeline. They're composable building blocks, not competing choices.

---

## Decision cheat sheet

| If... | Use |
|---|---|
| Steps are known and sequential | Prompt chaining |
| Input types are distinct | Routing |
| Subtasks are independent | Parallelization (sectioning) |
| One run isn't reliable enough | Parallelization (voting) |
| Subtask list is unknown until runtime | Orchestrator–workers |
| Clear eval criteria + iteration helps | Evaluator–optimizer |
| Steps genuinely can't be predicted | Autonomous agent |
| None of the above clearly applies | **One LLM call. Stop there.** |

---

## Action items for the team

1. **Audit existing pipelines** — for each one, name which pattern it actually is. Most "agents" turn out to be chains.
2. **Find the sequential bottleneck** — any step processing a list one item at a time in a single context is a candidate for orchestrator–workers.
3. **Add gates** — every chain should have programmatic checks between steps that stop execution on bad output.
4. **Review tool descriptions** — treat them as prompts. Test whether a new hire could use the tool correctly from the description alone.
