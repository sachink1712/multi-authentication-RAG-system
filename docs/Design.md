# Multi-Authenticator, Access-Controlled RAG Chatbot — Design Doc

Status: Draft v0.1 · Stack: Python 3.11+, FastAPI, LangGraph (PostgresSaver + Store), Postgres + pgvector, Langfuse, Pydantic v2, pytest, Docker Compose, GitHub Actions

## 1. Decisions at a glance

- **Authorization lives in Postgres, not in the LLM.** RLS (FORCE) on every tenant table, plus ACL predicates inside the retrieval SQL. No post-filtering anywhere.
- **One `Principal`, built only from a verified credential.** Frozen Pydantic model; never read from request body, headers like `X-User-Id`, prompts, or memory.
- **Graph state holds IDs, not content.** Checkpoints store chunk IDs and citations; content is re-fetched under RLS. A checkpoint never becomes an ACL bypass.
- **Threads are owned.** `thread_id` is bound to `(tenant_id, user_id)` in our own table and verified before every invoke (prevents IDOR on checkpoints).
- **Abstain by default under weak evidence.** Threshold calibrated on the golden set, not guessed.
- **Every gate is testable.** ACL leakage = 0 is a hard CI blocker.

## 2. Architecture

```
Client → FastAPI (authn middleware → Principal) → thread-ownership check
  → LangGraph (PostgresSaver) per-request config {principal, request_id}
      authenticate → guardrail_in → load_memory → query_analyzer
        → [rewrite | clarify | decompose | pass]
        → retrieve → rerank → generate → guardrail_out → write_memory → respond
  → Postgres (RLS) · pgvector · LangGraph Store · audit_log
  → Langfuse (1 trace/request, span/node)
```

Principal is injected server-side into `config["configurable"]`; clients cannot supply configurable keys. Each DB transaction runs `SELECT set_config('app.tenant_id', $1, true)` (and user, roles) so settings are transaction-local and cannot leak across pooled connections.

## 3. Identity & Access Control

**Authenticators (adapters → `Principal`)**

| Adapter | Verification | Notes |
| --- | --- | --- |
| OIDC/SSO | JWKS fetch + cache, check `iss`, `aud`, `exp`, `nbf`, signature alg allow-list | Claims → roles/tenant via config-driven mapping |
| JWT (first-party) | Same validator, separate issuer/keys | Reject `alg=none`, enforce `kid` |
| API key | Prefix + hashed secret lookup, constant-time compare, rotation + expiry | Key row carries tenant, roles, scopes; keys are never logged |

`Principal {user_id, tenant_id, roles: frozenset[str], scopes: frozenset[str], auth_method}`. Tenant comes from the token/key row only.

**Data layer**

- App connects as a non-owner role without `BYPASSRLS`; tables use `ENABLE` + `FORCE ROW LEVEL SECURITY`.
- `documents` and `chunks` carry `tenant_id`, `owner_id`, `acl_principals text[]` (user IDs, role names, groups). ACL is denormalized onto chunks so the filter is inside the vector/FTS query.

```sql
CREATE POLICY chunk_read ON chunks FOR SELECT USING (
  tenant_id = current_setting('app.tenant_id')::uuid
  AND (acl_principals && string_to_array(current_setting('app.principals'), ','))
);
```

- ACL changes on a document propagate to chunks transactionally; a nightly reconciler verifies chunk ACL equals document ACL.
- **Audit log:** append-only table; `UPDATE/DELETE/TRUNCATE` revoked and blocked by trigger; hash-chain (`prev_hash`) for tamper evidence; written over a separate connection so denial records survive request rollback. Records: hashed principal, request\_id, original + rewritten query, retrieved doc/chunk IDs, tool calls, decision (answer/abstain/deny). Raw query text may contain PII: redact before write, define retention.

**Failure modes:** pooled-connection setting leak (mitigated by transaction-local settings + test), stale ACL on chunks (reconciler), role-claim drift from IdP (short token TTL, mapping tests), key leakage (hash at rest, prefix for lookup, rotation).

## 4. Agent Graph

**State (Pydantic):** `principal_ref`, `messages` (trimmed), `memory_ctx`, `analysis`, `sub_queries`, `retrieved_ids`, `answer`, `citations`, `abstained`, `budget`.

