---
name: learning-log
description: Tracks concepts Damien has learned session by session — use to avoid re-explaining and to build on prior knowledge
metadata: 
  node_type: memory
  type: user
  originSessionId: c61ec033-bfa6-4006-aa1a-140a30fa0a24
---

## Concepts Learned

### 2026-07-17 — Backend receive path: poll→stream mechanics, baseline_ids dual-use, emit/SSE-send stack

Deep dive into the SRE receive side (sre_stream.py / sre_client.py / handler.py). Damien traced send-vs-receive as two HTTP calls, self-corrected the "no readiness race" reasoning, caught the `_send_follow_up` baseline puzzle, and pushed into how `_emit` physically sends bytes.

**SEND + RECEIVE = TWO SEPARATE HTTP CALLS (polling, not push).** `post_message`/`create_thread` (send) and `get_messages` (receive) are distinct REST requests. There is NO push-based stream to "hook into." NO readiness race: the agent WRITES its answer into the thread (durable server storage), not down a wire — so timing never causes a miss. If agent is slow → poll returns nothing-new → poll again; if agent already finished → poll returns it. The thread IS the buffer. Ordering in handler.py guarantees correctness: (1) get_messages → snapshot baseline_ids, (2) post question, (3) THEN stream() polls — even a 5ms answer is found because it's sitting in the thread.

**baseline_ids = "what existed BEFORE this turn," used for TWO jobs (Damien's catch — it's the SAME set reused):** Job A (streaming filter, `_process_messages` line 692): `if mid in self.baseline_ids: continue` → anything NOT in baseline = this turn's new content. Job B (send-time idempotency, `_send_follow_up` lines 76-86): if `post_message` throws (HTTP response LOST but write may have landed), instead of blind-retrying (→ duplicate turn), VERIFY via `_user_message_landed(baseline_ids, text)` = "did a NEW user msg matching my text appear?" baseline_ids is the "before" reference that detects whether the post landed. Both jobs = same principle "new since baseline" applied first to the QUESTION (dedup send), then to the ANSWER (filter stream).

