---
name: stateful-stateless-patterns
description: Deep-dive on stateful vs stateless API calls — protocol, endpoint, storage layers; conversationId vs sessionId; Claude Code's own architecture
metadata:
  type: reference
  origin: 2026-06-15 session — Copilot.tsx agentConversationId selection
---

# Stateful vs Stateless in API Calls — A Deep Dive

> Anchored on a line from `src/portal/frontend/src/pages/Copilot/Copilot.tsx:409`:
> ```typescript
> await agentChat.sendMessage(question, agentConversationId, activeTab.key, currentSessionId);
> ```
> This single line encodes nearly every concept below.

## TL;DR

- **Stateless** = each call is self-contained; server forgets you the moment it responds.
- **Stateful** = somewhere remembers what happened before so the next call can build on it.
- Most modern systems are **stateless at the protocol level** (HTTP, REST APIs) but **simulate statefulness** by passing IDs around (session IDs, conversation IDs, JWT tokens).
- `agentConversationId` is exactly this pattern: a handle to server-stored LLM context.

## The mental model: who remembers what?

Every interaction has a question: **who's holding the memory?**

```
┌──────────┐                    ┌──────────┐
│  Client  │ ──── request ────► │  Server  │
│ (browser)│                    │ (backend)│
│          │ ◄─── response ──── │          │
└──────────┘                    └──────────┘
```

For any piece of information that matters across multiple calls, one of three things must be true:
1. **Client remembers it** and sends it each time → server is stateless
2. **Server remembers it** keyed by some ID the client sends → server is stateful
3. **Nobody remembers** → it's lost (only valid for truly one-shot ops)

## 4 layers to separate

### Layer 1: Protocol (HTTP itself)

**HTTP is stateless by design.** Every request is independent. The server doesn't know request #2 came from the same person as request #1 unless the client *tells* it (cookie, auth header, etc.).

```
Request 1: GET /messages          → Server: "Who are you? Here's all messages."
Request 2: GET /messages/new      → Server: "Who are you? What's 'new' mean to you?"
```

The server has zero memory of Request 1 when Request 2 arrives. This is the foundation everything else is built on.

### Layer 2: Server-side endpoint design

#### Stateless endpoint
Request body/params contain **everything** needed to compute the response.

```http
POST /api/translate
{ "text": "Hello", "from": "en", "to": "fr" }
→ { "translated": "Bonjour" }
```

Call it 1000 times in any order, the result for the same input is identical. Functional purity at the API layer.

#### Stateful endpoint
The request references **server-stored context** by ID.

```http
POST /api/agent/chat
{ "conversationId": "abc-123", "message": "And what about my second question?" }
→ { "reply": "About React hooks, the answer is..." }
```

The server looks up `abc-123` and the response depends on that history. Same input + different `conversationId` = different response.

### Layer 3: Where does the state actually live?

#### A) Server memory (process-local) — fragile
```python
conversations = {}  # in-memory dict

@app.post("/chat")
def chat(req):
    conversations[req.conversationId].append(req.message)
    return generate_response(conversations[req.conversationId])
```

**Why this breaks**: Modern backends run as multiple instances behind a load balancer. Request #1 hits Instance A; Request #2 might hit Instance B, which has no memory of you. Also: instances restart, get autoscaled away. #1 "works on my machine" bug.

#### B) Shared database (Cosmos, Redis, Postgres) — robust
```python
@app.post("/chat")
def chat(req):
    convo = cosmos.get(req.conversationId)
    convo.messages.append({"role": "user", "content": req.message})

    response = generate_response(convo.messages)

    convo.messages.append({"role": "assistant", "content": response})  # save response too
    convo.updatedAt = now()
    cosmos.save(convo)

    return response
```

Any instance can serve the request because state lives in a shared external store. The Copilot portal uses this pattern with Cosmos DB.

Note the commit `perf(portal): defer Cosmos session write off the agent streaming critical path` — that's this pattern with an optimization: write to DB happens AFTER streaming finishes, so user sees tokens immediately.

