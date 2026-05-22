# Phase 3 — Agents & Tool Use — Extra Readings

A curated "further reading" list for the whole phase. These go deeper than the course
material. Every URL was verified to resolve at the time of writing (2026-05). They are
organized by topic; within each topic, primary sources (official docs, papers) come
first.

---

## Topic 7 — Tool Use / Function Calling

- **Define tools** — Anthropic (Claude Docs) — https://platform.claude.com/docs/en/agents-and-tools/tool-use/implement-tool-use
  The authoritative reference for Anthropic tool definitions, `tool_choice` modes, the
  tool-use system prompt, `input_examples`, and forced-tool-use behavior. Read this for
  the exact, current Anthropic semantics rather than relying on second-hand summaries.

- **Handle tool calls** — Anthropic (Claude Docs) — https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls
  Covers the `tool_use`/`tool_result` lifecycle, the strict ordering rules (tool_result
  blocks must come first in the user message), `is_error`, and structured/image/document
  result content — detail the material only summarizes.

- **Function calling guide** — OpenAI (API docs) — https://developers.openai.com/api/docs/guides/function-calling
  The OpenAI counterpart: `tool_choice` (`auto`/`none`/`required`/named/`allowed_tools`),
  `parallel_tool_calls`, the `tool`-role result message, and strict structured outputs.
  Read alongside the Anthropic page to internalize how provider-specific the surface is.

- **Writing tools for agents** — Anthropic Engineering — https://www.anthropic.com/engineering/writing-tools-for-agents
  A deep dive on tool *design* — consolidating operations, namespacing, response shaping,
  and token economy. The best long-form treatment of "the description is a prompt."

- **Building with extended thinking** — Anthropic (Claude Docs) — https://platform.claude.com/docs/en/build-with-claude/extended-thinking
  Explains how reasoning interacts with tool use, including the constraint that forced
  tool use (`tool_choice` `any`/`tool`) is incompatible with extended thinking — the
  per-model nuance Topic 7.6 now flags.

- **Model Context Protocol — Specification** — modelcontextprotocol.io — https://modelcontextprotocol.io/specification/2025-11-25
  The primary MCP spec (latest dated revision). Authoritative on primitives, JSON-RPC,
  capability negotiation, and authorization. Use the version selector to compare
  revisions (2024-11-05, 2025-03-26, 2025-06-18, 2025-11-25).

- **MCP Transports** — modelcontextprotocol.io — https://modelcontextprotocol.io/specification/2025-03-26/basic/transports
  The spec page where Streamable HTTP was introduced and HTTP+SSE deprecated — the
  primary source for the transport history Topic 7.8 corrects.

---

## Topic 8 — Agents & Harnesses

- **Building effective agents** — Anthropic Engineering — https://www.anthropic.com/engineering/building-effective-agents
  The single most influential practitioner article on the workflow-vs-agent distinction
  and on common patterns (prompt chaining, routing, orchestrator-workers, evaluator-
  optimizer). Topic 8's framing comes directly from here.

- **ReAct: Synergizing Reasoning and Acting in Language Models** — Yao et al., 2022 (arXiv) — https://arxiv.org/abs/2210.03629
  The original ReAct paper. Worth reading for the actual experiments (HotPotQA, Fever,
  ALFWorld, WebShop) that motivated interleaving reasoning with action.

- **How we built our multi-agent research system** — Anthropic Engineering — https://www.anthropic.com/engineering/multi-agent-research-system
  A candid production case study of an orchestrator/subagent system: what worked, the
  ~15× token cost, prompt-engineering the lead agent, and evaluation challenges. The best
  real-world complement to Topic 8.7.

- **Lost in the Middle: How Language Models Use Long Contexts** — Liu et al., 2023 (arXiv) — https://arxiv.org/abs/2307.03172
  The empirical paper behind "context rot" and the case for context management and
  subagent isolation in 8.5/8.6/8.8.

- **A practical guide to building agents** — OpenAI (PDF) — https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf
  OpenAI's counterpart to Anthropic's agents guide — covers agent design, guardrails, and
  orchestration patterns from a different vendor's perspective; good for triangulation.

- **OpenAI Agents SDK — Handoffs** — OpenAI — https://openai.github.io/openai-agents-python/handoffs/
  Concrete documentation of the handoff multi-agent topology: how control transfer is
  modeled as a tool, input filtering, and prompt conventions. Pairs with Topic 8.7.

- **Effective context engineering for AI agents** — Anthropic Engineering — https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
  Goes deeper than 8.5/8.6 on compaction, retrieval, and managing the context window over
  long agent runs.