**Query analyzer** (structured output, validated with repair, prompt versioned):

```
Analysis {label: clear|vague|ambiguous|multi_part, confidence: float,
          rewrite?: str, clarify_question?: str, sub_queries?: list[str]}
```

- `confidence ≥ τ_rewrite` → rewrite; below → ask exactly ONE clarifying question and end the turn (no second clarify on the same thread until the user answers); `multi_part` → decompose and fan out retrieval per sub-query (LangGraph `Send`; verify current API in the pinned version).
- Rewrite guard: entity/number retention check + embedding similarity to original; failure falls back to `pass`. Original and rewrite are both logged in the span and audit log.
- Analyzer sees conversation + memory, but memory is context only.

**Human-in-the-loop:** sensitive tools call LangGraph `interrupt()`; resume via `Command(resume=...)` with the approver identity re-verified from the resuming request's token. Approval payload is bound to the exact tool args (hash), so args cannot be swapped after approval.

**Guardrails:** `guardrail_in` (injection heuristics + classifier, PII flags, length limits); `guardrail_out` (citation presence, groundedness check, exfil patterns such as markdown image URLs, leaked system prompt). Both fail closed to a safe refusal.

## 5. Retrieval

```
query → (ACL + metadata WHERE) → [FTS top-N] + [vector top-N] → RRF (k=60) → cross-encoder rerank top-M → generate
```

- **Pre-filter, in-query.** Both legs share the same RLS/ACL predicate.
- **pgvector caveat:** an HNSW scan with a restrictive filter can return fewer than N rows (the filter is applied after the index scan). pgvector 0.8 added iterative index scans for this; verify the version and parameter names before relying on them. For small or highly restrictive tenants, fall back to exact scan; for very large tenants consider partitioning by `tenant_id`. Add a recall test with restrictive ACLs.
- **BM25 caveat:** native `tsvector` + `ts_rank_cd` is *not* true BM25. v1 uses it behind a `LexicalSearcher` interface; swap to a BM25 extension later if evals show lexical weakness (verify extension availability/licensing).
- **Rerank:** cross-encoder (self-hosted, config-selected) over top-50 → top-8. Runs only on rows that already passed ACL.
- **Abstain:** if top rerank score < τ\_abstain or fewer than k distinct supporting chunks, return an abstention with what was searched. τ values come from the golden set.
- **Citations:** answer claims map to chunk IDs; `guardrail_out` rejects uncited factual answers. Citation targets are re-verified against the principal's visibility at response time.
- **Ingestion:** parse → chunk (structure-aware, overlap config) → embed (model + version stored per chunk) → ACL inherit from source → idempotent upsert by content hash. Embedding model change = reindex job, never in-place mix.

## 6. Memory

| Tier | Store | Namespace | Write rule |
| --- | --- | --- | --- |
| Short-term | PostgresSaver | `thread_id` (owned) | Trim by tokens; summarize older turns into a rolling summary |
| Semantic | Store | `(tenant, user, "semantic")` | Gated extractor; dedupe by key + similarity; update in place with `updated_at`, `source_turn` |
| Episodic | Store | `(tenant, user, "episodic")` | Summary + timestamp + outcome per session; decays |
| Procedural | Store | `(tenant, user, "procedural")` | Versioned; written only after validation or explicit user approval |

- **Write-gate:** extractor proposes → rules + classifier (durable? user-specific? non-sensitive? non-contradictory?) → PII redaction → write. Contradiction with an existing fact creates an update with history, not a silent overwrite.
- **TTL/decay:** `expires_at` and `last_used_at` on every item; our own sweeper job enforces TTL (do not assume the Store provides it; verify in the pinned version).
- **User control:** `GET /memory`, `DELETE /memory/{id}`, `DELETE /memory` (all), authenticated via Principal; deletes are hard deletes and are audit-logged.
- **Isolation:** the Store is library-managed and not covered by our RLS. All access goes through a `MemoryService` that derives the namespace prefix from `Principal` (callers cannot pass it). Tech debt flag: add a DB-level guard (RLS or view) later; until then, cross-user isolation tests are mandatory.
- **Precedence:** memory is injected in a delimited block below system rules and retrieved-doc policy; it can shape tone/preferences but never widens access, changes tool permissions, or overrides instructions.

