# ARCHITECTURE — Autonomous Multi-Party Negotiation & Settlement Platform

Status: DESIGNED (Chat 1). Nothing here has been executed or verified. Diagram: `docs/architecture.png` (and `.jpg`). STATUS UPDATE: implemented and tested, see PROJECT_HANDOFF.md and docs/BENCHMARK_REPORT.md.

## 1. Key decisions and trade-offs

| # | Decision | Why | Trade-off |
|---|----------|-----|-----------|
| D1 | All binding logic lives in PostgreSQL functions (schema `api`, SECURITY DEFINER). n8n only orchestrates and calls LLMs. | Invariants must hold even if n8n or an LLM is compromised. | More SQL; n8n logic stays thin. |
| D2 | LLM agents are plain Groq chat calls with **no tools and no DB access**. n8n builds each agent's input from `api.get_agent_view(session, role)`. | "Agent cannot query the opponent's mandate" becomes structurally true, not a prompt promise. | Agents cannot self-serve data; orchestrator must feed it. |
| D3 | n8n connects via a dedicated Postgres role `n8n_runtime` that has **EXECUTE on `api.*` only** (no table rights). Never the service-role key. | Least privilege; compromised workflow still cannot read `private.*`. | Needs direct/pooler Postgres connection in n8n (not PostgREST). |
| D4 | One parameterised sub-workflow `WF_Party_Agent` (role = BUYER/SELLER) with **two separate prompt templates and configs** in `core.agent_configs`. | Independent prompts/objectives without duplicating plumbing. | Both agents share workflow code (not prompts). |
| D5 | Mandates are **per session and per party** (`private.mandates`, unique `(session_id, role)`), frozen at session creation. | One seller in many sessions never shares state or mandates. | Mandate data repeated per session. |
| D6 | Money as integer minor units (`price_per_unit_minor bigint`) + fixed session currency. | No float errors; no currency-swap attacks. | Conversion at the edges. |
| D7 | Only 5 terms are negotiable: price, quantity, delivery_date, payment_terms_days, quality_grade. Product, unit, currency are fixed public session context. | Small allowlisted attack surface. | Less expressive contracts (sufficient for B2B demo). |
| D8 | Per-session row lock (`SELECT … FOR UPDATE` on `core.negotiation_sessions`), READ COMMITTED, plus UNIQUE constraints as backstop. | Narrow scope: sessions of the same seller never block each other. | Requires every mutating function to take the lock first (rule L1). |
| D9 | Arbiter LLM sees **public data only**. It proposes terms; the DB validates against both mandates and returns a generic accept/reject. Max 3 proposals, then human review. | Arbiter needs no confidential data. | Residual leakage ≤ 3 binary signals (documented risk R2). |
| D10 | Benchmark has two modes: DETERMINISTIC (scripted agents/offers, pure DB + orchestrator logic) and LLM (real Groq agents). Reported separately. | Reproducibility where possible; honesty about variance. | Two runner paths. |
| D11 | Groq model names are config (`core.agent_configs.model`), selected and verified against current Groq docs in Chat 3. | Models change; avoid hard-coding stale names. | Decision deferred. |

## 2. Components

**Negotiation Orchestrator (`WF_Negotiation_Orchestrator`, n8n).** Webhook (header-auth) → `api.create_session` → loop: `api.next_turn` (DB decides whose turn / whether to stop) → `api.get_agent_view` → call `WF_Party_Agent` → `api.submit_action` → `api.evaluate_progress` → continue / escalate / settle. The loop count is bounded by the DB (`max_rounds`), n8n loop is a safety-capped (max_rounds+2) iterator. n8n never decides acceptance validity.

**Buyer Agent / Seller Agent (`WF_Party_Agent`).** Input: own mandate (from DB view), public context, current offer, sanitised history. Output: JSON `{action, terms, accept_offer_id, message, rationale}` validated by schema before submission. Opposing objectives: buyer minimises price/long payment terms; seller maximises price/short payment terms.

**Deterministic Mandate Enforcement (`api.submit_action` → `private.check_mandate`).** Re-reads the authoritative mandate from `private.mandates` inside the transaction; the agent supplies no authorization values. Checks every OFFER (proposer's own mandate) and every ACCEPT (acceptor's mandate against the exact stored terms).

**Arbiter (`WF_Arbiter`, n8n + `api.submit_arbiter_proposal`).** Triggered by DB-decided escalation reasons. Reads public offer history only. Proposal stored as offer by role ARBITER, validated against both mandates; becomes binding only if both parties then ACCEPT through the normal path.

**Settlement Engine (`api.settle_session`, internal, called inside `submit_action` transaction).** Verifies two acceptances with identical `terms_hash`, rechecks both mandates, inserts one immutable `core.settlements` row. No LLM involved.

**Benchmark Harness (`WF_Benchmark_Runner` + `bench.*` tables).** Loads scenarios, drives sessions through the same `api.*` functions, checks invariants and outcome ranges, writes `bench.results`.

