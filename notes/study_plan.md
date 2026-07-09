---
name: study-plan-agent-rampup
description: "2-week agent-first study plan (backend SWE) — agent core first, frontend last; emphasizes systems thinking"
metadata: 
  node_type: memory
  type: project
  originSessionId: 937afedc-80c6-4996-ae65-1c95651e50bd
---

## Study Plan: Agent Application Ramp-Up (revised 2026-06-11)

**Schedule**: ~2-3 hours/day, 2 weeks (10 study days)
**Goal**: Understand real-world agent architecture AND train systems thinking
**Approach**: Backend-first. Agent code (Python) before UI (React). Frontend is just a chat shell — the brain is in the MCP server + prompts + LLM loop.

---

### Systems Thinking — The Meta-Skill

Threaded through every day. **Systems thinking** = the ability to look at code and ask:
1. **What's the input, what's the output?** (interfaces, contracts)
2. **What changes when this runs?** (state, side effects)
3. **What happens if this fails?** (failure modes, recovery)
4. **What other components does this trust?** (dependencies, trust boundaries)
5. **What stays the same vs. what varies?** (invariants vs. parameters)
6. **Where would I cut this in half?** (composition, boundaries)

Each day ends with a "Systems lens" exercise applying these questions to that day's code.

---

### Week 1: Agent Core (Python-first)

- [x] **Day 1**: LLM agent fundamentals (concepts, no code). LLM, tool use, agent loop, system prompt, context window. Systems lens: "Why does the LLM not execute code itself?" (~1.5h) — DONE 2026-07-14 via quiz-format refresher. Strong across all 6; pushed into context-mgmt + memory architecture (Day 6+). Checkpoint met: can explain the agent loop without code.
- [x] **Day 2**: MCP server structure — diagnostics-mcp.py + tools (get_incident_metadata, search_relevant_docs, query_geneva_metrics, +get_grafana_dashboard/query_azure_resources_metrics). Systems lens: "What's the contract between MCP and Foundry?" (~2h) — DONE 2026-07-09. 8/8 verification quiz; every gap was a missing label, not a broken concept. Read real code: server→registrar→tool→client→Azure chain. Key wins: signature-as-prompt thesis, RAG-as-pattern (front-wheel-drive framing stuck), relevance-score-because-context-is-finite (derived himself), Foundry↔MCP contract.
- [x] **Day 3**: Prompt composition system — routing.yaml, _routing.py, flows, fragments. Where the agent's personality is defined. Systems lens: "Why YAML for routing instead of Python?" (~2h) — DONE 2026-07-09. 7.5/8 verification quiz. Read real code (routing.yaml, _routing.py, _helper.py, flows, fragments, troubleshoot_incident.py). Reversed his earlier tool-vs-prompt confusion (now correct). Killed one misconception: compose() output IS the complete prompt (fragments = all sections), nothing else assembles it. Gap: systems-lens answer was shallow (readability) vs deep (policy/mechanism separation).
- [ ] **Day 4**: Flask backend handler — copilot/__init__.py → handler.py → stream.py. The relay between browser and Foundry. Systems lens: "What does the handler own vs delegate?" (~2h)
- [ ] **Day 5**: SRE Agent (autonomous) — spec.yaml + troubleshoot-guidance-w365.md. Same MCP server, different host. Systems lens: "Why does the same tool set produce different behaviors?" (~2h)

### Week 2: Infrastructure + Just-Enough Frontend

- [ ] **Day 6**: Azure infrastructure — Managed Identity, Kusto, Key Vault, EV2. utils/credential.py, utils/kusto.py. Systems lens: "Why MI over passwords?" (~2h)
- [ ] **Day 7**: Just-enough frontend — only 3 concepts: components, props, fetch. Goal: scan a .tsx file and find "where does this call the backend." Skip useReducer/useCallback/useRef. (~1.5h)
- [ ] **Day 8-10**: PR reviews using /pr-learning-review skill. One PR per day. Learn by seeing changes in context.

### PR Reviews (Already Completed)

- [x] PR #15634281 (AVD support) — reviewed 2026-06-05.
- [x] PR #15790330 (Scheduled agent) — reviewed 2026-06-05.
- [x] PR #15792977 (Portal modernization) — reviewed 2026-06-08.

### Milestone Checkpoints

| Day | Checkpoint |
|-----|-----------|
| 1 | Can explain the agent loop without looking at code |
| 2 | Can name 5 tools and what they do |
| 3 | Given a new product, can sketch what files to create |
| 4 | Can trace a request from handler.py to Foundry and back |
| 5 | Can explain SRE Agent vs Portal Agent using the same MCP server |
| 6 | Can explain Managed Identity vs password auth |
| 7 | Can scan any .tsx file and find the fetch call |
| 10 | Can review a PR independently and explain the system impact |

---

**Current status**: Days 1–3 complete (2026-07-09/07-14). Day 4 next — Flask backend handler (the host side).
**Last updated**: 2026-07-09

**Why:** Damien is a backend SWE — Python feels familiar, React/JSX created friction. Agent core (MCP, prompts, Foundry relay) is all Python and is where the actual intellectual content of this repo lives. Frontend is just a chat shell.
**How to apply:** Check at the start of each study session. After each day's reading, do the "Systems lens" exercise — answer the 6 systems questions about that day's code. Mark days as done.
