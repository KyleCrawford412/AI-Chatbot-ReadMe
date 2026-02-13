# Engineering Decisions (Senior-Level Notes)

This doc is the "why" behind implementation choices. The prototype is intentionally scoped, but each choice maps cleanly to production paths.

## Decision 1: Split API and AI into separate services

**Chosen**
- `apps/api` for auth, transport, governance
- `apps/ai-service` for LangChain orchestration

**Why**
- Keeps security-critical logic (auth/tokens/rate limits) separate from model logic.
- Lets us scale AI workers independently from API/websocket workload.
- Makes it easier to swap providers/prompts without touching auth surfaces.

**Alternative considered**
- Single Node service with embedded AI client.

**Tradeoff**
- More operational overhead now (two services), much cleaner boundaries later.

## Decision 2: Socket auth via short-lived chatbot token

**Chosen**
- Frontend calls `POST /api/chatbot/token` after login.
- Socket handshake uses that short-lived token, not the access JWT.

**Why**
- Narrow scope + short TTL is safer if a transport token leaks.
- Decouples login session concerns from chat session concerns.

**Alternative considered**
- Reuse same access token everywhere.

**Tradeoff**
- One extra API call, better token hygiene.

## Decision 3: Deterministic availability simulation

**Chosen**
- `scheduler.py` implements deterministic slot logic for this prototype.

**Why**
- Demo reliability matters in assessments.
- Avoids external calendar dependency failures during evaluation.
- Preserves a clean interface to later replace with real provider integration.

**Alternative considered**
- Direct Google/Microsoft calendar integration.

**Tradeoff**
- Less realism now, more predictable demos and easier grading.

## Decision 4: Hybrid extraction (regex + prompted reply)

**Chosen**
- Regex for core booking slots (`service`, `date`, `time`, `contact_name`).
- LangChain prompt for conversational response generation.

**Why**
- Deterministic extraction reduces model drift on required fields.
- Prompted replies keep UX natural in multi-turn conversations.
- Fallback path works even when no provider key is configured.

**Alternative considered**
- Fully LLM-only extraction with schema parsing.

**Tradeoff**
- Regex is narrower linguistically, but much easier to reason about in prototype scope.

## Decision 5: Persist transcripts in backend DB

**Chosen**
- API appends session transcript snapshots to `chat_sessions`.

**Why**
- Enables debugging, analytics, and post-hoc conversation review.
- Keeps persistence out of frontend concerns.
- Gives a simple base for later compliance workflows.

**Alternative considered**
- Keep chat state only in browser/AI memory.

**Tradeoff**
- Slightly more write overhead, much better observability.

## Decision 6: Data model optimized for prototype-to-product path

**Chosen tables**
- `users`
- `appointments`
- `chat_sessions`

**Why**
- Normalized enough for SaaS growth.
- Indexes cover expected reads (auth lookup, appointment timeline, session activity).
- Optional `business_id` included for multi-tenant direction.

## Operational Notes

- Current AI memory is in-process; restart clears active memory.
- The DB remains source of truth for appointments and chat transcripts.
- Environment settings are intentionally explicit in `.env.example` for easier reviewer setup.

## If This Was Moving To Production

1. Move AI memory/state to Redis or Postgres-backed store.
2. Add retries + timeout budgets between API and AI.
3. Add typed contract validation between services.
4. Add metrics dashboards and distributed tracing.
5. Add stronger auth lifecycle (refresh token rotation and revocation).
