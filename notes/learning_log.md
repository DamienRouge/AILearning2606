---
name: learning-log
description: Tracks concepts Damien has learned session by session — use to avoid re-explaining and to build on prior knowledge
metadata: 
  node_type: memory
  type: user
  originSessionId: c61ec033-bfa6-4006-aa1a-140a30fa0a24
---

## Concepts Learned

### 2026-07-09 — Prompt layers, folder-vs-flow, ID timing, loop interleaving (Day 3 follow-up → Day 4 bridge)

Deep follow-up after the Day 3 quiz. Kept probing the prompt/loop boundary and reconstructed how prompt-invocation interleaves with the agent loop — the crux of agent architecture.

**THREE LAYERS of prompt (the "surely compose isn't everything?" instinct was right):**
- ① BASE agent prompt — the agent's PERMANENT identity/safety/tool-etiquette, scoped to the agent's whole existence, rarely changes. Owned by HOST (spec.yaml / agent config, Day 5).
- ② TASK prompt = `compose()` output — the PROCEDURE + output contract for ONE task-type (LSI vs CRI vs AVD), written ahead of time, changes per task-type. Owned by MCP. Contains: task-scoped role ("act as troubleshooter" = narrower hat than ①), ordered steps, which-tools-when, task rules, report format.
- ③ CONVERSATION — the live back-and-forth (user msg + tool calls + results), changes every turn, built by the HOST loop at runtime.
- `00-overview` restates identity in ② (even though ① also sets identity) because MCP is HOST-AGNOSTIC — can't assume the host framed it right, so self-contained. Base rule = "never fabricate" (personality); task rule = "fetch incident before metrics" (procedure).

**WHERE the 3 layers combine = the HOST, never MCP.** Host builds the API call: `system = ① + ②` (concatenated), `messages = ③` (the running list). MCP only produces ② and stops at `return`; has no idea ①/③ exist. Combining is NOT in _helper.py/troubleshoot_incident.py — those return ② and exit over HTTP.

**A "step" has 3 forms:** (1) a name-string in flow YAML (`lsi/01-fetch-incident`), (2) a `.md` file on disk (`_fragment_path`: name → `fragments/lsi/01-fetch-incident.md`), (3) a text section in the final prompt. `compose()` = read each step's file in YAML order, `"\n\n".join(bodies)`. The `### Step 1` headings INSIDE fragments are human-written Markdown, NOT the YAML `steps:` list (they rhyme but are different levels).

**Where the flow "leaves":** compose returns string → troubleshoot_incident prepends `# Troubleshoot Incident {id}` → registered MCP prompt → HTTP response → HOST.

**CORRECTION — folder is NOT a flow:**
- `fragments/lsi/` = a LIBRARY/drawer of bricks grouped by topic. A flow = a FILE (`flows/lsi-w365.yaml`) that LISTS which bricks in what order.
- Numbers (00-, 01-) = human sort-hints for editor ordering, NOT execution order. Execution order = the flow's explicit `steps:` list.
- Clincher: a flow pulls from MULTIPLE folders (lsi/ + shared/ + addons/w365/), so it can't BE the lsi folder. One folder of bricks feeds MANY flows. The naming rhymes (incident-type LSI / folder lsi/ / flows lsi-*) — 3 different things at 3 levels.

**Incident ID source — 2 cases:**
- Case A (webhook/scheduler): ID arrives STRUCTURED (IcM event field). Host passes it directly to troubleshoot_incident(12345) UP-FRONT. No extraction.
- Case B (human types free-form "look into incident 12345"): ID is EMBEDDED in prose → must be EXTRACTED. Modern agent design: THE LLM extracts it (reads the ID from the sentence, calls the prompt-builder with it) — a natural-language task, no regex.
- Extract loosely / validate strictly: regardless of source, MCP validates at the boundary (`_routing.py:67` `if not incident_id.isdigit(): raise`). MCP never trusts the extraction.

