# Access-Controlled Multi-Authenticator RAG Chatbot

Production-grade RAG chatbot where **identity comes only from a verified credential** and **access control is enforced in Postgres**, never by the LLM.

> Status: design complete, implementation in phases (see [Roadmap](#roadmap)). Commands below are the target interface, delivered by Phase 8.

## Guarantees

- One `Principal {user_id, tenant_id, roles, scopes}` from OIDC/SSO, JWT, or API key. Never read from prompts, bodies, or memory.
- Postgres RLS (`FORCE`) plus ACL pre-filter inside hybrid search. No post-filtering.
- Every answer is cited; weak retrieval means abstain.
- Memory is namespaced per tenant + user, and never overrides ACLs or system rules.
- Retrieved text and tool output are untrusted data. Sensitive tools require human approval.
- Immutable, hash-chained audit log: who asked what, what was retrieved, which tools ran.

## Stack

Python 3.11+, FastAPI, LangGraph (PostgresSaver + Store), Postgres + pgvector, Langfuse, Pydantic v2, pytest, Docker Compose, GitHub Actions. Metrics: OpenTelemetry, Prometheus, Grafana. Dev IdP: Keycloak.

## Architecture

```
Client -> FastAPI (authn -> Principal) -> thread-ownership check
  -> LangGraph (PostgresSaver)
     authenticate -> guardrail_in -> load_memory -> query_analyzer
       -> [rewrite | clarify | decompose | pass]
       -> retrieve -> rerank -> generate -> guardrail_out -> write_memory -> respond
  -> Postgres (RLS, pgvector, Store, audit_log)
  -> Langfuse (1 trace per request, 1 span per node/tool)
```

Retrieval: `ACL/metadata WHERE` -> FTS + vector -> RRF (k=60) -> cross-encoder rerank -> generate.

## Key decisions

| Area | Decision |
|---|---|
| IdP | Generic OIDC via config; tenant from `tenant_id` claim; groups mapped to roles by `role_map.yaml` |
| ACL | Flat principal set at query time; hierarchy expanded at ingest; `acl_pending` docs excluded (fail closed) |
| Scale target | 50 tenants, 5M chunks total, 1M per tenant; HNSW + iterative scan, exact scan for small filtered sets |
| Lexical search | Postgres FTS (`ts_rank_cd`) behind an interface; not true BM25, swappable |
| Retention | Audit 1y, checkpoints 30d, episodic 90d, semantic until deleted or 12 months unused; tenant-overridable |
| Alerting | OpenTelemetry to Prometheus + Grafana Alerting |
| Launch tools | `search_documents`, `get_document` (read); `create_ticket` (sensitive, HITL) |

Full rationale, failure modes, and risks: `docs/DESIGN.md`.

## Repo layout (planned)

```
app/
  api/            FastAPI routes, middleware, SSE streaming
  auth/           authenticators -> Principal, role_map loader
  db/             migrations, RLS policies, session/tx helpers
  ingest/         parse, chunk, embed, ACL inherit, re-ACL job
  retrieval/      hybrid search, RRF, rerank, abstention
  graph/          LangGraph nodes, state, routing
  memory/         MemoryService (semantic/episodic/procedural), TTL sweeper
  tools/          ToolSpec registry, authz wrapper, breaker, idempotency
  llm/            model router, fallback, structured-output repair
  guardrails/     input/output checks, injection defense, PII redaction
  observability/  Langfuse + OTel wiring, ID hashing
  config/         pydantic-settings, prompt version pins
evals/            golden datasets (versioned), runners, judges, red-team
tests/            unit, integration (real Postgres), security
docs/DESIGN.md
docker-compose.yml
```

## Quickstart (target)

```bash
cp .env.example .env          # set secrets; never commit .env
docker compose up -d          # postgres+pgvector, keycloak, langfuse, otel stack, api
make migrate                  # schema, RLS, audit triggers
make seed                     # demo tenants, users, documents
make test                     # unit + integration + security
make eval-pr                  # small golden subset (CI parity)
```

## Configuration

All config via environment / `pydantic-settings`; secrets from env or a secret manager only.

| Variable (illustrative) | Purpose |
|---|---|
| `DATABASE_URL` | App role, non-owner, no `BYPASSRLS` |
| `OIDC_ISSUER`, `OIDC_AUDIENCE`, `OIDC_JWKS_URL` | OIDC validation |
| `JWT_*`, `API_KEY_HASH_*` | First-party JWT and API key verification |
| `AUDIT_DATABASE_URL` | Separate connection and role for audit writes |
| `LANGFUSE_*` | Tracing, prompts, datasets |
| `USER_HASH_SECRET` | HMAC key for hashing `user_id` in logs |
| `MODEL_FALLBACK_ORDER`, `PROMPT_VERSIONS` | Externalized model and prompt pins |
| `RATE_LIMIT_*`, `BUDGET_*` | Per-user limits and token/cost budgets |

## Testing and evals

- **Every PR:** unit, integration (real Postgres), deterministic security suite, small golden subset.
- **Nightly:** full golden set, red-team (cross-user, privilege escalation, injection, identity spoofing).
- **Hard gates:** ACL leakage = 0, cross-user memory isolation = 100%, red-team 100% blocked.
- **Soft gates (calibrate on baseline):** faithfulness, citation accuracy, correct refusal, retrieval recall/MRR/nDCG, tool selection/argument accuracy.
- **Prompt or model change:** full regression against the golden set.
- **Online:** sampled traces scored by LLM-as-judge; periodic human spot-checks calibrate the judge.

## Security notes

- Set RLS context with transaction-local settings (`set_config(..., true)`) so pooled connections cannot leak identity.
- `thread_id` is bound to `(tenant_id, user_id)` and verified before every invoke.
- Checkpoints store chunk IDs and citations, not content; content is re-fetched under RLS.
- LangGraph Store tables are library-managed and outside RLS: all access goes through `MemoryService`, which derives the namespace from `Principal`.
- Log `user_id` only as HMAC-SHA256; redact PII before Langfuse and audit writes.

## Roadmap

| # | Phase | Exit criteria |
|---|---|---|
| 1 | Auth + RLS | All authenticators yield identical `Principal`; spoofing ignored; RLS holds under pooling; audit append-only |
| 2 | Ingestion + retrieval | ACL in-query; filtered recall verified; RRF + rerank; abstention; leakage = 0 |
| 3 | Graph | Typed nodes; routing tests; HITL resume with arg-binding; thread ownership |
| 4 | Memory | Gating, dedupe, TTL, view/delete, isolation, PII redaction |
| 5 | Harness | Tool authz, timeouts, retries, breaker, idempotency, fallback, limits, injection tests |
| 6 | Evals | Versioned datasets; CI gates block merges; judge calibrated |
| 7 | Observability | Full trace coverage; hashed IDs; alert drill passes |
| 8 | Deploy | Compose + GitHub Actions; migrations; secrets externalized; smoke + rollback |

## Known risks / tech debt

1. Store isolation is app-level (no RLS on library tables).
2. FTS is not BM25; revisit if lexical recall lags.
3. Filtered HNSW recall must be measured per tenant size.
4. Library APIs (LangGraph `interrupt`/`Send`/Store options, Langfuse callbacks, pgvector iterative scan) must be verified against pinned versions at implementation time.
5. Retention and erasure defaults need legal review before production.