## 7. Tools & Harness

```
ToolSpec {name, args_model: BaseModel, required_scopes, sensitive: bool,
          timeout_s, max_retries, idempotent: bool, breaker_config}
```

- **Authorization in the wrapper**, before execution: scopes/roles from `Principal`; denied calls are audit-logged and returned to the model as a neutral error.
- **Reliability:** per-tool timeout; retries with exponential backoff + jitter (e.g., tenacity); idempotency key = hash(thread, step, tool, args) stored with result; circuit breaker per tool/downstream (small in-house or a library; evaluate before adopting).
- **Model layer:** ordered fallback list from config; structured outputs validated with Pydantic and one repair attempt, then fail closed. Streaming via SSE; `guardrail_out` operates on the buffered final answer or a sliding window. Decision: buffer-then-stream for sensitive tenants, token-stream otherwise.
- **Limits:** per-user rate limits and daily token/cost budgets. v1 uses Postgres counters (no new infra); move to Redis if contention shows up. Budget exhaustion returns a clear error, not silent degradation.
- **Injection defense:** retrieved text and tool output are wrapped as quoted data with an explicit "untrusted" instruction; they never choose tools or tool args without schema validation; sensitive tools always require HITL regardless of model output; outbound links/images in answers are stripped or allow-listed.
- **Config:** `pydantic-settings`; secrets from env/secret manager only. Prompts live in Langfuse prompt management, pinned by version/label in config, with a local fallback file so CI is deterministic.

## 8. Evals (CI-gated)

Golden datasets are versioned in Langfuse datasets and mirrored to the repo (hash-pinned). Thresholds below are **starting placeholders**, to be calibrated on a measured baseline.

| Layer | Metrics | Initial gate |
| --- | --- | --- |
| Query rewrite | intent preservation, ambiguity-detection accuracy, clarify-vs-rewrite correctness | ≥ 0.90 / 0.85 / 0.85 |
| Retrieval | recall@k, MRR, nDCG, context precision; **ACL leakage** | no regression > 2 pts; **leakage = 0** |
| Rerank | uplift over RRF-only baseline | > 0, no regression |
| Generation | faithfulness, answer relevance, citation accuracy, correct refusal | ≥ 0.90 / 0.85 / 0.95 / 0.95 |
| Prompt | golden-set regression per prompt change | no regression |
| Memory | recall relevance, stale/conflict rate, cross-user isolation | isolation = 100% |
| Tools | selection accuracy, argument accuracy | ≥ 0.95 / 0.90 |
| Security red-team | cross-user, privilege escalation, injection, identity spoofing | 100% blocked |
| End-to-end | task success, p50/p95 latency, cost/request | budget-based |

- **PR pipeline:** unit + integration tests, deterministic security suite (real Postgres in Compose), small golden subset. **Nightly:** full golden set and red-team. Prompt/model change triggers the full regression.
- **Judge reliability:** LLM-as-judge at temperature 0, versioned judge prompts, majority of N runs for borderline scores; calibrate against human labels periodically (target agreement tracked).
- **Online:** sample production traces → judge → scores attached to the Langfuse trace; weekly human spot-check of a sample to detect judge drift.
- Security evals are pure tests (no LLM needed) wherever possible: ACL leakage is checked by SQL assertions across tenant/user matrices.

## 9. Observability (Langfuse)

- One trace per request (`trace_id = request_id`), a span per graph node and tool call, with tokens, cost, latency. p50/p95 per node from exported data.
- `user_id` is logged only as HMAC-SHA256 with a secret key (a plain hash is reversible by enumeration); tenant ID logged as a stable pseudonym. No raw PII in span inputs/outputs: redact before the callback.
- Eval scores and user feedback attach to traces by trace ID.
- **Alerts:** cost spike, error rate, ACL-denial anomalies, eval drift. Langfuse is not an alerting system by itself; export metrics (OpenTelemetry/Prometheus) and alert there. Decision: OpenTelemetry metrics to Prometheus + Grafana Alerting (see section 12).

## 10. Phases & Exit Criteria