**KEY INSIGHT — loop timing (guessed "loop hasn't started yet" — actually OPPOSITE):**
- In Case B, the agent loop has ALREADY STARTED by the time the LLM extracts the ID. Starting the loop = calling the LLM = the LLM can now read/extract. The LLM extracting the ID IS iteration 1 of the loop. There is NO separate pre-loop extraction phase.
- Sequence Case B: loop starts with ① only → iter 1: LLM extracts ID + invokes troubleshoot_incident(12345) → compose produces ② MID-LOOP → ② appended → iter 2: LLM sees workflow, calls get_incident_metadata. The "re-send everything" is just the loop's normal append-result-and-recall rhythm.
- Case A composes ② BEFORE the loop; Case B composes ② DURING the loop. Both converge to same state; only the TIMING of when ② exists differs.
- prompt-vs-tool framing of troubleshoot_incident is a HOST-DESIGN CHOICE: as a "prompt" it seeds up-front (Case A); as a "tool-like call" the LLM invokes it mid-loop (Case B). MCP doesn't care.

### 2026-07-09 — Prompt composition system (Day 3, real code + 7.5/8 quiz)

Read the actual prompt-composition tree. 7.5/8 on verification quiz. Notably REVERSED an earlier tool-vs-prompt confusion — now has the arrow right.

**The 3 building blocks (Lego analogy):**
- FRAGMENT = reusable piece of a prompt, one section, a .md file (`compose/fragments/lsi/00-overview.md`). = Lego brick. DRY for prose.
- FLOW = ordered list of fragments in a YAML recipe (`compose/flows/lsi-w365.yaml`). = instruction booklet.
- ROUTING = `routing.yaml` table mapping incident attributes → flow name. = the picker. First match wins, `on_no_match: raise`.

**Tool vs Prompt (had this BACKWARDS before, now correct):**
- Tool = fn the LLM CALLS during the loop. Prompt = pre-written instruction template that STARTS/shapes the session. Prompt comes first and CAUSES behavior; tool calls happen during as a consequence. `troubleshoot_incident` is a PROMPT-builder, NOT a tool.
- MCP servers expose BOTH tools AND prompts (`register_tools` + `register_prompts`).

**YAML↔Python bridge:** YAML is just text; `yaml.safe_load(path.read_text())` PARSES it into Python dict/list. After that line there's no more YAML, just normal dict access. General concept = serialization/deserialization; safe_load = won't execute embedded code.

**Systems-lens "why YAML not Python if/elif" (the DEEP answer):**
- Principle = separate POLICY (rules that change often) from MECHANISM (the engine that evaluates them — rarely changes).
- Concrete benefit: change routing WITHOUT a code change → without redeploy → without eng review; non-engineers (PM/on-call) can safely edit the table; stable engine stays locked. NOT just "easier to read" (symptom, not cause).

**Division of labor (route vs compose):**
- `_routing.py`: incident_id → flow name. Knows IcM/Kusto.
- `_helper.py`: flow name → prompt string. Knows NOTHING about IcM/routing. Pure composer.