#### C) Client-side (request itself carries the state) — server stays pure
```http
POST /api/chat
{
  "message": "And what about React?",
  "history": [
    { "role": "user", "content": "Tell me about hooks" },
    { "role": "assistant", "content": "Hooks are functions that..." }
  ]
}
```

Server stores **nothing**. Client sends entire conversation each time. Fully stateless — pure function `(history, new_message) → reply`.

### Layer 4: Why your code has BOTH patterns

Look at `Copilot.tsx`:
- `agentChat.sendMessage(question, agentConversationId, ...)` — only sends an **ID**, server looks up history
- `askChat.sendMessage(question, messages, ...)` — sends entire **history array**

#### Why "agent" is server-stateful (conversationId only)
- Long, complex conversations (many turns)
- Need to remember tool calls, intermediate reasoning, file references
- Backend manages tool execution context, sandbox state, citations
- Sending full history every turn would mean huge payloads, wasted bandwidth, consistency problems

#### Why "ask" is client-stateful (full messages)
- Shorter Q&A conversations
- Smaller payloads acceptable
- Lets client control history (edit, retry, branch)
- Server stays simple: pure function, no storage needed
- Cheaper to scale (no DB writes per turn)

This is a real engineering tradeoff and you see both halves of it 10 lines apart in the same file.

## The "conversationId" pattern lifecycle

```
Turn 1 (new conversation):
  Client → POST /agent { message: "Hi", conversationId: null }
  Server → creates conversation, stores it, returns { conversationId: "abc-123", reply: "Hello!" }
  Client → saves "abc-123" via dispatch({ type: "SET_AGENT_CONVERSATION_ID" })

Turn 2 (continuing):
  Client → POST /agent { message: "What's React?", conversationId: "abc-123" }
  Server → looks up "abc-123", finds prior turns, generates context-aware reply

Turn 3:
  Client → POST /agent { message: "And hooks?", conversationId: "abc-123" }
  Server → "abc-123" history now has 4 turns, replies coherently
```

The `conversationId` is a **handle** — like a coat check ticket. You don't carry the coat; you carry the number that points to the coat. The server holds the coat.

In `Copilot.reducer.ts`:

```typescript
case "SET_AGENT_CONVERSATION_ID":
    return { ...state, agentConversationId: action.payload };
```

This fires AFTER the first turn's response — server says "here's your handle," client saves it. Every subsequent send includes it.

```typescript
case "NEW_CHAT":
    return {
        ...state,
        // ...
        askConversationId: null,
        agentConversationId: null,  // throw away the handle
        // ...
    };
```

`NEW_CHAT` clears it. Next message → `conversationId: null` → server creates a brand new conversation. Clean slate.

## Concrete stateless / stateful examples

### Stateless

| Example | Why |
|---|---|
| `GET /api/weather?city=Seattle` | Pure function of query params |
| `POST /api/translate` with text in body | Self-contained input |
| `GET /api/users/123` | ID in URL, no hidden context |
| DNS lookup | Same query → same IP (mostly) |
| LLM `messages.create` API | Send full history each time; server forgets |

### Stateful

| Example | What's held | Where |
|---|---|---|
| Logged-in session | Who you are | Server session store OR signed cookie |
| Shopping cart | Items added | DB keyed by user/session ID |
| `agentConversationId` | Prior chat turns + agent context | Cosmos |
| Database transaction | Pending uncommitted writes | DB connection |
| Bank account balance | Money you have | Bank DB |
| TCP connection | Sequence numbers, window | Both client and server kernel |

### Sneaky middle cases

**JWT tokens (looks stateful, often stateless):**
```
Authorization: Bearer eyJhbGc...
```
Token CONTAINS user ID + permissions, cryptographically signed. Server verifies signature — no lookup needed. Each request is self-contained. **Stateless auth.**

