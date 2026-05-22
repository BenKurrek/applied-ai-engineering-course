# Topic 08 — Agents & Harnesses — Course Material

> Authored source material for the Applied AI Engineering course. Tutors teach from
> this file. Do not record student progress here.

## How to use

This file is taught sub-chapter by sub-chapter, in order. Each sub-chapter has core
teaching content (**Concept**), a list of **Key terms**, **Common misconceptions** to
pre-empt, a **Worked example** that makes the idea concrete, and **Check questions** the
tutor uses to confirm understanding before moving on. The **Exam Question Bank** at the
end is the pool the tutor draws from for the gated, closed-book exam (scored /100, pass
mark 85). This topic builds directly on Topic 7 (Tool Use): an agent is a loop wrapped
around the tool-use mechanism. Topic 8 is a core interview area — a graduate should be
able to whiteboard a harness end to end.

---

## 8.1 — What makes something an agent vs. a single LLM call

### Concept

A **single LLM call** is one shot: prompt in, completion out. It is stateless,
deterministic in structure (one request, one response), and the model has no ability to
*do* anything beyond producing text. Even prompt chains — call A's output feeds call B —
are still just a fixed, developer-defined pipeline. The *control flow is decided by you*,
in advance.

An **agent** is different in one decisive way: **the model itself decides the control
flow.** An agent is an LLM running in a **loop**, equipped with **tools**, **observing
the results** of its actions, and continuing until a **termination condition** is met.
The model chooses what to do next based on what it has seen so far — the sequence of
steps is not predetermined.

The minimal definition has three required ingredients:

1. **A loop** — the model is invoked repeatedly, not once.
2. **Tools** — the model can take actions that affect the world or gather information
   (Topic 7), and *feed the results back in*.
3. **Model-driven control** — the model decides each next step (which tool, whether to
   continue, when it's done). If a human or fixed script decides every step, it is a
   workflow, not an agent.

Anthropic frames a useful distinction: **workflows** are "systems where LLMs and tools
are orchestrated through predefined code paths," whereas **agents** are "systems where
LLMs dynamically direct their own processes and tool usage, maintaining control over how
they accomplish tasks." [1] Both are valid; agents trade predictability for flexibility.
(Note this workflow/agent split is one influential framing — Anthropic's — not a
universal industry definition; other sources draw the line differently.)

Why does the distinction matter? Because the properties change completely:

- **Cost/latency** — a single call is bounded and predictable. An agent runs an unknown
  number of model calls; cost and latency are *variable* and must be bounded explicitly
  (8.9).
- **Failure modes** — a single call can be wrong; an agent can *loop forever*, drift off
  goal, cascade errors, or overflow its context (8.8). New, harder failure modes.
- **Reproducibility** — even a single call at temperature 0 is *not* guaranteed
  deterministic. Temperature 0 makes sampling greedy (always take the argmax token), but
  the *logits* themselves can shift run to run: floating-point matrix multiplication is
  **not associative**, so a different batch size, sequence length, or GPU work-split
  reorders the additions and changes low-order bits; **MoE models** route tokens to
  experts in a way that depends on what *else* is in the serving batch; and providers
  silently change kernels, hardware, and model snapshots. So temp-0 is *low-variance*, not
  deterministic — see Topic 3.7 for the full treatment. An agent run then compounds this:
  it involves many sampled steps *and* external tool results, so **two runs of the same
  task rarely produce the same trajectory.** This makes agents hard to test and debug.
- **Evaluation** — you can't just check one output; you may need to evaluate the whole
  *trajectory* (8.11).

The practical guidance: **don't build an agent when a workflow will do.** Agents shine
when the task is open-ended, the number of steps can't be predicted, and the path
genuinely depends on intermediate results (e.g. a coding task, a research task). For
well-understood tasks with a fixed shape, a deterministic workflow is cheaper, faster,
and far more reliable.

### Key terms

- **Single LLM call** — one prompt-to-completion request; no loop, no model-driven control.
- **Workflow** — LLMs orchestrated through *developer-defined*, predetermined code paths.
- **Agent** — an LLM in a loop with tools, observing results, where *the model* decides each next step until a termination condition.
- **Control flow** — the sequence of steps taken; in an agent it is decided by the model, not hardcoded.

### Common misconceptions

- ❌ "Any LLM call that uses a tool is an agent." → ✅ A single tool call is not an agent. An agent needs a loop, observation of results, and model-driven control over what happens next.
- ❌ "Agents are always better than workflows." → ✅ Agents trade predictability, cost-control, and reproducibility for flexibility. For fixed-shape tasks a workflow is better.
- ❌ "A prompt chain is an agent." → ✅ A fixed chain has developer-defined control flow; the model doesn't decide the path, so it is a workflow.

### Worked example

Task: "Summarize this document." → single LLM call. Fixed shape, one step.

Task: "Translate this document, then check the translation for tone." → two-call
workflow. Two steps, but *you* decided there are exactly two and what they are.

Task: "Investigate why the nightly build is failing and fix it." → agent. The model must
read logs, decide which file to inspect, run tests, interpret failures, and iterate. The
number and order of steps is unknowable in advance and depends entirely on what it finds.

### Check questions

1. A pipeline calls model A to classify a ticket, then routes to one of three pre-written model-B prompts based on the class, then returns. It uses tools and runs the model twice. Is it an agent? Justify using the decisive criterion. — **Answer:** No — it is a workflow. Using tools and running the model more than once is not sufficient; the decisive criterion is *who decides the control flow*. Here the developer hardcoded the path (classify → fixed branch → return); the model never decides what to do next. An agent requires model-driven control over each next step, not just a multi-step pipeline.
2. An agent has a loop and model-driven control, but its tools' results are discarded and never appended to the context. Why does this fail to be a functioning agent even though two of the three ingredients are present? — **Answer:** The third ingredient — feeding tool results back so the model *observes* the consequences of its actions — is missing. Without observation the model cannot adapt its next decision to what happened; it is acting blind. The loop, tools, and model-driven control only produce agency when the model can see and react to results.
3. A teammate runs a new agent once on a task, it succeeds, and they conclude it is ready to ship. Explain, from the property that makes agents hard to test, why this conclusion is unsound. — **Answer:** Agents are non-deterministic — each run involves many sampled model steps and live external tool results, so two runs of the same task rarely follow the same trajectory. One success is a single sample from a distribution; a failure mode that appears in, say, 1 run of 10 is real but invisible in one run. Readiness needs a pass-rate over many runs (8.11), not a single happy-path result.

---

## 8.2 — The agentic loop: think → act → observe → repeat

### Concept

Every agent, regardless of framework, runs the same fundamental cycle:

1. **Think** — the model receives the current context (goal + history so far) and reasons
   about what to do next. It may emit reasoning text and then a decision.
2. **Act** — the model emits a tool-use request (Topic 7). The harness executes the
   corresponding tool.
3. **Observe** — the tool's result is captured and appended to the context as a
   tool_result. The model now "sees" the consequence of its action.
4. **Repeat** — the loop runs again with the enlarged context. Each iteration the context
   grows by one think-act-observe cycle.

The loop **terminates** when the model stops requesting tools and produces a final answer
(`stop_reason: end_turn`), or when a harness-level limit trips (8.9).

This is the heart of agency. It is a tight feedback loop: the agent does not plan the
whole task and execute blindly — it takes one step, *sees what happened*, and decides the
next step in light of that. This is what lets an agent recover from surprises, adapt to
errors, and handle tasks whose shape it could not have known in advance.

Mechanically, the loop is just the Topic 7.2 tool-use round-trip, repeated:

```
context = [system_prompt, user_goal]
while True:
    response = model(context, tools)          # THINK
    if response.stop_reason == "end_turn":
        return response                       # done
    context.append(response)                  # echo assistant turn
    for call in response.tool_use_blocks:     # ACT
        result = execute(call)
        context.append(tool_result(call.id, result))   # OBSERVE
    # loop → REPEAT
```

Two things to internalize:

- **The context is the agent's entire working memory.** Everything the agent "knows" at
  step N is whatever is in the context: the goal, every prior thought, every action,
  every observation. There is no hidden state. This is why context management (8.6) is
  central — the context both *grows every iteration* and is *all the memory there is*.
- **The loop is a feedback loop, not a script.** The model re-decides every iteration.
  This adaptivity is the strength; the unbounded, non-deterministic nature is the cost.

The "think" step is sometimes implicit (the model just emits a tool call) and sometimes
explicit (the model emits reasoning text first, or, with reasoning models, an internal
thinking block). Making the thinking explicit is the core idea of ReAct (8.3).

### Key terms

- **Agentic loop** — the repeated think → act → observe cycle that constitutes an agent's execution.
- **Think / Act / Observe** — reason about the next step / execute a tool / feed the result back into context.
- **Iteration / step / turn** — one pass through the loop; one model call plus its tool execution.
- **Working memory** — the agent's context window; the *only* state it has across iterations.

### Common misconceptions

- ❌ "The agent plans everything up front, then executes." → ✅ The defining feature of the loop is that the agent acts one step at a time and re-decides after each observation. (Upfront planning is one *option* — 8.4.)
- ❌ "The loop maintains hidden state between iterations." → ✅ All state is in the context window; there is no hidden memory. Anything not in the context is forgotten.
- ❌ "The loop ends when the task is done." → ✅ The loop ends when the *model* stops requesting tools (or a limit trips). A model can wrongly stop early or never stop — termination is not automatic correctness.

### Worked example

Goal: "What is the population of the capital of France?" A reasoning model agent:

- **Think:** "I need the capital of France, then its population." → **Act:** calls
  `search("capital of France")`. → **Observe:** "Paris."
- **Think:** "Now I need Paris's population." → **Act:** calls
  `search("population of Paris")`. → **Observe:** "~2.1 million (city proper)."
- **Think:** "I have the answer." → emits final text, `end_turn`. Loop terminates.

Three iterations, each one informed by the previous observation. The second search query
*depended* on the first observation — which is exactly why it could not have been a
single planned-ahead call.

### Check questions

1. Someone proposes an "optimization": have the model produce the full sequence of tool calls in one shot, then have the harness execute them all, skipping the per-step model calls. Which phase of the agentic loop does this delete, and what capability is lost? — **Answer:** It deletes the **Observe** phase (and the re-Think that follows it). The agent can no longer see the result of one action before choosing the next, so it cannot adapt to surprises, recover from errors, or handle a task whose later steps depend on earlier results. The loop's whole value is the feedback — it is a feedback loop, not a pre-planned script.
2. During a long run an agent "forgets" a fact it discovered 30 steps earlier and re-derives it. Given how agent state works, explain mechanically why this happens and what it implies for harness design. — **Answer:** An agent has no hidden state — everything it "knows" is whatever is in the context window. If the earlier fact scrolled out of effective range (or was evicted/compacted away), it is simply gone from the model's working memory. This is why context management (8.6) is a core harness job: what to keep, summarize, evict, or re-retrieve directly determines what the agent can still "remember."
3. An agent emits `end_turn` and the loop stops, but the task is not actually complete. Is this a bug in the loop's termination logic? Explain what `end_turn` does and does not guarantee. — **Answer:** It is not a loop bug — the loop is *designed* to stop when the model stops requesting tools (`end_turn`) or a limit trips. `end_turn` guarantees only that the model ended its turn with text; it does *not* guarantee task completion — the model can stop early or hallucinate completion. The fix is not in the loop mechanics but in adding success verification (8.9) on top of the natural exit.

---

## 8.3 — ReAct (reason + act)

### Concept

**ReAct** — short for **Reason + Act** — is the foundational agent pattern, from the 2022
paper *ReAct: Synergizing Reasoning and Acting in Language Models* (Yao et al., arXiv
2210.03629). [2] Its core insight: instead of having the model *either* reason *or* act,
**interleave** the two. At each step the model produces an explicit **thought**
(reasoning in natural language) and *then* an **action** (a tool call), sees the
**observation**, and repeats: Thought → Action → Observation → Thought → ...

Why interleaving helps:

- **Reasoning before acting** — the explicit thought lets the model plan the immediate
  next step, decompose the problem, and decide *which* tool and *why*. This is
  chain-of-thought (Topic 5.3) applied inside the loop.
- **Acting grounds the reasoning** — the observation injects real, external facts back
  in, correcting the model when its reasoning drifts from reality. Pure chain-of-thought
  with no actions can confidently reason itself into a wrong answer (hallucinated facts);
  pure acting with no reasoning fumbles tool selection. ReAct couples them so each fixes
  the other's weakness.
- **It produces a legible trace** — the thought/action/observation log is human-readable,
  which makes debugging and trajectory evaluation (8.11) far easier.

The classic ReAct format is a text trace:

```
Thought: I need to find when the Eiffel Tower was completed.
Action: search("Eiffel Tower completion year")
Observation: The Eiffel Tower was completed in 1889.
Thought: The question also asks its height. I should search that.
Action: search("Eiffel Tower height")
Observation: 330 metres.
Thought: I have both facts. I can answer.
Answer: Completed 1889, 330 m tall.
```

**ReAct today.** The original paper relied on prompting the model to emit that literal
text format, often with few-shot examples. Modern tool-use APIs (Topic 7) implement ReAct
*natively*: the `tool_use`/`tool_result` mechanism *is* the Action/Observation loop, and
the model's text (or a reasoning model's thinking block) *is* the Thought. You rarely
hand-write the ReAct prompt format anymore — but the *pattern* (interleaved reasoning and
acting, grounded by observations) is exactly what every modern agent does. ReAct is the
conceptual ancestor of the agentic loop in 8.2.

A related point: **reasoning models** (extended-thinking models, Topic 14.6) changed the
balance — they do substantial reasoning internally, so the "Thought" step is partly
inside the model. But the interleaving with real observations is still essential;
internal reasoning alone cannot ground itself in external facts.

### Key terms

- **ReAct (Reason + Act)** — the agent pattern that interleaves explicit reasoning (Thought) with tool actions (Action) and their results (Observation).
- **Thought / Action / Observation** — the three repeating elements of a ReAct trace.
- **Grounding** — using real observations from actions to keep the model's reasoning anchored to reality.

### Common misconceptions

- ❌ "ReAct requires the literal `Thought:/Action:/Observation:` text format." → ✅ That was the original prompting technique; modern tool-use APIs implement the same pattern natively. ReAct is the *pattern*, not the format.
- ❌ "ReAct is just chain-of-thought." → ✅ Chain-of-thought is reasoning only. ReAct *interleaves* reasoning with real actions, so observations can correct the reasoning.
- ❌ "Reasoning models made ReAct obsolete." → ✅ They moved the 'reason' step partly inside the model, but interleaving with external observations is still required — internal reasoning can't ground itself in outside facts.

### Worked example

Without ReAct (pure reasoning): asked "What's the current weather in Denver?", the model
*reasons*: "Denver is high-altitude, probably cool and dry, likely around 15°C." —
plausible-sounding, possibly wrong, and ungrounded.

