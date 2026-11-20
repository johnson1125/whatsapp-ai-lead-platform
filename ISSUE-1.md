# Implement WhatsApp Cloud API webhook with idempotency, retries, and integration tests.

## Context and objective

This is the first implementation task for the WhatsApp AI Lead Management SaaS. The repository currently contains planning documentation and a .NET `.gitignore`; there is no application or test suite. Build the initial backend and a reliable inbound webhook pipeline. Do not assume any services or credentials already exist.

Implement in a feature branch and open a PR targeting the default branch with `Closes #1`. This brief is intended for Devin, but does not imply Devin has been granted repository access or started work.

## Scope and design decisions

- Use ASP.NET Core on .NET 10 LTS, EF Core/Npgsql, PostgreSQL, and a hosted background worker. Pin the SDK and compatible package versions in source control.
- Use PostgreSQL as both the durable inbox and source of uniqueness. Redis is part of the proposed wider architecture but is not required for this issue and must not be the correctness boundary.
- Implement inbound text messages and message status events. Retain metadata for recognized unsupported message types without downloading media. AI, RAG, frontend, live outbound sends, billing, and production deployment are excluded.
- Configure at least two synthetic tenant/WhatsApp phone-number mappings for tests. No public tenant-registration endpoint is required. Unknown business/phone mappings must never create tenant data.

## Technical tasks

### 1. Bootstrap and configuration

- [ ] Create the solution, API, application/infrastructure boundaries, unit tests, and integration tests. Prefer a small modular implementation over speculative abstractions.
- [ ] Add validated options for app secret, verification token, database connection, tenant mappings, payload limit, worker settings, and retry policy. Verification token and app secret must be distinct. Fail safely when required configuration is missing.
- [ ] Add Docker Compose for PostgreSQL, EF migrations, placeholder-only configuration examples, and liveness/readiness endpoints. Database unavailability must fail readiness without exposing credentials.

### 2. Verification and ingress

- [ ] Implement `GET /webhooks/whatsapp`: require `hub.mode=subscribe`, the configured `hub.verify_token`, and a present `hub.challenge`; return the unchanged challenge as plain text with 200. Return 400 for missing/malformed required parameters and 403 for an invalid token/mode. Never log the token/query string.
- [ ] Implement `POST /webhooks/whatsapp`: require `X-Hub-Signature-256` in `sha256=<hex>` form. Compute HMAC-SHA256 over the exact raw body bytes using the app secret; compare decoded fixed-length digests in constant time before JSON parsing or persistence. Reject missing/malformed/mismatched signatures with 401 and no writes.
- [ ] Bound request size (proposed default 1 MiB, configurable); return 413 for oversized requests and 400 for authenticated malformed JSON or invalid supported envelope structure. Enforce limits for chunked bodies as well as Content-Length.
- [ ] Parse every entry/change/message/status item in a batched `whatsapp_business_account` envelope. Support unknown optional fields for forward compatibility; never treat each HTTP body as one message.
- [ ] Resolve tenant identity using configured business account and phone-number IDs. Persist supported mapped events and quarantine authenticated unmapped/unsupported events with bounded retention and no domain effects. Return 200 after durable quarantine; count and document these cases. Supported events missing required identity fields return 400 with the whole batch rolled back.
- [ ] Persist the entire accepted batch in one PostgreSQL transaction. Return 200 only after commit, including duplicates. Return 503 if durable storage cannot commit. No LLM calls or outbound network processing before acknowledgement.

### 3. Durable inbox and event identity

- [ ] Model event ID/key, tenant, business and phone IDs, event type, provider message ID, provider timestamp, minimized payload, processing status, attempt count, next-attempt time, lease owner/expiry, timestamps, and sanitized error category.
- [ ] Enforce database unique constraints over canonical event keys scoped by tenant and channel. Inbound message identity uses tenant + phone ID + message ID + event kind. Status identity additionally includes status name and provider timestamp so sent/delivered/read are not collapsed into one event. Document handling for otherwise ambiguous payloads and unsupported events; use a stable canonical digest fallback only where no provider identity exists.
- [ ] Use conflict-safe inserts so duplicate events do not abort valid new events in the same batch. Concurrent requests across two application instances must persist a single logical event.
- [ ] Deduplicate individual events, not only whole request hashes. Rebatched and reordered deliveries must remain idempotent.

### 4. Worker, retries, and recovery

- [ ] Claim work atomically with PostgreSQL row locking (for example SKIP LOCKED), persistent leases, and a documented reclaim policy. Two workers must not concurrently apply the same event.
- [ ] Normalize inbound messages into tenant-scoped conversation/message records and persist status history. Prevent stale or out-of-order statuses from regressing the current status; explicitly define handling of failed versus delivered/read events.
- [ ] Commit all domain writes and the inbox completed marker in one transaction with defensive uniqueness constraints. A crash after commit but before worker acknowledgement must cause no duplicate domain effects on replay.
- [ ] Classify transient failures for retry and permanent failures for quarantine. Proposed policy: at most 8 total attempts, capped exponential delay starting at 5 seconds, maximum 15 minutes, with jitter; make configurable and document semantics. These are internal defaults, not Meta delivery guarantees.
- [ ] Persist attempts and next-attempt time across restarts. Exhausted events enter a dead-letter state with sanitized failure metadata. Use an injectable clock/delay/random source so tests do not sleep through backoff.
- [ ] Provide a documented operator replay command or restricted local procedure that preserves event identity and audit history. Do not expose an unauthenticated replay endpoint. Replaying completed events must not duplicate effects.