**Cookies (looks stateless, often stateful):**
```
Cookie: sessionid=abc123
```
Just an ID. Server has `sessions` table mapping `abc123` → user data. **Stateful auth.**

**HTTP/2 connection reuse (sneakily stateful at network layer):**
HTTP is stateless but underlying TCP/TLS connection is reused for performance. Hidden state at a lower layer.

## Why this matters in practice

1. **Scaling** — Stateless servers scale horizontally trivially. Stateful servers need sticky sessions or shared storage.
2. **Debugging** — Stateless calls reproduce by replaying the exact request. Stateful calls require recreating prior state.
3. **Caching** — Stateless responses cache aggressively (CDN). Stateful responses usually can't.
4. **Failure handling** — Stateless fail: just retry. Stateful fail: may have partial state (half-written conversation, half-charged credit card).
5. **Cost** — LLM stateful agents are easier on bandwidth, expensive on storage. Stateless LLM APIs mean prompts grow linearly with conversation length — every turn re-sends and re-tokenizes everything.

## Q&A from the session

### Q: Do we save the response too, or only the user message?

**Yes, almost always.** Why:
- Next turn needs response as context — Turn 3 needs to see what assistant said at Turn 2
- History display on refresh
- Audit/compliance (legal, debugging, abuse review)
- Analytics (response quality, token cost, user satisfaction)
- Possibly training data (anonymized)

**Exceptions:**
- Truly ephemeral chat modes (privacy is the feature)
- Streaming optimization tricks (save after streaming finishes, not during)
- Some regulated industries forbidding storage of certain content

### Q: If client loses conversationId, is server state wasted?

**Yes — orphaned data.** Solutions, layered:

1. **URL param** (e.g., `?session=abc-123` — Copilot does this via `navigateChat`) — survives refresh, bookmarkable, shareable
2. **localStorage / IndexedDB** — survives refresh + closed tab, tied to browser profile
3. **Server-side index keyed by user** — strongest fix. On login, fetch "all my conversations." This is the sidebar conversation list. NOT just a UI feature — it's the recovery mechanism.

Despite safety nets, true orphans happen (anonymous users, bugs). Cleanup via TTL: "delete conversations not accessed in 90 days," "delete anonymous sessions after 7 days." Trades off storage cost vs user expectation.

### Q: Is Claude Code (myself) stateful or stateless?

**The Anthropic API is fully stateless; the local harness is stateful.**

Per turn, the harness:
1. Collects: full conversation history + new user message + system prompt + tool definitions + memory files
2. Sends ALL to API (stateless call)
3. API returns response, forgets everything
4. Harness stores response in conversation history for next turn

Repo content is NOT auto-sent. Only:
- Initial summary: git status, recent commits, tree info, CLAUDE.md, memory
- Files I `Read` (content comes back as tool result, enters context)
- Search hits from `Grep` (only matches, not whole files)

Context window size matters — `claude-opus-4-8[1m]` gives 1M tokens (~750k words), enough for a small-to-medium codebase.