| # | Phase | Exit criteria (tests pass) |
| --- | --- | --- |
| 1 | Auth + RLS | Each authenticator → identical `Principal`; spoofed header/body identity ignored; RLS blocks cross-tenant/user reads under pooled connections; audit append-only enforced |
| 2 | Ingestion + retrieval | ACL inside query; recall with restrictive filters; RRF + rerank; abstention; ACL leakage = 0 on matrix |
| 3 | Graph | Node contracts typed; analyzer routing tests; HITL interrupt/resume with arg-binding; thread ownership enforced |
| 4 | Memory | Gating, dedupe, TTL sweeper, view/delete, cross-user isolation, PII redaction |
| 5 | Harness | Tool authz, timeouts, retries, breaker, idempotency, fallback, rate/cost limits, injection tests |
| 6 | Evals | Datasets versioned; CI gates block merge; judge calibration set |
| 7 | Observability | Trace/span coverage, hashed IDs, alerts firing in drill |
| 8 | Deploy | Compose + GH Actions; migrations; secrets externalized; smoke + rollback |

Each module follows the working rule: a design note (design, trade-offs, failure modes, under 10 lines) before code; its test and eval land in the same PR.

## 11. Risks & Tech Debt

1. **Store isolation is app-level** (no RLS on library tables): mitigated by `MemoryService` + tests; revisit.
2. **FTS is not BM25:** may under-perform on rare terms; interface allows swap.
3. **Filtered HNSW recall:** needs measurement per tenant size; fallback strategies above.
4. **Checkpoint contents** can persist sensitive data: IDs only, plus retention and deletion on user/tenant offboarding.
5. **Streaming vs output guardrails** conflict: buffer policy per tenant.
6. **Thread of PII in logs/audit:** redaction coverage must be tested, not assumed.
7. **Eval cost/flakiness** in CI: small PR subset, nightly full run, cached judge outputs.
8. **Library API drift** (LangGraph `interrupt`/`Send`/Store index and TTL options, Langfuse callback, pgvector iterative scan): pin versions, and verify each against current docs at implementation time. Nothing here should be treated as confirmed until checked.

## 12. Resolved Decisions

All six former open questions are decided below as defaults, chosen without stakeholder input. Each is config-driven, so changing one does not require a redesign.

| # | Topic | Decision | Trade-off / note |
| --- | --- | --- | --- |
| 1 | IdP and claims | Generic OIDC via config (issuer, JWKS URL, audience, claim names). Dev IdP: Keycloak in Compose; prod-compatible with Entra ID, Okta, Auth0. Tenant from a `tenant_id` claim (fallback: issuer-to-tenant map); roles from `roles`; groups from `groups`, mapped to roles by a versioned `role_map.yaml`. | IdP-agnostic and reviewable. Entra group-overage claims need a Graph lookup: out of v1, token rejected and logged. |
| 2 | ACL model | Flat principal set at query time (user ID, roles, groups). Folder/doc hierarchy is expanded at ingest onto chunks; an ACL change triggers an async re-ACL job. | One indexed array-overlap (GIN) per query. During re-ACL the document is flagged `acl_pending` and excluded from retrieval (fail closed). |
| 3 | Scale target | 50 tenants, 5M chunks total, 1M per tenant max. HNSW + iterative scan; exact scan when the filtered set is under 10k chunks; partition by `tenant_id` only when a tenant passes 1M chunks. | No premature partitioning; thresholds revisited after Phase 2 recall benchmarks. |
| 4 | Retention and erasure | Audit 1 year hot, then archived with its hash chain; checkpoints 30 days; episodic 90 days; semantic until deleted or 12 months unused. All tenant-overridable. User delete cascades to memory and checkpoints within 24 h. | Audit holds only pseudonymous IDs and redacted queries, so erasure means destroying the pseudonym key mapping, not editing the log. Needs legal review before production. |
| 5 | Alert backend | OpenTelemetry metrics to Prometheus + Grafana Alerting (in Compose). Langfuse stays for traces, evals, prompts. | Langfuse is not an alert engine; swappable for a managed stack later. |
| 6 | Launch tools | `search_documents` and `get_document` (read-only, non-sensitive); `create_ticket` (`tickets:write`, sensitive, HITL). Rule: any tool with an effect outside this system is sensitive. | One sensitive tool exercises interrupt/resume, arg-binding and idempotency with a small blast radius. |
