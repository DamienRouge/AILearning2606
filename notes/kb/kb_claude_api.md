---
name: kb-claude-api
description: "Pointer — for concrete Claude/Anthropic API details (model ids, pricing, params, streaming, tool use, MCP, caching, token counting), invoke the `claude-api` skill, which is authoritative and kept current. This file only exists so [[kb-claude-api]] cross-links resolve."
metadata: 
  node_type: memory
  type: reference
  originSessionId: 18235b91-c1a6-4331-b2d7-3bd32217d84f
---

# Claude / Anthropic API — see the `claude-api` skill

> This is a **pointer**, not a primer. The other kb-* files cross-link `[[kb-claude-api]]` when they touch the concrete API surface (model ids, pricing, message params, streaming format, tool-use blocks, MCP, prompt caching, token counting, model migration).

**For anything concrete about the Claude/Anthropic API, invoke the `claude-api` skill** (via the Skill tool) — it is the authoritative, version-current reference and is maintained by Anthropic. Do NOT answer Claude-API specifics from memory; the skill exists precisely because model ids/pricing/params drift.

Conceptual context lives in the primers:
- The tool-use protocol, `stop_reason`/`tool_use`/`end_turn`, the agent loop, roles, MCP-as-standard → [[kb-agent-loop]].
- SSE streaming format and consuming a token stream → [[kb-streaming-pipeline]].
- Embeddings / vector search (the API's embedding side) → [[kb-rag-retrieval]].

Current-model quick facts (verify against the skill before relying on them): the newest Claude models as of 2026 are the **Claude 5 family**, **Opus 4.8**, and **Haiku 4.5**. Model ids — Fable 5: `claude-fable-5`, Opus 4.8: `claude-opus-4-8`, Sonnet 5: `claude-sonnet-5`, Haiku 4.5: `claude-haiku-4-5-20251001`. When building AI apps, default to the latest, most capable model for the task.
