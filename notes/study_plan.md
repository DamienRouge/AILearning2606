---
name: study-plan-agent-rampup
description: "2-week study plan for ramping up on this repo's agent architecture, React/Flask/LLM/MCP/Azure — tracks progress day by day"
metadata: 
  node_type: memory
  type: project
  originSessionId: 937afedc-80c6-4996-ae65-1c95651e50bd
---

## Study Plan: Agent Application Ramp-Up (started 2026-06-08)

**Schedule**: ~2-3 hours/day, 2 weeks (10 study days)
**Goal**: Understand real-world agent architecture patterns AND become productive in this repo

---

### Week 1: Foundations + Frontend-to-Backend Chain

- [ ] **Day 1-2**: React/TypeScript essentials — components, useState, useReducer, useEffect, useCallback, useRef, arrow functions. Read QuestionInput.tsx and FollowupChips.tsx, trace props from parent.
- [ ] **Day 3**: Flask backend essentials — routes, blueprints, request object, Response. Read copilot/__init__.py → handler.py → stream.py in order.
- [ ] **Day 4**: Full frontend-to-backend chain — trace a single question from QuestionInput.tsx:70 → Copilot.tsx:355 → useAgentChat.ts:390 → __init__.py:35 → handler.py:72 → stream.py → back to useAgentChat.ts:435.
- [ ] **Day 5**: SSE, streaming, and reducer pattern — understand how streaming data flows into the UI. Add console.log in applyStreamChunk and observe in DevTools.

### Week 2: Agent/LLM + MCP + Repo Architecture

- [ ] **Day 6**: LLM agent fundamentals — LLM, tool use/function calling, agent loop, system prompt, context window. Conceptual, no code.
- [ ] **Day 7**: MCP (Model Context Protocol) — MCP server vs host, tool schemas. Read execute_kusto_query.py, search_relevant_docs.py, query_geneva_metrics.py.
- [ ] **Day 8**: Prompt composition system — routing.yaml → _routing.py → flows (lsi-w365.yaml, lsi-avd.yaml) → fragments. Understand inheritance and edits.
- [ ] **Day 9**: Azure infrastructure — Azure Functions, App Service, Managed Identity, Kusto, EV2, Key Vault. Read credential.py and kusto.py.
- [ ] **Day 10**: SRE Agent (autonomous agent) — spec.yaml, subagent-config.json, troubleshoot-guidance-w365.md. Compare portal agent vs SRE agent using same MCP tools.

### Ongoing: PR Reviews

- [x] PR #15634281 (AVD support) — reviewed 2026-06-05. Teaches: adding a new product (routing, compose flows, Kusto data source, tests).
- [x] PR #15790330 (Scheduled agent) — reviewed 2026-06-05. Teaches: SRE Agent subagents (spec.yaml, handoffs, scheduled tasks, deploy scripts).
- [x] PR #15792977 (Portal modernization) — reviewed 2026-06-08. Teaches: frontend/backend interaction (new endpoints, SSE streaming, React patterns).
- [ ] Review one more PR each week after the plan completes.

### Milestone Checkpoints

| Day | Checkpoint |
|-----|-----------|
| 2 | Can read any .tsx file and understand component structure |
| 3 | Can trace a request from fetch() to @api.route() to handler |
| 5 | Can explain the full path from keypress to streaming response |
| 7 | Can explain MCP and name 5 tools in this repo |
| 8 | Given a new product, can sketch what files to create |
| 10 | Can explain how SRE Agent and Portal use the same MCP server differently |

---

**Current status**: Day 1-2 in progress. Covered useReducer, components, props, parent-child, .tsx, and compared to DI. Also covered web fundamentals (fetch, POST, SSE, Promises) and Cosmos DB session flow. See [[learning-log]].
**Last updated**: 2026-06-11

**Why:** Damien is ramping up on this repo as a backend SWE — needs to understand the full agent stack (React frontend → Flask backend → MCP server → LLM) to contribute code and review PRs. See [[user-profile]].
**How to apply:** Check this plan at the start of each study session. Mark items as done. If stuck, ask Claude to walk through the specific day's files.