With ReAct: **Thought:** "I cannot know current weather; I must look it up." →
**Action:** `get_weather("Denver")` → **Observation:** "2°C, snowing." → **Thought:** "I
have the real value." → **Answer:** "2°C and snowing." The action grounded the reasoning
and corrected a confident-but-wrong guess.

### Check questions

1. Describe the specific failure mode of an agent that *only acts* (no reasoning step) and the specific failure mode of one that *only reasons* (pure chain-of-thought, no actions). How does ReAct's interleaving address each? — **Answer:** Act-only fumbles tool selection — with no thought it picks wrong tools or wrong arguments because it never plans which tool and why. Reason-only can confidently hallucinate — pure chain-of-thought has no external facts, so it can reason itself fluently into a wrong answer. ReAct interleaves them: the Thought improves tool selection/planning, and the Observation from each Action injects real facts that correct drifting reasoning. Each leg fixes the other's failure.
2. A developer reads the 2022 ReAct paper, sees the `Thought:/Action:/Observation:` text format, and concludes modern agents must prompt the model to emit exactly that format. Why is this conclusion outdated, and what is the right way to think about ReAct today? — **Answer:** The literal text format was the original *prompting technique*. Modern tool-use APIs implement the same pattern natively — the `tool_use`/`tool_result` mechanism *is* the Action/Observation loop, and the model's text or reasoning block *is* the Thought. ReAct today is the underlying *pattern* (interleaved reasoning and acting grounded by observations), not a required prompt string.
3. "Reasoning models do extended internal thinking, so ReAct's reasoning step is now redundant." Assess this claim. — **Answer:** Partly true, mostly wrong. Reasoning models do move the "reason" step partly inside the model, so you rely less on explicit Thought text. But the *interleaving with real observations* is not redundant — internal reasoning, however extensive, cannot ground itself in external facts. Without Actions feeding back real results, even a strong reasoner can confidently produce ungrounded answers. ReAct's act-observe coupling remains essential.

---

## 8.4 — Planning: upfront vs. interleaved; decomposition; subagents

### Concept

For non-trivial tasks the agent needs a **plan** — a strategy for how to get from goal to
done. There are two ends of a spectrum:

**Upfront (plan-then-execute).** The agent first produces a complete plan — an ordered
list of steps — and then executes them. *Pros:* the plan is legible, reviewable (a human
or another check can vet it before any action), and gives structure to a long task;
keeps the agent on track and reduces drift. *Cons:* it is brittle — the world rarely
matches the plan; step 3 may reveal that steps 4–7 were wrong. A pure upfront plan
followed blindly handles surprises badly.

**Interleaved (plan-as-you-go).** The agent plans the immediate next step, acts, observes,
then re-plans. This is the ReAct style (8.3). *Pros:* maximally adaptive — every decision
uses the latest information. *Cons:* the agent can lose the thread of a long task,
"forget" the overall goal, or wander (goal drift, 8.8).

In practice, good agents **combine both**: form a rough upfront plan for structure and
direction, then execute interleaved, **re-planning when reality diverges** from the plan.
A common concrete pattern is a maintained todo list — the agent writes a plan, works
through it, and updates it as it learns (this is what coding agents like Claude Code do).

**Decomposition** is the skill underneath planning: breaking a large, vague goal into
smaller, concrete, individually-achievable subtasks. Good decomposition makes each step
tractable and makes progress measurable. Poor decomposition (steps too big, too vague, or
wrongly ordered) is a leading cause of agent failure.

**Subagents.** A powerful decomposition technique: instead of one agent doing everything,
an **orchestrator** agent spawns **subagents**, each handling one subtask, often with its
own fresh context and a focused tool set. Why this helps:

- **Context isolation** — each subagent works in a clean, focused context window. The
  orchestrator never sees the subagent's noisy intermediate steps, only its final result.
  This is the biggest win: it fights context bloat and "lost in the middle."
- **Specialization** — a subagent can have a tailored prompt and a minimal tool set for
  its job (e.g. a "search" subagent, a "code-writing" subagent).
- **Parallelism** — independent subtasks can run as concurrent subagents.

The cost: more model calls (more tokens, more latency), coordination overhead, and the
risk that information needed by one subagent is stranded in another's context. Subagents
help most when subtasks are *separable* and each generates a lot of intermediate context
the parent doesn't need. (Full multi-agent treatment in 8.7.)

**Reflection and self-critique.** ReAct (8.3) interleaves *acting* with reasoning, but it
does not, on its own, make the agent step back and ask "is what I just produced actually
good?" **Reflection** patterns add exactly that: an explicit step where the agent (or a
separate critic) **evaluates its own recent output or trajectory, identifies problems,
and feeds that critique back in** before continuing or retrying. It is planning applied
*backwards* — re-planning informed by a judgment of what went wrong.

Three patterns, increasingly structured:

- **Self-critique** — after producing a result, the agent is prompted to critique it
  ("review the draft above; list errors and weaknesses"), then revise. A single
  generate → critique → revise pass.
- **Verifier / generate-and-check** — a *separate* check decides whether the output is
  acceptable. The verifier can be **deterministic** (run the tests, validate the schema,
  execute the code) or an **LLM-as-judge** (Topic 9.5). Deterministic verifiers are far
  more trustworthy when available — a passing test suite is ground truth; a model
  critiquing itself can be wrong or sycophantic.
- **Reflexion-style loops** — from the *Reflexion* paper (Shinn et al., 2023 [6]): the
  agent attempts the task, a verifier produces a signal (pass/fail, error output), and
  the agent writes a **natural-language reflection** on *why* it failed — that reflection
  is added to the context for the **next attempt**, so the agent learns from the failure
  within the run without any weight update. It is iterative: attempt → evaluate →
  reflect → retry.

**Why reflection helps:** it catches error cascades (8.8) early — a bad intermediate
result is criticized and corrected *before* later steps build on it — and it turns a
verifier signal into a concrete improvement rather than a blind retry.

**The costs and limits.** Reflection multiplies model calls (every critique and retry is
more tokens and latency — relevant to the cost model in 8.12). And **self-critique with
no external verifier is weak**: a model judging its own work shares the blind spots that
produced the work, and can "reflect" itself into a confident wrong answer or loop on a
non-improvement. Reflection is strongest when the critique signal comes from *outside* the
model — a real test run, a schema validator, a tool result — i.e. when it is *grounded*,
the same lesson as ReAct. Use a deterministic verifier whenever the task admits one.

### Key terms

- **Upfront planning (plan-then-execute)** — produce a full plan first, then execute it.
- **Interleaved planning** — plan the next step, act, observe, re-plan; the ReAct style.
- **Decomposition** — breaking a large goal into smaller, concrete, achievable subtasks.
- **Subagent** — a child agent spawned to handle a subtask, typically with its own fresh context and focused tools.
- **Orchestrator** — the parent agent that decomposes the task, spawns subagents, and integrates their results.
- **Reflection / self-critique** — an explicit step where the agent evaluates its own recent output, identifies problems, and feeds that critique back before continuing or retrying.
- **Verifier** — a separate check (deterministic — tests, schema validation; or an LLM-as-judge) that decides whether output is acceptable.
- **Reflexion** — a pattern (Shinn et al., 2023) where a verifier signal drives a natural-language reflection that is added to context for the next attempt; iterative attempt → evaluate → reflect → retry.

### Common misconceptions

- ❌ "A good agent makes a perfect plan and follows it." → ✅ Plans rarely survive contact with reality; the best agents re-plan as they learn. Pure plan-then-execute is brittle.
- ❌ "Subagents are mainly about parallel speed." → ✅ The biggest benefit is *context isolation* — keeping each subagent's noisy intermediate work out of the parent's context. Parallelism is a secondary bonus.
- ❌ "More decomposition is always better." → ✅ Over-decomposition adds coordination overhead and call cost; steps that are too granular waste tokens and lose the thread.
- ❌ "Self-critique always improves the output." → ✅ A model critiquing its own work shares the blind spots that produced it; reflection is reliable only when the critique signal is *grounded* — from a deterministic verifier (tests, schema) or a real tool result — not the model marking its own homework.
- ❌ "Reflexion retrains or updates the model." → ✅ No weights change. Reflexion adds a natural-language reflection on the failure *to the context* for the next attempt — in-context learning within the run, not fine-tuning.

### Worked example

Task: "Write a competitive analysis of three products." A monolithic agent does all the
research and writing in one context — by the time it writes, the context is bloated with
research dumps for all three products and quality degrades.

Subagent design: an **orchestrator** decomposes into "research product A/B/C" + "synthesize
report." It spawns three research **subagents** (parallel, fresh contexts), each returning
a *concise summary* of its product. The orchestrator's context only ever holds three tidy
summaries — not the raw research — and the synthesis step works on clean input. Context
isolation directly improves output quality.

### Check questions

