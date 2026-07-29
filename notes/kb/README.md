# Knowledge-base primers

General-knowledge primers on the core technologies behind an LLM agent product, each with the **WCX Engineering Copilot** repo (and this Claude Code session) as worked examples. Written during an agent-architecture ramp-up.

Each primer follows the same shape:
- **PART 0 — Why this matters** (philosophy, deep principles, real-world connection, what breaks without it)
- **PART 1 — General knowledge** (transferable to any system / job / interview)
- **PART 2 — Worked example** (how the WCX repo instantiates it)
- **PART 3 — Transferable takeaways**

Markdown is the source of truth; identical PDFs are in [`pdf/`](pdf/) for offline/printable reading.

## Start here
**[Harness engineering](kb_harness_engineering.md)** is the **capstone** — the 7-layer harness stack (loop → tools → context → safety → orchestration → verification → observability) plus prompt-assembly / cost / human-in-the-loop, and the 5 foundational laws. It ties every other primer together and explains where each one fits. Read the others for depth; read this one for the map.

## The primers

| Primer | Topic |
|--------|-------|
| [Harness engineering](kb_harness_engineering.md) ⭐ | **Capstone.** The layered harness architecture + laws; how a token-predictor becomes a reliable agent |
| [Agent loop](kb_agent_loop.md) | LLM agents: tool-use, the loop & stop conditions, prompt roles, context/statelessness, framework landscape |
| [Streaming pipeline](kb_streaming_pipeline.md) | SSE vs WebSocket vs polling, framing buffers, generators, the snapshot→diff→stream trick, backpressure |
| [Storage model](kb_storage_model.md) | Stateless servers, SQL vs NoSQL, metadata-vs-content split, conversation memory, persistence across reload |
| [Frontend architecture](kb_frontend_architecture.md) | React for a backend dev: declarative UI, props/hooks/reducer, one-way flow, memoization |
| [RAG & retrieval](kb_rag_retrieval.md) | RAG, embeddings, ANN indexes, chunking, retriever vs reranker, hybrid search, MCP |
| [Auth & infra](kb_auth_infra.md) | Secret zero, workload/managed identity, OAuth2/OIDC/JWT, OBO/token exchange, secret managers |
| [Concurrency & async](kb_concurrency_async.md) | Concurrency vs parallelism, event loops, async/await, green threads (gevent/greenlets), the GIL |
| [Distributed patterns](kb_distributed_systems_patterns.md) | Statelessness, idempotency, retries/backoff, caching, policy-vs-mechanism, the managed-service boundary |
| [Claude API](kb_claude_api.md) | Pointer to the authoritative Claude/Anthropic API reference |

## How they connect
```
                    Harness engineering (capstone: the whole stack)
                                     │
   ┌──────────────┬─────────────┬────┴─────┬──────────────┬───────────────┐
 Agent loop    Streaming     Storage    RAG &         Auth & infra   Concurrency
 (L1 loop,     (deliver      (L3 memory, retrieval    (under every   (L5 parallel
  L2 tools)     output)       L5 state)  (L2 tools)    tool call)     sub-agents)
                                     │
                        Distributed patterns
              (idempotency, statelessness, retries — the loop's backbone)
```

**Trust note:** general-knowledge sections are standard industry concepts. Repo-specific claims carry a "Trust status" line noting what was verified against source; code drifts, so re-grep symbols before relying on any file:line reference. External-harness examples (Cursor, Devin, LangGraph, etc.) are illustrative "widely-reported shape," not verified internals.