**Cost implications:**
- Turn 50 of a long session sends ~50× the data of Turn 1
- Prompt caching makes re-sent prefixes cheaper (cached prefix isn't re-tokenized)
- Auto-compact summarizes old turns when context fills

This is "Layer 3-C" in action: stateless server, stateful client, conversation length costs money.

### Q: agentConversationId vs currentSessionId — what's the difference?

**They live at different abstraction layers.**

```
┌─────────────────────────────────────────────────────────┐
│ Session (user-facing concept)                            │
│ "My conversation about the Cosmos refactor"              │
│ ID: sessionId = "sess-abc-123"                           │
│                                                          │
│   ┌─────────────────────────────────────────────────┐  │
│   │ AgentConversation (LLM-facing concept)          │  │
│   │ The actual back-and-forth turns with the agent   │  │
│   │ ID: agentConversationId = "conv-xyz-789"        │  │
│   └─────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

- **Session** = user-visible container. Sidebar entry "My chat about React hooks." Owns title, pinned status, creation timestamp.
- **AgentConversation** = LLM runtime context. The messages the agent remembers, tool call history, intermediate reasoning.

#### Why split them?

If they were the same thing, these common ops would be impossible:
1. Rename a session — metadata change shouldn't touch LLM context
2. Switch agents mid-session — same session, different agent runtime (`SET_TAB` in Copilot.reducer.ts clears `agentConversationId` but not `currentSessionId`)
3. Reload from sidebar — fetch messages by sessionId, server attaches a fresh conversationId
4. Pin/unpin — metadata only
5. List all sessions — query sessions collection, don't load LLM state
6. Cost accounting — sessions can group multiple conversations

#### `SET_TAB` proof in Copilot.reducer.ts

```typescript
case "SET_TAB":
    return {
        ...state,
        activeTab: action.payload,
        agentConversationId: null,   // reset LLM context
        isPromptListExpanded: false,
    };
```

Switching tabs (products) clears `agentConversationId` but not `currentSessionId`. The user-facing session is still the same one, but the agent has changed — different "minds" can't share a conversation.

#### `LOAD_SESSION` proof

```typescript
case "LOAD_SESSION":
    if (state.currentSessionId !== action.payload.sessionId) {
        return state;
    }
    return {
        ...state,
        messages: action.payload.messages,
        agentConversationId: action.payload.conversationId ?? null,  // from server
        // ...
    };
```

Server returns BOTH:
- `messages` — rendered chat history (display)
- `conversationId` — LLM context handle (continuing)

#### Persistence requirements for session

1. **Across refresh** — URL param (`?session=...`)
2. **Across login sessions** — Cosmos store keyed by userId
3. **Across devices** — same Cosmos store, queryable from any device
4. **Across agent restarts** — Cosmos again
5. **For audit** — indefinite retention with access controls

`agentConversationId` could be ephemeral (regenerated per resume) — but `sessionId` MUST be persistent because it's the user's mental anchor for their work.

#### The killer insight

**A single session can have multiple agent conversations over time.**

- Monday: session "Cosmos refactor", sessionId=`sess-abc`, agentConversationId=`conv-001` (Agent v1.0)
- Tuesday: deploy Agent v1.1. User reopens session.
  - sessionId=`sess-abc` (same — user sees same thread)
  - agentConversationId=`conv-002` (NEW — old agent context gone, fresh start)
- Yesterday's messages display for context, but agent's "live memory" is fresh.

Or: user switches Agent → Ask mode mid-session. Same sessionId, no agentConversationId (Ask is stateless on server).

Same principle as a Slack channel — fixed identity even though the server process handling it changes.

## Mental checklist for any API call

1. **Does response depend on prior calls?** If yes → stateful, identify how state is referenced (ID? cookie? token?).
2. **Where is that state stored?** Server memory (fragile), DB (robust, costly), or in each request (no server burden but bigger payloads)?
3. **What happens on retry?** Stateless → safe. Stateful → must be idempotent or risk double-effects (e.g., "charge card").
4. **What happens if server forgets?** Stateful → conversation lost; need fallback. Stateless → nothing to lose.
5. **What happens if client forgets?** Stateful with server-held state → user clears cookies, conversation orphaned. Stateless → user starts over.

## The Copilot one-liner, fully decoded

```typescript
await agentChat.sendMessage(question, agentConversationId, activeTab.key, currentSessionId);
```

- `question` — new content (always per-call)
- `agentConversationId` — handle to **server-side LLM context** (stateful at server)
- `activeTab.key` — which agent/product to route to (stateless metadata)
- `currentSessionId` — handle to **server-side persistent session** (stateful at server)

Two different state handles, both stored in client React state, both pointing to server-side records, with carefully designed lifecycle (new chat clears them, loading a session restores them).

Once you see this pattern in one codebase, you see it in every chat product, every SaaS app, every game backend: **statelessness at the protocol, statefulness via handles, persistence in shared stores.** That's the entire architecture of modern web backends.