### 5. Security and observability

- [ ] Redact message bodies, phone numbers, signatures, access tokens, secrets, and verification queries from default logs and exception responses. Use synthetic fixtures only.
- [ ] Apply explicit tenant filters and constraints in worker/domain access; test attempted cross-tenant collision and access. Quarantine data must not be queryable through business APIs.
- [ ] Add structured correlation using safe event identifiers and metrics for accepted/duplicate/rejected/quarantined events, backlog age, worker attempts, failures, and dead letters. Avoid unbounded per-tenant/message metric labels.
- [ ] Document raw-payload minimization, retention, purge, encryption, and replay tradeoffs. Do not claim production compliance or exactly-once delivery.

## Acceptance criteria

All criteria must be demonstrated in the PR:

1. Valid GET verification returns the exact challenge; invalid verification never returns it.
2. Signed valid POST batches are durably accepted; invalid signatures produce no database writes. Byte-level body tampering, whitespace changes, and malformed signature encodings are rejected.
3. Sequential and concurrent duplicate delivery produces one inbox event and one domain effect per logical event, including rebatched deliveries and multi-worker execution.
4. Storage failure yields 503 without false acknowledgement. No accepted supported event is lost when the API or worker restarts.
5. Transient failures retry on schedule and recover; permanent/exhausted work is quarantined and replayable without duplicate effects. Worker lease expiry and crash recovery are exercised.
6. Batched new + duplicate events are all handled correctly; malformed supported items roll back the batch. Unknown fields/types and unknown mappings follow the documented quarantine contract.
7. Out-of-order status events cannot regress current state, and tenant/channel boundaries cannot collide or leak.
8. A clean checkout can restore, build, migrate, run, and execute all automated tests with documented commands and Docker, without real Meta credentials or paid services.
9. CI runs build and tests on pull requests and pushes to the default branch with least-privilege permissions; the PR includes a passing run and actual command results.

## Testing requirements

- Unit tests: raw-byte HMAC validation, strict signature parsing, verification parameters, canonical event keys, tenant mapping, status progression, retry classification/backoff bounds, and secret redaction.
- Integration tests: ASP.NET Core WebApplicationFactory plus a real ephemeral PostgreSQL instance (for example Testcontainers). EF InMemory/SQLite alone is insufficient for uniqueness, locking, migrations, and transaction behavior.
- Cover: valid/invalid GET; valid/missing/malformed/incorrect signatures; tampered body; malformed JSON/envelope; oversized fixed and chunked bodies; multi-entry/multi-change batches; unsupported media metadata; unknown fields/mappings; sequential/concurrent duplicates; same message ID in separate tenants/channels; duplicate plus new event in one batch; status variations; database outage/rollback; transient/permanent/exhausted failures; restart recovery; lease reclaim; two worker instances; safe replay; logging redaction.
- Inject a failure before commit and simulate replay after commit to demonstrate atomicity. Verify domain rows and durable inbox state, not just HTTP codes.
- Keep tests deterministic with bounded polling and fake clocks where appropriate. No live calls to Meta, a model provider, or customer accounts.
- A manual Meta sandbox smoke test is optional and must be reported as unperformed when credentials are unavailable; it cannot substitute for CI tests.

## Expected deliverables

- [ ] Reviewable PR referencing #1, implementation summary, design tradeoffs, limitations, and real validation evidence.
- [ ] Buildable solution, SDK/package pins, migrations, local PostgreSQL Compose setup, and placeholder configuration.
- [ ] Verified webhook endpoints, durable inbox, idempotent domain processing, persistent retries, lease recovery, dead letters, and safe replay procedure.
- [ ] Unit/integration fixtures and test suite plus a passing GitHub Actions workflow.
- [ ] `docs/webhook-operations.md`: local setup, exact commands, synthetic signed request example, Meta callback/subscription setup, endpoint response table, event-key design, retry/replay runbook, and troubleshooting.
- [ ] README updated to distinguish implemented, tested, manually verified, and future functionality. Include no fabricated test results or production claims.

## Starting instructions for Devin

Read README and this brief, inspect the default branch, then implement on `feat/issue-1-whatsapp-webhook`. Select compatible supported dependency versions and verify current Meta documentation linked in README. Prefer the contract above; record any justified adjustment in the PR. Never commit credentials, contact customers, send live WhatsApp messages, or deploy a production service for this task. If access or a necessary prerequisite is missing, report the exact blocker instead of marking acceptance criteria complete.