**MISCONCEPTION KILLED:** kept guessing "maybe compose() only makes PART of the system prompt." WRONG. `compose()` output (+ the `# Troubleshoot Incident {id}` header) IS the COMPLETE system prompt. The fragments ARE all the sections. (Host may wrap with conversation scaffolding, but that's host-side.)

**Flow inheritance (`extends` + `edits`):**
- = class inheritance for prompts. `extends: lsi-w365` inherits parent's step list; `edits` = child's modifications on top.
- `_apply_edit` (_helper.py:94-116) is the interpreter. Fixed grammar of 4 verbs: {after,insert}|{before,insert}|{replace,with}|{remove}. YAML supplies data, Python supplies meaning.
- `_resolve_steps` is RECURSIVE (multi-level extends) + cycle guard raises on A→B→A.
- Problem solved: no copy-paste of 15-step flows per variant; parent improvements auto-inherited. DRY for whole flows.

**Full lifecycle + 3 triggers:** (1) HUMAN interactive, (2) SCHEDULER/cron (`diagmcp-log-auto-fix.md` — wakes on timer), (3) EVENT/webhook (incident fires → autonomous SRE agent = Day 5). All host-side; troubleshoot_incident is TRIGGER-AGNOSTIC.

### 2026-07-09 — Retrieval deep-dive (Day 2 extension: RAG internals, vector math, wrapper pattern)

**Wrapper/adapter pattern (utils/search.py):** TWO classes named `SearchClient`: Azure's (imported `as AzureSearchClient` — MS SDK) and the repo's own (wraps Azure's). The `as` alias is the giveaway. Why wrap: simpler interface (4 params vs ~10), encapsulate vector-query setup, uniform logging/error handling, swappability. Ownership ladder: tool → wrapper (ours) → SDK (MS) → Azure AI Search service (MS, rented).

**gRPC:** alternative to REST; binary Protobuf vs JSON, faster, not human-readable. Example of a swappable transport hidden inside a client.

**Client caching (resource.py):** clients cached in a dict inside the ResourceManager singleton, keyed by config tuple. Lazy singleton cache: "in dict? return it : build once, store, return." Building a client does an auth handshake (expensive). Double-check locking = thread safety.

**Vector distance (stage 1):** comparison is QUERY vector vs DOC-CHUNK vectors — the query is the "ground of relevance." Cosine similarity = ANGLE between vectors (ignores length), range -1..1. `k_nearest_neighbors=50` = top 50 nearest chunk vectors.

**CHUNKS not documents:** docs SPLIT INTO CHUNKS at indexing; each chunk has its own vector. System ranks/returns CHUNKS, not whole docs. Two chunks from the same doc score differently; no aggregation. This repo = Design A (chunk-level): `record["Content"] = result.get("content")` returns chunk passages + `title`. Granularity ladder: document(title) → chunk(content) → caption(best sentences).

**Stage 1 vs Stage 2 — the real mechanistic difference:**
- Stage 1 (retriever): query & chunk embedded SEPARATELY, compared by vector distance. Fast, blunt. Misses negation ("restart" vs "do NOT restart" look identical as vectors).
- Stage 2 (reranker): query & chunk fed TOGETHER into one model, uses ATTENTION across both → one relevance number. Slow, sharp. "Cross-encoder."
- Blurb-match millions→50, then careful-read those 50.

**Return metadata mapped to stages:** `@search.score` = STAGE 1 blunt; `@search.reranker_score` = STAGE 2 sharp (0-4); `@search.captions` = STAGE 2 best sentences (extractive). Scores surfaced for debugging/telemetry.

**Context-loss = central weakness of chunk-RAG (spotted unprompted):** LLM does NOT reliably backfill missing context — if the detail is in an unreturned chunk, it's gone, and the LLM may hallucinate. Only legit backfill = LLM re-calls search with a refined query. Real fixes are in RETRIEVAL DESIGN: chunk overlap, parent-document retrieval, return multiple chunks (≤20 here), semantic-boundary chunking.

**Precision fix:** "the agent loop invokes the tool; Foundry is the RUNTIME that runs the loop." Agent side = LLM + loop + config; MCP side = tools; meet over HTTP. Test for which side a file is on: under `src/mcp-servers/`? → MCP side.

### 2026-07-09 — MCP server structure (Day 2, real code + 8/8 verification quiz)

Read the actual diagnostics MCP code. 8/8 on final quiz — every gap was a missing label, not a broken concept.

**MCP architecture (3 layers by job):**
- SERVER = `diagnostics-mcp.py` (~55 lines): `FastMCP(..., stateless_http=True)` + `register_tools` + `register_prompts`. Rest is deployment (uvicorn + auth middleware, prod-only). FastMCP implements the tool_use/tool_result protocol so you register plain Python fns.
- REGISTRAR = `core/tools/__init__.py`: auto-discovery via `pkgutil.iter_modules` — drop a .py file with a public fn = registered, NO manual list. Guards: `__module__` check, `_should_register()` gate, uniform logging wrapper (decorator pattern).
- TOOLS = individual fns. Signature + docstring + Pydantic `Field(description=)` ARE the schema/prompt the LLM sees.

**Key concepts locked:**
- Signature-as-prompt thesis: docstring is the LLM's user manual; vague/wrong → wrong tool call.
- Pydantic = "type hints that actually do something" — runtime validation + auto-generates JSON schema. `ge=1, le=20` = ≥1 and ≤20; enforced before code runs; context-window guardrail.
- Separation of concerns: tool assembles → `resource.py` hands a cached client → the CLIENT CLASS does the real HTTP (3 hops deep). Swap transport by editing only the client.
- ResourceManager singleton: one global instance, caches expensive clients keyed by config, double-check locking.
- RAG = a PATTERN, not a technology ("front-wheel drive, not the engine"). R=`search_relevant_docs.py`→Azure AI Search; A=`tool_result` injection (Foundry); G=LLM reply. Only R is code you write. Why RAG exists: raw LLM answers from fuzzy training memory + hallucinates; real source-of-truth → read not recall → accurate + citable.
- Search index = pre-built lookup structure (back-of-book analogy); docs chunked+embedded OFFLINE once, query embedded + nearest-neighbor lookup ONLINE.
- Azure AI Search = standalone managed service = the "R" engine, rented not built. `SEMANTIC_VECTOR_HYBRID` = vector + reranker + keyword at once.
- Retriever vs reranker 2-stage funnel; scores `@search.score` + `@search.reranker_score`. Same cheap-wide-then-expensive-sharp funnel as agent-loop cap.
- Relevance score exists BECAUSE context is finite — must rank + take top-K (≤20).
- Statelessness recurs at every layer (LLM, MCP server, HTTP): push state to ONE place, keep rest stateless → scalable + crash-resilient + simple.
- Foundry ↔ MCP contract: Foundry owns LLM + agent loop (decides WHICH tool + WHEN); MCP owns tools (DOES the work); over HTTP. You deploy the MCP server separately + register its URL in a Foundry agent config.

### 2026-07-14 — LLM agent fundamentals (Day 1, quiz-format refresher)

Covered all 6 Day-1 fundamentals via quiz-then-fill-gaps. Pushed past the syllabus into context-management / memory-architecture.

**LLM core:**
- LLM = next-token predictor: text in → probability distribution → sample one token → append → repeat. Pure function, stateless, can't *do* anything.
- Tokens (not words) — word-pieces from a fixed ~100k vocab; billing + context measured in tokens.
- Embeddings — each token → vector of thousands of numbers; meaning = geometry (king-man+woman≈queen).
- Attention (Transformer) — every token attends to every other token; source of both power and the n² cost limit.
- Temperature — sampling randomness (0 = always top token).

**Agents:**
- Why agents exist — bridge raw LLM's 4 gaps: frozen knowledge, no live data, can't take actions, hallucination. Agent = LLM decides, agent code does.
- Tool use = 4-step structured conversation: describe tools as JSON schemas → model emits `tool_use` (decides, then STOPS) → agent/MCP runs code → result appended as `tool_result` + re-sent.
- Agent loop — agent runs the `while` loop; re-sends full conversation each iteration. Stop condition = output type: `tool_use` → loop; `end_turn` → stop. Plus max-iterations cap.

**System prompt + roles:** system prompt = fixed authoritative rules (cacheable, reusable); user message = per-turn variable input. Roles are tagged (`role:` field), flattened into special delimiter tokens (`<|system|>`/`<|user|>`), and the model learned during training to weight them differently — real but learned wall (why prompt injection is possible).

**Context window + statelessness:**
- 1M context = HARD wall (API rejects on overflow). "Memory" = agent re-sends full message list every call; model re-reads from scratch.
- Overflow strategies are agent code's job: truncation (lossy), summarization (lossy), RAG/retrieval.
- Why context can't grow infinitely: attention is O(n²) — doubling context quadruples compute. Sparse/linear attention pushes it but trades quality.
- Independently re-derived RAG ("keep memory server-side, curate, pull relevant slice"). External store grows infinitely (cheap DB); context sent stays small.
- Brainstorm "summarize a life into tokens" → tiered self-consolidating memory (episodic → semantic → identity), background "sleep" consolidation, retrieval by relevance+recency+importance, forgetting as the core algorithm.

### 2026-06-15 — Render, state changes, event chain, stateful vs stateless

**React rendering:**
- "Render" = take data + component/template → produce visual output. Not just frontend: Flask `render_template`, ASP.NET Razor do server-side rendering; word originates from 3D graphics. In React day-to-day: 99% means "React runs component function → produces new DOM."
- Re-render is triggered by reference change, not value change. React compares old vs new state with `Object.is` (≈ `===`). **Mutating state in place doesn't trigger re-render** — that's why reducers always return `{...state, ...}` instead of `state.x = y`.
- Reconciliation = React diffs new JSX vs old, patches only the changes to real DOM. This is why React stays fast despite frequent re-renders.

**Event → setter chain (Copilot example, end-to-end):**
1. User presses Enter → browser fires native keydown
2. Fluent UI `<ChatInput>` catches it, calls `onSubmit` prop
3. `QuestionInput.onSubmit` runs `onSend()` (prop)
4. Parent's `sendQuestion()` runs `dispatch({type: "SET_INPUT", payload: ""})` + kicks off fetch
5. fetch streams chunks → each chunk dispatches more actions (`APPEND_MESSAGE`, `UPDATE_LAST_ASSISTANT`)
6. Each dispatch → reducer returns new state object → React detects reference change → re-render
- Synthetic events: React attaches ONE listener at document root and delegates, instead of attaching listeners per element. `<button onClick={...}>` doesn't create a real DOM listener — it registers with React's delegation system.
- Controlled component pattern: DOM input is just a mirror of React state. React is source of truth; on each keystroke `onChange` → `setQuestion` → state update → React re-renders input with new value.

**Stateful vs stateless (the big one — see [stateful-stateless-patterns.md](stateful-stateless-patterns.md) for full deep-dive):**
- HTTP itself is stateless — every request independent; server has zero memory of prior requests unless client tells it (cookie/header/token).
- Stateless endpoint: response computed purely from request body/params; same input → same output. Easy to scale, cache, retry.
- Stateful endpoint: response depends on server-stored context referenced by an ID. Hard to scale horizontally, must use shared store.
- **3 places state can live**: server memory (fragile, breaks behind load balancer), shared DB like Cosmos/Redis (robust), or in the request itself (server stays pure).
- Same file in Copilot.tsx has BOTH patterns: `agentChat.sendMessage(question, agentConversationId, ...)` sends just an ID (server-stateful), `askChat.sendMessage(question, messages, ...)` sends full history (client-stateful). Why split: agents have long convos with tool state (worth server storage); Ask is shorter Q&A (cheaper to stay stateless).
- **conversationId is a "coat check ticket"** — you don't carry the coat (conversation history), you carry the number. Server holds the coat.
- JWT vs cookie auth: JWT is stateless (token itself contains user info, cryptographically signed); cookie is usually stateful (just an ID; server looks up session in DB).
- **Saving responses**: yes, always save the AI's response too (not just user message) — needed for next turn's context, UI display on refresh, audit/compliance, analytics. Recent portal commit `defer Cosmos session write off the agent streaming critical path` = save response but AFTER streaming finishes so user sees tokens immediately.
- **Lost conversationId problem**: if client only holds ID in component state, refresh/close orphans the server-side record. Solutions: URL param (`?session=...` — your portal does this), localStorage, server-side index keyed by userId (the sidebar conversation list IS the recovery mechanism). Cleanup jobs delete truly orphaned records via TTL.
- **Claude Code itself**: Anthropic API is stateless — every turn re-sends full conversation history. The harness (local CLI) is the "stateful client" that accumulates history, system prompt, tool defs, memory files, recently-read file contents. Repo is NOT auto-sent; only what gets Read/Grep'd enters context. Prompt caching makes re-sent prefixes cheap; auto-compact summarizes old turns when context fills.

**Session vs Conversation (subtle but critical):**
- `currentSessionId` = user-facing container ("My chat about Cosmos refactor"). Owns title, pinned status, sidebar position, creation time. Persistent across refresh / device / agent restart.
- `agentConversationId` = LLM runtime context handle (the actual back-and-forth memory, tool call history, intermediate reasoning).
- Why split: rename session (metadata only), switch agents mid-session (`SET_TAB` clears `agentConversationId` but keeps `currentSessionId`), reload session from sidebar (fetch messages by sessionId, server returns a `conversationId` to continue with), pin/unpin (metadata), list all sessions (no need to load LLM state).
- Killer insight: **a single session can have multiple agent conversations over time**. Deploy a new agent version → user reopens same session → sees yesterday's messages, but agent's live memory is fresh. Same way a Slack channel keeps its identity even as the server process handling it changes.
- Persistence needs: across refresh (URL param), across sessions (Cosmos), across devices (Cosmos keyed by userId), across agent restarts (Cosmos), for audit (indefinite retention).

### 2026-06-11 — React fundamentals + web concepts

**React / TypeScript:**
- `useReducer` pattern — single (state, action) → newState function vs. many useState calls; why it's called "reducer" (from Array.reduce — you "reduce" a stream of actions into a single state value)
- Parent-child component relationship — JSX nesting, not OOP inheritance; composition ("has a") not inheritance ("is a"). Analogy: nesting boxes — the outer box decides what goes inside and what data to hand down.
- Props — data and callbacks passed parent → child; one-way data flow. Child can't reach up and grab parent state. If child needs to affect parent, parent passes a callback (e.g. `onSend={sendQuestion}` — QuestionInput calls it, but the function lives in Copilot).
- `.tsx` vs `.ts` — tsx allows JSX (HTML-like syntax in TypeScript). Without JSX you'd write `React.createElement("div", ...)` instead of `<div>...</div>`.
- React props ≈ dependency injection — QuestionInput doesn't know WHERE question text lives (useReducer) or WHAT happens on Enter (fetch + SSE + Cosmos). It just calls `onSend()` and trusts the parent. Same principle as backend DI.

**Web fundamentals:**
- `fetch` — browser's built-in HTTP request function. Analogy: fetch = the phone (the tool), GET/POST = the reason for calling (asking a question vs. sending a package). Defaults to GET; you set `method: "POST"` when sending data in the body.
- GET vs POST — GET for retrieving (params in URL, visible, limited size), POST for sending data (params in body, not in URL, no size limit)
- Promises — JS representation of a future value; three states: pending (in flight), fulfilled (got it), rejected (failed). Two ways to use: `.then()/.catch()` callback style, or `async/await` which reads like synchronous code but doesn't block the browser.
- SSE (Server-Sent Events) — not a separate protocol, just a standard built on HTTP. Server sends `data:` lines separated by blank lines. One-way (server → client). Compared to: regular HTTP (one request, one full response), SSE (one request, server streams chunks), WebSockets (separate `ws://` protocol, two-way, more complex). SSE is natural for AI chat — client sends one question, server pushes tokens as the model generates.

**General programming:**
- Dependency Injection — instead of a class creating what it needs, you pass it in from outside. Example: `OrderService` hardcoding `new SqlOrderRepository()` (can't swap, can't test) vs. receiving `IOrderRepository` via constructor (caller decides: real DB in prod, fake in tests). React parallel: QuestionInput receives `onSend` via props instead of importing the send logic itself. Core principle: "don't create your own dependencies — receive them."

**Codebase-specific:**
- CopilotState reducer — what each field tracks, why currentSessionId exists
- Cosmos DB session storage — metadata only (id, title, userId, conversationId); messages live in Foundry
- Session creation flow — handler.py builds doc, greenlet persists async, title refined via LLM after stream ends
