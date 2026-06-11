---
name: learning-log
description: Tracks concepts Damien has learned session by session — use to avoid re-explaining and to build on prior knowledge
metadata: 
  node_type: memory
  type: user
  originSessionId: c61ec033-bfa6-4006-aa1a-140a30fa0a24
---

## Concepts Learned

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