## 3. Data flow (happy path)

1. Business system POSTs `create_session` (mandates come from this trusted call, never from an LLM) → `api.create_session` writes `core.negotiation_sessions`, `private.mandates` ×2, audit event. State `ACTIVE`.
2. Orchestrator calls `api.next_turn(session)` → returns `{role, round_no, allowed_actions}` or a terminal/escalation instruction.
3. Orchestrator calls `api.get_agent_view(session, role)` → own mandate + public context + current offer + sanitised history (counterparty `message` wrapped as untrusted, rationales excluded).
4. `WF_Party_Agent` calls Groq; output validated against the action schema (n8n) — then **re-validated in SQL**.
5. `api.submit_action(session, role, request_id, action_json)`: lock session row → idempotency check → state/turn/ownership checks → schema/allowlist validation → mandate check → insert offer/acceptance → (if ACCEPT valid) settle → audit event. Any failure rolls back everything.
6. `api.evaluate_progress(session)` classifies CONVERGING / STUCK / OSCILLATING / ROUND_LIMIT using public data only; DB moves session to `ARBITRATION` when needed.

## 4. Trust boundaries

| Zone | Trusted? | Can reach |
|------|----------|-----------|
| Business API caller (create_session) | Authenticated (header secret) | `api.create_session` only |
| n8n workflows | Semi-trusted code | `api.*` via `n8n_runtime` |
| Groq LLM output | **Untrusted** | Nothing directly; only through schema validation + `api.submit_action` |
| Counterparty `message` text | **Untrusted** | Shown to other agent only inside a data-delimited, length-bounded field; never in system prompt |
| PostgreSQL `api`/`private`/`core` | Trusted enforcement | — |

## 5. Data contracts

**Terms (negotiable, canonical key order, used for `terms_hash = sha256(canonical_json)`):**
```json
{
  "price_per_unit_minor": 4200,
  "quantity": 500,
  "delivery_date": "2026-12-15",
  "payment_terms_days": 30,
  "quality_grade": "A"
}
```
Rules: integers only for money/quantity; `delivery_date` ISO date ≥ session start; `payment_terms_days ∈ {0,15,30,45,60}`; `quality_grade ∈ {A,B,C}`; unknown keys → reject + security event.

**Fixed session context (public):** `product_sku`, `unit`, `currency ∈ {USD,EUR,BDT}`, `max_rounds` (default 10), `max_invalid_attempts` (default 2), progress parameters.

**Mandate (private, per session/party):**
- BUYER: `max_unit_price_minor`, `min_quantity`, `max_quantity`, `latest_delivery_date`, `min_payment_terms_days`, `acceptable_grades[]`.
- SELLER: `min_unit_price_minor`, `min_quantity`, `max_quantity`, `earliest_delivery_date`, `max_payment_terms_days`, `offered_grades[]`.

**Agent action (LLM output, untrusted):**
```json
{"action":"OFFER|ACCEPT|REJECT|WALK_AWAY",
 "terms":{...} | null,
 "accept_offer_id":"uuid | null",
 "message":"<=280 chars, public",
 "rationale":"<=500 chars, never shown to counterparty"}
```
`request_id` is generated by the orchestrator (not the model). Binding data = `terms` only; `message`/`rationale` are never used for decisions.

## 6. Session state machine (enforced in SQL)

`CREATED → ACTIVE → SETTLED`
`ACTIVE → ARBITRATION → (ACTIVE_ARB_ACCEPT) → SETTLED | HUMAN_REVIEW | FAILED_NO_DEAL`
`ACTIVE → HUMAN_REVIEW` (security event / invalid responses), `ACTIVE → FAILED_NO_DEAL` (WALK_AWAY)
`CREATED|ACTIVE|ARBITRATION → CANCELLED` (operator). Terminal: SETTLED, FAILED_NO_DEAL, CANCELLED; HUMAN_REVIEW exits only via operator function. Any other transition raises an exception. During arbitration, the only allowed actions are ACCEPT/REJECT of the arbiter proposal.

Escalation triggers (DB-decided): round limit; same `terms_hash` repeated ≥2 times by one party; A-B-A oscillation; gap not shrunk by ≥`min_progress_bps` over last 3 rounds; ≥`max_invalid_attempts` invalid actions in a turn; missing mandatory terms; any CRITICAL security event.

## 7. Critical invariants

