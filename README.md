# WhatsApp AI Lead Platform

A proposed multi-tenant SaaS for small and medium-sized businesses to capture WhatsApp enquiries, qualify leads, and coordinate follow-ups with AI assistance and human oversight.

## Project status

**Planning and repository bootstrap only.** This repository contains documentation and a .NET-oriented `.gitignore`. No application, database migrations, webhook endpoint, AI integration, deployment, or passing test suite has been implemented. Architecture, features, and safeguards below are proposed requirements, not working capabilities. There is no live demo or production service.

The first implementation milestone is **Issue #1: Implement WhatsApp Cloud API webhook with idempotency, retries, and integration tests.** Read [the implementation brief](ISSUE-1.md) before starting. Publication of this repository does not provision Devin access or start a Devin session.

## Product scope

Planned capabilities:

- Capture inbound messages and delivery events using the official Meta WhatsApp Cloud API.
- Shared inbox, conversation ownership, lead profiles, tags, notes, qualification fields, pipeline stages, and follow-up reminders.
- AI-generated conversation summaries, suggested responses, and lead qualification with human handoff.
- Retrieval-augmented generation (RAG) over each business's approved knowledge base, with source references and a fallback when evidence is insufficient.
- Tenant administration, role-based access, audit history, operational dashboards, and usage controls.
- Later: approved message templates, consent-aware outbound messaging, CRM integrations, and subscription billing.

Automatic replies, outbound campaigns, billing, and the dashboard are outside Issue #1. Messaging must respect Meta policies, applicable consent requirements, template rules, and customer-service-window restrictions; verify current policies before release.

## Proposed architecture

Start with a modular ASP.NET Core application and a durable background worker. Avoid splitting into microservices before operational need is demonstrated.

| Component | Proposed responsibility |
| --- | --- |
| ASP.NET Core on .NET 10 LTS | Webhook ingress, future authenticated APIs, dependency injection, health endpoints |
| Meta WhatsApp Cloud API | Official inbound events and future outbound messages; configurable Graph API version |
| PostgreSQL with EF Core/Npgsql | Tenant mappings, durable webhook inbox, processing state, messages, leads, and audit records |
| Background worker | Claim durable events, normalize messages/statuses, retry recoverable failures, quarantine exhausted work |
| Redis | Future caching and rate limiting; optional wake-up hints, never the sole source of durable events or deduplication |
| RAG service and model adapter | Future tenant-scoped retrieval, bounded prompts, response suggestions, and evaluation |
| PostgreSQL/pgvector or separate vector store | Future document embeddings with tenant and document access filters; final choice deferred |
| Object storage | Future attachments and source documents with restricted access and retention policies |
| OpenTelemetry and structured logs | Correlation, request metrics, worker failures, retry age, and backlog monitoring |

Proposed flow: Meta webhook → signature validation → PostgreSQL inbox transaction → HTTP acknowledgement → background normalization → lead/conversation updates → future RAG suggestions and human review.

RAG and model calls must stay outside the synchronous webhook path. Never claim exactly-once network delivery: the design assumes duplicate and out-of-order delivery and provides idempotent database effects.

## Security and privacy requirements

These controls remain to be implemented and verified:

- Validate POST signatures using the Meta app secret over the original request bytes; the GET verification token is a separate secret.
- Resolve tenants from server-controlled WhatsApp business/phone-number mappings. Do not accept a client-provided tenant ID as authority.
- Enforce tenant isolation in application queries, database constraints, background jobs, object storage, and future vector retrieval; test negative cross-tenant cases.
- Use HTTPS, least-privilege database and application identities, secret management, and credential rotation. Keep secrets out of commits, fixtures, request logs, and CI output.
- Add authenticated administrator APIs and role-based authorization before exposing business functionality. Webhooks use signature verification rather than end-user authentication.
- Minimize stored personal data, redact logs, protect backups, define retention/deletion policies, and assess Malaysia PDPA obligations before launch.
- Treat retrieved documents and message text as untrusted input. Future AI tools require authorization checks, prompt-injection defenses, output review, and explicit boundaries on autonomous actions.
- Add dependency scanning, secure CI permissions, audit records, abuse limits, and recovery procedures before production.

## Webhook reliability requirements

- Acknowledge only after durable persistence, or after confirming an already-persisted duplicate. Return a retryable failure if persistence fails.
- Atomically persist all supported events in a batch; a partial transaction must not lose the remainder when Meta retries.
- Enforce event-level uniqueness in PostgreSQL. Redis or an in-memory cache alone cannot guarantee idempotency across restarts and concurrent replicas.
- Retry background processing with bounded exponential backoff and jitter, persistent attempt counters, and reclaimable leases. Separate Meta delivery retries from internal processing retries.
- Commit processing state and domain database effects atomically. Future external effects require an outbox and provider-specific reconciliation; they are outside this milestone.
- Quarantine poison events, provide a safe replay procedure, and prevent old status events from regressing current state.
- Monitor acceptance failures, duplicate counts, backlog age, retry attempts, dead letters, and processing latency without logging message bodies.

Exact event keys, response behavior, test cases, and delivery requirements are specified in [the Issue #1 brief](ISSUE-1.md).

## Implementation roadmap

| Milestone | Planned outcome | Status |
| --- | --- | --- |
| 0 | Repository documentation and first implementation brief | Bootstrap documentation |
| 1 | Signed webhook verification, durable inbox, idempotent processing, retries, integration tests, CI | Not implemented; Issue #1 |
| 2 | Tenant onboarding, authentication, shared inbox, lead model, and roles | Planned |
| 3 | Knowledge ingestion, tenant-scoped RAG, response suggestions, evaluations, and human handoff | Planned |
| 4 | Consent-aware outbound messaging, reminders, templates, and CRM integrations | Planned |
| 5 | Billing, operational hardening, backups, security review, and controlled pilot | Planned |

No timeline, revenue, customer adoption, reliability SLA, or completed functionality is claimed.

## Development handoff

There are no runnable setup commands yet because no solution exists. For Issue #1, Devin should create the solution, pin the supported SDK and package versions, add PostgreSQL local infrastructure and migrations, and document exact restore/build/test/run commands. Use synthetic data and a local fake event processor; real Meta credentials and a live WhatsApp account must not be required for automated tests.

Suggested structure, to be created by Issue #1:

```text
src/WhatsAppLead.Api/
src/WhatsAppLead.Application/
src/WhatsAppLead.Infrastructure/
tests/WhatsAppLead.UnitTests/
tests/WhatsAppLead.IntegrationTests/
docs/
```

Keep changes on a feature branch and submit a reviewable pull request referencing Issue #1. Do not deploy or add live secrets. Update this README only to describe capabilities actually implemented and checks actually run.

## Primary references

- [Meta WhatsApp Business Platform](https://developers.facebook.com/documentation/business-messaging/whatsapp/overview/)
- [Meta webhook verification and signatures](https://developers.facebook.com/docs/graph-api/webhooks/getting-started/)
- [Microsoft .NET releases and support](https://learn.microsoft.com/en-us/dotnet/core/releases-and-support)
- [ASP.NET Core integration testing](https://learn.microsoft.com/en-us/aspnet/core/test/integration-tests)

Verify current Meta payload schemas, retry behavior, policy requirements, and Graph API version during implementation. Proposed internal retry defaults in Issue #1 are application decisions, not claims about Meta's retry schedule.