**get_messages paging (Damien's restatement, corrected):** (a) skip is a COUNT not an id — `tail_skip = max(0, len(baseline_ids) - _TAIL_SKIP_MARGIN)` (sre_stream.py:864), a positional OFFSET ("skip first N msgs"), sent as `skip=N` (sre_client.py:237), NOT an id value. (b) `page_size=200` = PAGING per HTTP response (data plane caps at 200), NOT a per-poll cap on new msgs — `get_messages` LOOPS `while len < max_messages(2000)`, extending 200-msg pages until a short page ends it (sre_client.py:236-247). (c) NO client re-sort here — asks server `orderby:timestamp`, extends pages in arrival order (already ascending); Day-5 `_chronological_messages` re-sort is elsewhere/defensive. Core insight right: `start_skip` skips immutable baseline → fetch only tail → per-poll cost CONSTANT regardless of thread length.

**`_process_messages` does NOT receive only-new messages (Damien's wrong assumption, fixed — this was the "not adding up" crux).** Input = the TAIL SNAPSHOT = full current state of recent msgs, INCLUDING already-seen-and-emitted ones. Each poll re-fetches the SAME growing answer msg (stable id), just longer. That's WHY filters exist: input is repeated snapshots, not deltas. Two-filter dedupe: baseline_ids (drops pre-turn msgs) + emitted-length/payload trackers (drops already-emitted PARTS of this-turn msgs).

**THE EMITTED-LENGTH DIFF (backend mirror of the Day-5 client diff-engine), sre_stream.py:744-751:** `already = self._emitted_text_len.get(mid, 0)` → chars of THIS msg already forwarded to browser in prior polls. `if len(text) > already: delta = text[already:]; self._emitted_text_len[mid] = len(text); yield emit(content=delta)`. `_emitted_text_len` = handler-INSTANCE dict `{msg_id → chars_sent}`, persists across all polls of the turn, updated (line 748) right after emit → next poll reads the new high-water mark. `dict.get(mid, 0)` = value if key exists else 0 (Damien guessed right; avoids KeyError on first sight). NO duplicate msgs IN a list — the SAME id RECURS ACROSS polls, longer each time; the per-id bookmark converts "same msg now longer" → "emit only new suffix." Walkthrough: poll1 text="The error"(0→9 emit "The error"); poll2 "The error rate is"(9→17 emit " rate is"); poll4 unchanged(22>22 false → emit NOTHING = dedupe). Tools use the parallel `_emitted_tool_payloads {mid → fingerprint tuple}` (status/name/output/args...): re-emit only when payload CHANGES, identical polls skipped (tools appear whole, not char-by-char).

**HOW `_emit` "SENDS" — it DOESN'T; sending is 4 layers, _emit owns only layer 1 (Damien's stretch Q):** (1) `_emit`→`format_sse` (models.py:113) = `f"data: {json.dumps(msg.to_payload())}\n\n"` — turns StreamingMessage OBJECT into an SSE-format STRING, sends nothing. The `\n\n` = SAME delimiter the frontend `buffer.indexOf("\n\n")` splits on (both ends agree — full circle). (2) `yield self._emit(...)` — stream() is a GENERATOR (`-> Generator[str]`); hands out ONE frame string then PAUSES with state frozen (Day-4 yield). (3) `Response(generator, mimetype="text/event-stream")` (handler.py:142) — FLASK/WSGI iterates the generator and writes+flushes each chunk to the socket = THE ACTUAL SEND (your code never touches the socket). Headers: `text/event-stream` (browser parses SSE incrementally), `keep-alive` (hold TCP open), `Cache-Control:no-cache`, and CRITICAL `X-Accel-Buffering:no` (tells nginx NOT to batch — else answer appears all-at-once not typing). (4) OS TCP socket → bytes over the held-open connection browser opened via fetch → client `reader.read()` receives them. So "send" = framework-driven: you write a generator yielding SSE strings, register as Flask streaming Response, Flask turns each yield into a flushed socket write. Server-side MIRROR of the client SSE read-loop: `format_sse` builds `data:...\n\n` here; `indexOf("\n\n")` parses it there — same contract, opposite ends.

### 2026-07-16 — Just-enough frontend: finding the backend call (Day 7, ~4.5/6 quiz)

Short day. Goal = scan a .tsx and find "where does it call the backend." Damien traced the FULL chain unprompted (impressive). One core React concept fuzzy (passing functions via props) — fixed; it's the key to 3 questions.

**THE FULL FRONTEND→BACKEND CHAIN (4 files):** QuestionInput.tsx `onSubmit → onSend()` → Copilot.tsx `sendQuestion()` → useAgentChat.ts `sendMessage()` (builds ctx StreamContext) → `consumeAgentStream(ctx, convId, signal => agentChatApi(request, signal))` → Copilot.api.ts `agentChatApi` → `fetch("/copilot/agent/chat", {method:POST, body:JSON.stringify(options)})` → backend Day-4/5 route `@api.route("/agent/chat", POST)` → handle_conversation_sre. SSE frames stream back; consumeAgentStream = frontend counterpart to backend stream() — reads `data:{...}` frames, dispatches to React state → UI updates token-by-token.

**THE DAY-7 SKILL — find the backend call:** (1) look for `fetch(` (or EventSource/axios/`*Api` fn); (2) read the URL string → tells you which endpoint; (3) check method (POST=send, GET=fetch); (4) if no fetch, it's a component calling a prop callback — follow the callback UP or find an imported `*Api` fn. THIS repo centralizes ALL fetch in `Copilot.api.ts` (agentChat/resume/feedback/session CRUD) — components never fetch directly. So "where's the backend call?" = "look in *.api.ts".

**KEY CONCEPT FIXED — passing a function as a value via props (Damien thought `onSend={sendQuestion}` involved calling/return-value):** A function is a VALUE you can pass like a number. `onSend={sendQuestion}` (NO parens) = PASSES the function itself into the child's onSend slot — does NOT call it, no return value involved. `onSend()` inside child (QuestionInput.tsx:70) = NOW run it, on submit. Parent LENDS function to child; child calls it later. Analogy: giving your phone number (pass) vs dialing now (call). Same function, TWO names: `sendQuestion` in parent (defined there), `onSend` in child (prop slot it arrived in). Child is generic — knows only "some onSend action", doesn't care parent calls it sendQuestion → reusability.

**3 WAYS FUNCTIONS HOOK ACROSS FILES (Damien mixed these up — corrected):**
- PROPS (down, parent→child): `onSend={sendQuestion}` in Copilot.tsx:721 passes fn down to QuestionInput. ONLY between components (parent/child JSX).
- HOOK-RETURN: `const agentChat = useAgentChat(...)` → hook RETURNS `{sendMessage, ...}` → parent calls `agentChat.sendMessage`. Hook = fn (name starts `use`) packaging reusable stateful logic.
- IMPORT (direct): `import { agentChatApi } from "../Copilot.api"` → call it directly. NOT a prop (agentChatApi isn't a component).
- CRITICAL: "import" ≠ "prop". Import = make a name available in this file (both hooks and api fns get imported). Prop = pass value parent→child via JSX. Unrelated concepts.

**Component types:** QuestionInput = "dumb"/presentational component — renders the input box, shouts "send!" upward via onSend, holds NO backend logic. Logic (build request + session/user info + call API) lives higher (Copilot → useAgentChat). = separation of concerns again.

**Session APIs connect to Cosmos (Day 4/5 callback):** listSessionsApi/getSessionApi/deleteSessionApi/updateSessionApi = frontend side of the Cosmos session METADATA (the "coat-check catalog") — sidebar list/rename/delete/pin → these fetch calls → backend session store → Cosmos.

### 2026-07-16 — Rigor pass: typing effect = repeated full re-render of state.messages; allMessages purpose; chunk kinds

Same-day precision session. Damien demanded rigid, code-anchored answers (no loose prose, no undefined jargon, cross-compare every definition, corrected my misuse of "assistant" as narration and my wrong anchor of line 703). These are the VERIFIED facts.

**THE TYPING EFFECT = React re-rendering `state.messages` many times per second (NO separate animation).** Per TEXT chunk: `parsed.content` (the fragment) → `appendTextDelta` grows `agentState.assistantSegments` → `textFromSegments` rebuilds `agentState.assistantContent` (useAgentChat.ts:366-371) → `updateAssistantMessage` copies onto `agentState.assistantMessage` (staging object, lines 264-273) → `dispatch(UPDATE_LAST_ASSISTANT, {...assistantMessage})` — spread makes a NEW object reference (line 275) → reducer replaces the last `role:assistant` message in `state.messages` with a NEW array (Copilot.reducer.ts:166-172) → new reference makes React re-run `messages.map` (Copilot.tsx:752) → the changed reply renders via `<ReactMarkdown>` (CopilotResponse.tsx:258). Repeating this dozens of times/sec IS the typing effect. The words are NOT rendered chunk-by-chunk directly, and the screen does NOT read `assistantMessage` directly — it reads `state.messages` (whose entry is a COPY of assistantMessage). The `LatencyLoader` pulse is a separate CSS thing; the text appearing is not animated.

**CHUNK KINDS DIFFER — the typing story is TEXT-chunks only (correction to an over-general claim).** `applyStreamChunk` branches on `parsed.event`: (a) TEXT chunk `event==="message" && parsed.content` (line 364) → appends a text fragment → segments GROW → typing. (b) TOOL-START `run_status`+`status==="in_progress"` (line 283) → `upsertToolSegment` adds/updates a TOOL segment (text unchanged) → re-render shows a tool CARD, not typing. (c) TOOL-COMPLETE `run_status`+non-pending tools (line 299) → replaces a tool segment; a `reasoning` one may via `reconcileReasoningTool` REMOVE/replace text segments → segments can SHRINK. (d) COMPLETED `status==="completed"` (line 348) → no content change, stamps id, dispatches `STOP_REQUEST`. So "every chunk grows segments / shows one fragment more" is FALSE in general — true only for text chunks; tool chunks re-render too but produce cards, and reasoning-complete can shrink.

**PURPOSE OF `ctx.allMessages` = input to `snapshotMessages` for the DETACH/background path, NOT the normal render.** On the ATTACHED (normal) path every chunk dispatches to the reducer (line 275) and the screen renders from `state.messages`; `allMessages` is NOT read for rendering. On DETACH (`ctx.detached===true`, switch chats mid-stream) `updateAssistantMessage` takes the ELSE branch → `saveToBackground` instead of dispatch (lines 276-277); reducer stops tracking this stream, so `allMessages` is what holds the full conversation for `snapshotMessages` (`base=[...ctx.allMessages]` + live agentState message stitched on top, lines 114-129). So `allMessages` is the detach-path counterpart of `state.messages`; on the normal path it's essentially dormant. (Corrected my earlier over-claim that allMessages is central to every turn — it is not.)

**CORRECTION — line 703 `ctx.allMessages = currentMessages` is inside `detachStream` (line 695), NOT the per-turn send path.** It runs ONLY on detach (switching away from a streaming chat), not on every turn. `allMessages` is seeded at ctx creation (`[userMessage]` line 620, or `base` on load line 671) and OVERWRITTEN with the full current list only on detach. I had wrongly cited 703 as the per-turn copy mechanism.

**`allMessages` HOLDS WHOLE TURNS, both roles (not one turn, not fragments).** Each element is one `IChatMessage` with `role` = `ChatRole.USER` OR `ChatRole.ASSISTANT`, interleaved = the conversation transcript. NOT "all previous LLM responses" (that omits the user messages — half the list). NOT the fragments of one reply (those are `agentState.assistantSegments`, one scale down INSIDE a single `role:assistant` message). Proof: code must SEARCH for `role===ChatRole.ASSISTANT` (useAgentChat.ts:120-122) — pointless if the list were LLM-only. Two scales to keep straight: `allMessages` grows by EXCHANGES (turns); `assistantSegments` grows by CONTENT WITHIN ONE reply (fragments merged to ~few segments).

**HISTORY CARRIER ACROSS TURNS = reducer `state.messages` (durable, whole session), NOT `allMessages` (per-turn, discarded with its ctx).** `state.messages` accumulates via `APPEND_MESSAGE` (`[...state.messages, payload]`, reducer:147 — fires per user msg + per reply msg) and `UPDATE_LAST_ASSISTANT` (per chunk). It survives every turn until page reload; then durable history = SERVER (SRE thread + Cosmos pointer), refilled via `LOAD_SESSION`. A prior turn's reply reaches a later turn THROUGH `state.messages`, not through `allMessages` passing itself.

**`allMessages` snapshot timing: captured at turn START, so it EXCLUDES the reply being generated.** The current reply lives in `agentState` during its turn; `snapshotMessages` = frozen `allMessages` (the settled past) + live `agentState` message stitched on. The reply enters "the history" only as the NEXT turn's frozen past. So "entire conversation so far" at turn start = everything up to & including the new USER message, minus the answer about to stream.

### 2026-07-16 — Agent-mode frontend→SRE full trace (QuestionInput → useAgentChat → SRE thread → render)

Long guided trace of the agent-mode chat path, front to back. Damien drove with precise "where exactly / who owns this" questions and self-corrected several models (ctx lifetime, reducer vs allMessages, stop-vs-resume). Builds on Day 4 (host) + Day 5 (SRE) — this session = the FRONTEND side those days didn't cover.

**COMPONENT CHAIN (send path):** QuestionInput.tsx (dumb textbox, raw text only, fires `onSend()` with NO args) → Copilot.tsx `sendQuestion` (reads `currentInput` from ITS OWN state, resolves agent name, no prompt built) → useAgentChat `sendMessage` (wraps into `IAgentRequest`) → `agentChatApi` fetch POST `/copilot/agent/chat`. Parent owns state+handlers, child just reports events up (data-down/events-up, revisits Day-1 6/11 props/DI).

**PROMPT COMPOSITION IS NOT IN THE REPO — it's in the remote SRE Agent service (Azure).** Frontend sends only `{message.content (raw text), conversationId, product, agent NAME}`. NO system prompt / history / tool list in the request. The service looks up the agent by name → loads its stored `instructions` → composes context → runs LLM. Two-phase: PHASE 1 registration (`setup_sre_agent.py apply`, run once by human/CI, PUTs YAML to `/api/v2/extendedAgent/agents/{name}`, `_SPEC_TO_WIRE` renames `system_prompt`→`instructions`); PHASE 2 per-request (backend picks agent NAME at handler.py:260 `get_agent(product)` dict-lookup, sends name in `startMessage.agent`, service injects instructions inline+deterministic — verified per sre_client.py docstring). "Where's the injection?" = inside the managed service, across a REST boundary, NOT in this codebase. (Corrected Damien's assumption that system_prompt is native/foundational: it's a ROLE not a level — there's likely a hidden platform system prompt PLUS the team's scenario-specific one stacked on top.)

**SYSTEM PROMPT = a FIELD, not the file.** Each `scripts/agents/*.yaml` = a full agent DEFINITION (name + system_prompt + tools + mcp_tools + agent_type + skills). Only the `system_prompt:` field is the system prompt. ~9 YAMLs = ~9 peer personas (general W365/CDP/AVD + `-TroubleshootIncident` specialists). `setup_sre_agent.py` = the UPLOADER, not a prompt.

**"IS THERE A FLOW DEFINITION?" — depends on the agent.** General agent (EngCopilot-W365.yaml): `agent_type: Autonomous`, NO scripted flow — model improvises steps from system_prompt + tool descriptions (trained agentic behavior: "can't answer → investigate first" is a LEARNED prior, needs no instruction). Specialist (`-TroubleshootIncident.yaml`): the system_prompt CONTAINS an explicit written playbook — `## Troubleshooting Workflow` with ordered named Steps, a condition TABLE (owning team = X → load IncidentKB), report formats, confidence formula. So "scenario guideline" DOES exist, in 3 tiers: (1) baked into specialist's system_prompt, (2) TSG docs fetched at RUNTIME via `fetch_engineeringhub_doc_by_url`/`search_relevant_docs`, (3) `get_prompt_template` named snippets stored server-side. A "scenario" = a system prompt; same model+tools, different playbook. `-TroubleshootIncident.yaml` is authoritative ONLY for the W365-incident flow — NOT a "primary guideline"; README treats all YAMLs as uniform peers.

**ctx = per-TURN record, NOT per-session (Damien's correction).** New `ctx` created every `sendMessage` (useAgentChat.ts:615), holds BOTH directions: `question` (outgoing), `agentState` (incoming response container), `allMessages` (per-turn transcript snapshot), `controller` (abort), `sessionId`/`streamKey`/`detached`. Discarded when turn ends (:589). `agentState` (IAgentStreamState) = the response container: `assistantContent` (growing answer string), `assistantMessage` (the on-screen message being built — "bubble" = UI slang for a message's display box), `assistantSegments`, `assistantTools`, `isToolInProgress`.

**SEND + RECEIVE ARE THE SAME CALL.** `consumeAgentStream(ctx, conversationId, (signal)=>agentChatApi(request, signal))` — arg 3 is a DEFERRED callback (thunk/recipe), NOT a chain and agentChatApi is NOT called at that line. consumeAgentStream OWNS the flow: it creates the AbortController, then RUNS the recipe (`makeRequest(controller.signal)` :428) which is where agentChatApi's fetch (the actual SEND) fires, then reads the SSE response. Same consumer reused for resume via `agentResumeApi` → `/copilot/agent/resume` (dependency-injection/inversion-of-control). `await` spans the whole stream.

**SSE READ LOOP (RESP→LOOP→AS, useAgentChat.ts:430-515):** `response.body.getReader()` → `await reader.read()` blocks for next byte-block (`{done, value}`) → `TextDecoder("utf-8").decode(value, {stream:true})` bytes→string (stream:true holds partial multi-byte chars across chunks; decoder reused ONCE for this) → append to `buffer` (string that survives across reads) → cut COMPLETE frames on `\n\n` (`indexOf`/`slice`; network chunks DON'T align to frame boundaries, buffer reassembles) → per line: skip blank/`:` (SSE heartbeat comments), strip `data: ` prefix, `[DONE]` end marker, `JSON.parse` → `applyStreamChunk(parsed, agentState, signal, ctx)` = THE APPEND. `resetStreamTimeout()` per read (90s silence = dead connection → abort).

**SEGMENTS not one string (agentSegments.ts):** answer INTERLEAVES text + tool calls in order, so stored as ordered `IMessageSegment[]` (`{type:"text"...}` | `{type:"tool"...}`). `assistantContent` string is DERIVED via `textFromSegments` (reduce, concat text segs only). `appendTextDelta`: grow last segment IF it's text AND same `sourceMessageId`, else push new segment. All PURE (`const next=[...segments]`, never mutate — React needs new reference to re-render). `sourceMessageId` = the seam: SRE streams chain-of-thought as a SEPARATE message that mid-stream looks identical to answer text (leaks); tagging by source id keeps them separable so `reconcileReasoningTool` can later swap leaked reasoning text for a collapsed "Thought process" block.

**LIVE STREAMING RENDER (not after-done):** every chunk → mutate agentState → `dispatch(UPDATE_LAST_ASSISTANT)` → React re-renders the message with longer text → user sees it type out. When `ctx.detached` → `saveToBackground` instead of dispatch (same agentState, different destination).

**STOP vs RESUME vs DETACH (Damien conflated — corrected):** STOP = `controller.abort()` via `signal` = kills the fetch, TERMINAL, no pause/resume. RESUME (`resumeStream` → `/agent/resume`) = DIFFERENT scenario: reconnect to a STILL-GENERATING server turn after PAGE RELOAD wiped client state; re-streams current answer. DETACH (`detachStream`) = switch chats mid-stream → mark `detached=true`, keep streaming into agentState but route to background store, remove controller from abort-list so chat-switch doesn't kill it. REATTACH = return to chat → `detached=false`, re-add controller, resume UI routing. Enables multiple NON-INTERFERING concurrent turns (each own ctx in a `Map` keyed by streamKey, own fetch/TCP stream, own controller — no shared state = no cross-talk). Same-chat second send BLOCKED by `sendQuestion`'s `if(isLoading||effectiveStreaming) return` guard; cross-chat allowed via detach.

**allMessages purpose (client-side history) — WHY keep full history if SSE only carries latest answer:** the STREAM/REQUEST only needs the latest (server thread holds real history), but the UI must RENDER the whole message list and DETACH must snapshot the whole conversation. `allMessages` = display/persistence cache, NOT request context. `snapshotMessages(ctx)` = prior messages + current agentState-built message swapped in at the end.

**REDUCER STATE = the session-long client transcript (Damien: "what IS reducer state, does it accumulate allMessages?").** Reducer = (state, action)→newState pattern. `CopilotState.messages: IChatMessage[]` = the on-screen transcript. Does NOT copy ctx.allMessages — builds its OWN list via actions: `APPEND_MESSAGE` (`[...state.messages, payload]` = THE accumulation, fires per user msg + per assistant msg), `UPDATE_LAST_ASSISTANT` (replaces last assistant, fires per chunk). ctx.allMessages (per-turn copy) and state.messages (durable client transcript) are fed the same messages INDEPENDENTLY. Per turn: dispatch APPEND user → chunks dispatch UPDATE_LAST_ASSISTANT → turn ends, ctx discarded, state.messages KEEPS everything.

**HISTORY ACROSS RELOAD/DAYS — reducer state does NOT survive (in-memory only, resets to [] on refresh).** Durable history = SERVER: SRE thread holds transcript, Cosmos holds session pointer (id + conversationId + title). On reopen: sidebar lists sessions from Cosmos → click → backend `get_messages` from thread via conversationId → `dispatch(LOAD_SESSION)` drops FULL transcript into state.messages → all turns re-render. The conversation-ID-in-URL is what survives refresh and lets reload find the chat. SAME pattern as ChatGPT (Damien asked): `/c/<id>` in URL → fetch conversation on load → history reappears. Universal chat-app architecture: client state = live disposable view; server = durable record; an id (in URL) links them. Mental model: reducer `state.messages` = live view of CURRENT session; SRE thread + Cosmos = permanent record; within-session accumulation = reducer (APPEND_MESSAGE); across-session = server, refilled via LOAD_SESSION.

#### Same-day follow-up — token→pixel render path + segment/delta mechanics (drilled deeper)

Second pass same day drilling the RECEIVE→RENDER half harder. Damien corrected several of his own models (delta-diffing, segment purity, frame boundaries, "why append"). Complements the trace above with the exact pixel path.

**agentState = LIVE scratchpad for the ONE reply streaming now; allMessages = settled HISTORY.** All IAgentStreamState fields are PROJECTIONS of `assistantSegments` (the master record): `assistantContent` = text-only projection, `assistantTools` = flat order-independent list, `isToolInProgress` = "tool running now?" flag, `assistantMessage` = the IChatMessage for UI/history, `workingConversationId` = backend id. Segments are truth; everything else re-derived. agentState `null` before stream starts (line 666-672), set null in finally when it ends (folded into allMessages).

**SEGMENTS ARE PURE — discriminated union, text OR tool, never both (Damien: "could one be mixed?").** Enforced by construction (appendTextDelta only makes text, upsertToolSegment only makes tool) AND by TypeScript (`.content` on a tool segment = compile error).

**BIGGEST CORRECTION — frontend does NOT compute deltas; BACKEND already sends them.** Damien's model was "receive full text, cross-check against stock, find what's new, append it." WRONG. "Delta" = the new fragment (`"Hello"`→`" world"`→`"!"`); backend streams these already-chopped as separate SSE events. `parsed.content` (useAgentChat.ts:366-368) IS the fragment; frontend just APPENDS — no diffing. Function is `appendTextDelta` not `computeDelta`. This is the browser-side view of the SAME delta the Day-5 host-side diff-engine PRODUCED from polled snapshots — two ends of one pipe. "Frontend appends" = frontend is ACCUMULATOR, backend is PRODUCER (fires fragments, moves on). WHY append (Damien: "only to track what's new?"): NO — the answer arrives in pieces but you need the WHOLE thing to display; each fragment alone is meaningless; append = ASSEMBLE the full reply → saved to allMessages at end.

**appendTextDelta grow-vs-new (agentSegments.ts:35-50):** on a text delta, if LAST segment is text from SAME sourceMessageId → concat onto it; else push NEW segment. One segment grows across MANY frames. textFromSegments (line 23) = FULL REBUILD from scratch each time (not incremental). **Merging loses FRAME BOUNDARIES (deliberately — arbitrary network-chunking artifacts, no meaning) but NEVER order** (only merges into last / pushes at end, never mid-insert or reorder) — a tool between two text runs stays exactly where it happened. Preserved: text content, order-vs-tools, source id. Analogy: taping adjacent pieces of a torn letter — lose tear-lines, keep word order, don't mix two letters.

**FULL PATH token→pixels (repeats per chunk = the live-typing effect, NO special animation):** (1) applyStreamChunk appends to segment → updateAssistantMessage copies onto assistantMessage → (2) `dispatch({type:"UPDATE_LAST_ASSISTANT", payload:{...assistantMessage}})` (line 275) — spread = NEW object ref for React → (3) reducer (Copilot.reducer.ts:166) walks BACKWARDS to last assistant msg, swaps into NEW array `[...state.messages]` (new ref → re-render) → (4) Copilot.tsx:752 `messages.map` → renderMessage → CopilotResponse → (5) CopilotResponse.tsx:243-259: groupSegments → tool groups `<ToolActivityGroup>`, text groups `<ReactMarkdown>{group.content}</ReactMarkdown>` (why replies show formatted markdown); `{isStreaming && !lastIsActiveTools}` adds `<LatencyLoader>` pulse. Dispatch only fires when `!ctx.detached`.

**PERF — CopilotResponse is React.memo (CopilotResponse.tsx:318-331):** custom comparator returns false (=re-render) only when message.content OR the last text segment's content changed → avoids re-rendering all messages on every fragment. Depends on same-source text being grouped into one growing segment (cheap "did last segment change?" check).

### 2026-07-15 — Azure infrastructure: auth, Managed Identity, Key Vault, OBO (Day 6, ~5/6 quiz)

Read credential.py + keyvault.py (diagnostics MCP). The auth story Damien had brushed past since Day 2 (that `get_azure_credential()`). Checkpoint met: can explain MI vs password auth. ~5/6; two deep gaps filled (what MI physically is; OBO cache-key security).

**SECRET ZERO problem:** to access secrets you need a secret, but that first secret must live somewhere (code/config/env) → stealable, leaks, expires, gets committed. Every password is a liability.

**MANAGED IDENTITY = the solution (kills secret zero).** `get_azure_credential()` (credential.py:46): prod → `ManagedIdentityCredential(client_id=AZURE_CLIENT_ID)`, NO password anywhere. WHAT MI ACTUALLY IS (Damien asked, guessed IP/MAC — wrong): two pieces — (1) an identity registered in Entra/AAD (a "service principal" = a user account for an app, gets an oid, can be granted roles like "read this Key Vault"); (2) the resource proves it's that identity via a PRIVATE LOCAL METADATA ENDPOINT (169.254.169.254, instance metadata service) that the Azure FABRIC answers — only reachable from INSIDE that resource. So identity comes from BEING the resource (only it can reach its own endpoint), not from holding a secret. Not IP/MAC (spoofable, network-level) — it's the platform vouching from underneath (infra level). Analogy: password = house key (copyable, must re-key if lost); MI = recognized by the security guard (nothing to carry/steal/rotate). Systems-lens answer: MI beats passwords because there's no secret to store/leak/expire/rotate.

**client_id is NOT a secret:** `AZURE_CLIENT_ID` = a NAME (username) for WHICH assigned identity to use (a resource can have several). Knowing it gets an attacker nothing — they still can't BE the resource. Safe in env vars. Identifier vs credential.

**LOCAL vs PROD (`is_debug_mode()` = DEBUG_MODE=true OR no AZURE_CLIENT_ID):** local → `ChainedTokenCredential(Environment, AzureCli, VSCode, AzurePowerShell, InteractiveBrowser)` = tries each until one works = "use whatever login the dev already has (az login / VS Code sign-in / browser)". Why local can't use MI: laptop ISN'T an Azure resource — no fabric, no metadata endpoint, no provisioned identity. MI only exists inside Azure. Same get_azure_credential() = personal identity locally, resource's MI in prod.

**KEY VAULT = where unavoidable secrets live.** MI removes MOST secrets but some services need their own credential. Elegant chain: app uses MI (passwordless) to unlock Key Vault (keyvault.py:27 `self._credential = get_azure_credential()`), Key Vault hands over the secret. `get_secret`/`get_certificate_bytes`. Key Vault = Azure's dedicated access-controlled audited secret store; the ONE place secrets are allowed; access gated by identity (MI) + logged.

**WHY IcM NEEDS A CERT (not MI) — Damien's quick question:** MI only works when the target service TRUSTS Entra/AAD tokens (Kusto, Key Vault, Azure Search all do). IcM authenticates with CERTIFICATES, not Entra tokens — a different/older auth system that doesn't integrate with MI. Cert = IcM's required credential (a crypto key pair, stronger than a password). Chain: MI unlocks Key Vault → Key Vault gives cert → app presents cert to IcM. So MI is the master key that avoids storing ANY secret in the app, even for cert-based services.

**ON-BEHALF-OF (OBO) = act AS the user, not the app.** `get_obo_credential` (credential.py:93). Why: sometimes must access data with the USER's OWN permissions (user only sees incidents they're authorized for), not the app's broad perms. Flow: (1) user's access token arrives in Authorization header (from Day-4 host), (2) app proves ITS identity via MI using Federated Identity Credential/FIC (mi_credential.get_token("api://AzureADTokenExchange/.default") as client_assertion), (3) Azure EXCHANGES user token → service-scoped token so app calls Kusto AS that user. Requires OBO_TENANT_ID/OBO_CLIENT_ID/OBO_MI_CLIENT_ID.

**OBO CACHE KEY = verified oid, NOT raw token (the security gem Damien missed):** OBO credential cached (TTLCache, 10min, skip repeat AAD round-trips). MUST key by the VERIFIED oid (AAD object id Azure validated), NEVER the raw user token (user-controlled). If keyed by raw token → crafted/colliding token could fetch ANOTHER user's cached credential → user A acts as user B = cross-user leak. Principle: only trust SERVER-VERIFIED identity as a cache key, never user-supplied strings. (Damien guessed it was about expiry — no, TTL handles freshness separately; key choice is purely cross-user isolation.)

**NonClosingCredential (ties to Day-2 caching):** Kusto client closes its credential's HTTP transport on `.close()`. But a SHARED/cached credential (used by other in-flight requests within TTL) must NOT be torn down by one per-request client closing. So NonClosingCredential wraps a shared credential, makes `.close()` a no-op, delegates everything else (__getattr__). Same adapter pattern as Day-2 SearchClient; protects a shared resource from premature teardown. "What happens under concurrency?"

**EV2 (brief, no code):** Express v2 = Microsoft's internal SAFE-DEPLOYMENT pipeline (staged regional rollouts, safe-deployment practices, approvals). Saw it Day 5 (`Build-SubAgentEv2.ps1`). Orthogonal to auth — the "how it gets deployed" layer.

**Three credential modes, one function + OBO:** local = personal login (chained); prod-as-app = MI (no secret); prod-as-user = OBO (user's token exchanged, their perms). One-line: agent proves identity without passwords via MI (Azure vouches for the resource → secret zero gone); unavoidable secrets live in Key Vault (unlocked by MI); OBO acts as the user when their permissions must apply.

**MI TOKEN MECHANISM — precise (Damien pushed hard to fully understand, corrected one misconception):** The RESOURCE does NOT generate the token (his initial guess) — ENTRA does. Flow: (1) app asks the LOCAL ENDPOINT (169.254.169.254, on-machine "receptionist" program Azure runs) for a token; (2) local endpoint REQUESTS the token from Entra, proving the machine's identity using crypto material the platform planted (app never sees it); (3) Entra ISSUES + cryptographically SIGNS the token; (4) flows back to app; (5) app sends token to Kusto; (6) Kusto trusts it because it carries ENTRA'S unforgeable signature (verified via Entra's public key) — NOT because "the resource made it." Two security pillars: (a) local endpoint only reachable from INSIDE the machine (so getting a token requires already BEING the resource); (b) proof material held by the endpoint, not app code (app holds NO secret). Hotel keycard analogy: Entra=front desk (issues cards), local endpoint=in-room kiosk only you can reach, app=you, Kusto=amenity door that checks the front desk made the card.

**AZURE FABRIC = the platform (clarified after confusion — avoid the word, say "platform"):** the datacenter management software that runs/hosts all physical machines — the "landlord" software. Role in MI = STEP 0 / setup: it created the local endpoint, planted the identity-proof material, and recorded "this machine = this identity" (it's the trusted party that told Entra the assignment). "Platform vouches from underneath" = trust comes from the layer BELOW the app (which hosts it), not from anything the app holds; an attacker can't fake it without being Azure itself. You never touch the fabric day-to-day — just call get_azure_credential(); it matters only for understanding WHY MI is trustworthy. File it as: "the Azure platform layer underneath that provisioned my machine's identity."

### 2026-07-14 — SRE agent definition (spec.yaml) + tool placement (Day 5 part 2, completes Day 5)

Second half of Day 5 (part 1 = SRE streaming internals, logged same day). Had to RE-TEACH spec.yaml slowly — Damien correctly said "I haven't learned spec.yaml" after I jumped to a quiz on it (presenting ≠ learning; don't quiz before absorption). This half = the agent DEFINITION.

**spec.yaml = an agent's "job description" (declarative config, `kind: AgentConfiguration`):** answers which tools + which instructions + what behavior mode + name. Same declarative-YAML idea as Day 3 routing.yaml (data describing intent, not code). This is the FIRST time Damien saw an agent DECLARED explicitly — the portal agent's definition was hidden in Foundry; the SRE agent's is a readable file. This is where "layer ① base agent prompt" (mentioned since Day 3) finally lives.

**Two tool lists = allow-lists (GRANTED access, NOT copied/inherited — Damien asked):**
- `tools:` (~65) = built-in Azure SRE PLATFORM tools (IcM ops GetIncidentDetails/PostDiscussionEntry, charting PlotBarChart, ExecutePythonCode, CreateFixPullRequest, pipelines, SearchMemory). Hosted/implemented by Azure; spec just REFERENCES by name. Analogy: badge granted access to shared printer (not your own copy).
- `mcpTools:` = the diagnostics MCP server's tools (Day 2!) — `diagnostics-mcp_search_relevant_docs`, `_query_geneva_metrics`, `_execute_kusto_query`. Prefix names the source server. Same allow-list/grant model.

**WHERE the two lists RUN (key split, ties to Day 1 tool-use):** LLM calls both IDENTICALLY (emit tool_use → get result). Executor differs: native `tools` run on Azure SRE platform; `mcpTools` run on THIS repo's MCP server. LLM doesn't know/care which — the tool abstraction hides the executor.

**instructions = INJECTED AT BUILD TIME (Damien noticed spec has only a comment, no instructions field):** spec.yaml is the TEMPLATE/output, not the source. Pointer lives in `subagent-config.json` (PromptFile field), NOT in spec.yaml. Chain: subagent-config.json (PromptFile=troubleshoot-guidance-w365.md) → Day-3 prompt COMPOSER resolves it → Build-SubAgentEv2.ps1 injects text into `instructions` before deploy. So SRE composes at BUILD time (baked in once at deploy); portal composes at REQUEST time (live per-request). Same composer, different timing.

**agentType: Autonomous** = the behavior switch. Autonomous = acts on its own (incident/cron triggered, no human per step) vs interactive (portal waits for human). ScheduledTask (kind: ScheduledTask, `cronExpression: 30 2 * * *` = 2:30 AM daily, agentMode: autonomous) = the clock trigger (Day-3 three-triggers made concrete).

**SYSTEMS-LENS ANSWER "why same tools → different behaviors":** behavior lives in the DEFINITION around the tools, NOT the tools. Three differences: (1) INSTRUCTIONS (portal: help-a-human/conversational; SRE: investigate-end-to-end + post-to-IcM + open-PR), (2) TOOL MIX (SRE has ACTION tools CreateFixPullRequest/PostDiscussionEntry the portal chat lacks), (3) AUTONOMY (autonomous vs interactive). Same search_relevant_docs, different agent → different behavior. Concrete: incident at 3AM → portal does nothing (needs human); SRE wakes, searches runbook, checks metrics, posts findings, maybe opens PR — no human.

**TOOL PLACEMENT (corrected Damien's "tools between frontend and backend" framing):** tools are NOT between frontend/backend and NOT near the frontend at all. A tool = a function the LLM calls during the loop to touch REAL SYSTEMS (IcM, Kusto, Geneva, repos) — on the AGENT/PLATFORM side. Frontend only WATCHES the narration stream by (SSE). "Process data and call functions" = right; they connect LLM→outside-world, not frontend→backend. Placement: browser ↔ host (SSE) ↔ agent(LLM+loop) → tools → real systems. Tools live at the far right.

**WEEK 1 COMPLETE (Days 1-5): the full system Damien can now trace** — MCP server (Day 2 tools + Day 3 prompts, SHARED) used by TWO hosts: portal (Day 4, interactive, human-driven, SSE stream) and SRE agent (Day 5, autonomous, spec.yaml-defined, cron/incident-triggered, polling stream). Behavior differs by host+definition, tools are shared.

### 2026-07-14 — SRE Agent host + polling/streaming internals (Day 5 part 1, current branch code)

Repo moved ahead of the Day-4 snapshot: on branch `lucaszhang/agent-mode-sre-backend`, `stream.py` was replaced by `sre_stream.py` (~50KB) + rewritten `handler.py`. The generic Foundry push-stream (`responses.create(stream=True)`) is GONE; the current SRE-agent-mode backend uses REST POLLING + optional SignalR hub. Deep multi-session dive into the polling internals. Damien found real engineering tensions unprompted (memory bounds, ordering scope, multi-decider gate) and self-corrected two of his own models (hub-vs-polling inverted; "one message per turn").

**Architecture shift (old stream.py → new sre_stream.py):** backend = Azure SRE Agent data plane (not Foundry Responses API). Conversation unit = THREAD (`conversationId == thread_id`). Two handlers: `handle_conversation_sre` (new/continued) + `handle_resume_sre` (reattach to in-flight turn after page reload). Border into SRE agent = `create_thread`/`post_message` (send) + POLL `get_messages` (receive) — NOT one pushed stream.

**Hybrid: polling backbone + hub fast-path (Damien had this INVERTED — corrected):**
- POLLING = mandatory backbone, always runs, carries tool cards/reasoning/structure + is the complete fallback + streams text when hub off. Cannot be removed.
- HUB (SignalR) = OPTIONAL accelerator, only for fast text tokens + authoritative `SignalProcessingComplete` completion signal. System works fully without it (just laggier text + heuristic completion). NOT "main source with polling as backup" — polling is source of truth, hub is turbo for text.
- When hub text on, REST text SUPPRESSED (`_suppress_rest_text`) to avoid double-emit.

**Who decides if hub is "on" = THREE independent deciders, all ANDed (sre_stream.py:844 `use_hub_text = use_signalr and hub_text_enabled and not self.resume`):**
1. NETWORK: `use_signalr = listener.start()` — did the WebSocket physically connect? (firewall/proxy/auth can block it)
2. OPS CONFIG: `get_hub_text_streaming()` — feature flag operators set on/off (policy-in-config, no redeploy)
3. REQUEST TYPE: `not resume` — hub can't REPLAY text generated before it connected, so resume falls back to REST (whose first poll re-emits the full current answer). Hub is a LIVE channel with no memory of past pushes.
- Any false → polling streams text. That's WHY polling is the non-removable backbone (hub has 3 ways to be off).

**Poll interval (adaptive):** `_POLL_INTERVAL_SECONDS = 0.8` normally; `_POST_COMPLETE_POLL_SECONDS = 0.3` once completion_seen (answer imminent, poll faster). Ceilings: `_MAX_STREAM_SECONDS=600` hard cap, `_FIRST_RESPONSE_TIMEOUT=150` give up if nothing. `_HEARTBEAT_INTERVAL=10s` → `_SSE_HEARTBEAT` frame keeps SSE connection warm through quiet gaps (else intermediary idle-timeout drops it, client misreads as end-of-turn).

**TURN = MANY messages (Damien thought 1 — CORRECTED):** one turn produces: user msg + one message per tool call + one per reasoning block + (usually) ONE answer-text message. Only the answer-text message GROWS char-by-char (same id, `text` lengthens each poll). Tool/reasoning messages are "DISCRETE" = separate self-contained units appearing whole, not lengthening. So a turn = several discrete messages + one growing message (NOT one-per-word, NOT one total).

**Message is NOT a stream — it's a GROWING RECORD.** Server assigns each message a stable `id` + `timestamp`. `_chronological_messages` sorts AMONG the turn's many messages (by timestamp), NOT within a single message. `get_messages` requests `orderby: timestamp`; client re-stitches across pages.

**THE DIFF-ENGINE DELTA TRICK (how snapshots fake a stream) — the core insight:**
- portal = memory-light DIFF ENGINE. Per message id it remembers only `_emitted_text_len {id: charcount}` (an int) + `_emitted_tool_payloads {id: fingerprint}`. NOT the content.
- Each poll: `already = _emitted_text_len.get(mid, 0); if len(text) > already: delta = text[already:]; emit(content=delta)`. Forwards ONLY the new suffix even though the poll returned the WHOLE current text. (git diff analogy: repeated diffs of a growing file = a stream of additions.)
- Browser receives "The error"+" rate"+" is 4.1%", appends each → sees token-by-token stream. A "stream" = sequence of appendable chunks over time; browser doesn't care the source was snapshots. Tools = same principle keyed on payload FINGERPRINT (status/output/args) instead of char-length.

**baseline_ids = the turn-mapping + idempotency mechanism:** snapshot all existing message ids BEFORE posting (handler.py:335). Everything NOT in baseline_ids = THIS turn's response. Maps response→question positionally (new-since-baseline = this turn). ALSO enables "never duplicate a turn": on lost `post_message` HTTP response, `_send_follow_up` VERIFIES whether the msg landed (a new-since-baseline user msg matching text) instead of blind-retrying → if it landed, continue streaming, never re-post. Idempotency via verification, not blind retry.

**WHO holds context server-side = the THREAD (not the loop, not the LLM).** Thread = durable Azure-managed storage (a row/doc in SRE Agent service's own DB), holds ordered messages = the conversation memory the LLM reasons over. Reached ONLY via REST (create_thread/post_message/get_messages); portal holds just the thread_id. Adding new question to context = SERVER's job when it gets post_message(thread_id, text) — NOT the host (Damien guessed host; it's the opposite — thread model = server owns/grows context, host stays stateless about content).

**COSMOS vs THREAD (both stores, different data, NOT duplicates):** thread (Azure/SRE) = conversation CONTENT; Cosmos (portal) = session METADATA (title/product/updatedAt) + POINTER to thread_id. Cosmos's most important job = the user's SESSION DIRECTORY/index (powers sidebar list, titles, recovery-after-refresh). Maps user→sessions→thread_id.

**CONTEXT-WINDOW LIMIT is AGENT-side, invisible to host:** host has NO context concern (just streams deltas, never assembles history+question). Thread = unlimited storage (no token limit). LLM = where 1M limit bites. Agent LOOP bridges the gap (decides how much thread history to send, truncate/summarize). Portal entirely out of context-management.

**MEMORY guards (Damien's `all_messages` catch — real concern, handled):** `get_messages` builds `all_messages` list — real transient memory, bounded 3 ways: `max_messages=2000` (count cap, `while len < 2000`), `start_skip`/tail (streaming polls fetch only CURRENT TURN tail, skip baseline history → constant per-poll cost regardless of thread length), `page_size=200` (per-response cap). Rebuilt+discarded each poll, NOT accumulated. Huge single message = transient per-poll bytes (fetched, diffed, discarded), host retains only the char-count. Count bounded; single-message size not (different axis).

**gevent + doc vocabulary:** `gevent` = "green+event" concurrency lib; greenlets = lightweight cooperative green-threads (not OS threads), cheap (thousands), fit web servers with many connections; `gevent.spawn(fn, arg)` = run concurrently, don't block; gunicorn gevent worker runs the app this way (also why polling `sleep` is non-blocking). Debug dev server doesn't drive the gevent hub → `if get_debug_flag(): inline else: gevent.spawn` (handler.py:318). `doc` = "document" = a Cosmos record (JSON object); Cosmos = NoSQL document store; `build_session_doc`/`persist_session_doc` = build/save the session metadata record.

### 2026-07-14 — Flask backend handler + streaming/SSE deep-dive (Day 4, host side, ~6.5/8 quiz)

Read the actual portal backend (the HOST for the human path). Big streaming/SSE/yield/WebSocket exploration. Checkpoint met: can trace a request handler→Foundry→back and name the border crossing.

**The host = portal backend (NOT the agent, NOT Foundry).** It's a relay + translator + bookkeeper between browser and Foundry. Repo architecture: `src/portal/` (frontend + backend relay, Cosmos bookkeeping) | `src/mcp-servers/diagnostics/` (tools+prompts) | `src/sre-agent/` + Foundry (agent config). Portal stores only SESSION METADATA in Cosmos, NOT messages (those live in Foundry conversation).

**3-layer chain (route → handler → stream):**
- ROUTE `copilot/__init__.py`: `@api.route("/agent/chat", POST)` → checks JSON → delegates to `handle_conversation_agent`. Owns ONLY the URL binding. (Sibling routes: /conversation=Ask mode, /followups, /feedback.)
- HANDLER `agent/handler.py`: `handle_conversation_agent` (NOTE: _agent not _sre; Foundry uses CONVERSATIONS not threads). Numbered steps: (1) agent ready? (2) validate message fail-fast (3) product→agent name (4) get/create conversation + Cosmos session (5) StreamingHandler.stream → SSE response.
- STREAMER `agent/stream.py`: calls Foundry, translates events→SSE.

**THE BORDER CROSSING = `responses.create(...)` (stream.py:175):** `get_openai_client().responses.create(conversation=id, input=msg, extra_body={agent_reference:{name}}, truncation="auto", stream=True)`. Everything BEFORE = portal/host; everything that call triggers (loop, LLM, MCP tools, prompt layers ①②③) = Foundry, far side. `for event in response_stream` (line 183) = portal receiving Foundry's streamed events back. This one call is the MCP↔host boundary made literal. `truncation="auto"` = Day-1-Q6 context overflow handled by one param.

**Host owns vs delegates (Day 4 systems-lens):**
- OWNS: HTTP/route/parsing, validation, product→agent selection, session metadata (Cosmos), conversation TICKET mgmt (create/pass id), translating events→SSE, telemetry/title-gen/error-sanitizing.
- DELEGATES to Foundry: the agent loop, calling LLM, composing/combining prompt layers, holding conversation history, calling MCP tools, context-window mgmt. One-liner: host = relay + bookkeeper, NOT a brain. After responses.create() everything learned Days 1-3 lives on the far side.

**The 3 IDs:** `user_oid` (who, from flask.g/Entra) → owns many `sessionId` (durable chat container in Cosmos: title/pinned/sidebar) → points to `conversationId` (live LLM history in FOUNDRY, the "coat check ticket"). One session can have MANY conversations over time (new agent version → reopen session → see old messages from Cosmos but fresh Foundry memory). Split because state at different rates lives in different stores (separation of concerns).

**Coat-check ticket (Damien nailed):** browser sends conversationId → continued chat, portal passes the ticket, Foundry holds the coat (history). No id → new EMPTY conversation created (fresh ticket, no coat yet).

**Fail-fast validation — WHY (he missed the SSE reason):** (a) never trust the client — UI blocking empty msg is cosmetic; anyone can curl the endpoint, backend MUST re-validate (boundary validates, "extract loosely validate strictly"). (b) SSE-SPECIFIC: streaming is a ONE-WAY DOOR — once you send the opening frame + 200 OK, headers are on the wire, can't cleanly error. So validate BEFORE stream() (handler.py:96 before :145) while you can still reject cleanly.

**Background write — WHY (he was unfamiliar):** `gevent.spawn(store.persist_session_doc, doc)` (handler.py:140) = run concurrently, don't wait. Cosmos write = network, 50-200ms. Inline → user waits staring at nothing before first token. Background → id generated locally (instant), kick off write, stream immediately. = the "defer Cosmos session write off the streaming critical path" commit from his 6-15 notes. Fail-soft: failed write logged+tolerated, chat proceeds.

**Streamer's real job = ADAPTER (he got half):** translates FOUNDRY EVENTS (`response.output_text.delta`, `output_item.done`, etc. — NOT SSE) → SSE FRAMES (`data:{...}\n\n`). Two DIFFERENT vocabularies (not "SSE to SSE"). Same pattern as Day-2 `SearchClient` wrapper/adapter: translate between two systems' languages. `_handle_event` dispatcher maps each Foundry event type → SSE frame; you watch the agent loop from outside (tool calls show as events → UI shows "running tool").

**STREAMING (big deep-dive):**
- = deliver data INCREMENTALLY over time; ONE API call whose RESPONSE arrives in pieces over ONE held-open connection. Opens on POST (question goes up), SAME connection stays open, answer streams DOWN, closes only after last token. NOT closed between Q and A. Damien's analogy: cut meat into cubes, pass through a tube to the cook (vs throwing one big chunk).
- BOTH sides stream: browser↔portal (SSE format, proven by `mimetype="text/event-stream"` handler.py:50) AND portal↔Foundry (Responses-API events). Portal relays one→other near-real-time. Corrected his framing: NOT "call vs stream" — both are single calls whose responses arrive in pieces.
- SSE = one-way (server→browser); WebSocket = two-way (Slack-style chat apps, both push); video (Netflix/YT) = HTTP segments usually NOT sockets; video calls = WebRTC. Streaming = general concept, many techs.
- Why SSE here not WebSocket: AI answer is one-way (ask once, stream reply); browser doesn't push mid-answer. SSE simpler.
- `_make_sse_response` (handler.py:41) = local helper wrapping a generator in Flask Response with SSE headers: `text/event-stream`, `Connection: keep-alive`, `X-Accel-Buffering: no` (don't buffer, flush each chunk immediately).
- `responses.create()` naming: "responses" = the API name (OpenAI Responses API, Foundry implements it); `.create` = create/START a new response (doesn't exist yet — you're starting generation). `stream=True` → returns a stream object immediately (iterable of events as produced) instead of blocking for the full answer.

**`yield` (he never understood it):** = like return but PAUSES the function (freezing local state + position) instead of ending; resumes from that exact spot when asked for next value. Function with yield = a GENERATOR. Necessary for streaming (not just "suitable"): without it, stream() would build the ENTIRE answer then return all-at-once = no streaming. Each yield (stream.py:254) hands out one SSE frame the instant it's ready → pause → Flask flushes it → resume for next. Teppanyaki chef: cook one piece, serve, pause, cook next (vs cook whole meal then bring). "yields events" = hand out one event at a time, pausing between, so caller sends each onward immediately.

### 2026-07-09 — Prompt layers, folder-vs-flow, ID timing, loop interleaving (Day 3 follow-up → Day 4 bridge)

Deep follow-up after the Day 3 quiz. Damien kept probing the prompt/loop boundary and reconstructed how prompt-invocation interleaves with the agent loop — the crux of agent architecture. These are the refinements/corrections from that session.

**THREE LAYERS of prompt (his "surely compose isn't everything?" instinct was right):**
- ① BASE agent prompt — the agent's PERMANENT identity/safety/tool-etiquette, scoped to the agent's whole existence, rarely changes. Owned by HOST (spec.yaml / agent config, Day 5).
- ② TASK prompt = `compose()` output — the PROCEDURE + output contract for ONE task-type (LSI vs CRI vs AVD), written ahead of time, changes per task-type. Owned by MCP. Contains: task-scoped role ("act as troubleshooter" = narrower hat than ①), ordered steps, which-tools-when, task rules, report format.
- ③ CONVERSATION — the live back-and-forth (user msg + tool calls + results), changes every turn, built by the HOST loop at runtime.
- `00-overview` restates identity in ② (even though ① also sets identity) because MCP is HOST-AGNOSTIC — can't assume the host framed it right, so self-contained. Base rule = "never fabricate" (personality); task rule = "fetch incident before metrics" (procedure).

**WHERE the 3 layers combine = the HOST, never MCP.** Host builds the API call: `system = ① + ②` (concatenated), `messages = ③` (the running list). MCP only produces ② and stops at `return`; has no idea ①/③ exist. Combining is NOT in _helper.py/troubleshoot_incident.py — those return ② and exit over HTTP.

**A "step" has 3 forms:** (1) a name-string in flow YAML (`lsi/01-fetch-incident`), (2) a `.md` file on disk (`_fragment_path`: name → `fragments/lsi/01-fetch-incident.md`), (3) a text section in the final prompt. `compose()` = read each step's file in YAML order, `"\n\n".join(bodies)`. "Displayed" = sections stacked top-to-bottom. The `### Step 1` headings INSIDE fragments are human-written Markdown, NOT the YAML `steps:` list (they rhyme but are different levels).

**Where the flow "leaves":** compose returns string → troubleshoot_incident prepends `# Troubleshoot Incident {id}` → registered MCP prompt → HTTP response → HOST. Exits at the same HTTP door as any MCP response.

**CORRECTION — folder is NOT a flow (he initially thought lsi/ folder = a flow, numbered files = its steps):**
- `fragments/lsi/` = a LIBRARY/drawer of bricks grouped by topic. A flow = a FILE (`flows/lsi-w365.yaml`) that LISTS which bricks in what order.
- Numbers (00-, 01-) = human sort-hints for editor ordering, NOT execution order. Execution order = the flow's explicit `steps:` list.
- Clincher: a flow pulls from MULTIPLE folders (lsi/ + shared/ + addons/w365/), so it can't BE the lsi folder. One folder of bricks feeds MANY flows (lsi-w365, lsi-cdp, lsi-avd...). The naming rhymes (incident-type LSI / folder lsi/ / flows lsi-*) which is why it's confusing — 3 different things at 3 levels. (He then got this right: "flow has its own folder, each flow's steps refer to fragment folder.")

**Incident ID source — 2 cases (his question surfaced this):**
- Case A (webhook/scheduler): ID arrives STRUCTURED (IcM event field). Host passes it directly to troubleshoot_incident(12345) UP-FRONT. No extraction.
- Case B (human types free-form "look into incident 12345"): ID is EMBEDDED in prose → must be EXTRACTED. Modern agent design: THE LLM extracts it (reads the ID from the sentence, calls the prompt-builder with it) — it's a natural-language task, no regex. (Mechanism 2 = code/regex extraction, old-school, less flexible.)
- Extract loosely / validate strictly: regardless of source, MCP validates at the boundary (`_routing.py:67` `if not incident_id.isdigit(): raise`). MCP never trusts the extraction.

**KEY INSIGHT — loop timing (he guessed "loop hasn't started yet" — actually OPPOSITE):**
- In Case B, the agent loop has ALREADY STARTED by the time the LLM extracts the ID. Starting the loop = calling the LLM = the LLM can now read/extract. The LLM extracting the ID IS iteration 1 of the loop. There is NO separate pre-loop extraction phase — the LLM only acts INSIDE the loop.
- Sequence Case B: loop starts with ① only → iter 1: LLM extracts ID + invokes troubleshoot_incident(12345) → compose produces ② MID-LOOP → ② appended → iter 2: LLM sees workflow, calls get_incident_metadata. His "extract → compose → back to Case A's send-everything" = correct, and that "re-send everything" is just the loop's normal append-result-and-recall rhythm (iter 2 with ② now in messages).
- Case A composes ② BEFORE the loop; Case B composes ② DURING the loop (as iter-1's result). Both converge to same state (LLM with ①+② following workflow); only the TIMING of when ② exists differs.
- prompt-vs-tool framing of troubleshoot_incident is a HOST-DESIGN CHOICE: as a "prompt" it seeds up-front (Case A); as a "tool-like call" the LLM invokes it mid-loop to fetch its own instructions (Case B). MCP doesn't care — answers "give me the prompt for 12345" before OR during the loop. Some hosts have frontend extract ID + invoke up-front (collapses B into A).

### 2026-07-09 — Prompt composition system (Day 3, real code + 7.5/8 quiz)

Read the actual prompt-composition tree. 7.5/8 on verification quiz. Notably REVERSED his earlier tool-vs-prompt confusion — now has the arrow right. Half-point off = shallow systems-lens answer. One misconception killed (see below).

**The 3 building blocks (Lego analogy):**
- FRAGMENT = reusable piece of a prompt, one section, a .md file (`compose/fragments/lsi/00-overview.md` etc.). = Lego brick. DRY for prose.
- FLOW = ordered list of fragments in a YAML recipe (`compose/flows/lsi-w365.yaml`). = instruction booklet.
- ROUTING = `routing.yaml` table mapping incident attributes → flow name. = the picker choosing which booklet. First match wins, `on_no_match: raise`.

**Tool vs Prompt (he had this BACKWARDS before, now correct):**
- Tool = fn the LLM CALLS during the loop (Day 2). Prompt = pre-written instruction template that STARTS/shapes the session. Prompt comes first and CAUSES behavior; tool calls happen during as a consequence. `troubleshoot_incident` is a PROMPT-builder, NOT a tool.
- MCP servers expose BOTH tools AND prompts (`register_tools` + `register_prompts`). `PROMPTS = [...]` list at bottom of troubleshoot_incident.py registers prompt-builders.

**YAML↔Python bridge:** YAML is just text; `import yaml` + `yaml.safe_load(path.read_text())` PARSES it into Python dict/list. After that line there's no more YAML, just normal dict access (`doc.get("extends")`). General concept = serialization/deserialization; safe_load = won't execute embedded code.

**Systems-lens "why YAML not Python if/elif" (the DEEP answer he needs to internalize):**
- Principle = separate POLICY (rules that change often: which incident→which flow) from MECHANISM (the engine that evaluates them: `_matches`, the resolve loop — rarely changes).
- Concrete benefit: change routing WITHOUT a code change → without redeploy → without eng review; non-engineers (PM/on-call) can safely edit the table; stable engine stays locked. NOT just "easier to read" (that's the symptom, not the cause).

**Division of labor (route vs compose):**
- `_routing.py`: incident_id → flow name. Knows IcM/Kusto. `resolve_flow` → `get_icm_identifier` (Kusto query to classify: type/service/team/monitor) → walk routing.yaml rules → flow name.
- `_helper.py`: flow name → prompt string. Knows NOTHING about IcM/routing. Pure composer.
- Separated = separation of concerns; either changes independently.

**MISCONCEPTION KILLED (his recurring hedge):** he kept guessing "maybe compose() only makes PART of the system prompt, other sections assembled elsewhere — division of labor?" WRONG. `compose()` output (+ the `# Troubleshoot Incident {id}` header) IS the COMPLETE system prompt. The fragments ARE all the sections (identity, workflow steps, privacy, report format all listed in the flow). Nothing else assembles it on the MCP side. `return f"# ...\n\n" + compose(flow)` = the finished prompt. (Host may wrap with conversation scaffolding, but that's host-side, not another MCP builder.)

**Flow inheritance (`extends` + `edits`):**
- = class inheritance for prompts. `extends: lsi-w365` inherits parent's step list; `edits` = child's modifications applied on top.
- `_apply_edit` (_helper.py:94-116) is the interpreter. Fixed grammar of 4 verbs: {after,insert}|{before,insert}|{replace,with}|{remove}. e.g. {after:X,insert:Y} → `steps.insert(index_of(X)+1, Y)`. YAML supplies data, Python supplies meaning. Unrecognized shape → fail loud.
- `_resolve_steps` is RECURSIVE (multi-level extends: grandparent→parent→child fully unwinds) + cycle guard raises on A→B→A.
- Problem solved: no copy-paste of 15-step flows per variant; parent improvements auto-inherited by children. DRY for whole flows.

**troubleshoot_incident(id) precise walkthrough:**
1. `resolve_flow(id)` — Kusto-classify incident, match rules → flow name. (Does NOT call get_incident_metadata — only classifies for routing.)
2. `compose(flow)` — `_resolve_steps` (extends+edits) → read each fragment .md, concatenate → COMPLETE prompt.
3. `return f"# Troubleshoot Incident {id}\n\n" + composed` — stamps the specific ID on top (fragments are generic re "target ID"; header supplies WHICH id).
4. Returns string to caller (HOST). Function ends at `return` — does NOT send to LLM or start loop.

**Full lifecycle + 3 triggers (which side of MCP↔host boundary):**
- 3 triggers, all host-side, all end at same door: (1) HUMAN interactive (portal/IDE, click/Enter), (2) SCHEDULER/cron (the `diagmcp-log-auto-fix.md` scheduled task — LIVE in his branch, wakes on timer, "last 1d"), (3) EVENT/webhook (incident fires in IcM → webhook → autonomous SRE agent = Day 5). Webhook = "reverse API, don't call us we'll call you."
- Lifecycle: incident event [host] → host invokes troubleshoot_incident [MCP] → resolve+compose [MCP] → return complete prompt [MCP→host] → host runs AGENT LOOP [host] → loop calls LLM [host] → LLM (not host!) calls first tool get_incident_metadata because the prompt told it to [LLM→MCP tool].
- `troubleshoot_incident` is TRIGGER-AGNOSTIC — doesn't know/care if human/cron/webhook drove it. Same as tools are trigger-agnostic. MCP = request-response, stateless, its side of HTTP.

### 2026-07-09 — Retrieval deep-dive (Day 2 extension: RAG internals, vector math, wrapper pattern)

Extended follow-up session drilling into HOW retrieval actually works. Damien asked relentlessly precise questions and caught two of MY imprecisions (Foundry-vs-agent-loop wording; ambiguity in chunk-vs-doc explanation). Push for concrete/precise, name patterns and files explicitly.

**Wrapper/adapter pattern (utils/search.py):**
- TWO classes named `SearchClient`: Azure's (`from azure.search.documents import SearchClient as AzureSearchClient`, line 8 — MICROSOFT's SDK, pip pkg) and the repo's own `SearchClient` (line 21 — OURS, wraps Azure's). The `as` alias is the giveaway of the wrapper pattern (avoids name clash).
- Why wrap: simpler interface (our .search takes 4 params vs Azure's ~10), encapsulate the clever vector-query setup, add uniform logging/timing/error handling, swappability (change provider → edit only this file).
- Ownership ladder: tool (ours) → SearchClient wrapper (ours) → AzureSearchClient SDK (MS) → Azure AI Search service (MS, rented). The wrapper is the SEAM between our code and Azure's.

**gRPC:** alternative to REST for service-to-service calls; binary Protobuf vs JSON, faster, not human-readable. Only mentioned as an example of a swappable transport hidden inside a client.

**Client caching (resource.py):** clients cached in a dict inside the ResourceManager singleton, keyed by config tuple e.g. `(service_name, index_name)`. Pattern = lazy singleton cache: "in dict? return it : build once, store, return." Why: building a client does an auth handshake (`get_azure_credential()` — network/token/cert), expensive to redo per call. Double-check locking = thread safety.

**Vector distance math (the retriever, stage 1):**
- Comparison is QUERY vector vs DOC-CHUNK vectors — the query is the "ground of relevance." Relevance is ALWAYS relative to the query, never abstract.
- Cosine similarity = measures ANGLE between vectors (ignores length): dot(a,b)/(|a||b|). Range -1..1: 1=same meaning, 0=unrelated, -1=opposite. Direction encodes meaning; cosine ignores magnitude so long/short docs on same topic still match.
- `k_nearest_neighbors=50` (search.py:44) = literally "top 50 nearest chunk vectors." `text=query` = embed this query; `fields="contentVector"` = compare against docs' contentVector field.

**CHUNKS not documents (key correction Damien pushed on):**
- Docs are SPLIT INTO CHUNKS at indexing time; each chunk gets its own vector. The system ranks/returns CHUNKS, not whole docs. "Document relevance" is almost a misnomer.
- Two chunks from the SAME doc score differently (relevant paragraph high, irrelevant low). No aggregation step — each chunk stands alone; winners are just the winning chunks.
- Design A (chunk-level, this repo) vs Design B (parent-doc retrieval). This repo = Design A: proven by `record["Content"] = result.get("content")` (line 59) returning chunk passages + `title` as source label, NOT whole docs. Granularity ladder: document(title) → chunk(`content`) → caption(best sentences).

**Stage 1 vs Stage 2 — the real mechanistic difference (Damien nailed this):**
- Stage 1 (retriever): query & chunk embedded SEPARATELY/individually, compared by vector distance. Fast, blunt. Squishing to one vector loses detail (misses negation: "restart" vs "do NOT restart" look identical as vectors).
- Stage 2 (reranker): query & chunk fed TOGETHER into one model, uses ATTENTION across both (every query token attends to every doc token) → one relevance number. Slow, sharp. Catches nuance/negation/"actually answers". "Cross-encoder."
- Both use the SAME query as reference — difference is METHOD (compare two summaries vs read both side-by-side), not what's compared. Blurb-match millions→50, then careful-read those 50.

**Return-value metadata (search_relevant_docs.py:61-65) mapped to stages:**
- `@search.score` = STAGE 1 blunt score (vector+keyword fused, unbounded ~0-4+).
- `@search.reranker_score` = STAGE 2 sharp joint-reading score (0-4 Azure semantic scale).
- `@search.captions` = STAGE 2 by-product (`query_caption="extractive"`, search.py:56): the best SENTENCE(s) extracted verbatim from the chunk — zooms finer than chunk. Same stage as reranker, different product (text vs number).
- Both scores surfaced mainly for debugging/telemetry (compare stage1 vs stage2 ranking).

**Context-loss = the central weakness of chunk-RAG (Damien spotted this unprompted):**
- LLM does NOT reliably backfill missing context. It CANNOT retrieve what it wasn't given — if the detail is in an unreturned chunk, it's gone, and the LLM may hallucinate to fill the gap (silent wrong answer). The LLM is NOT a safety net.
- Only legit backfill = LLM re-calls the search tool with a refined query (agent loop), but depends on it noticing the gap.
- Real fixes are in RETRIEVAL DESIGN, not the LLM: chunk overlap (repeat edge tokens), parent-document retrieval, return multiple chunks (this repo: up to 20), semantic-boundary chunking. Chunk-RAG trades context-completeness for precision + token-efficiency; that trade has a real failure mode.

**Precision fix logged:** "the agent loop invokes the tool; Foundry is the RUNTIME that runs the loop." Not "Foundry calls the tool." Agent side = LLM + loop + config (what to do/when); MCP side = tools (what can be done); meet over HTTP. Mechanical test for which side a file is on: is it under `src/mcp-servers/`? → MCP side. Everything read so far is 100% MCP side; agent-side config lives in `src/sre-agent/` (Day 5).

### 2026-07-09 — MCP server structure (Day 2, real code + 8/8 verification quiz)

Read the actual diagnostics MCP code. Scored 8/8 on final verification quiz — every gap was a missing *label* (file name, pattern name), never a broken concept. Understands in depth; can reason about pieces even without the jargon.

**MCP architecture (3 layers by job):**
- SERVER = `diagnostics-mcp.py` (~55 lines): `FastMCP("diagnostics-mcp", stateless_http=True)` + `register_tools(mcp)` + `register_prompts(mcp)`. Rest is deployment (uvicorn + auth middleware, prod-only). FastMCP = the framework that implements the tool_use/tool_result protocol so you just register plain Python fns.
- REGISTRAR = `core/tools/__init__.py`: auto-discovery via `pkgutil.iter_modules` walking the folder — drop a .py file with a public fn = it's registered, NO manual list. Guards: only registers fns defined in the module (`__module__` check), `_should_register()` opt-out gate, wraps every tool in uniform logging (decorator pattern; `query_geneva_metrics` on skip-params list).
- TOOLS = individual fns. Signature + docstring + Pydantic `Field(description=)` ARE the schema/prompt the LLM sees.

**Key concepts locked:**
- Signature-as-prompt thesis (Day 2's core): docstring is the LLM's user manual; vague/wrong → wrong tool call. Damien stated this perfectly.
- Pydantic = "type hints that actually do something" — runtime validation + auto-generates the JSON schema. `ge=1, le=20` = ≥1 and ≤20 (greater/less-than-or-equal); enforced before code runs; doubles as context-window guardrail (model can't request 500 docs).
- Separation of concerns / layering: tool assembles → `resource.py` hands a cached client → the CLIENT CLASS (e.g. `IcmClient` in utils/icm.py) does the real HTTP. Real query is 3 hops deep, not in the tool. Benefit: swap transport by editing only the client.
- ResourceManager singleton (`resource.py`): one global instance, caches expensive clients (Kusto/Search/IcM/KeyVault) keyed by config, double-check locking for thread-safety. Pattern: build expensive clients once, reuse across calls.
- RAG = a PATTERN, not a technology ("front-wheel drive, not the engine" — framing stuck hard). R=`search_relevant_docs.py`→Azure AI Search; A=`tool_result` injection (Foundry plumbing, not his code); G=LLM reply (Foundry loop). Only R is code you write. Damien derived *why RAG exists* himself: raw LLM answers from frozen/fuzzy training memory + hallucinates; feeding real source-of-truth → read not recall → accurate + citable.
- Search index = pre-built lookup structure (back-of-book-index analogy); docs chunked+embedded OFFLINE once, query embedded + nearest-neighbor lookup ONLINE (fast). `config.doc_search_index` picks per-product index.
- Azure AI Search = standalone managed service = the "R" engine, rented not built. Does embedding + vector index + reranker + keyword. Repo calls it via `SEMANTIC_VECTOR_HYBRID` (one enum) → vector + semantic-reranker + keyword all at once.
- Retriever vs reranker (2-stage funnel): retriever = fast/blunt, embeds query & doc SEPARATELY, compares vectors, wide top-N; reranker = slow/sharp, reads query+doc TOGETHER, re-scores shortlist. Why two: reranker too expensive for millions, retriever too blunt for final order. Scores visible: `@search.score` (vector) + `@search.reranker_score`. Same cheap-wide-then-expensive-sharp funnel as agent-loop cap.
- Relevance score exists BECAUSE context is finite — can't dump all docs, must rank + take top-K (`top=max_records`, ≤20). Damien connected this causal link himself.
- Statelessness recurs at every layer (LLM, MCP server, HTTP) for the same reason: push state to ONE place (Foundry holds conversation), keep rest stateless → scalable + crash-resilient + simple. `stateless_http=True`.
- Foundry ↔ MCP contract: Foundry owns LLM + agent loop (decides WHICH tool + WHEN, runs the Q4 while-loop); MCP owns tools (DOES the work). Communicate over HTTP. Foundry provides the loop — you DON'T hand-write it; you deploy the MCP server separately + register its URL in a Foundry agent config. Neither knows the other's internals.

### 2026-07-14 — LLM agent fundamentals (Day 1, quiz-format refresher)

Covered all 6 Day-1 fundamentals via quiz-then-fill-gaps. Damien answered strongly throughout and pushed past the syllabus into context-management / memory-architecture (Day 6+ territory).

**LLM core:**
- LLM = next-token predictor: text in → probability distribution over vocabulary → sample one token → append → repeat. Pure function, stateless, can't *do* anything.
- Tokens (not words) — word-pieces from a fixed ~100k vocab; billing + context measured in tokens. Code/UUIDs tokenize inefficiently.
- Embeddings — each token → vector of thousands of numbers; meaning = geometry (king-man+woman≈queen). Same mechanism as `search_relevant_docs`.
- Attention (Transformer) — every token attends to every other token, weighted by relevance; source of both power and the n² cost limit.
- Temperature — sampling randomness (0 = always top token).

**Agents:**
- Why agents exist — bridge raw LLM's 4 gaps: frozen knowledge, no live data, can't take actions, hallucination. Agent = LLM decides, agent code does.
- Tool use = 4-step structured conversation: (1) describe tools as JSON schemas in context, (2) model emits `tool_use` block (decides tool+args, then STOPS — never executes), (3) agent/MCP runs the real code, (4) result appended as `tool_result` message + re-sent. MCP = registry + transport (name→code).
- Agent loop — agent runs the real `while` loop; re-sends full conversation each iteration. Stop condition = the model's output *type*: `stop_reason: tool_use` → loop; `end_turn` (plain text) → stop. Plus a max-iterations safety cap.

**System prompt + roles (Damien's own question):**
- System prompt = fixed, authoritative behavior rules (cacheable, reusable); user message = per-turn variable input. Separated for priority/trust, stability-vs-variability, reusability.
- How the model knows system vs user: roles are *tagged* (`role:` field), *flattened into special delimiter tokens* (`<|system|>`/`<|user|>`), and the model *learned during training* to weight them differently. Real but learned wall — why prompt injection is possible.

**Context window + statelessness (Damien reached the frontier):**
- 1M context = HARD wall (API rejects on overflow, not auto-shrink). Counts system+history+tool results. "Memory" = agent re-sends full message list every call; model re-reads from scratch (amnesia + notebook).
- Overflow strategies are *agent code's* job: truncation (lossy), summarization (extra LLM call, lossy), RAG/retrieval.
- Why context can't just grow infinitely: attention is O(n²) — doubling context quadruples compute. That's the hard limit Damien intuited. Sparse/linear attention pushes it but trades quality.
- Damien independently re-derived RAG ("keep memory server-side, curate, pull relevant slice") — the external-memory pattern. Resolves the paradox: external store grows infinitely (cheap DB); context sent stays small (relevant slice only). This IS how his own MEMORY.md system works.
- Brainstorm "summarize a life into tokens" → answer: tiered self-consolidating memory (episodic → semantic → identity), background "sleep" consolidation, retrieval by relevance+recency+importance, forgetting as the core algorithm not a bug. A scaled-up version of the session's own memory system.

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