| ID | Invariant | Enforced by | Planned test |
|----|-----------|-------------|--------------|
| INV-01 | Buyer never bound at price > max; seller never at price < min; same for quantity, delivery, payment, grade | `private.check_mandate` inside `submit_action` | Boundary tests at min/max ±1 |
| INV-02 | Mandates read only from `private.mandates`; no function parameter carries authorization values | Function signatures; code review | Negative test: extra keys rejected |
| INV-03 | A party cannot act for the other (role derived from session participant, turn-checked) | `submit_action` ownership + turn check | Wrong-role/wrong-turn tests |
| INV-04 | Action valid only in permitted state | State machine function | Tests per state |
| INV-05 | Settlement requires two acceptances with identical `terms_hash`, referencing the *current* offer | `settle_session` + UNIQUE constraints | Stale/hash-mismatch tests |
| INV-06 | Rejected operation leaves no binding rows | Single transaction; exceptions roll back | Row-count assertions |
| INV-07 | One settlement per session | `UNIQUE(session_id)` + session lock | Double-accept race test |
| INV-08 | Arbiter proposal binding only after both ACCEPT, each re-checked | Same `submit_action` path | Arbiter out-of-mandate test |
| INV-09 | Session data (offers, mandates, state) isolated per session | All functions keyed by `session_id`; composite FKs | Same-seller/different-buyer test |
| INV-10 | Duplicate `request_id` returns the stored result, causes no second effect; same id with different payload → conflict | `core.idempotency_keys` | Duplicate webhook tests |
| INV-11 | Settlements and audit events are immutable | Triggers blocking UPDATE/DELETE, no grants | Update/delete attempts fail |
| INV-12 | Opponent mandate unreachable by agents/n8n_runtime (tables), error messages, logs | No tools for LLM; role grants; generic errors; redaction | Role-privilege test, error-text scan |
| INV-13 | Round count ≤ `max_rounds`, enforced in DB | `next_turn`/`submit_action` | Round-limit test |

## 8. Authoritative source for each binding decision

| Decision | Authority |
|----------|-----------|
| Party's limits | `private.mandates` row (from trusted `create_session`) |
| Whether an offer/acceptance is allowed | `api.submit_action` (`check_mandate`, state, turn) |
| Whose turn / round number | `core.negotiation_sessions` (DB) |
| Exact contract terms | `core.offers.terms` + `terms_hash` |
| Whether both parties agreed | `core.acceptances` (same hash) |
| Settlement | `core.settlements` (created only by `settle_session`) |
| Escalation / stuck | `api.evaluate_progress` (SQL, public data) |
| Arbiter outcome validity | Same mandate check; Arbiter text is advisory |
| Duplicate detection | `core.idempotency_keys` |
| Audit truth | `core.audit_events` (DB-written), not LLM explanations |

## 9. Concurrency and idempotency

- **L1:** every mutating `api` function first runs `SELECT … FROM core.negotiation_sessions WHERE id=$1 FOR UPDATE`. No other lock is taken (no deadlocks; sessions never block each other).
- Isolation: READ COMMITTED + the row lock; UNIQUE constraints (`settlements.session_id`, `offers(session_id, round_no)`, `acceptances(session_id, role)`) as second line of defence.
- Idempotency key = `(session_id, role, request_id)`; stores `request_hash` and response JSON. Replays return the stored response. n8n retries reuse the same `request_id`.
- Conflict handling: lock waits use `lock_timeout` (e.g. 5 s); timeout returns a retryable error.

## 10. Logging and confidentiality

- `core.audit_events` payloads contain only public terms, decisions, check *outcomes* (e.g. `PRICE_ABOVE_MAX` without the limit value), hashes, latency, model/token metadata.
- Agent `rationale` stored in `private.offer_rationales` (not in audit payloads, not in counterparty view).
- Self-leak scan: if `message`/`rationale` contains the proposer's own exact mandate values, message is replaced by `[redacted]` and a `SELF_LEAK_ATTEMPT` security event is logged.
- Rejection errors to an agent name only its own violated field category; no opposing information.
- n8n execution data may contain the agent's *own* mandate: set `WF_Party_Agent` to not save successful-execution data; document that n8n access = confidential access (R3).

## 11. Residual risks (to be tracked)

- R1: Paraphrased leaks ("around forty dollars") are not caught by the exact-match scanner.
- R2: Arbiter feedback loop leaks ≤3 bits; mitigated by cap and generic error.
- R3: n8n execution logs and Groq (third-party processor) see each party's own mandate in model input.
- R4: Counterparty pressure can still push a model toward the edge of its own mandate (economically poor but allowed). Benchmark measures this; it is not a safety violation.

## 12. Consistency review (done by design inspection)

- Arbiter has no mandate access (D9) yet proposals must satisfy mandates → resolved via DB validation with generic feedback.
- "Agents cannot query DB" vs "agents need own mandate" → resolved: orchestrator supplies only own mandate through `get_agent_view`.
- Seller reused across sessions vs isolation → mandates keyed by session (D5), locks keyed by session (D8).
- Settlement without LLM vs "both parties accept the same terms" → proposer's OFFER counts as implicit acceptance (mandate-checked at proposal), counterparty ACCEPT references `offer_id` + `terms_hash`.
- Orchestrator loop vs "LLM must not control loop" → DB `next_turn` is the only loop authority.