1. A team ships an agent that produces a full step-by-step plan and then executes it exactly, never revising. On unpredictable tasks it frequently fails partway through. Diagnose the planning choice and prescribe the fix, naming the trade-off they were trying to get and the one they sacrificed. — **Answer:** They chose pure upfront (plan-then-execute) planning. Its benefit is structure and a reviewable plan; its cost is brittleness — plans rarely survive contact with reality, and following one blindly handles surprises badly. Fix: keep a rough upfront plan for direction but execute interleaved, *re-planning when reality diverges* (a maintained todo list is the common concrete pattern). They wanted legibility/structure and sacrificed adaptivity; the combination recovers both.
2. A developer adds subagents purely to make a task run faster and is surprised that, for their tightly-sequential task, it does not help. What did they misunderstand about the *primary* reason subagents exist? — **Answer:** They treated subagents as a parallelism/speed tool. The primary benefit is **context isolation** — each subagent works in a fresh, focused window and returns only a concise result, keeping its noisy intermediate work out of the parent's context (fighting bloat and lost-in-the-middle). Parallelism is a secondary bonus that only materializes for *independent* subtasks; for a sequential task there is little speed to gain, but the isolation benefit can still apply.
3. Give a concrete example of *poor* decomposition of the goal "improve our app's onboarding," and explain which decomposition failure (too big, too vague, or mis-ordered) it shows and why it would hurt the agent. — **Answer:** Example of *too vague / too big*: a single subtask "make onboarding better" — it is not concrete or individually achievable, gives no measurable progress, and the agent cannot tell when it is done. Example of *mis-ordered*: "ship the redesign" before "identify where users drop off" — the agent acts before it has the information the action depends on. Good decomposition yields concrete, achievable, correctly-ordered, measurable subtasks; the failure modes above are a leading cause of agent failure because each step becomes intractable or premature.
4. A coding agent writes a fix, immediately re-reads its own diff, declares it "looks correct," and moves on — yet the fix is wrong. A teammate proposes adding a Reflexion-style loop instead. Explain what is weak about the current self-critique and what specifically a Reflexion loop adds. — **Answer:** The current step is *ungrounded* self-critique: the model judges its own output using the same reasoning that produced the bug, so it shares the blind spot and rubber-stamps the error. A Reflexion-style loop adds an *external verifier signal* — run the test suite — and uses its result: on failure the agent writes a natural-language reflection on *why* it failed, that reflection is added to context, and it retries. The improvement is twofold: the pass/fail signal is ground truth (a real test run, not the model's opinion), and the reflection turns that signal into a concrete correction for the next attempt rather than a blind retry. Deterministic verifier beats self-judgment whenever the task admits one.

---

## 8.5 — Memory: short-term (context) vs. long-term (external store)

### Concept

An agent has two distinct kinds of memory, and conflating them is a common mistake.

**Short-term memory = the context window.** Everything the agent is currently "thinking
with" — the system prompt, the goal, and every thought/action/observation so far. Its
properties: it is **volatile** (gone when the run ends), **bounded** (fixed token limit),
**degrades before the limit** ("lost in the middle," Topic 4.4), and **costs tokens on
every model call** (it is re-sent each turn — Topic 1.7). Short-term memory is the
agent's RAM.

**Long-term memory = an external store.** A database, vector store, file, or knowledge
base outside the context window that **persists across runs and sessions**. Its
properties: **durable**, effectively **unbounded**, but **not automatically visible** —
the agent only "remembers" something from long-term memory if it explicitly **retrieves**
it back into the context window. Long-term memory is the agent's disk.

The flow between them: the agent **reads** from long-term memory into context when it
needs something (a retrieval tool — this is RAG, Topic 10, applied to agent memory), and
**writes** to long-term memory when it learns something worth keeping (a memory-write
tool, or an end-of-run summarization step).

Why long-term memory matters:

- **Cross-session continuity** — a coding agent that remembers project conventions, a
  support agent that recalls a user's past tickets.
- **Context-window relief** — instead of keeping everything in context (expensive,
  degrading), offload to a store and retrieve on demand.
- **Learning over time** — accumulate facts, preferences, and lessons across runs.

**Summarization / compaction** is the bridge. When the context grows too large, the
harness summarizes older turns into a compact form — either kept in context (compaction)
or written to long-term memory (and re-retrievable later). The art is summarizing without
dropping information a later step will need — *and* without breaking the structural rules
of the messages array; the mechanics are covered in detail in 8.13. Some systems also use a **scratchpad** —
notes the agent writes for itself, sitting between pure short-term and pure long-term.

Key trade-off: anything in **short-term memory** is *immediately available but costly and
crowding the window*; anything in **long-term memory** is *cheap to keep but invisible
until retrieved*. Designing agent memory is deciding what lives where and engineering the
read/write paths.

### Key terms

- **Short-term memory** — the context window; volatile, bounded, re-sent every call. The agent's RAM.
- **Long-term memory** — an external persistent store (DB, vector store, files); durable, unbounded, but invisible until retrieved. The agent's disk.
- **Retrieval** — reading from long-term memory back into the context window.
- **Summarization / compaction** — condensing older context to free space, optionally persisting it to long-term memory.
- **Scratchpad** — notes the agent writes for itself during a run.

### Common misconceptions

- ❌ "The agent automatically remembers everything it has ever done." → ✅ It only knows what is currently in its context window. Long-term memory is invisible unless explicitly retrieved.
- ❌ "Long-term memory means a bigger context window." → ✅ Long-term memory is an *external store*; it is unbounded precisely *because* it is not in the context window. Retrieval moves slices of it in.
- ❌ "Summarization is lossless." → ✅ Summarization necessarily drops detail; the risk is dropping something a later step needs. It is a deliberate, lossy trade-off.

### Worked example

A customer-support agent. Within one conversation, the user's last five messages live in
**short-term memory** (the context). When the agent needs the user's purchase history, it
calls a retrieval tool that pulls records from a database — **long-term memory** read into
context. At the end of the conversation, the harness writes a one-paragraph summary
("user reported X, resolved by Y, prefers email contact") to the user's profile —
**long-term memory write**. Next week, when the same user returns, that summary is
retrieved into the new conversation's context, giving continuity without keeping the
entire old transcript around.

### Check questions

1. For each item, say whether it belongs in short-term or long-term memory and why: (a) the user's last three messages in this conversation; (b) a coding agent's knowledge of project conventions it should reuse next week; (c) the raw 200-step trace of a finished subtask. — **Answer:** (a) Short-term — it is the live context the agent is reasoning with right now; volatile and fine to lose at run end. (b) Long-term — it must persist across runs/sessions, so it belongs in an external store and be retrieved when relevant. (c) Long-term (or discarded) — keeping a 200-step raw trace in context bloats the window and degrades reasoning; if any of it is needed later it should be summarized and offloaded. The principle: immediately-needed-now → short-term; persists-or-too-bulky → long-term.
2. A support agent "has" a user's full ticket history in a database but answers as if the user is brand new. The database is fine. What is the actual bug? — **Answer:** The agent never *retrieved* the history into its context. Long-term memory is an external store that is invisible to the model unless an explicit retrieval step pulls it into the context window. The agent only "knows" what is currently in context — having data in a store is not the same as the agent using it. The missing piece is the read path (a retrieval tool / step), not the data.
3. An engineer enables aggressive context compaction and reports the agent now sometimes "loses the plot" mid-task. Explain the trade-off they ran into and how to mitigate it without abandoning compaction. — **Answer:** Compaction is inherently lossy — condensing older turns necessarily drops detail, and aggressive settings can drop *load-bearing* information a later step needed (the goal, a key finding). The trade-off is window space vs. retained detail. Mitigation: compact less aggressively; preserve high-value items (the goal, decisions, open questions) verbatim while summarizing the rest; or write dropped detail to long-term memory so it can be re-retrieved rather than lost outright.

---

## 8.6 — Anatomy of the harness: loop, dispatch, context mgmt, retries, guardrails, logging

### Concept

The model is only one part of an agent. The **harness** is *everything around the model*
— the deterministic code that turns a model that emits tokens into a system that reliably
gets work done. Being able to whiteboard a harness is a core interview skill. Its
components:

1. **The loop** — the think-act-observe cycle (8.2). The driver that repeatedly calls the
   model and processes responses.

2. **Tool dispatch** — the router that takes a `tool_use` block, maps `name` to the actual
   function, validates `input` against the schema, executes it (often concurrently for
   parallel calls — Topic 7.3), and packages the result as a `tool_result`. This is also
   where **deterministic guardrails** on tool execution live (permission checks,
   allowlists — Topic 7.4).

3. **Context management** — keeps the context within budget and effective: ordering
   content well, compacting/summarizing old turns (8.5), evicting stale data, retrieving
   on demand. Without this, every agent eventually overflows (8.8). This is where Topic 4
   (context engineering) meets the agent.

4. **Retries & error handling** — wraps model calls and tool calls in retry logic with
   exponential backoff (for rate limits — Topic 12.5 — and transient failures), timeouts,
   and fallbacks (e.g. a cheaper/faster model if the primary is down — Topic 12.6). Tool
   errors are turned into tool_results, not crashes (Topic 7.5).

5. **Guardrails** — checks on inputs, outputs, and actions: input filtering, output
   validation, action gating (require approval for dangerous tools — 8.10), and the
   deterministic enforcement that makes the agent safe even if the model misbehaves
   (defense in depth — Topic 13).

6. **Termination logic** — the conditions that stop the loop: success detection, and the
   budget/step/time limits that stop a runaway agent (8.9).

7. **Logging / observability** — recording every step: each model call (prompt,
   completion, tokens, latency, cost), each tool call (args, result, duration), and the
   full trajectory. Because agents are non-deterministic, you *cannot* debug them without
   detailed traces. The concrete structure for this — a trace of nested spans — is
   covered in 8.14. This feeds evaluation (8.11) and production observability (Topic 12.7).

The mental model: **the model provides judgment; the harness provides reliability,
safety, and control.** A frontier model with a flimsy harness is an unreliable demo; a
solid harness around a modest model is a shippable product. When something goes wrong in
production, the fix is usually in the harness, not the model.

A framing worth remembering: the harness is where you *enforce* what the prompt only
*requests*. The prompt asks the model to behave; the harness guarantees the limits.

### Key terms

- **Harness** — all the deterministic code surrounding the model that makes an agent reliable: loop, dispatch, context management, retries, guardrails, termination, logging.
- **Tool dispatch** — the harness component that routes tool_use blocks to functions, validates inputs, executes, and packages results.
- **Context management** — keeping the context within budget and effective via ordering, compaction, eviction, retrieval.
- **Observability/tracing** — recording every model and tool call and the full trajectory for debugging and evaluation.

### Common misconceptions

- ❌ "The agent is basically just the model." → ✅ The model is one component; the harness — loop, dispatch, context mgmt, retries, guardrails, logging, termination — is most of the engineering and most of the reliability.
- ❌ "A better model removes the need for a good harness." → ✅ Better models help, but bounding cost, enforcing safety, managing context, and handling errors are harness jobs no model solves for you.
- ❌ "Logging is optional polish." → ✅ Agents are non-deterministic; without detailed per-step traces you cannot debug, evaluate, or trust them. Logging is core infrastructure.

### Worked example

Whiteboard sketch of a coding-agent harness:

```
┌─ HARNESS ─────────────────────────────────────────────┐
│  LOOP:  while not done and within budget:              │
│    1. call model(context, tools)        ← retries/backoff/fallback │
│    2. if end_turn → done                ← TERMINATION  │
│    3. for each tool_use:                ← DISPATCH     │
│         validate args → check permissions (GUARDRAIL)  │
│         execute (sandboxed) → tool_result              │
│    4. append results; if context > budget → COMPACT    │
│    5. log step (prompt, tools, tokens, cost, latency)  │
└────────────────────────────────────────────────────────┘
        MODEL = judgment.   HARNESS = reliability + safety.
```

### Check questions

1. A startup upgrades from a mid-tier model to a frontier model and expects their flaky agent to become reliable; it does not. Using the "model provides judgment, harness provides reliability" principle, explain why a better model did not fix it and where the fix actually lives. — **Answer:** Reliability, safety, cost-bounding, context management, and error handling are *harness* jobs — no model solves them for you. A better model improves judgment but cannot bound a runaway loop, enforce permissions, compact context, or retry a failed call; if those harness components are weak the agent stays flaky. The fix is in the harness (loop, dispatch, context management, retries, guardrails, termination, logging), not the model. A frontier model with a flimsy harness is still an unreliable demo.
2. An agent intermittently crashes on transient API rate-limit errors, and on long tasks its output quality silently degrades. Name the harness component responsible for each problem and what it should do. — **Answer:** The crashes are the **retries & error-handling** component's job — wrap model/tool calls with exponential-backoff retries, timeouts, and fallbacks so a transient rate-limit does not kill the run. The silent quality degradation is the **context management** component's job — compact/summarize old turns, evict stale data, and retrieve on demand so the context stays within budget and effective before lost-in-the-middle sets in. Two different components, two different failures.
3. A teammate says detailed per-step logging is "nice-to-have polish we can add later." Give the specific reason logging is core infrastructure *for agents in particular*, not just general good practice. — **Answer:** Agents are non-deterministic — every run takes a different trajectory — so a failure you saw once may not reproduce on demand. Without a recorded per-step trace (each model call's prompt/completion/tokens/cost/latency and each tool call's args/result/duration) you cannot debug *that specific* failed run, you cannot do trajectory evaluation (8.11), and you cannot trust the system in production. For agents, logging is what makes the system observable at all — not optional polish.

---

## 8.7 — Multi-agent systems: orchestrator/worker, handoffs, when it helps vs. hurts

### Concept

A **multi-agent system** uses several LLM-driven agents that cooperate on a task, rather
than one monolithic agent. Two dominant topologies:

**Orchestrator / worker (manager-worker).** A lead **orchestrator** agent decomposes the
task, delegates subtasks to **worker** subagents (each with a focused context and tool
set), and integrates their results. Hierarchical: workers report up; the orchestrator
holds the overall plan. This is the subagent pattern from 8.4 formalized. Anthropic's
multi-agent Research system is a public example — a lead agent spawns parallel search
subagents; Anthropic reports it used roughly 15× the tokens of a single chat interaction,
underscoring the cost multiplier below. [3]

**Handoffs (peer / sequential).** Control passes from one agent to another, each
specialized for a phase or domain. A support system might hand a conversation from a
general triage agent to a billing agent to a technical agent. The "baton" of control
moves; one agent is active at a time. (OpenAI's Agents SDK provides a first-class
`handoff` primitive for exactly this — a handoff is exposed to the model as a tool, and
the receiving agent takes over the conversation. [4])

**Why multi-agent can help:**

- **Context isolation** — each agent has a clean, focused window; the orchestrator never
  drowns in workers' intermediate steps (the 8.4 argument).
- **Specialization** — tailored prompts and minimal tool sets per role improve reliability.
- **Parallelism** — independent subtasks run concurrently, cutting wall-clock time.
- **Separation of concerns** — easier to develop, test, and reason about one focused agent.

**Why multi-agent can hurt — and often does:**

- **Cost & latency multiply** — each agent is its own model-call stream; a multi-agent
  run can cost many times a single-agent run for the same task.
- **Coordination overhead & error propagation** — a mistake or misunderstanding in one
  agent's output cascades into others (error cascades — 8.8). Information needed
  downstream can be stranded in an upstream agent's context.
- **Lost shared context** — agents don't share a context window; conveying everything one
  agent knows to another is lossy and hard.
- **Harder to debug** — more moving parts, more non-determinism, more trajectories.

The honest guidance (and a common interview point): **most tasks do not need multi-agent;
a single well-designed agent with good tools is simpler and more reliable.** Reach for
multi-agent when the task is genuinely *parallelizable* and subtasks are *separable* with
clean interfaces, or when each subtask produces a lot of intermediate context that would
otherwise pollute a shared window. If subtasks are tightly coupled and constantly need
each other's state, multi-agent *adds* failure modes without the benefits — keep it
single-agent. Multi-agent is a tool for *separable* problems, not a default architecture.

### Key terms

- **Multi-agent system** — multiple LLM-driven agents cooperating on a task.
- **Orchestrator/worker** — a lead agent decomposes and delegates to worker subagents and integrates results; hierarchical.
- **Handoff** — control of a task passes from one specialized agent to another; one agent active at a time.
- **Error cascade** — one agent's mistake propagating into and corrupting other agents' work.

### Common misconceptions

- ❌ "Multi-agent systems are more capable, so prefer them." → ✅ They multiply cost, latency, and failure modes. A single well-designed agent is the right default; multi-agent is for genuinely separable tasks.
- ❌ "Subagents share context with the orchestrator." → ✅ Each agent has its own context window; passing knowledge between them is explicit and lossy.
- ❌ "Multi-agent always means parallelism." → ✅ Handoff systems are sequential — control moves between specialized agents one at a time.

### Worked example

Good fit: a research task — "compare the privacy policies of 5 vendors." Five worker
subagents each read one vendor in parallel (separable, each generates lots of context),
return concise summaries; an orchestrator synthesizes. Multi-agent wins clearly.

Poor fit: "refactor this module and keep all tests passing." The steps are tightly
coupled — every change depends on the current state of every other file. Splitting it
across agents that don't share context produces inconsistent changes and broken tests. A
single agent with file and test tools is simpler and more reliable here.

### Check questions

1. For each system, name the multi-agent topology and justify: (a) a customer-support flow that moves a chat from a triage agent to a billing specialist; (b) a system where a lead agent fans out five research tasks at once and merges the results. — **Answer:** (a) is a **handoff** topology — control passes sequentially from one specialized agent to another, one active at a time, the "baton" moving down the line. (b) is **orchestrator/worker** — hierarchical: a lead decomposes the task, delegates to parallel worker subagents with focused contexts, and integrates their results. The tell is whether control is *transferred* (handoff) or *delegated-and-reported-up* (orchestrator/worker).
2. A team's multi-agent refactoring system produces inconsistent edits across files and is hard to debug. Name the two multi-agent-specific failure modes most likely at work and explain the mechanism of each. — **Answer:** **Lost shared context** — the agents don't share a context window, so each is blind to the others' changes and they make mutually inconsistent edits. **Error cascades** — one agent's wrong assumption or bad output feeds the others, who build on it, compounding the mistake. (Harder debugging follows from more moving parts and more non-determinism.) Both stem from splitting a *coupled* task across agents that cannot see each other's state.
3. You are told "multi-agent is more advanced, so default to it." Rebut this and give the actual decision rule, with one task that fits multi-agent and one that does not. — **Answer:** Multi-agent is not a default — it multiplies cost, latency, and failure modes; a single well-designed agent is the right default. Reach for multi-agent only when subtasks are genuinely *separable/parallelizable* with clean interfaces, or each generates lots of intermediate context that would pollute a shared window. Fits: "summarize the privacy policies of 5 vendors" — separable, each generates bulk context. Does not fit: "refactor this module and keep tests green" — tightly coupled, every change depends on shared state; splitting it across non-context-sharing agents breaks consistency.

---

## 8.8 — Failure modes: loops, context overflow, error cascades, goal drift

### Concept

Agents fail in ways single LLM calls cannot. Knowing the catalog — and the defenses — is
core competence.

**1. Infinite / unproductive loops.** The agent repeats the same action, or oscillates
between two states, without progress: re-running a failing tool, re-searching the same
query, "I'll try X" → X fails → "I'll try X." Caused by an unrecoverable situation the
model doesn't recognize, or an ambiguous goal. *Defenses:* hard step/iteration limits
(8.9); loop detection (flag repeated identical actions); make tool errors actionable
(Topic 7.5) so the model can change tack; surface "no progress" signals.

**2. Context overflow.** The think-act-observe loop grows the context every iteration.
On a long task it exceeds the window — a hard error — or, worse, *silently* degrades well
before the limit ("lost in the middle," context rot — Topic 4.4): the agent forgets the
goal, forgets earlier findings, repeats work. ("Lost in the middle" is from Liu et al.,
2023 [5], which found models retrieve information best when it sits at the start or end
of the context and worst when it is buried in the middle.) *Defenses:* context management (8.6) —
compaction/summarization, eviction, retrieval-on-demand; subagents for isolation (8.4);
monitor context-window utilization.

**3. Error cascades.** An early mistake — a misread observation, a wrong assumption, a
bad tool result — becomes an input to later steps, which build on it, compounding the
error. By the end the agent is confidently far off. Worse in multi-agent systems where
one agent's bad output feeds others (8.7). *Defenses:* verification steps (check
intermediate results before building on them); ground claims in tool observations;
checkpoints; keep trajectories short enough to inspect.

**4. Goal drift.** Over a long run the agent gradually loses sight of the original
objective — pulled off course by an interesting subproblem, an injected instruction, or
just the original goal scrolling far up a long context and losing salience. It ends up
solving a *different* problem. *Defenses:* re-state the goal periodically (keep it salient
— recency, Topic 5.6); a maintained plan/todo list it re-reads; checkpoints comparing
progress to the goal.

**5. Tool misuse.** Wrong tool, wrong arguments, calling a tool when it shouldn't or
failing to when it should. Usually a tool-description problem (Topic 7.7) or too large /
ambiguous a tool set. *Defenses:* better descriptions, fewer/clearer tools (least
privilege), schema validation.

A cross-cutting truth: these failure modes **interact and compound**. Context overflow
causes goal drift causes unproductive loops. And because agents are non-deterministic, a
failure mode that appears in 1 run of 20 is still real — you find these with evaluation
(8.11) and observability (8.6), not by running once and seeing it work.

### Key terms

- **Infinite/unproductive loop** — the agent repeats actions without making progress.
- **Context overflow** — the growing context exceeds the window or silently degrades effectiveness before the hard limit.
- **Error cascade** — an early mistake compounding as later steps build on it.
- **Goal drift** — the agent gradually losing track of its original objective over a long run.
- **Tool misuse** — selecting the wrong tool or supplying wrong/malformed arguments.

### Common misconceptions

- ❌ "If the agent works once, it works." → ✅ Agents are non-deterministic; a failure mode that shows in 1 of 20 runs is real and must be found via evaluation, not a single happy-path run.
- ❌ "Context overflow only matters at the hard token limit." → ✅ Effectiveness degrades well before the limit (lost in the middle / context rot); the silent degradation is the bigger danger.
- ❌ "These failure modes are independent." → ✅ They compound — overflow causes drift causes loops; fixing one in isolation is often insufficient.

### Worked example

A debugging agent is asked to fix a failing test. The test fails because of an
environment issue the agent can't change. It tries to edit the code, the test still
fails, it edits again, fails again — an **unproductive loop**. Meanwhile the context fills
with repeated failure output (**context overflow**), the original task description
scrolls out of salient range (**goal drift**), and it starts "fixing" unrelated code.
Defenses that would have caught it: a 15-step budget cap (8.9); loop detection on
identical edits; periodic goal re-statement; and the tool returning an *actionable* error
("test fails due to missing env var DB_URL — this is an environment issue, not a code
bug") so the model recognizes the situation and stops or asks for help.

### Check questions

1. An agent tasked with "audit our dependencies for security issues" ends a long run having instead written a detailed performance-optimization proposal. Name the failure mode, explain the most likely mechanism, and give one defense. — **Answer:** Goal drift — the agent gradually lost sight of its original objective. Likely mechanism: the original task description scrolled far up a long context and lost salience, and/or an interesting subproblem (a slow dependency) pulled it off course; an injected instruction is another possible cause. Defense: periodically re-state the goal to keep it salient (recency), and maintain a todo list the agent re-reads against the goal at checkpoints.
2. The course says context overflow is *more* dangerous before the hard token limit than at it. Explain this seemingly backwards claim. — **Answer:** Hitting the hard limit raises an explicit error you can catch and handle. The pre-limit danger is *silent*: well before the limit, effectiveness degrades (lost in the middle / context rot) — the agent quietly forgets the goal and earlier findings and repeats work, producing worse output with *no error raised*. A loud failure you can detect; a silent degradation you may ship. That asymmetry is why the pre-limit zone is the bigger danger.
3. A debugging agent's run goes wrong as follows: a tool keeps failing, the agent re-tries the same edit repeatedly, the context fills with repeated failure output, and it starts editing unrelated files. Identify the chain of failure modes and explain how fixing only one of them is insufficient. — **Answer:** Three interacting modes: an **unproductive loop** (re-trying the same failing edit), **context overflow** (repeated failure output bloats the window), and **goal drift** (the task scrolls out of salience, so it edits unrelated files). They compound — overflow worsens drift, drift worsens looping. Fixing one in isolation (say, just a step cap) bounds the runaway but leaves the drift and overflow; robust handling needs several defenses together (step limit + loop detection + context management + actionable tool errors).

---

## 8.9 — Termination and budget limits

### Concept

A single LLM call ends on its own. An agentic loop does **not** — left alone, it can run
forever, burn unbounded tokens and money, or hammer downstream systems. Deciding *when to
stop* is a first-class harness responsibility, and it has two sides.

**Success termination — stopping when done.** Normally the loop ends when the model stops
requesting tools and emits a final answer (`end_turn`). But "the model said it's done"
≠ "the task is actually done" — the model can stop early (give up, hallucinate
completion) or never stop. So success detection may also include explicit checks: a
`task_complete` tool the model must call, output validation against the goal/schema, or a
verification step. The loop's *natural* exit is the model ending its turn; robust harnesses
*verify* that exit.

**Budget termination — stopping when out of resources.** Independent of success, the
harness enforces hard limits so a misbehaving agent cannot run away:

- **Step / iteration limit** — a maximum number of loop iterations (e.g. 20). The
  bluntest, most important guardrail; directly bounds infinite loops (8.8).
- **Token budget** — a cap on total tokens consumed across the run; bounds cost directly.
- **Wall-clock / time limit** — a maximum run duration; protects latency SLAs.
- **Cost budget** — a hard dollar cap, often derived from the token budget.
- **Tool-call limits** — caps on total calls, or per-tool (e.g. at most 5 web searches).

When a budget limit trips, the loop **must stop gracefully** — not crash. Good behavior:
return the best partial result, a clear "stopped: hit step limit" status, and the
trajectory so far, so the caller (or a human) can decide what to do. Silently stopping,
or crashing, is bad.

**Why budgets are non-negotiable:** without them, a single bug, an ambiguous task, an
adversarial input, or a loop (8.8) translates *directly* into an unbounded bill and
unbounded latency. Budgets convert "catastrophic runaway" into "bounded failure with a
partial result." They are the safety net under every other failure mode in 8.8.

Tuning: limits too tight cause real tasks to be cut off prematurely; too loose and a
runaway costs a lot before stopping. Set them from observed real-task distributions
(e.g. "95% of real tasks finish within 12 steps → cap at 25") and monitor how often they
trip — frequent tripping signals either a bug or mis-set limits.

### Key terms

- **Success termination** — the loop ending because the task is (believed) complete, typically `end_turn`, ideally verified.
- **Budget termination** — the loop being stopped by a resource limit regardless of success.
- **Step/iteration limit** — a hard cap on loop iterations; the primary runaway guardrail.
- **Token / cost / time budget** — hard caps on tokens, dollars, and wall-clock per run.
- **Graceful stop** — halting with a clear status and best partial result rather than crashing.

### Common misconceptions

- ❌ "The loop ends when the task is done." → ✅ The loop ends when the *model stops requesting tools* — which may not coincide with the task being done. Success should be verified, and budget limits exist precisely because the model may never stop.
- ❌ "Budget limits are a nice-to-have." → ✅ They are mandatory — without them a bug or adversarial input becomes an unbounded bill and unbounded latency.
- ❌ "Hitting a limit should throw an error." → ✅ It should stop *gracefully* — return a clear status and the best partial result and trajectory so the caller can react.

### Worked example

A research agent is given an unanswerable question. It keeps searching, finding nothing
conclusive, searching again. The harness has a 20-step limit and a $0.50 cost budget. At
step 20 the loop stops, and instead of crashing it returns: `status: "stopped — step
limit reached"`, the best partial findings, and the full trajectory. The cost is bounded
at well under $0.50; a human reviews the trajectory and sees the question was
ill-posed. Without the limit the agent would have searched indefinitely, racking up cost
with no end.

### Check questions

1. Success termination and budget termination are two separate mechanisms. Explain why a harness needs *both* rather than just one — give a failure each one alone misses. — **Answer:** Success termination (verifying the task is actually done, beyond the model's `end_turn`) catches the model stopping early or hallucinating completion — but it cannot stop an agent that *never* says it is done. Budget termination (step/token/time/cost caps) stops a runaway regardless of the model — but it says nothing about whether the result is correct, so alone it can let a confidently-wrong agent "finish." You need success verification for correctness *and* budget caps as the unconditional safety net.
2. A team sets a generous token budget as their only limit. Describe a runaway scenario that this single limit handles poorly, and which additional limit would address it. — **Answer:** An agent stuck calling a slow external tool can run for many minutes while consuming relatively few tokens — the token budget barely moves, so the run blows the latency SLA before the budget trips. A **wall-clock / time limit** addresses this directly. (Similarly, a per-tool call limit would catch an agent hammering one downstream system.) Different limits bound different runaway dimensions; relying on one leaves the others uncovered.
3. When a step limit trips, one harness throws an exception and another returns a status object with partial results and the trajectory. Argue which is correct and what is lost with the other. — **Answer:** Returning a status object with the best partial result and the full trajectory is correct — it converts a runaway into a *bounded, inspectable* failure the caller or a human can act on. Throwing an exception (or stopping silently) discards the partial progress and the trajectory, so the caller learns only "it failed" with no material to diagnose the ill-posed task or mis-set limit. Graceful stop preserves the information; crashing destroys it.

---

## 8.10 — Human-in-the-loop checkpoints

### Concept

Full autonomy is not always desirable. **Human-in-the-loop (HITL)** means inserting
points where a human must review, approve, correct, or provide input before the agent
proceeds. It is the deliberate decision *not* to let the agent run unsupervised through
certain steps.

**Why HITL exists:** agents are probabilistic and can be wrong, manipulated (prompt
injection — Topic 13), or confidently mistaken. For **high-stakes, irreversible, or
costly actions**, the expected cost of an error outweighs the friction of asking a human.
HITL puts a human judgment gate in front of those actions.

**Where checkpoints go:**

- **Before irreversible / dangerous actions** — sending money, deleting data, sending
  external communications, deploying code, executing trades. The agent *proposes* the
  action; a human *approves* before the harness executes it. (This is the request/execute
  gap of Topic 7.4 used deliberately.)
- **At plan approval** — for a long or expensive task, a human reviews the agent's plan
  before any execution (ties to upfront planning — 8.4).
- **On low confidence / ambiguity** — when the agent is unsure or the task is
  underspecified, it escalates to a human instead of guessing.
- **On budget or limit triggers** — when a limit trips (8.9), hand off to a human.
- **Final-output review** — a human signs off before the result is delivered or acted on.

**Patterns of human involvement:**

- **Approve / reject** — human gates a proposed action (most common).
- **Edit** — human modifies the agent's proposed action or plan before it proceeds.
- **Escalation / handoff** — the agent gives up control to a human entirely.
- **Confirmation** — a lightweight "are you sure?" for medium-stakes actions.

**The core trade-off: autonomy vs. safety/control.** More checkpoints = safer and more
controllable, but slower, costlier in human time, and less scalable. Fewer checkpoints =
faster and more autonomous, but riskier. The right level is **risk-calibrated**: gate the
irreversible, costly, or externally-visible actions; let reversible, low-stakes,
internal actions run autonomously. A coding agent might autonomously edit files and run
tests (reversible, sandboxed) but require approval to push to production.

A critical point: HITL is a **safety layer, not a substitute for harness guardrails**. A
human approving a long stream of opaque actions will rubber-stamp them (approval fatigue).
HITL works best on a *small number of clearly-presented, high-stakes* decisions, backed
by deterministic guardrails (Topic 7.4) for everything else. The human is a checkpoint,
not the whole defense.

### Key terms

- **Human-in-the-loop (HITL)** — inserting human review/approval/input points into the agent's execution.
- **Approval gate / checkpoint** — a point where the agent must pause for human sign-off before proceeding.
- **Escalation** — the agent handing control to a human when unsure, blocked, or out of budget.
- **Approval fatigue** — humans rubber-stamping approvals when asked too often, defeating the checkpoint.

### Common misconceptions

- ❌ "HITL replaces the need for code-level guardrails." → ✅ It complements them; deterministic guardrails (Topic 7.4) handle the bulk, HITL handles a small number of high-stakes decisions. Humans alone rubber-stamp.
- ❌ "More human checkpoints are always safer." → ✅ Too many cause approval fatigue and kill the throughput benefit of automation; checkpoints should be risk-calibrated.
- ❌ "HITL means a human watches the whole run." → ✅ It means humans gate *specific* high-stakes steps; the rest runs autonomously.

### Worked example

A finance-ops agent reconciles invoices. It autonomously reads invoices, matches them to
purchase orders, and flags discrepancies — all reversible, low-stakes, so no checkpoint.
But the step "issue a payment" is irreversible and high-stakes: the agent *proposes* the
payment (amount, recipient, linked invoice) and the harness pauses at an **approval gate**;
a human reviews the clearly-presented proposal and approves or rejects before the payment
tool is executed. One well-placed checkpoint on the one dangerous action — not a human
watching every step.

### Check questions

1. A travel agent can search flights, hold a fare, charge the user's card, and email the itinerary. Decide for each action whether it needs a human checkpoint, and state the principle you used. — **Answer:** Search flights and hold a fare are reversible/low-stakes → run autonomously (deterministic guardrails only). Charge the card is irreversible and costly → human approval gate before execution. Emailing the itinerary is externally visible → at least a confirmation, arguably an approval, since it leaves the system. Principle: checkpoints are *risk-calibrated* — gate the irreversible, costly, or externally-visible actions; let reversible, low-stakes, internal actions run autonomously.
2. A design adds a human approval step before *every* tool call to "maximize safety." Explain why this can make the system *less* safe in practice. — **Answer:** Asking a human to approve many actions — most of them low-stakes — causes **approval fatigue**: the reviewer stops reading carefully and rubber-stamps, so the checkpoint that genuinely matters (the rare high-stakes one) also gets waved through. It also destroys the throughput benefit of automation. More checkpoints are not monotonically safer; safety comes from a *small number of clearly-presented* high-stakes gates the human actually scrutinizes.
3. "We have human approval on dangerous actions, so we don't need deterministic code-level guardrails." Refute this and state the correct division of labor. — **Answer:** Wrong because humans cannot be the bulk defense — flooded with approvals they rubber-stamp (approval fatigue), and they are not present for every action. Correct division: deterministic code-level guardrails (allowlists, permission checks, sandboxing, limits) handle the *bulk* of enforcement reliably and unconditionally; HITL handles a *small number* of clearly-presented, genuinely high-stakes decisions where human judgment adds value. HITL complements guardrails; it does not replace them.

---

## 8.11 — Evaluating agents: trajectory eval vs. final-outcome eval

### Concept

Evaluating an agent is harder than evaluating a single LLM call, because an agent has a
*process*, not just an output. There are two complementary lenses (this section is an
on-ramp to Topic 9).

**Final-outcome evaluation.** Judge only the end result: did the agent achieve the goal?
For "fix the failing test" — do the tests now pass? For "book a flight" — was the right
flight booked? *Pros:* it is what ultimately matters; often objectively checkable (tests
pass, file exists, value correct); simple. *Cons:* it is **blind to *how***. An agent can
reach a correct outcome by luck, by an absurdly expensive 50-step path, or by an unsafe
route. Outcome-only eval can't distinguish a robust agent from a lucky one, and gives
little signal on *where* a failing agent went wrong.

**Trajectory evaluation.** Judge the *process* — the full sequence of thoughts, tool
calls, and observations. Questions it answers: Did the agent pick the right tools? Were
the tool arguments correct? Was the path efficient or did it wander? Did it loop or
backtrack? Did it follow safety constraints? Did it recover well from errors? *Pros:* rich
diagnostic signal — it tells you *why* and *where*, catches "right answer for wrong
reasons," and measures efficiency and safety. *Cons:* harder to do — there is often no
single "correct" trajectory (many valid paths exist), so it usually needs an LLM-as-judge
(Topic 9.5) or rubric scoring (Topic 9.6) over the trace, plus good logging (8.6).

**You need both.** Final-outcome eval tells you *whether* the agent works; trajectory eval
tells you *why*, *how reliably*, *how efficiently*, and *how safely*. Outcome alone:
"passed but I don't know if it'll pass next time or how much it cost." Trajectory alone:
"the process looked good but did it actually solve the task?" Production agent eval
combines them, plus operational metrics — **cost per task, latency, step count, tool-error
rate, intervention rate** — and is run **repeatedly per task** because of non-determinism
(8.8): a single run tells you almost nothing; pass-rate over many runs (a pass@k-style
notion — Topic 9.4) is the meaningful measure.

What to evaluate over: a **golden set** of representative tasks (Topic 9.2), each with a
checkable success criterion. Run the agent N times per task; report outcome pass-rate
plus trajectory-quality and operational metrics. Use this as a **regression gate** — every
change to the prompt, tools, model, or harness must be re-evaluated, because any of them
can shift agent behavior (Topic 9.3).

**Standard agent benchmarks.** Beyond your own golden set, public benchmarks let you
compare models and harnesses on shared tasks. Two worth knowing:

- **τ-bench (tau-bench)** [7] — evaluates an agent in **multi-turn tool-use conversations
  with a (simulated) user**, in realistic domains like retail and airline support. It
  scores **final-outcome** correctness (did the database end in the right state?) against
  policy rules, and reports **pass^k** — the probability that the agent succeeds on *all
  k* independent attempts of the same task — explicitly measuring **reliability/consistency**,
  not just average pass rate. It directly embodies the "run it many times" lesson.
- **BFCL (Berkeley Function-Calling Leaderboard)** [8] — focuses on **function-calling
  accuracy**: does the model pick the right function and produce correct, well-typed
  arguments, across simple, parallel, multiple-choice, and multi-turn cases, including
  cases where the model should call *nothing*. It is closer to single-call tool-use
  correctness (Topic 7) than to full long-horizon trajectory evaluation.

The takeaway: these benchmarks are a useful *external* yardstick, but they are not a
substitute for evaluating *your* agent on *your* tasks — a golden set tailored to your
domain and your tools is still the thing that gates a release.

### Key terms

- **Final-outcome evaluation** — judging only whether the agent achieved the end goal.
- **Trajectory evaluation** — judging the *process*: tool choices, arguments, efficiency, safety, error recovery.
- **Operational metrics** — cost per task, latency, step count, tool-error rate, intervention rate.
- **Pass-rate / pass@k** — fraction of repeated runs that succeed; the meaningful success measure given non-determinism.
- **τ-bench / BFCL** — public agent benchmarks: τ-bench evaluates multi-turn tool-use with a simulated user and reports pass^k (all-k-attempts reliability); BFCL evaluates function-calling accuracy (right function, correct arguments).
- **pass^k** — the probability an agent succeeds on *all* k attempts of a task; a reliability/consistency measure (distinct from pass@k, which credits *any* success in k).

### Common misconceptions

- ❌ "If the agent produced the right answer, it passed." → ✅ Outcome-only eval can't tell a robust agent from a lucky one, or a cheap path from a 50-step one — trajectory and cost matter.
- ❌ "There is one correct trajectory to compare against." → ✅ Usually many valid paths exist; trajectory eval typically uses rubric/LLM-judge scoring of the trace, not exact-match.
- ❌ "One successful run proves the agent works." → ✅ Agents are non-deterministic; meaningful evaluation runs each task many times and reports a pass-rate.

### Worked example

Two coding agents both make the failing test pass — identical **final outcome: pass**.
**Trajectory eval** reveals: Agent A read the error, opened the right file, made a
one-line fix, ran the test — 4 steps, $0.03, clean. Agent B searched the web, edited five
unrelated files, broke and re-fixed two of them, looped once, and stumbled into the fix —
22 steps, $0.40, fragile. Outcome eval rates them equal; trajectory + operational eval
shows Agent A is the one to ship. Run each 20 times: A passes 19/20, B passes 11/20 — the
pass-rate confirms what the trajectory analysis suggested.

### Check questions

1. An agent books the correct flight, so final-outcome eval scores it a pass. Give two distinct things that could still be seriously wrong with that run, and name which evaluation lens would surface each. — **Answer:** It could have taken a wildly inefficient 40-step, high-cost path (surfaced by **trajectory eval** + operational metrics — step count, cost per task), and it could have reached the booking via an *unsafe* route, e.g. ignoring a safety constraint or skipping a required approval (surfaced by **trajectory eval** checking safety-constraint adherence). Outcome eval is blind to both because it only checks the end state, not *how* it was reached.
2. Why does running an agent eval once per task give "almost no information," and what is the meaningful measure instead? — **Answer:** Agents are non-deterministic — sampled model steps and live tool results make each run a different trajectory — so one run is a single draw from a distribution and could be a lucky pass or an unlucky fail. The meaningful measure is a **pass-rate over many runs per task** (a pass@k-style notion); a 90% mean with low variance is a different thing from a 90% mean with high variance, and one run cannot tell them apart.
3. Trajectory evaluation cannot use exact-match against a single "correct" trajectory the way some output evals do. Explain why, and what is used instead. — **Answer:** For almost any non-trivial agent task there are *many* valid trajectories — different tool orders and paths can all be correct — so there is no single reference trace to match against. Instead trajectory eval uses rubric scoring or an LLM-as-judge over the recorded trace, assessing qualities (right tools, correct arguments, efficiency, safety, error recovery) rather than checking equality. This is also why good per-step logging (8.6) is a precondition for trajectory eval.
4. τ-bench reports **pass^k** rather than ordinary pass@k. Explain the difference and why pass^k is the more honest number for judging whether an agent is *production-ready*. — **Answer:** pass@k credits the agent if it succeeds on *at least one* of k attempts — it measures best-case capability. pass^k requires the agent to succeed on *all k* attempts — it measures consistency/reliability. For production readiness, what matters is that the agent succeeds *every* time a user runs the task, not that it *could* succeed on a lucky try; a high pass@k with a low pass^k means a capable but unreliable agent. pass^k surfaces exactly the non-determinism risk (8.8) that one run hides — which is why a benchmark aimed at real deployments reports it.

---

## 8.12 — The cost model of a growing trajectory

### Concept

A dangerous intuition: "an agent that takes 30 steps costs about 30× a single LLM call."
It does not. It costs **far more** — and understanding *why* is essential for budgeting,
for setting the limits in 8.9, and for knowing when an agent is the wrong tool.

The root cause is **statelessness** (Topic 1.7). The model has no memory between calls,
so **every step re-sends the entire conversation so far** — the system prompt, the tool
definitions, the goal, and *every* prior thought, tool call, and tool result. The context
is not a fixed prompt that gets answered once; it is a buffer that **grows every
iteration**, and the *whole grown buffer* is the input to the next call.

**The derivation.** Suppose each think-act-observe step adds a roughly constant `c` tokens
to the context (the model's reasoning + the tool_use block + the tool_result). Let the
fixed prefix (system prompt + tool definitions + goal) be `P` tokens. Then the **input
tokens billed on step n** are:

```
input(n) = P + (n − 1)·c        ← everything accumulated before step n
```

The total **input tokens** billed across an N-step run is the sum over every step:

```
total_input = Σ_{n=1..N} [ P + (n−1)·c ]
            = N·P  +  c·(0 + 1 + 2 + … + (N−1))
            = N·P  +  c·N(N−1)/2
```

That second term, `c·N(N−1)/2`, is **quadratic in N**. The trajectory's own content is
billed *triangularly*: step 1's tokens are re-sent in steps 2…N, step 2's in steps 3…N,
and so on — early steps are paid for again and again. A "30× a single call" estimate only
counts each step's tokens *once*; the reality re-bills them every subsequent turn.

**Put numbers on it.** Take `P = 2,000` (prompt + tools) and `c = 1,000` tokens per step.

- A **single call**: ~`2,000 + 1,000 = 3,000` input tokens.
- A **30-step agent**, naive estimate (30× single call): ~`90,000` input tokens.
- A **30-step agent**, actual: `30·2,000 + 1,000·30·29/2 = 60,000 + 435,000 = 495,000`
  input tokens — about **5.5× the naive estimate**, and ~165× a single call, not 30×.
- Double it to **60 steps** and the quadratic term alone (`1,000·60·59/2 ≈ 1.77M`) is
  *four times* the 30-step term: doubling the steps roughly **quadruples** the
  trajectory-driven cost.

A few important refinements:

- **Output tokens** are billed once per step (the model generates each step's output
  only once) and are usually priced higher per token — but they don't accumulate, so they
  are the *linear* part of the bill. The **quadratic blow-up is on the input side**, from
  re-sending the history.
- **Prompt caching** (Topic 6) attacks the *fixed prefix* `P` — the system prompt and
  tool definitions can be cached so they are cheap to re-read each turn. It helps the
  `N·P` term and, with incremental caching of the growing conversation, can soften the
  quadratic term substantially — but a cache read is cheaper, not free, and cache entries
  expire, so caching mitigates the curve rather than flattening it.
- **Latency tracks cost.** Each step's prefill processes that whole growing input, so
  per-step latency *also* climbs as the trajectory grows — a long agent gets slower as it
  goes, not just more expensive.

**What this implies for design:**

1. **Step count is the dominant cost lever.** Because cost is ~quadratic in N, *reducing
   steps* (better tools, better planning, less wandering) saves far more than shaving
   tokens per step. Halving the steps roughly quarters the trajectory cost.
2. **Context management is cost control, not just quality control.** Compaction (8.13)
   and subagent isolation (8.4) cap how large the re-sent context `c·n` can get — they
   bend the quadratic curve back toward linear. This is a second, equal reason to do the
   8.6 context-management job, beyond avoiding lost-in-the-middle.
3. **Budgets must assume the quadratic.** A token/cost budget (8.9) set from "steps ×
   single-call cost" will be blown long before the step limit trips. Estimate budgets
   from the `N·P + c·N(N−1)/2` shape, or empirically from real run distributions.

The headline to remember: **a long agent run is super-linear in its own length** because
a stateless API re-bills the whole growing trajectory every step. That is *the* economic
fact of agent engineering.

### Key terms

- **Trajectory** — the full accumulated context of an agent run: prompt, tools, goal, and every step's thought/action/observation.
- **Quadratic cost growth** — total input tokens over an N-step run scale as `N·P + c·N(N−1)/2`; the trajectory-content term is quadratic in N because each step's tokens are re-sent on every later step.
- **Re-billing / triangular cost** — early steps' tokens are paid for again on every subsequent call; over N steps the trajectory content is billed ~N²/2 times its size.
- **Fixed prefix (P)** — system prompt + tool definitions + goal; re-sent every step (the `N·P` linear term), and the primary target of prompt caching.

### Common misconceptions

- ❌ "A 30-step agent costs ~30× a single call." → ✅ It costs far more — typically several times that — because a stateless API re-sends the *whole growing trajectory* every step, so the trajectory's tokens are billed quadratically in the step count.
- ❌ "The quadratic blow-up is in output tokens." → ✅ Output is generated once per step and does not accumulate — it is the linear part. The quadratic term is *input*: the re-sent conversation history.
- ❌ "Prompt caching makes the cost linear." → ✅ Caching makes re-reading the fixed prefix (and cached history) *cheaper*, softening the curve, but cache reads are not free and entries expire. Caching mitigates the quadratic; it does not erase it.
- ❌ "To cut agent cost, shave tokens per step." → ✅ Because cost is ~quadratic in step count, *reducing the number of steps* (and bounding context size via compaction) saves far more than trimming per-step tokens.

### Worked example

A research agent averages 40 steps. The team estimates cost as `40 × (single-call cost)`
and sets a token budget on that basis. With `P ≈ 3,000` and `c ≈ 1,200` per step:

- Their estimate: `40 × (3,000 + 1,200) = 168,000` input tokens.
- Actual: `40·3,000 + 1,200·40·39/2 = 120,000 + 936,000 = 1,056,000` input tokens —
  **6.3× their estimate.**

Their budget trips constantly on perfectly normal runs, and they wrongly conclude the
agent is "buggy." The bug is the cost model. Fixes: re-estimate the budget from the
`N·P + c·N(N−1)/2` shape; enable prompt caching on the 3,000-token prefix; and add
compaction (8.13) so `c·n` stops growing without bound past, say, step 25 — which bends
the back half of the run from quadratic toward linear.

### Check questions

1. A teammate says "our agent does 20 steps; a single call costs $0.01, so budget about $0.20 per run." Explain precisely why this under-estimates the true cost, and write the expression they should have used. — **Answer:** The estimate counts each step's tokens exactly once. But the API is stateless, so every step re-sends the *entire conversation so far* — step 1's tokens are billed again in steps 2…20, step 2's in 3…20, and so on. The trajectory content is billed *triangularly*, ~quadratically in the step count. The correct shape for total input tokens is `N·P + c·N(N−1)/2`, where `P` is the fixed prefix (system prompt + tools + goal) and `c` is the tokens added per step. The `c·N(N−1)/2` term is quadratic and is exactly what the naive ×N estimate omits.
2. Cost is roughly quadratic in step count. Two proposed savings: (a) trim each step's reasoning to use 20% fewer tokens; (b) improve planning so the agent finishes in 20 steps instead of 30. Which saves more, and why does the cost model make this not even close? — **Answer:** (b) wins decisively. The trajectory-cost term is `c·N(N−1)/2`. Option (a) scales `c` down by 0.8 → ~20% off that term. Option (b) takes `N` from 30 to 20: `20·19/2 = 190` vs `30·29/2 = 435` → the term drops to ~44% of its value, a ~56% cut — and it also cuts the linear `N·P` term. Because the dominant term is quadratic in `N`, reducing step count beats reducing per-step tokens; halving steps roughly quarters the trajectory cost.
3. An engineer enables prompt caching and expects agent cost to become linear in step count. Explain what caching does and does not change about the cost curve. — **Answer:** Caching targets *re-reading the same tokens cheaply*. It directly helps the fixed-prefix `N·P` term (system prompt + tool definitions read at a cache discount every step) and, with incremental caching of the growing conversation, can make re-reading the accumulated history much cheaper too — softening the quadratic term. But it does not make cost linear: a cache *read* is discounted, not free, and cache entries expire (so a long or paused run re-pays full price), and the trajectory still grows every step. Caching bends the quadratic curve down; it does not flatten it. Bounding context size via compaction (8.13) is what actually limits how large the re-sent context can get.

---

## 8.13 — Context compaction mechanics

### Concept

Section 8.5 introduced compaction as the bridge between short- and long-term memory, and
8.12 showed that bounding context size is also *cost* control. This section is the
**mechanics**: compaction is not "the harness magically shrinks the context" — it is a
concrete transformation of the messages array that you must perform *correctly*, and the
single hardest constraint is **not breaking tool_use/tool_result pairing**.

**What compaction is.** When the trajectory grows past a threshold (a token count, a step
count, a fraction of the window), the harness rewrites the older portion of the messages
array into a smaller form, while keeping the run able to continue. Two broad strategies,
usually combined:

- **Summarization** — replace a span of old turns with a model-written summary of what
  happened in them (decisions made, facts found, files touched, open questions).
- **Eviction / pruning** — drop low-value content outright: stale tool results, superseded
  intermediate data, verbose observations no longer relevant.

**The pairing constraint — the hard part.** Recall from Topic 7.2: a `tool_use` block (in
an assistant message) and its matching `tool_result` block (in the following user message)
are a **linked pair**, joined by `tool_use_id`. The API enforces structural rules:

- Every `tool_use` block **must** be followed by a `tool_result` with the matching id.
- A `tool_result` **must** have a preceding `tool_use` with that id.

Naïve compaction breaks this constantly. If you "drop the oldest 10 messages" by a flat
window, you can easily delete an assistant message containing a `tool_use` while keeping
the user message with its `tool_result` — now you have an **orphaned tool_result**, and
the next API call is **rejected as malformed**. Or you summarize across a boundary and
leave a `tool_use` with no result. **Compaction must operate on whole tool_use/tool_result
pairs, never splitting one.**

**The rules for correct compaction:**

1. **Treat a tool_use/tool_result pair as one atomic unit.** When you summarize or evict,
   the unit you remove or replace is the *pair* (and any reasoning text bound to it) —
   never one half.
2. **Preserve the structural prefix verbatim.** The system prompt, tool definitions, and
   the original goal are not candidates for compaction — they are load-bearing every turn.
3. **Compact a *contiguous middle span*, not scattered messages.** Keep the recent N
   turns intact (recency matters — the model needs fresh detail to act now) and keep the
   prefix intact; summarize the contiguous span *between* them. The replacement summary
   is itself a normal text message (typically a user message, or appended to the goal).
4. **The summary must carry forward what later steps need** — decisions, discovered
   facts, the current plan/todo state, unresolved errors. This is the lossy art of 8.5;
   getting it wrong causes goal drift and re-work (8.8).
5. **Optionally offload before evicting.** Write the dropped detail to long-term memory
   (8.5) so it can be re-retrieved, rather than lost outright.

A subtle case: if the *most recent* turn is itself a tool_use awaiting its result, you
finish that pair before compacting — you never compact a half-open step.

### Key terms

- **Compaction** — rewriting the older portion of the messages array into a smaller form (summary + eviction) while keeping the run continuable.
- **tool_use/tool_result pairing** — the API-enforced rule that each tool_use block has exactly one matching tool_result (by `tool_use_id`); compaction must preserve it.
- **Orphaned tool_result** — a tool_result whose paired tool_use was removed (or vice versa); produces a malformed, API-rejected request.
- **Atomic compaction unit** — a whole tool_use/tool_result pair (plus bound reasoning); the indivisible thing compaction removes or replaces.
- **Contiguous-middle compaction** — keep the prefix and the recent N turns verbatim; summarize only the contiguous span between them.

### Common misconceptions

- ❌ "Compaction is just truncating the oldest messages." → ✅ A flat truncation routinely splits a tool_use from its tool_result, producing an orphaned block and an API-rejected request. Compaction must remove/replace whole pairs.
- ❌ "The harness handles compaction; I don't need the mechanics." → ✅ Whatever does it must obey the pairing rules and choose what to preserve; if you build or configure a harness, this is your correctness problem, not magic.
- ❌ "Summarize the whole history into one blob." → ✅ Recent turns must stay verbatim (the model needs fresh detail to act), and the prefix must stay verbatim (it is load-bearing). You summarize the *contiguous middle*, not everything.
- ❌ "A tool_result can stand alone if it's informative." → ✅ The API requires every tool_result to have a matching preceding tool_use; an orphan is structurally invalid regardless of how useful its content is.

### Worked example

An agent's trajectory, oldest to newest (each line one message):

```
[0] system  : system prompt + tool definitions          ← PREFIX, never compact
[1] user    : goal — "audit dependencies for CVEs"       ← PREFIX, never compact
[2] assistant: text + tool_use(id=t1, list_dependencies)
[3] user    : tool_result(t1) → 240-line dependency list   ┐ pair A
[4] assistant: text + tool_use(id=t2, check_cve, lodash)
[5] user    : tool_result(t2) → "lodash 4.17.20: CVE-... " ┐ pair B
[6] assistant: text + tool_use(id=t3, check_cve, express)
[7] user    : tool_result(t3) → "express 4.18.1: no known CVE" ┐ pair C
[8] assistant: text + tool_use(id=t4, check_cve, axios)
[9] user    : tool_result(t4) → "axios 0.21.1: CVE-2021-... high"  ┐ pair D
```

Context is too large; you compact. **Wrong way** — "drop messages [2]–[5]": this deletes
the assistant message [4] holding `tool_use(t2)` but, if done sloppily by a flat token
window, can leave message [5]'s `tool_result(t2)` — an **orphaned tool_result**, request
rejected. Even dropping [2]–[5] cleanly throws away that *pair B found a CVE* — a
load-bearing finding.

**Right way.** Keep the prefix `[0],[1]` verbatim. Keep the recent turns `[8],[9]`
verbatim (the model is mid-step on axios). Compact the **contiguous middle** — the whole
pairs A, B, C (`[2]–[7]`) — into one replacement message, removing *whole pairs only*:

```
[0] system   : (unchanged)
[1] user     : (unchanged)
[2'] user    : "[Compacted summary of steps 1–3] Listed 240 dependencies.
                Checked lodash → CVE-2021-23337 (prototype pollution, high).
                Checked express 4.18.1 → no known CVE. axios check in progress."
[8] assistant: text + tool_use(id=t4, check_cve, axios)
[9] user     : tool_result(t4) → "axios 0.21.1: CVE-2021-... high"
```

Every remaining `tool_use` (`t4`) still has its matching `tool_result`; no orphans. The
lodash CVE — a load-bearing finding — survived in the summary. The 240-line dependency
list, no longer needed verbatim, was dropped (and could be written to long-term memory if
a later step might need it). The run continues correctly with a much smaller context.

### Check questions

1. A harness compacts by deleting the oldest 30% of messages by token count, and the very next API call fails as "malformed." Explain the most likely structural cause and the rule that prevents it. — **Answer:** The flat 30%-by-tokens cut almost certainly fell *inside* a tool_use/tool_result pair — it deleted an assistant message containing a `tool_use` block while keeping the following user message with the matching `tool_result` (or vice versa). That leaves an **orphaned tool_result** (a result with no preceding tool_use of that id), which the API rejects as malformed. The rule: compaction must treat each tool_use/tool_result pair as an *atomic unit* and only remove or replace whole pairs — never split one.
2. Describe what compaction must keep verbatim and what it may summarize, and justify each choice. — **Answer:** Keep verbatim: (1) the **prefix** — system prompt, tool definitions, goal — because it is load-bearing on every turn and re-sent regardless; and (2) the **most recent N turns**, because the model needs fresh, full-detail context to choose its next action well (recency). Summarize: the **contiguous middle span** between them — older completed steps whose full detail is no longer needed turn-to-turn — replacing whole tool_use/tool_result pairs with a summary. The summary must still carry forward decisions, found facts, plan state, and open errors, or the agent drifts and re-does work.
3. When compacting, why is it not enough for the summary to merely be "short" — and what is the failure if it omits, say, a CVE that an earlier tool result found? — **Answer:** Compaction is *lossy*, and the danger is dropping *load-bearing* information, not just length. If the summary omits a CVE an earlier step discovered, that fact is now gone from the agent's only working memory (the context) — the agent will behave as if it never found it: it may re-run the check (wasted steps, more quadratic cost — 8.12), or, worse, conclude the dependency is safe and skip remediation. A correct summary is *short and complete on the load-bearing items* — decisions, findings, plan state, open errors — summarizing away only genuinely low-value detail; offloading dropped detail to long-term memory is the safety net.

---

## 8.14 — Observability: trace and span structure

### Concept

Section 8.6 named logging as a core harness component and 8.11 made it a precondition for
trajectory evaluation. This section makes it concrete: *what structure* do you record so a
failed agent run can actually be reconstructed?

The wrong answer is "print log lines." Flat, unstructured logs from a non-deterministic,
deeply-nested agent run are nearly impossible to reconstruct after the fact. The right
structure is the one borrowed from distributed-systems observability: a **trace** built of
nested **spans**.

- A **trace** is the record of *one complete agent run* — one task, start to finish. It
  has a unique `trace_id`.
- A **span** is one *timed unit of work* inside the run — a model call, a tool execution,
  a compaction pass, a subagent invocation. Each span has its own `span_id` and a
  `parent_span_id`, so spans form a **tree** that mirrors the agent's actual structure: a
  subagent's spans nest under the orchestrator span; a tool execution nests under the loop
  step that triggered it.

This tree is what lets you answer "what happened, in what order, nested how" for a run you
cannot reproduce on demand (non-determinism, 8.8).

**A concrete span schema.** Every span, regardless of type, carries:

```json
{
  "trace_id":       "trc_9f2a...",        // the run this belongs to
  "span_id":        "spn_4b81...",        // this span
  "parent_span_id": "spn_1a00...",        // null for the root span
  "name":           "model_call",         // or tool_call, compaction, subagent, ...
  "span_type":      "llm",                 // llm | tool | compaction | subagent | guardrail
  "start_time":     "2026-05-22T14:03:01.220Z",
  "end_time":       "2026-05-22T14:03:04.880Z",
  "status":         "ok",                  // ok | error
  "step_index":     7,                     // which loop iteration (8.2)
  "attributes":     { ... },               // type-specific fields, below
  "error":          null                   // populated on status=error
}
```

The **`attributes`** differ by `span_type` — record the fields you would need to debug
*that* kind of step:

- **`llm` span** (a model call): `model`, `input_messages` (or a content hash + a pointer
  to the stored payload), `output` (text and any tool_use blocks), `stop_reason`,
  `input_tokens`, `output_tokens`, `cached_tokens`, `cost_usd`, `latency_ms`,
  `temperature`. Tokens and cost per call are what let you reconstruct the 8.12 cost
  curve for a real run.
- **`tool` span** (a tool execution): `tool_name`, `tool_use_id` (ties it to the model
  call that requested it — Topic 7.2), `arguments`, `result` (or `is_error` + message),
  `duration_ms`. Recording `tool_use_id` is what links a tool span to its parent llm span.
- **`compaction` span**: `tokens_before`, `tokens_after`, `messages_removed`,
  `summary_length` — so you can see *what the agent forgot and when* (a frequent root
  cause of goal drift — 8.8).
- **`guardrail` span**: which check ran, `decision` (`allow` | `block` | `escalate`),
  and why — so a blocked or escalated action is visible in the trace.
- **`subagent` span**: the child `trace_id`, the subtask given, the result returned — the
  parent's view of a delegated unit (8.7).

**The run-level record** (the root span / trace summary) carries the things you sort and
alert on: `task`, `final_status` (`success` | `failed` | `stopped: step_limit` | …),
`total_steps`, `total_tokens`, `total_cost_usd`, `wall_clock_ms`, `tool_error_count`,
`intervention_count` — the operational metrics of 8.11.

**Why this structure specifically:**

- **Reconstruction** — the span tree replays the exact sequence and nesting of a run you
  cannot reproduce. You can see step 7 called the wrong tool, step 8 built on its bad
  result (error cascade — 8.8), and exactly when compaction dropped the goal.
- **It is the substrate for evaluation** — trajectory eval (8.11) runs over the recorded
  span tree; without it there is no trace to score.
- **It localizes cost and latency** — per-span tokens/cost/duration show *which* steps
  and tools are expensive or slow, instead of a single opaque run total.
- **It is correlatable** — `trace_id` / `span_id` / `parent_span_id` is the same model
  production tracing tools (and the OpenTelemetry GenAI conventions) use, so agent traces
  slot into the same observability backend as the rest of your system (Topic 12.7).

A practical note: store large payloads (full message arrays, big tool results) out of the
span itself — keep a hash or a pointer in the span and the blob in object storage — so the
trace stays queryable and cheap while the full detail is still retrievable.

### Key terms

- **Trace** — the structured record of one complete agent run, identified by a `trace_id`.
- **Span** — one timed unit of work within a run (a model call, tool execution, compaction, subagent), with `span_id` and `parent_span_id`.
- **Span tree** — spans nested via `parent_span_id` so the structure mirrors the agent's actual execution (subagent spans under the orchestrator, tool spans under loop steps).
- **Span attributes** — type-specific recorded fields (tokens/cost for an llm span, arguments/result for a tool span, before/after sizes for a compaction span).
- **Run-level metrics** — per-trace totals: final status, total steps, total tokens/cost, wall-clock, tool-error and intervention counts.

### Common misconceptions

- ❌ "Plain text log lines are enough to debug an agent." → ✅ A non-deterministic, deeply-nested run needs a structured trace of nested spans; flat logs cannot be reliably reconstructed into the run's actual sequence and nesting.
- ❌ "One log entry per run is fine." → ✅ You need a span per unit of work (each model call, tool call, compaction, subagent) — that granularity is what localizes which step failed, looped, or was expensive.
- ❌ "Just put the full message arrays in every span." → ✅ That bloats the trace store and slows queries; keep a hash/pointer in the span and the large payload in object storage.
- ❌ "Observability is a production-only concern." → ✅ The same span/trace structure is the substrate trajectory evaluation (8.11) runs over — it is needed in development and CI, not only in production.

### Worked example

A coding agent run fails: the final answer is wrong. The flat-log version is a 4,000-line
text dump — effectively unreadable. The **trace** version: one `trace_id`, a root span
showing `final_status: "success"` (the model wrongly thought it was done — 8.2), 18 child
spans.

Querying the span tree: span at `step_index: 6` is a `tool` span,
`tool_name: "run_tests"`, `status: "error"`, `is_error: true` — the tests failed. The
`llm` span at `step_index: 7` shows the model *read* that failure but its output
misinterpreted it. A `compaction` span at `step_index: 11` shows `messages_removed: 8` and
a `summary_length` that — on inspection — dropped the original failing-test detail. From
`step_index: 12` on, the agent is "fixing" the wrong thing — a visible **error cascade**
seeded at step 7 and worsened by a lossy compaction at step 11. None of that is findable
in a flat log; the span tree makes the failure's *origin and propagation* explicit. The
per-span `cost_usd` totals also show step 7–18 burned 70% of the run's cost re-sending a
trajectory built on the bad result — the 8.12 curve, made visible.

### Check questions

1. Why is `parent_span_id` essential rather than just recording a flat ordered list of spans? Give a concrete agent structure where a flat list loses critical information. — **Answer:** `parent_span_id` makes the spans a *tree* that mirrors the agent's real execution structure; a flat ordered list only preserves time order. Concrete case: an orchestrator with three subagents (8.7). In a flat list you see a stream of model and tool spans interleaved in time and cannot tell which tool call belonged to which subagent — the nesting is lost. With `parent_span_id`, each subagent's spans nest under its `subagent` span, so you can isolate one subagent's behavior. Tool spans nesting under the loop step that requested them is the same point at smaller scale.
2. An agent run produced a wrong answer. Name three specific span types you would inspect and what each would tell you about the failure. — **Answer:** (1) **`tool` spans** — check `status`/`is_error` and `result`: a failed or wrong tool result is a common root cause, and `tool_use_id` ties it to the model call that requested it. (2) **`llm` spans** — check the `output` and `stop_reason` around the suspected step: did the model misread a tool result, pick the wrong tool, or wrongly emit `end_turn`? (3) **`compaction` spans** — check `tokens_before/after` and `summary_length`: a lossy compaction that dropped a load-bearing fact (8.13) causes the agent to "forget" and go wrong afterward. Together they let you locate where the trajectory first went bad and how it propagated (error cascade).
3. A team stores the full input message array inside every `llm` span. As runs get longer their trace store becomes slow and expensive to query. Explain the cause via the 8.12 cost model and give the standard fix. — **Answer:** By 8.12, each step's input message array is the *whole growing trajectory*, so step n's array is ~`P + (n−1)·c` tokens; storing the full array in every span re-stores the trajectory ~quadratically — the trace store grows like the run's own quadratic cost curve. The standard fix: keep only a content hash (or a pointer/key) in the span and store the large payload once in object storage. The span tree stays small and fast to query; the full message array is still retrievable on demand via the pointer.

---

## Topic 08 — Exam Question Bank

A pool the tutor draws from for the gated exam. Mixed-format, scored /100, 85 to pass.

### True / False

1. A two-call pipeline — classify input, then run a branch chosen by a hardcoded `if` on the class — qualifies as an agent because it runs the model more than once and uses a tool. — **Answer:** False. Running the model multiple times and using tools are not sufficient; an agent requires *model-driven* control over each next step. Here the developer hardcoded the path, so it is a workflow.
2. If an agent needs a fact it discovered earlier but that fact has scrolled out of its context, it can still rely on having "experienced" it. — **Answer:** False. An agent has no hidden state — it knows only what is currently in the context window. A fact no longer in context is simply gone unless re-retrieved from long-term memory.
3. Because modern tool-use APIs implement ReAct natively, the reasoning ("Thought") aspect of ReAct no longer matters. — **Answer:** False. The APIs remove the need to hand-write the literal text format, but the *pattern* — interleaving reasoning with action, grounded by observation — still matters; the Thought is now the model's text or reasoning block.
4. Writing a fact to an agent's long-term store during a run guarantees the agent will use that fact later in the same run. — **Answer:** False. Long-term memory is invisible until explicitly *retrieved* back into context. A write alone does nothing; the agent must read it in for it to have any effect.
5. A team can safely omit budget/step limits if their agent has tested reliably on their golden set. — **Answer:** False. Budget limits are mandatory regardless of test results — an untested input, an ambiguous task, an adversarial prompt, or a loop turns directly into an unbounded bill and latency. Passing a golden set does not bound worst-case runs.
6. For a task whose subtasks are tightly coupled and constantly need each other's state, a multi-agent design is usually the more reliable choice. — **Answer:** False. Tightly-coupled subtasks are exactly where multi-agent *hurts* — agents don't share a context window, so it produces inconsistent results and error cascades. A single agent is more reliable here; multi-agent suits *separable* tasks.
7. Two agents with the same final-outcome pass rate are equally good and either can be shipped. — **Answer:** False. Equal outcome rates can hide very different trajectories — one cheap/clean/safe, one expensive/looping/unsafe — and different variance. Trajectory and operational eval are needed to choose.
8. When the agentic loop exits on `end_turn`, that confirms the task was completed correctly. — **Answer:** False. `end_turn` means only that the model stopped requesting tools; it can stop early or hallucinate completion. Correct completion must be separately verified.
9. A 30-step agent run costs roughly 30× a single LLM call. — **Answer:** False. Because the API is stateless, every step re-sends the whole growing trajectory; total input tokens scale ~quadratically (`N·P + c·N(N−1)/2`), so a 30-step run typically costs several times the naive 30× estimate.
10. Compacting an agent's context by deleting the oldest messages by a flat token count is a safe default. — **Answer:** False. A flat cut routinely splits a tool_use block from its matching tool_result, producing an orphaned block that makes the next API call malformed. Compaction must remove or replace whole tool_use/tool_result pairs.
11. Self-critique — having a model review and revise its own output with no external check — reliably improves results. — **Answer:** False. A model critiquing its own work shares the blind spots that produced it. Reflection is reliable only when grounded by an external signal — a deterministic verifier (tests, schema validation) or a real tool result.
12. An agent trajectory is best recorded as one structured trace of nested spans, not a flat stream of text log lines. — **Answer:** True. A non-deterministic, deeply-nested run needs a span tree (spans linked by `parent_span_id`) to be reconstructable; flat logs cannot reliably recover the run's sequence and nesting.

### Multiple Choice

1. Two systems both call an LLM repeatedly with tools. System X follows a fixed developer-written sequence of steps; System Y lets the model choose each next step from what it has observed. Which is the agent, and on what criterion?
   A. X — it is more predictable
   B. Y — the model, not the developer, decides the control flow
   C. Both — both use tools and loop
   D. Neither — both are workflows
   — **Answer:** B. The decisive criterion is *who controls the flow*. Y is the agent (model-driven control); X is a workflow despite looping and using tools.

2. An "optimization" has the model emit a full list of tool calls once, then the harness runs them all without further model calls. What does this break?
   A. Nothing — it is strictly faster
   B. It removes the Observe/re-Think step, so the agent can no longer adapt to results
   C. It removes the Act step
   D. It only affects logging
   — **Answer:** B. Pre-committing all calls deletes observation and re-decision — the agent loses the ability to adapt to surprises, recover from errors, or handle result-dependent steps. The loop's value is the feedback.

3. A task's subtasks are tightly coupled and constantly need each other's intermediate state. Using subagents here would most likely:
   A. Improve reliability through context isolation
   B. Hurt — knowledge stranded in separate contexts causes inconsistent results and cascades
   C. Have no effect either way
   D. Guarantee a speedup
   — **Answer:** B. Subagents help when subtasks are *separable*; for tightly-coupled work, separate non-shared contexts strand needed state and cause inconsistency and error cascades. Context isolation is a benefit only when isolation is appropriate.

4. An agent must recall a user's preferences from previous sessions weeks ago. Where must that information live, and what must happen for the agent to use it?
   A. In the context window; nothing extra — it is always present
   B. In an external long-term store; it must be explicitly retrieved into context
   C. In the system prompt; it updates itself
   D. In the model's weights; it is memorized during the conversation
   — **Answer:** B. Cross-session information belongs in long-term memory (an external store); it is invisible to the model until a retrieval step pulls it into the context window.

5. An agent repeats the same failing tool call over and over. The most direct harness defense is:
   A. A larger context window   B. A step/iteration limit plus loop detection
   C. A more powerful model   D. Disabling parallel tool calls
   — **Answer:** B. A hard step cap bounds the runaway and loop detection flags repeated identical actions.

6. On long runs an agent's output quality drops with no error raised — it forgets the goal and repeats work. Which harness component is responsible for preventing this, and why is it not the retry component's job?
   A. Retry logic — it should retry the degraded steps
   B. Context management — it compacts/evicts/retrieves to keep context effective; retries only handle failed calls, not a bloated-but-successful context
   C. Tool dispatch — it should route around the problem
   D. Logging — it should detect and fix the drift
   — **Answer:** B. Silent degradation is a context problem (lost-in-the-middle), addressed by context management. Retry logic only re-attempts *failed* calls; here calls succeed while the context quietly rots.

7. A step limit trips mid-run. The best harness behavior is to:
   A. Throw an unhandled exception so the caller notices
   B. Silently stop and return nothing
   C. Stop gracefully, returning a clear "stopped: step limit" status, the best partial result, and the trajectory
   D. Raise the limit and continue
   — **Answer:** C. Graceful stop converts a runaway into a bounded, inspectable failure the caller can act on; an exception or silent stop discards the partial progress and trajectory needed to diagnose it.

8. An agent reaches the correct final answer but took a 40-step, high-cost, partly-unsafe path. Which evaluation approach reveals this, and why does the other miss it?
   A. Final-outcome eval reveals it; trajectory eval only checks the answer
   B. Trajectory eval reveals it; final-outcome eval only checks the end state, not how it was reached
   C. Neither can detect path quality
   D. Both detect it equally well
   — **Answer:** B. Final-outcome eval is blind to *how* — it only checks whether the goal was met. Trajectory (plus operational) eval inspects the process: step count, cost, tool choices, safety adherence.

9. An agent run is dominated by input-token cost that grows much faster than the step count. The mechanism is:
   A. The model generates more output each step
   B. A stateless API re-sends the entire growing trajectory every step, so the trajectory's tokens are billed ~quadratically in the step count
   C. The tool definitions grow each step
   D. Temperature increases over the run
   — **Answer:** B. Statelessness forces re-sending the whole conversation each step; total input tokens scale as `N·P + c·N(N−1)/2`. The quadratic term is the re-billed trajectory history. Output (A) is generated once per step and is the linear part.

10. To meaningfully cut the cost of a long agent run, the highest-leverage lever is:
   A. Trimming a few tokens from each tool description
   B. Reducing the number of steps (better tools/planning) and bounding context size via compaction
   C. Lowering the temperature
   D. Using a larger context window
   — **Answer:** B. Cost is ~quadratic in step count, so reducing steps — and capping how large the re-sent context grows via compaction — bends the curve down far more than per-step token trimming. A larger window (D) does not reduce what is billed.

11. A harness compacts context by deleting the oldest 25% of messages by token count, and the next API call is rejected as malformed. The most likely cause is:
   A. The context is still too large
   B. The cut split a tool_use block from its matching tool_result, leaving an orphan
   C. The system prompt was removed
   D. The model temperature is too high
   — **Answer:** B. A flat by-token cut routinely falls inside a tool_use/tool_result pair, leaving an orphaned block the API rejects. Compaction must operate on whole pairs and keep the prefix and recent turns intact.

12. The most reliable form of agent self-improvement among these is:
   A. The model critiques its own output with no external check
   B. A Reflexion-style loop driven by a deterministic verifier (e.g. a real test run), with a natural-language reflection fed into the next attempt
   C. Raising the temperature so the model "thinks differently" on retry
   D. Re-running the same prompt until it succeeds
   — **Answer:** B. A grounded, external verifier signal (a real test run) is trustworthy in a way self-critique (A) is not, and the reflection turns the failure signal into a concrete correction. D is a blind retry; C does not address the cause.

13. In an agent observability trace, what makes the spans a *tree* rather than a flat list, and why does it matter?
   A. `start_time` ordering — it preserves chronology
   B. `parent_span_id` linking each span to its parent — it reconstructs the run's actual nesting (subagent spans under the orchestrator, tool spans under their loop step)
   C. `trace_id` — it groups spans by run
   D. `span_type` — it categorizes spans
   — **Answer:** B. `parent_span_id` builds the tree that mirrors the agent's real execution structure; a flat ordered list keeps only time order and loses which tool call belonged to which subagent or loop step.

14. τ-bench reports **pass^k**. What does pass^k measure that ordinary pass@k does not?
   A. The single best run out of k
   B. Whether the agent succeeds on *all* k attempts — i.e. reliability/consistency
   C. The average cost over k runs
   D. The number of tool calls per run
   — **Answer:** B. pass@k credits *any* success in k attempts (best-case capability); pass^k requires success on *every* attempt, surfacing the non-determinism risk that matters for production readiness.

### Short Answer

1. You are given two systems for "respond to a customer email": System A always runs detect-language → draft-reply → tone-check in that fixed order; System B reads the email and decides for itself whether to look up an order, escalate, or just reply. Classify each as agent or workflow and explain what would have to change about System A to make it an agent. — **Model answer:** A is a workflow — the developer fixed the three-step path; the model never decides what happens next. B is an agent — the model chooses each next step from what it observes. To make A an agent, control of the path must move to the model: instead of a hardcoded sequence, the model would decide (in a loop, with tools and observed results) whether to detect language, look something up, escalate, or reply — the steps and their order becoming model-driven rather than predetermined.
2. An agent "remembers" the user's last three messages but not a preference the user stated last month, even though that preference is in a database. Explain, in terms of the two memory types, why the recent messages are available and the month-old preference is not — and what single mechanism closes the gap. — **Model answer:** The recent messages are in **short-term memory** — the live context window the model reasons with directly. The month-old preference sits in **long-term memory** — an external store that is durable but *invisible to the model until retrieved*. Having it in a database is not enough; the agent only knows what is in context. The mechanism that closes the gap is a **retrieval step/tool** that reads the relevant long-term record into the context window.
3. A reasoning-model agent does extensive internal thinking but has no tools, and it confidently returns a wrong, out-of-date fact. Explain which half of ReAct it is missing and why the internal reasoning could not save it. — **Model answer:** It is missing the **Act/Observe** half — interleaved actions and the real observations they return. ReAct couples reasoning with grounding: actions inject external facts that correct reasoning when it drifts. Internal reasoning, however extensive, cannot ground itself — it has no channel to the outside world, so it can fluently and confidently reason to a wrong answer. The fix is not "more thinking" but adding tool actions whose observations anchor the reasoning in reality.
4. An agent built on a frontier model is unreliable in production: it occasionally runs away on cost, sometimes ignores a permission rule, and is impossible to debug after the fact. For each of the three symptoms, name the harness component that is missing or weak and what it should do. — **Model answer:** Cost runaway → **termination logic** is missing/weak — it should enforce step/token/cost/time budgets and stop gracefully. Ignored permission rule → **guardrails / tool dispatch** is missing/weak — permission checks and allowlists must be deterministic code in the dispatch layer, not a prompt instruction. Impossible to debug → **logging/observability** is missing — per-step traces of model and tool calls are required because agents are non-deterministic. The through-line: the model supplies judgment, but reliability, safety, and debuggability are harness jobs no model provides.
5. A team runs each agent task once on their golden set and reports the pass/fail. Explain what is statistically wrong with this and what a sound evaluation reports instead — and why a "90% pass rate" can still be insufficient information. — **Model answer:** One run per task is a single sample from a non-deterministic distribution — a lucky pass or unlucky fail tells you almost nothing, and a failure mode appearing in a minority of runs stays invisible. A sound evaluation runs each task many times and reports a **pass-rate** (pass@k-style) plus trajectory-quality and operational metrics. Even "90%" is insufficient without variance: a 90% mean with high run-to-run variance is materially worse and riskier than a steady 90%, and outcome rate alone still hides cost, efficiency, and safety of the trajectories.

6. Derive why an N-step agent costs much more than N× a single LLM call. State the total-input-token expression, identify which term is quadratic and why, and name one design lever and one infrastructure lever that each address it. — **Model answer:** The API is stateless, so every step re-sends the entire conversation accumulated so far. With a fixed prefix `P` (system prompt + tool definitions + goal) and ~`c` tokens added per step, step n is billed `P + (n−1)·c` input tokens; summing over N steps gives `total_input = N·P + c·N(N−1)/2`. The `c·N(N−1)/2` term is **quadratic in N** — each step's tokens are re-billed on every later step (triangular re-billing). A naive ×N estimate counts each step's tokens once and so badly under-estimates. *Design lever:* reduce step count (better tools/planning) and bound context growth with compaction, which bends the quadratic toward linear. *Infrastructure lever:* prompt caching on the fixed prefix (and incremental caching of the history), which makes the re-sent tokens cheaper to read though not free.

7. Explain the single hardest correctness constraint when compacting an agent's context, what goes wrong if it is violated, and the rule that prevents it. — **Model answer:** The hard constraint is **preserving tool_use/tool_result pairing**. Each tool_use block (assistant message) has exactly one matching tool_result (following user message), linked by `tool_use_id`, and the API structurally requires every tool_use to have its result and every tool_result to have its tool_use. Naïve compaction — e.g. dropping the oldest messages by a flat window — splits a pair, leaving an **orphaned tool_use or tool_result**; the next API call is then rejected as malformed. The rule: treat each tool_use/tool_result pair as an *atomic unit* and only remove or replace whole pairs — keep the prefix (system prompt, tools, goal) and the recent N turns verbatim, and summarize only the contiguous middle span, replacing complete pairs with a summary that still carries forward decisions, findings, plan state, and open errors.

### Long Answer

1. Whiteboard the anatomy of an agent harness: name each component and what it does, then defend the principle "the model provides judgment, the harness provides reliability" by arguing what specifically would go wrong if you removed each component but kept a frontier model. — **Model answer / rubric:** Should enumerate and explain the components AND tie each to a concrete failure-on-removal: the loop (without it there is no agent — one shot, no iteration); tool dispatch (without it tool_use blocks are never routed/validated/executed, and deterministic action guardrails have nowhere to live); context management (without it the context overflows or silently rots on long runs); retries/error handling (without it a transient rate-limit or timeout kills the whole run); guardrails (without it a wrong or injected tool request is acted on — no enforcement below the model); termination logic (without it a misbehaving agent runs away on cost/latency unbounded); logging/observability (without it a non-deterministic failure cannot be debugged or evaluated). The principle: the model contributes judgment/reasoning, but reliability, safety, cost control, context management, and error handling are deterministic harness jobs no model solves for you. Strong answers conclude that a frontier model with a flimsy harness is an unreliable demo, and that production fixes usually land in the harness, not the model.
2. A research agent given an under-specified, long-running task fails badly. Walk through the most likely *chain* of failure modes from start to finish, explain why they compound rather than occur independently, give a coordinated set of defenses, and explain why fixing only one defense is insufficient. — **Model answer / rubric:** A plausible chain: the ambiguous goal leads the agent to wander into a subproblem; the context grows and **context overflow** sets in (silent lost-in-the-middle degradation before any hard limit); the original goal loses salience → **goal drift** (it starts solving a different problem); a misread or stale observation becomes an input to later steps → **error cascade**; unable to progress, it repeats actions → **unproductive/infinite loop**; **tool misuse** may worsen things throughout. Should explicitly explain the *compounding*: overflow worsens drift, drift worsens looping, cascades feed all of them — not independent failures. Coordinated defenses: step/iteration limits and loop detection; context management/compaction and subagent isolation; periodic goal re-statement and a maintained todo list; verification of intermediate results and grounding in observations; actionable tool errors; fewer/clearer tools with schema validation. Should stress that a single defense (e.g. just a step cap) bounds the runaway but leaves drift and overflow intact — the modes compound, so defenses must too — and that non-determinism means these are found via repeated evaluation and observability, not one happy-path run.
3. Discuss when to use a multi-agent system versus a single agent, covering both topologies and the trade-offs. — **Model answer / rubric:** Topologies: orchestrator/worker (hierarchical — lead decomposes, delegates to parallel workers with focused contexts, integrates results) and handoffs (sequential — control passes between specialized agents). Benefits: context isolation, specialization, parallelism, separation of concerns. Costs: multiplied cost/latency, coordination overhead, error cascades, lost shared context, harder debugging. Guidance: a single well-designed agent is the right default; reach for multi-agent only when subtasks are genuinely separable/parallelizable with clean interfaces or generate lots of intermediate context that would pollute a shared window. For tightly-coupled tasks, multi-agent adds failure modes without benefit. Strong answers give a good-fit example (parallel research) and a poor-fit example (coupled refactor).
4. Explain trajectory evaluation versus final-outcome evaluation, why both are needed, and how non-determinism shapes agent evaluation. — **Model answer / rubric:** Final-outcome eval = did it achieve the goal (often objectively checkable, simple, but blind to *how* — can't distinguish a lucky/expensive/unsafe success). Trajectory eval = judges the process: tool choices, argument correctness, efficiency, looping, safety adherence, error recovery (rich diagnostics on *why/where*, but harder — no single correct path, needs LLM-judge/rubric scoring over traces plus good logging). Both needed: outcome says *whether*, trajectory says *why/how reliably/how efficiently/how safely*. Non-determinism: each task must be run many times; success is a pass-rate (pass@k-style), not a single pass/fail. Should also mention operational metrics (cost/task, latency, step count, tool-error rate, intervention rate) and using the eval as a regression gate for any prompt/tool/model/harness change.
5. Discuss human-in-the-loop checkpoints: why they exist, where to place them, the central trade-off, and why they don't replace code-level guardrails. — **Model answer / rubric:** Why: agents are probabilistic and can be wrong, injected, or confidently mistaken; for high-stakes/irreversible/costly actions the expected error cost outweighs the friction of asking a human. Where: before irreversible/dangerous actions (payments, deletes, deploys, external comms), at plan approval, on low confidence/ambiguity, on limit triggers, at final-output review. Patterns: approve/reject, edit, escalation/handoff, confirmation. Trade-off: autonomy/speed/scalability versus safety/control — risk-calibrated, gating only irreversible/costly/externally-visible actions while letting reversible low-stakes actions run autonomously. Why not a replacement for guardrails: humans asked to approve many or opaque actions suffer approval fatigue and rubber-stamp; deterministic code-level guardrails must handle the bulk, with HITL reserved for a small number of clearly-presented high-stakes decisions.

6. Explain the cost model of a long agent run and the role of context compaction. Derive why cost is super-linear in step count, explain how compaction both controls cost *and* threatens correctness, and state the rules that make compaction safe. — **Model answer / rubric:** *Cost derivation:* the API is stateless, so each step re-sends the whole accumulated trajectory; with a fixed prefix `P` and ~`c` tokens added per step, step n is billed `P + (n−1)·c` input tokens, and `total_input = N·P + c·N(N−1)/2`. The `c·N(N−1)/2` term is quadratic in N — each step's tokens are re-billed on every later step — so an N-step agent costs far more than N× a single call (a "30×" estimate can be several times low). Output is generated once per step (linear); the quadratic blow-up is on the input side. *Compaction as cost control:* compaction caps how large the re-sent context `c·n` grows, bending the back half of a run from quadratic toward linear; subagent isolation does the same; prompt caching makes the prefix and history cheaper to re-read. *Compaction as correctness threat:* compaction is lossy — a summary that drops a load-bearing fact causes the agent to "forget" and re-derive or go wrong (goal drift, error cascade). And mechanically it can produce a *malformed request*: if it splits a tool_use block from its tool_result it leaves an orphan the API rejects. *Rules for safe compaction:* treat each tool_use/tool_result pair as an atomic unit and only remove/replace whole pairs; keep the prefix (system prompt, tools, goal) and the recent N turns verbatim; summarize only the contiguous middle span; ensure the summary carries forward decisions, findings, plan state, and open errors; optionally offload dropped detail to long-term memory for re-retrieval. Strong answers connect all three: compaction is simultaneously a cost lever (8.12) and a correctness hazard (8.13), and the pairing rule is the non-negotiable mechanical constraint.

### Applied Scenario

1. Your production research agent occasionally costs 30× the normal amount on certain queries and the run takes 8 minutes. Investigation shows it searches, finds nothing conclusive, and searches again — for dozens of iterations. What failure modes are present and what do you add to the harness? — **Model answer / rubric:** Failure modes: an unproductive/infinite loop (repeating searches without progress), likely context overflow as repeated results pile up, and possibly goal drift. Harness additions: a hard step/iteration limit and a token/cost budget (8.9) so the run stops gracefully with a partial result and clear status; loop detection to flag repeated near-identical searches; context management/compaction so repeated results don't bloat the window; make the search tool return actionable signals ("no results — query may be unanswerable") so the model can stop or escalate; consider a HITL escalation when a limit trips. Should note the agent should be re-evaluated over many runs to confirm the fix.

2. A teammate proposes splitting a "refactor this service and keep all tests green" task across five specialized agents (one per module) to speed it up. Critique this. — **Model answer / rubric:** Poor fit for multi-agent. The subtasks are tightly coupled — a change in one module affects interfaces, tests, and behavior in others — and the agents don't share a context window, so each is blind to the others' changes. Result: inconsistent edits, broken cross-module contracts, error cascades, and lost shared context, while cost and coordination overhead multiply. Multi-agent helps for *separable* tasks; this is the opposite. A single agent with file-edit and test-run tools, a maintained plan, and a step budget is simpler and more reliable. (Speedup, if needed, comes from parallel *tool* calls within the one agent, not parallel agents.)

3. You are designing a coding agent that can edit files, run tests, and deploy to production. Where do you place human-in-the-loop checkpoints and where do you rely on autonomy and code-level guardrails? — **Model answer / rubric:** Autonomy for reversible, low-stakes, sandboxed actions: editing files and running tests in an isolated workspace — no checkpoint, just deterministic guardrails (sandbox, path allowlists, resource limits). HITL approval gate before the irreversible/high-stakes action: deploying to production — the agent proposes the deploy, a human reviews a clear summary (diff, test results) and approves before the harness executes it. Optionally a plan-approval checkpoint for large changes and final-output review. Should stress: gate only the genuinely irreversible/externally-visible step to avoid approval fatigue; deterministic guardrails (least-privilege tools, sandboxing — Topic 7.4/13) handle everything else; HITL complements, not replaces, those guardrails.

4. Two candidate agents for a customer-support task both achieve a 90% final-outcome pass rate on your golden set. How do you decide which to ship? — **Model answer / rubric:** Final-outcome parity is not enough — go to trajectory and operational evaluation. Compare: average step count and tool calls (efficiency), cost per task, latency, tool-error rate, how often each loops or backtracks, how well each recovers from errors, whether each respects safety constraints and escalates appropriately, and human-intervention rate. Run each many times per task and inspect the pass-rate distribution and variance — a 90% mean with high variance is worse than a steady 90%. The agent with the cleaner trajectories, lower cost/latency, fewer unsafe actions, and lower variance is the one to ship. Should mention using this comparison as a regression baseline going forward.

5. A production agent intermittently returns wrong answers; the failure does not reproduce when you re-run the task. Your harness currently emits flat text log lines. Describe how you would re-architect observability so this failure becomes debuggable, including the concrete structure you would record. — **Model answer / rubric:** Flat logs cannot reconstruct a non-deterministic, deeply-nested run, and the failure won't reproduce on demand — so you must capture each run as it happens in a *structured* form. Re-architect to a **trace of nested spans**: one `trace_id` per run; a `span` per unit of work (model call, tool execution, compaction pass, subagent), each with `span_id` and `parent_span_id` so the spans form a *tree* mirroring the agent's real structure (tool spans under their loop step, subagent spans under the orchestrator). Each span records `span_type`, start/end time, `status`, `step_index`, and type-specific `attributes`: an `llm` span carries `model`, input/output (or a hash + pointer to the stored payload), `stop_reason`, input/output/cached tokens, `cost_usd`, `latency_ms`; a `tool` span carries `tool_name`, `tool_use_id`, `arguments`, `result`/`is_error`; a `compaction` span carries `tokens_before/after` and `summary_length`; a `guardrail` span carries the decision. The root span carries run-level metrics (final status, total steps/tokens/cost, wall-clock, tool-error and intervention counts). With this, the specific failed run is fully reconstructable — you can see which step picked a wrong tool, which observation was misread, where an error cascaded, and whether a lossy compaction dropped a needed fact — and the same trace is the substrate trajectory evaluation (8.11) runs over. Strong answers note storing large payloads out-of-line (hash/pointer in the span, blob in object storage) and that `trace_id`/`span_id` correlate with the rest of the system's observability (Topic 12.7).

---

## Sources

[1] Anthropic — Building effective agents (workflow vs. agent definitions; orchestrator-workers) — https://www.anthropic.com/engineering/building-effective-agents
[2] Yao et al. — ReAct: Synergizing Reasoning and Acting in Language Models (arXiv 2210.03629) — https://arxiv.org/abs/2210.03629
[3] Anthropic — How we built our multi-agent research system (orchestrator/subagent pattern; ~15× token usage) — https://www.anthropic.com/engineering/multi-agent-research-system
[4] OpenAI — Handoffs (OpenAI Agents SDK documentation) — https://openai.github.io/openai-agents-python/handoffs/
[5] Liu et al. — Lost in the Middle: How Language Models Use Long Contexts (arXiv 2307.03172) — https://arxiv.org/abs/2307.03172
[6] Shinn et al. — Reflexion: Language Agents with Verbal Reinforcement Learning (arXiv 2303.11366) — https://arxiv.org/abs/2303.11366
[7] τ-bench: A Benchmark for Tool-Agent-User Interaction in Real-World Domains (arXiv 2406.12045) — https://arxiv.org/abs/2406.12045
[8] Berkeley Function-Calling Leaderboard (BFCL) — https://gorilla.cs.berkeley.edu/leaderboard.html
