# Municipal Court Management — Phased Development Plan

> Project: 377-municipal-court-management · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the three data-model suggestions. The database design adopts **Data Model Suggestion 1 (Entity-Centric Normalized Relational)** as the canonical schema — it maps cleanly to NIEM Justice, OASIS LegalXML ECF 5.01, and NCSC JTC functional standards, satisfies CJIS/NIST audit requirements, and produces state-reporting exports from typed columns. Where Suggestion 1 already uses `JSONB` overflow columns (court config, sentencing detail, vehicle data, address), we keep them — this is the targeted hybrid use that Suggestion 2 advocates without sacrificing relational integrity for the analytics-critical case/hearing/disposition/fine/payment chain.

---

## Core Requirements (synthesised)

- **What it does:** An AI-native, open-source case management system (CMS) for small and mid-sized municipal courts, covering the full lifecycle: citation intake → defendant dedup → docket scheduling → disposition/sentencing → fines & fee cashiering → DMV/state reporting → document generation → records retention. Ships with a defendant-facing public portal and an open OpenAPI 3.1 + webhook surface from day one.
- **Primary personas:** court clerk / deputy clerk (daily docketing & cashiering), cashier (payment intake), presiding/associate judge (bench calendar, dispositions), court administrator (reporting, config), records manager (retention/expungement), and the **defendant/public** (portal: case lookup, plea, pay, reschedule).
- **Key differentiators:** open source + open API (incumbents are closed and partner-gated); AI-native (citation OCR, case classification/statute lookup, plain-language summaries, multilingual defendant assistant, no-show prediction, cashiering anomaly detection, drafting assistance); affordable for sub-50k-population jurisdictions; WCAG 2.2 AA multilingual portal; standards-first (NIEM/ECF/NCSC) rather than proprietary screen flows.
- **MVP scope (from features.md "Must-have"):** case intake (manual + CSV/e-citation), defendant dedup, docket scheduling with continuances + notices, disposition & sentencing, cashiering with PCI-scoped processor + payment plans, DMV/state reporting export (jurisdiction-configurable), document templates with merge fields + audit-logged generation, defendant public portal (lookup/plea/pay/reschedule), RBAC + full audit log + records retention scheduler.
- **v1.1 (Should-have):** warrant issuance/recall with LE notification, configurable workflow engine, virtual-hearing integration + recording archival, AI citation OCR & classification, plain-language multilingual defendant assistant, embedded analytics (caseload/clearance/revenue/equity disparity).
- **Backlog (Nice-to-have):** jury management, STT hearing transcripts w/ diarisation, AI drafting for orders/warrants/notices, e-signature provider integration, code-enforcement/parking inbound connectors, predictive scheduling/no-show intervention engine.
- **Deployment model:** **Hybrid** — single-binary-ish Docker Compose stack for self-hosted small courts, and the same image deployable to managed cloud (multi-court tenancy via `court_id`). No proprietary dependencies required to run core.
- **Integration surface:** REST + webhooks (OpenAPI 3.1); payment processor (Stripe hosted checkout as reference, pluggable to Tyler Payments / nCourt); DMV abstract export (FTA/FTC/conviction/reinstatement); state court reporting (Texas OCA / California JBSIS pluggable); LE RMS/CAD inbound citation feed; SMS/email notifications (Twilio/SMTP); virtual hearings (Zoom for Government); e-signature (DocuSign); LLM provider (pluggable — OpenAI/Anthropic/local) plus an MCP server exposing case-data tools.
- **Standards the build must implement:** OASIS LegalXML ECF 5.01 (+4.01 compat) for filing identifiers/messages; NIEM Justice identifiers on core entities; NCSC JTC functional coverage; WCAG 2.2 AA on public surfaces; PCI-DSS v4.0 scope minimisation (processor delegation, SAQ-A); CJIS Security Policy v6.0 audit fields for warrant/criminal-justice data; NIST SP 800-53 AU controls (audit_log); ISO 15489 retention; PDF/A (ISO 19005) archival; OpenAPI 3.1, JSON Schema 2020-12, OAuth 2.0/OIDC, JWT (RFC 7519), TLS 1.3.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | **Python 3.12** | The product is AI-heavy (OCR, classification, summarisation, no-show modelling, MCP tooling). Python has the strongest LLM/ML/OCR ecosystem (Anthropic/OpenAI SDKs, `pytesseract`/cloud OCR, scikit-learn, MCP Python SDK) while remaining productive for CRUD + reporting. Keeps the AI engine and the CMS in one language. |
| API framework | **FastAPI** | Generates OpenAPI 3.1 + JSON Schema 2020-12 automatically (a stated day-one differentiator), async for webhook/LLM I/O, Pydantic v2 validation aligns with our typed NIEM/ECF payloads. |
| Data validation | **Pydantic v2** | Request/response models, config models, ECF/NIEM DTOs, JSONB-column shape validation (covers the "JSONB bypasses CHECK constraints" trade-off from Suggestion 1). |
| ORM / DB access | **SQLAlchemy 2.0 (async) + Alembic** | Mature, supports Postgres native types (UUID, JSONB, arrays, partitioning DDL via raw migration ops), explicit migrations for the 12-table schema. |
| Database | **PostgreSQL 16** | Required by the chosen schema: `gen_random_uuid()`, JSONB, GIN indexes, partial indexes, `RANGE` partitioning on `audit_log`, `TEXT[]`. Single store for OLTP + reporting at municipal scale. SQLite is unsuitable (no JSONB ops/partitioning) — Postgres runs fine in one Docker container for self-host. |
| Task queue / async jobs | **Celery + Redis** | Async, retryable, scheduled work: notice/reminder dispatch, DMV/state-report batch export, AI OCR/classification jobs, retention/expungement sweeps, no-show scoring, payment webhook reconciliation. Redis doubles as cache + rate-limit store. |
| Scheduler | **Celery Beat** | Cron-style triggers for daily warrant-auto-issue sweeps, retention purge eligibility, reminder windows, reconciliation. |
| Object storage | **S3-compatible (MinIO self-host / AWS S3 cloud)** via `boto3` | Documents, generated PDFs (PDF/A), hearing recordings. Keeps large blobs out of Postgres; `documents.file_url` holds the object key. |
| Frontend (staff) | **React 18 + TypeScript + Vite, TanStack Query + Router, shadcn/ui + Tailwind** | Role-tailored back-office dashboards (clerk/judge/admin/cashier), live calendar, bench view. SPA against the REST API. |
| Frontend (public portal) | **Next.js 15 (App Router) + Tailwind** | Server-rendered for SEO/perf/accessibility; WCAG 2.2 AA is far easier to guarantee with SSR + progressive enhancement; multilingual via `next-intl`. Separate deployable from staff SPA to keep the public attack surface minimal. |
| PDF generation | **WeasyPrint (HTML→PDF) + Ghostscript/`ocrmypdf` for PDF/A** | Merge-field templates rendered from HTML; converted to PDF/A (ISO 19005) for archival documents (orders, judgements, abstracts). |
| OCR | **Tesseract (`pytesseract`) baseline + pluggable cloud OCR (AWS Textract / GCV)** | Citation OCR works offline for self-host; cloud OCR optional for accuracy. |
| LLM / AI | **Provider-abstracted client (Anthropic + OpenAI), prompts versioned in repo** | Classification, statute lookup, plain-language summaries, drafting, defendant assistant, anomaly detection. Provider behind an interface so courts can choose / self-host. |
| AI tool exposure | **MCP server (Python MCP SDK)** | Exposes case lookup / schedule / draft tools to LLM clients per `standards.md`; clean boundary for the defendant assistant and clerk copilots. |
| Auth (staff) | **OIDC / SAML 2.0 federation + local fallback, JWT sessions** | State-agency SSO (SAML still common), OIDC for modern IdPs; `users.idp_provider`/`idp_subject` already in schema. |
| Auth (public portal) | **Email/OTP verification + scoped JWT** | Defendants self-register against `defendants.portal_email`; low-friction, no password reuse risk. |
| Payments | **Stripe hosted Checkout (reference) behind a `PaymentProcessor` interface** | PCI-DSS SAQ-A scope (card data never touches our servers); pluggable to Tyler Payments / nCourt via the same interface; webhook signature verification. |
| Notifications | **Twilio (SMS) + SMTP/SendGrid (email) behind a `NotificationChannel` interface; mail = generated PDF queue** | Hearing notices/reminders, delinquency notices; multilingual content. |
| Testing | **pytest + pytest-asyncio + httpx test client; Vitest + Playwright (frontend)** | Unit/integration/E2E. `testcontainers-python` spins real Postgres/Redis for integration tests; `axe-core` via Playwright for WCAG checks. |
| Code quality | **Ruff (lint+format), mypy (strict), `bandit` (security), pre-commit** | Government-grade quality (ISO/IEC 25010); `bandit` + OWASP ASVS checks for a security-sensitive domain. |
| Package manager | **uv (Python), pnpm (JS)** | Fast, reproducible lockfiles. |
| Containerisation | **Docker + Docker Compose; multi-stage images** | Self-host one-command deploy; same images to cloud. |
| Observability | **structlog (JSON logs), OpenTelemetry traces, Prometheus metrics** | Audit-adjacent operational logging distinct from the legal `audit_log`. |
| Secrets / config | **Pydantic Settings (env) + `.env`; KMS/Secrets Manager in cloud** | 12-factor; FIPS-validated crypto modules where state procurement requires. |

### Project Structure

```
municipal-court-management/
├── README.md
├── docker-compose.yml                 # postgres, redis, minio, api, worker, beat, staff-web, portal-web
├── docker-compose.prod.yml
├── .env.example
├── pyproject.toml                     # uv-managed; ruff/mypy/pytest config
├── uv.lock
├── alembic.ini
├── Makefile                           # make dev / test / lint / migrate / seed
├── docs/
│   ├── openapi/                        # exported openapi.json (CI-generated artifact)
│   ├── architecture.md
│   └── compliance/                     # PCI scope, CJIS audit mapping, WCAG conformance notes
├── backend/
│   ├── Dockerfile
│   ├── alembic/
│   │   └── versions/                   # one migration per phase increment
│   └── src/court/
│       ├── main.py                     # FastAPI app factory, router mount, middleware
│       ├── config.py                   # Pydantic Settings
│       ├── db/
│       │   ├── base.py                 # async engine/session
│       │   ├── models/                 # SQLAlchemy models (courts, users, defendants, cases, …)
│       │   └── repositories/           # data-access layer per aggregate
│       ├── schemas/                    # Pydantic DTOs (request/response, JSONB shapes)
│       ├── api/
│       │   ├── deps.py                 # auth, current_user, court scoping, RBAC guards
│       │   ├── routers/                # cases, defendants, hearings, dispositions,
│       │   │                           #   finances, payments, documents, warrants,
│       │   │                           #   reporting, admin, portal, webhooks, ai
│       │   └── errors.py               # RFC 9457 problem+json handlers
│       ├── services/                   # business logic
│       │   ├── intake/                 # manual + CSV + e-citation import, dedup
│       │   ├── scheduling/             # docket engine, continuances, notices
│       │   ├── disposition/            # plea/verdict, sentencing, fine assessment
│       │   ├── finance/                # fines/fees ledger, payment plans, reconciliation
│       │   ├── payments/               # PaymentProcessor interface + Stripe adapter
│       │   ├── documents/              # template render, PDF/A, e-sign
│       │   ├── warrants/               # issue/recall, NCIC tracking
│       │   ├── reporting/              # DMV abstracts, state reporting, analytics
│       │   ├── retention/              # ISO 15489 schedules, expungement
│       │   ├── notifications/          # channels, templating, multilingual
│       │   ├── rbac/                   # roles, permissions, policy checks
│       │   └── audit/                  # audit_log writer, CJIS classification
│       ├── ai/
│       │   ├── provider.py             # LLM provider abstraction
│       │   ├── ocr.py                  # OCR abstraction
│       │   ├── prompts/                # versioned prompt templates
│       │   ├── classify.py, summarise.py, draft.py, noshow.py, anomaly.py, assistant.py
│       │   └── mcp_server.py           # MCP tool/resource definitions
│       ├── integrations/
│       │   ├── ecf/                    # ECF 5.01/4.01 message (de)serialisation
│       │   ├── niem/                   # NIEM identifier mapping
│       │   ├── dmv/                    # abstract format adapters
│       │   ├── state_reporting/        # texas_oca, california_jbsis adapters
│       │   ├── rms/                    # LE RMS/CAD inbound
│       │   └── virtual_hearing/        # Zoom for Gov adapter
│       └── workers/
│           ├── celery_app.py
│           └── tasks/                  # notices, exports, ai jobs, retention, reconcile
├── frontend-staff/                     # React + Vite SPA
│   └── src/{routes,components,api,hooks,lib}
├── frontend-portal/                    # Next.js public portal (WCAG 2.2 AA, i18n)
│   └── src/{app,components,lib,messages}
└── tests/
    ├── unit/
    ├── integration/                    # testcontainers postgres/redis/minio
    ├── e2e/                            # Playwright (staff + portal)
    └── fixtures/                       # sample citations, CSV imports, ECF messages
```

---

## Phase 1: Foundation, Tenancy & Audit Spine

### Purpose
Stand up the runnable skeleton: containerised Postgres/Redis/MinIO, the FastAPI app factory, configuration, the `courts`/`users` tables, RBAC primitives, and the immutable `audit_log` with CJIS classification. Everything downstream writes to `audit_log` and is scoped by `court_id`, so this spine must exist first. After this phase the system boots, migrates, authenticates a local user, scopes all queries by court, and records audit entries.

### Tasks

#### 1.1 — Project scaffold, config, and Docker stack
**What:** uv-managed backend, Docker Compose (postgres/redis/minio/api/worker/beat), Pydantic Settings, health endpoint.

**Design:**
- `config.py` exposes `Settings`:
```python
class Settings(BaseSettings):
    database_url: PostgresDsn
    redis_url: RedisDsn
    s3_endpoint: str; s3_bucket: str; s3_access_key: str; s3_secret_key: SecretStr
    jwt_secret: SecretStr; jwt_ttl_minutes: int = 60
    environment: Literal["dev", "staging", "prod"] = "dev"
    llm_provider: Literal["anthropic", "openai", "none"] = "none"
    model_config = SettingsConfigDict(env_file=".env", env_prefix="COURT_")
```
- `main.py` `create_app()` mounts routers, exception handlers (RFC 9457 `application/problem+json`), structlog + OTel middleware, and request-scoped `correlation_id`.
- `GET /healthz` → `{status, db, redis, s3}` checking each dependency.
- `Makefile`: `dev`, `migrate`, `test`, `lint`, `seed`.

**Testing:**
- `Unit: Settings loads from env → correct typed fields, defaults applied`
- `Unit: missing COURT_DATABASE_URL → ValidationError naming the field`
- `Integration (testcontainers): GET /healthz with all deps up → 200, all "ok"`
- `Integration: redis down → /healthz returns 503 with redis "error"`

#### 1.2 — Database base, `courts` and `users` migrations
**What:** Async SQLAlchemy engine/session, Alembic baseline migration creating `courts` and `users` exactly per Data Model Suggestion 1.

**Design:**
- Models mirror the DDL (UUID PK, JSONB `config_json`/`permissions`, all CHECK constraints as enums). Court config shape validated by a Pydantic model on read/write:
```python
class CourtConfig(BaseModel):
    timezone: str = "America/Chicago"
    business_hours: BusinessHours
    courtrooms: list[str] = []
    judges: list[JudgeRef] = []
    case_number_format: str = "{year}-{type}-{seq:05d}"
    fine_grace_period_days: int = 30
    payment_plan_max_months: int = 12
    warrant_auto_issue_days: int = 60
    state_reporting_format: str | None = None
    dmv_reporting_enabled: bool = False
    virtual_hearing_platform: Literal["zoom_gov","teams","none"] = "none"
    notification_channels: list[Literal["email","sms","mail"]] = ["email"]
```
- Migration creates indexes from the DDL (`idx_courts_state`, `idx_users_role`, etc.).
- `repositories/base.py` `Repo` always requires `court_id` and rejects cross-court access.

**Testing:**
- `Integration: alembic upgrade head → courts & users tables + all indexes exist (introspect pg_indexes)`
- `Unit: CourtConfig with bad case_number_format placeholder → ValidationError`
- `Integration: insert user with role not in enum → IntegrityError`
- `Integration: UNIQUE(court_id, email) violated → IntegrityError`

#### 1.3 — Audit log (partitioned) and audit writer service
**What:** `audit_log` `RANGE`-partitioned by `created_at`, monthly partitions auto-created, and an `AuditWriter` used by all services.

**Design:**
- Migration creates the partitioned parent + a helper to create monthly partitions (Celery Beat task `ensure_audit_partitions` provisions next month).
- `AuditWriter.record(...)`:
```python
async def record(self, *, court_id: UUID, actor_type: ActorType, actor_id: str,
                 action: str, entity_type: str, entity_id: UUID,
                 changes: dict | None = None, ip: str | None = None,
                 cjis_relevant: bool = False, cjis_category: str | None = None,
                 pii_accessed: bool = False, session_id: str | None = None) -> None
```
- CJIS auto-classification: writes touching `warrants`, `defendants` criminal identifiers, or NCIC fields set `cjis_relevant=true` + category in {access, modification, deletion, query}.

**Testing:**
- `Integration: record() → row in correct monthly partition`
- `Integration: warrant modification → cjis_relevant=true, category="modification"`
- `Unit: PII-bearing entity_type → pii_accessed=true`
- `Integration: ensure_audit_partitions → next month partition created idempotently`

#### 1.4 — Auth, sessions, and RBAC guards
**What:** Local + OIDC/SAML stub auth issuing JWTs, dependency-injected `current_user`, court scoping, and role/permission guards.

**Design:**
- `POST /auth/login` (local) → JWT (`sub`, `court_id`, `role`, `perms`, `exp`).
- `deps.current_user` decodes JWT, loads user, asserts `status="active"`.
- `require(role=..., perm=...)` guard reads `users.role` + `users.permissions` JSONB (e.g. `can_issue_warrants`, `can_accept_payments`, `max_fine_waiver_cents`).
- Role→default-permission matrix defined in `rbac/policy.py`; per-user JSONB overrides.

**Testing:**
- `Integration: login valid creds → 200 + JWT with correct claims`
- `Integration: expired JWT → 401`
- `Integration: clerk hits judge-only endpoint → 403, audit "access_denied" recorded`
- `Unit: permission resolution merges role defaults + user overrides correctly`

---

## Phase 2: Defendants, Deduplication & Case Intake

### Purpose
Implement the entry point of the lifecycle: defendant records with deduplication and the central `cases` table with manual + CSV + e-citation import. After this phase a clerk can create/look up a defendant (with dedup suggestions) and file a case with a generated case number, statute, and citation data. This is the heart of "case initiation" from NCSC functional standards.

### Tasks

#### 2.1 — Defendant CRUD and deduplication engine
**What:** `defendants` table operations plus a dedup matcher that flags likely duplicates before insert.

**Design:**
- Models per DDL (JSONB `address_json`, denormalised counters).
- Dedup score over (last_name, first_name, dob, dl+dl_state, ssn_last_four) — weighted; exact DL match = strong, name+dob fuzzy (`rapidfuzz`) = medium.
```python
@dataclass
class DedupCandidate: defendant_id: UUID; score: float; matched_on: list[str]
def find_duplicates(court_id, payload) -> list[DedupCandidate]  # score desc, threshold 0.6
```
- `POST /defendants?check_dupes=true` returns `409` with candidates if score ≥ 0.85 unless `force=true`.
- `POST /defendants/{id}/merge` sets `status="merged"`, `merged_into_id`, repoints cases/fines/warrants/payments to survivor, recomputes counters; fully audit-logged.

**Testing:**
- `Unit: identical DL+state → score ≥ 0.85, matched_on includes "drivers_licence"`
- `Unit: typo'd surname + same dob → medium score via rapidfuzz`
- `Integration: create with high-confidence dupe and force=false → 409 + candidates`
- `Integration: merge → child rows repointed, counters recomputed, audit entries written`

#### 2.2 — Case creation, numbering, and statute capture
**What:** `cases` CRUD with court-config-driven case-number generation and lifecycle status.

**Design:**
- Number generator applies `courts.config_json.case_number_format` with a per-court, per-year, per-type sequence (advisory lock to avoid races).
- `CaseCreate` Pydantic DTO covers citation_number, statute_code/description, offence_date/location, filing_source, issuing_officer/agency/ori, `vehicle_json` (validated by `VehicleInfo` model for traffic types).
- Status state machine enforced in `disposition`/`scheduling` services later: `filed → pending → set_for_hearing → continued/in_progress → disposed → closed` (+ reopened/appealed/transferred/expunged/sealed). Phase 2 implements `filed`/`pending`.
- NIEM/ECF identifier fields populated when present.

**Testing:**
- `Unit: format "{year}-{type}-{seq:05d}" → "2026-TR-00042"`
- `Integration: concurrent case creation → no duplicate case_number (advisory lock)`
- `Integration: UNIQUE(court_id, case_number) enforced`
- `Unit: traffic case missing plate in vehicle_json → ValidationError`

#### 2.3 — CSV and e-citation batch import
**What:** Async import job that ingests CSV / e-citation feeds, runs dedup, creates defendants + cases, and returns a per-row result report.

**Design:**
- `POST /imports` (multipart CSV or JSON e-citation batch) → enqueues Celery task, returns `import_id`.
- Column-mapping profile per court stored in config; e-citation maps from a documented field set (officer ORI, statute, plate, defendant identity).
- Task writes an `ImportResult`: `{row, status: created|matched_existing|error, case_id?, defendant_id?, message}`; partial success allowed.
- `GET /imports/{id}` → progress + per-row results.

**Testing:**
- `Integration: 100-row CSV, 10 dupes → 90 created, 10 matched_existing, audit entries`
- `Integration: row with invalid statute/missing required field → that row "error", others proceed`
- `Fixture: sample e-citation batch → cases created with filing_source="e_citation"`
- `Integration: GET /imports/{id} reports correct counts`

---

## Phase 3: Docketing, Scheduling & Notices

### Purpose
Add calendaring — the daily workload of the court. Auto-schedule hearings against judge/courtroom availability, manage continuances as a linked chain, and generate/dispatch hearing notices and reminders. After this phase a case moves to `set_for_hearing`, appears on a judge's calendar, and the defendant receives a notice.

### Tasks

#### 3.1 — Hearing scheduling engine
**What:** `hearings` CRUD plus an availability-aware scheduler.

**Design:**
- Models per DDL (continuance chain via `continued_from_id`, virtual fields, `no_show_probability`).
- `propose_slots(court_id, case_type, hearing_type, earliest)` reads court business hours, courtrooms, judge divisions, existing hearings; returns conflict-free `(date, time, courtroom, judge_id)` slots respecting `duration_minutes`.
- `POST /hearings` validates no double-booking of judge/courtroom/time; sets case → `set_for_hearing`, updates denormalised `cases.next_hearing_id/date`.
- `GET /calendar?judge_id&date` and `?courtroom&date` for bench/courtroom views.

**Testing:**
- `Unit: propose_slots excludes occupied judge/courtroom slots`
- `Integration: booking overlapping slot → 409`
- `Integration: scheduling sets case.status="set_for_hearing" and next_hearing_date`
- `Unit: business-hours boundary (slot crossing close time) excluded`

#### 3.2 — Continuance workflow
**What:** Reschedule a hearing while preserving history.

**Design:**
- `POST /hearings/{id}/continue` → marks original `continued`, creates new hearing with `continued_from_id`, increments `cases.continuance_count`, requires `continuance_reason` + `continuance_requested_by`.
- Optional court policy cap (`max_continuances`) → warns/blocks.

**Testing:**
- `Integration: continue → new hearing chained, count incremented, original status "continued"`
- `Integration: continue beyond policy cap → 422 with reason`
- `Integration: chain traversal returns full continuance history ordered`

#### 3.3 — Notice & reminder generation and dispatch
**What:** Generate hearing notices/reminders and dispatch via channel abstraction; track sent flags.

**Design:**
- `NotificationChannel` interface (`email`, `sms`, `mail`) with multilingual templates keyed by `defendants.preferred_language`.
- On scheduling, enqueue notice; Celery Beat `send_due_reminders` dispatches reminders N days before (court config).
- Updates `hearings.notice_sent/at`, `reminder_sent/at`; writes audit entries; `mail` channel renders a PDF and queues to a print batch.

**Testing:**
- `Integration (mocked Twilio/SMTP): schedule → notice sent, notice_sent=true`
- `Unit: Spanish preferred_language → Spanish template selected`
- `Integration: send_due_reminders only targets hearings inside the window, idempotent`
- `Integration: channel send failure → retried, audit records failure then success`

---

## Phase 4: Dispositions, Fines/Fees & Cashiering

### Purpose
Deliver the financial and judicial core: record pleas/verdicts and sentencing, assess fines & fees, take payments through a PCI-scoped processor, manage payment plans, and reconcile revenue. This is where municipal courts spend most of their operational time and where incumbents are strongest, so correctness and auditability are paramount.

### Tasks

#### 4.1 — Disposition & sentencing entry
**What:** `dispositions` CRUD that closes the case and emits fine/fee assessments.

**Design:**
- Models per DDL with `sentence_json` validated by:
```python
class Sentence(BaseModel):
    fine_cents: int = 0; court_costs_cents: int = 0; surcharges_cents: int = 0
    community_service_hours: int = 0; community_service_deadline: date | None = None
    probation_months: int = 0; probation_conditions: list[str] = []
    defensive_driving: bool = False; dd_deadline: date | None = None
    licence_suspension_days: int = 0; jail_days: int = 0
    jail_suspended: bool = False; restitution_cents: int = 0
```
- `POST /cases/{id}/disposition` records disposition, sets case → `disposed`, sets `cases.disposition_id`, and (in 4.2) auto-creates `fines_fees` rows from the sentence + court fee schedule. Sets `dmv_action`/`dmv_abstract_type` when applicable (suspension/conviction).
- Appeal: `POST /dispositions/{id}/appeal` → status `appealed`, case `appealed`.

**Testing:**
- `Unit: Sentence with negative fine_cents → ValidationError`
- `Integration: guilty disposition → case "disposed", fines_fees rows created from sentence + fee schedule`
- `Integration: dismissal → no fines, dmv_abstract_type="dismissal"`
- `Integration: appeal → disposition "appealed", case "appealed"`

#### 4.2 — Fines/fees ledger & payment plans
**What:** `fines_fees` ledger with balances, waivers, and payment-plan scheduling.

**Design:**
- Each charge row tracks `amount/paid/waived/balance_cents` and status. Court fee schedule (config) maps `charge_type` → default amounts.
- `POST /fines/{id}/waive` enforces `users.permissions.max_fine_waiver_cents`; records `waived_by`/`waive_reason`.
- `POST /cases/{id}/payment-plan` builds `plan_json` (monthly, start/end, instalments) respecting `payment_plan_max_months`; sets affected charges `in_payment_plan`.
- Beat task `flag_delinquent_fines` marks overdue → triggers delinquency notice (Phase 3 channel) and eligibility for collections / warrant (Phase 6).

**Testing:**
- `Unit: waiver above user cap → 403`
- `Integration: payment plan exceeds court max months → 422`
- `Integration: delinquency sweep flags only past-due outstanding charges`
- `Unit: balance = amount - paid - waived recomputed on every mutation`

#### 4.3 — PCI-scoped payments & cashiering
**What:** `payments` via `PaymentProcessor` interface (Stripe hosted Checkout reference) + in-person cashiering; receipts; reconciliation.

**Design:**
- `PaymentProcessor` interface: `create_checkout(amount, case, fine_ids) -> {url, ref}`, `verify_webhook(payload, sig) -> Event`. **No card data ever stored** — only `processor_name`/`processor_ref` (PCI SAQ-A).
- `POST /payments/checkout` (online) returns hosted URL; webhook `POST /webhooks/payments` (signature-verified) marks payment `completed`, applies to `fines_fees`, generates receipt (`receipt_number`, PDF `receipt_url`), updates case/defendant balances.
- In-person: `POST /payments/manual` (cash/check/money_order) by cashier; `drawer_id`; sets `received_by`.
- Reconciliation report + `revenue_posted`/`gl_account` posting hook to ERP (export file or webhook).

**Testing:**
- `Integration (mocked Stripe): checkout → URL+ref returned, payment "pending"`
- `Integration (mocked): valid webhook signature → payment "completed", fines applied, balances updated`
- `Integration (mocked): invalid webhook signature → 401, no state change`
- `Integration: cash payment by user lacking can_accept_payments → 403`
- `Integration: reconciliation report sums completed payments by method for date range`
- `Security: assert no card-number/CVV fields exist on payments model or DTOs`

---

## Phase 5: Documents, Templates & Records Retention

### Purpose
Generate, store, and govern the lifecycle of court documents. Produce merge-field documents (notices, orders, judgements, abstracts) as PDF/A, store them in object storage, and enforce ISO 15489 retention with statutory expungement workflows. After this phase every case action can produce an audit-logged, archival document, and old records age out lawfully.

### Tasks

#### 5.1 — Document storage & template rendering
**What:** `documents` CRUD over S3, plus merge-field templates → HTML → PDF/A.

**Design:**
- `documents` per DDL; blobs in S3 keyed `{court_id}/{case_id}/{doc_id}.{ext}`; `file_url` = object key.
- Templates stored as `documents` with `is_template=true` + `template_merge_fields` (declared variables). `render(template_id, context)` validates required fields present, renders via Jinja→HTML→WeasyPrint→PDF, then `ocrmypdf --output-type pdfa` → `format="pdf_a"`.
- Generation is audit-logged (who/what/when), satisfying "audit-logged generation".

**Testing:**
- `Unit: render with missing required merge field → error listing the field`
- `Integration: render order template → PDF/A in S3, document row format="pdf_a", audit entry`
- `Integration: confidential doc not returned to public-scoped requester`
- `Fixture: golden template + context → stable rendered text (snapshot)`

#### 5.2 — E-signature integration
**What:** Route generated documents (orders/judgements) through an e-sign provider.

**Design:**
- `ESignProvider` interface (DocuSign adapter): `send_for_signature(doc, signer) -> envelope_id`; webhook updates `signed`/`signer_name`/`signed_at`/`signature_ref`.

**Testing:**
- `Integration (mocked DocuSign): send → envelope_id stored, status pending`
- `Integration (mocked): completion webhook → signed=true, signed_at set`

#### 5.3 — Records retention & expungement
**What:** Retention scheduler + expungement workflow per statutory rules.

**Design:**
- Per-court retention schedule (config) maps `case_type`/`document_type` → retention years → sets `cases.retention_date`, `documents.retention_expires`.
- Beat task `flag_destruction_eligible` sets `destruction_eligible=true` when expired; `flag_expungement_eligible` sets `cases.expungement_eligible` per rules (e.g. dismissals after N days, completed deferred adjudication).
- `POST /cases/{id}/expunge` (records_manager + judge) → status `expunged`, anonymises defendant linkage where required, generates expungement order, audit-logged with CJIS category "deletion".
- Destruction job removes S3 blobs for eligible documents, sets `destroyed_at` (metadata row retained for audit).

**Testing:**
- `Integration: retention sweep flags only past-expiry records`
- `Integration: dismissed case past threshold → expungement_eligible=true`
- `Integration: expunge → status "expunged", expungement order generated, CJIS deletion audit`
- `Integration: destruction job deletes blob, keeps metadata + destroyed_at`

---

## Phase 6: Warrants, Workflow Engine & Virtual Hearings (v1.1)

### Purpose
Add the enforcement and operational-flexibility layer: warrant issuance/recall with NCIC tracking and LE notification, a clerk-configurable workflow engine for case-type routing, and virtual-hearing orchestration with recording archival. These are the headline v1.1 differentiators that move the product beyond MVP.

### Tasks

#### 6.1 — Warrant issuance & recall with NCIC tracking
**What:** `warrants` lifecycle tied to FTA/FTP and bench-warrant triggers.

**Design:**
- Models per DDL; `POST /warrants` requires `can_issue_warrants`; generates warrant doc; optional `ncic_entry` with `ncic_entry_number`; places `dmv_hold_placed`.
- Beat `auto_issue_warrants` proposes bench warrants after `warrant_auto_issue_days` of FTA/FTP (proposal, not auto-final, unless court opts in).
- `POST /warrants/{id}/recall` → status `recalled`, verifies NCIC removal, releases DMV hold; CJIS-audited.
- LE notification via webhook/RMS connector.

**Testing:**
- `Integration: issue without permission → 403`
- `Integration: issue with NCIC → ncic fields + cjis_relevant audit`
- `Integration: recall → status recalled, dmv_hold_released, NCIC removal recorded`
- `Integration: auto_issue proposes only after threshold for FTA cases`

#### 6.2 — Configurable workflow engine
**What:** Clerk-defined, case-type-specific routing rules (no vendor required).

**Design:**
- Declarative rules (stored per court): `on <event> if <condition> then <actions>` where events = case/hearing/payment/disposition transitions, actions = schedule_hearing, send_notice, assess_fee, propose_warrant, generate_document.
- Rule DSL validated by Pydantic; evaluated by a deterministic engine on domain events (emitted by services).

**Testing:**
- `Unit: rule parse — invalid action → ValidationError`
- `Integration: "on disposition=guilty then generate_document(order)" fires`
- `Integration: condition false → no action; audit shows rule evaluated`

#### 6.3 — Virtual hearing integration & recording archival
**What:** Zoom-for-Government adapter creating meetings for virtual/hybrid hearings; archive recordings as documents.

**Design:**
- `VirtualHearingProvider.create_meeting(hearing) -> {join_url, meeting_id}` populates `hearings.virtual_link/virtual_meeting_id`.
- Recording webhook stores recording in S3, creates `documents` row (`document_type="hearing_recording"`, `format` mp4), links `hearings.recording_url`.

**Testing:**
- `Integration (mocked Zoom): virtual hearing → join_url + meeting_id stored`
- `Integration (mocked): recording webhook → document row + recording_url set`

---

## Phase 7: AI Engine — OCR, Classification, Summaries & Assistant (v1.1)

### Purpose
Ship the AI-native advantages. All AI output flows through `ai_suggestions` as reviewable, confidence-scored, explainable suggestions (human-in-the-loop) — never silently mutating the legal record. Adds citation OCR, case classification/statute lookup, plain-language multilingual summaries, the defendant assistant (via MCP), no-show prediction, and cashiering anomaly detection.

### Tasks

#### 7.1 — AI provider abstraction, prompt registry & suggestion store
**What:** `LLMProvider`/`OCRProvider` interfaces, versioned prompts, and `ai_suggestions` CRUD with accept/reject.

**Design:**
- `ai_suggestions` per DDL; every AI result persisted with `confidence`, `model_id/version`, `explanation`, `detail_json`, `target_entity_type/id`, status `pending`.
- `POST /ai/suggestions/{id}/accept` applies the change to the target entity (audited, `actor_type="ai"` for origin + accepting user); `reject` records reviewer.
- Prompts versioned under `ai/prompts/` with a registry mapping `suggestion_type → prompt+schema`; outputs validated against JSON Schema 2020-12.

**Testing:**
- `Unit: provider returns malformed JSON → suggestion not created, error logged`
- `Integration: accept suggestion → target entity updated, both AI and user audit entries`
- `Integration: confidence outside 0..1 rejected by DB CHECK`

#### 7.2 — Citation OCR & structured extraction
**What:** OCR a paper/PDF citation document → extracted fields → `citation_ocr` suggestion to pre-fill case creation.

**Design:**
- `OCRProvider.extract_text(blob)`; LLM structures text into `{citation_number, statute, offence, plate, defendant fields}` with per-field confidence (matches the `detail_json` example in Suggestion 1).
- Clerk reviews; accept → populates `CaseCreate`.

**Testing:**
- `Fixture: sample scanned citation → suggestion with extracted_fields + per-field confidence`
- `Integration: low overall confidence → flagged for mandatory human review`
- `Integration: accept → case prefilled from extracted fields`

#### 7.3 — Case classification, statute lookup & plain-language summaries
**What:** Classify narrative → case_type + candidate statute; summarise orders/judgements in plain language, multilingual.

**Design:**
- `classify(narrative) -> {case_type, statute_candidates[], confidence}`; statute lookup against a court statute table/config.
- `summarise(disposition) -> plain_language` rendered in defendant `preferred_language`; stored as suggestion, surfaced on portal once accepted/auto-approved per court policy.

**Testing:**
- `Unit: speeding narrative → case_type "traffic_moving", statute candidate present`
- `Integration: summary generated in Spanish for es defendant`
- `Integration: ambiguous narrative → multiple statute candidates, lower confidence`

#### 7.4 — Defendant assistant via MCP
**What:** Conversational multilingual assistant answering eligibility/payment/hearing-prep questions, backed by an MCP server exposing read-only case tools.

**Design:**
- `mcp_server.py` exposes tools: `lookup_case(case_number, last_name)`, `get_balance(case_id)`, `next_hearing(case_id)`, `payment_options(case_id)` — **read-only, court-scoped, no PII beyond the authenticated defendant's own cases**.
- Assistant chat endpoint orchestrates LLM + MCP tools; refuses legal advice, routes to plain-language explanations.

**Testing:**
- `Integration (mocked LLM): "when is my hearing?" → calls next_hearing tool, returns date`
- `Security: assistant cannot access another defendant's case via tool (court+identity scoped)`
- `Integration: legal-advice question → safe deflection response`

#### 7.5 — No-show prediction & cashiering anomaly detection
**What:** Score `hearings.no_show_probability`; flag cashiering anomalies and missing dispositions.

**Design:**
- No-show model (start: logistic regression on history — prior FTAs, days-to-hearing, notice delivered, case_type) writing `no_show_probability`; high scores propose extra reminder (workflow action).
- Anomaly detector over `payments` (unreconciled patterns, off-hours voids, drawer mismatches) and `cases` with hearings completed but no disposition → `cashiering_anomaly`/`missing_disposition` suggestions.

**Testing:**
- `Unit: defendant with prior FTAs → higher probability`
- `Integration: completed hearing without disposition → missing_disposition suggestion`
- `Integration: voided payment off-hours → cashiering_anomaly suggestion`

---

## Phase 8: Reporting, DMV/State Exports & Equity Analytics (v1.1)

### Purpose
Deliver the outputs that justify the system to administrators and regulators: standard operational reports, jurisdiction-configurable DMV abstracts and state court reporting exports, and embedded equity/disparity analytics — an underserved differentiator.

### Tasks

#### 8.1 — Operational reports & dashboards
**What:** Caseload, clearance rate, revenue/aging, judge performance reports via SQL (using the cross-domain queries from Suggestion 1) + API + staff dashboards.

**Design:**
- `GET /reports/clearance?from&to`, `/reports/revenue?from&to`, `/reports/caseload`, `/reports/aging` returning typed JSON + CSV export.
- Implemented as parameterised SQL (the clearance-rate and reconciliation queries from the data model doc), court-scoped.

**Testing:**
- `Integration: clearance rate matches hand-computed fixture`
- `Integration: revenue report groups by method/date, excludes non-completed`
- `Integration: CSV export round-trips numeric values`

#### 8.2 — DMV abstract & state reporting exports
**What:** Pluggable exporters (FTA/FTC/conviction/reinstatement abstracts; Texas OCA / California JBSIS formats) driven by `courts.config_json.state_reporting_format`.

**Design:**
- `StateReportExporter` interface + per-state adapters; `DMVAbstractExporter` builds abstracts from `cases.dmv_abstract_type`/disposition `dmv_action`, sets `dmv_abstract_sent`/`dmv_sent_at`.
- Beat batch job produces the configured format; NIEM identifiers used where the format requires them. ECF-aligned where applicable.

**Testing:**
- `Fixture: disposed conviction → Texas OCA record matches golden output`
- `Integration: DMV abstract export sets dmv_abstract_sent + audit`
- `Unit: unsupported state_reporting_format → clear configuration error`

#### 8.3 — Equity & disparity analytics
**What:** Disparity monitoring across case-type outcomes, fines, and dispositions.

**Design:**
- Aggregations over `dispositions`/`fines_fees`/`defendants` (using only lawful, configurable attributes) producing disparity indicators with documented methodology; surfaced to administrators with caveats.
- Privacy-safe: small-cell suppression to avoid re-identification.

**Testing:**
- `Integration: disparity report computes outcome distribution by configured dimension`
- `Unit: cells below suppression threshold are masked`

---

## Phase 9: Public Defendant Portal (WCAG 2.2 AA, Multilingual)

### Purpose
Deliver the accessible, multilingual, mobile-first public surface that incumbents handle poorly: case lookup, plea entry, online payment, and hearing-reschedule requests. This is a primary differentiator and must meet ADA Title II / WCAG 2.2 AA.

### Tasks

#### 9.1 — Portal auth & case lookup
**What:** Defendant self-registration (email/OTP) and secure lookup of their own cases.

**Design:**
- `POST /portal/register` (citation_number + last_name + dob → OTP to email/phone) → verified `defendants.portal_*` flags + scoped JWT.
- `GET /portal/cases` returns only the authenticated defendant's cases with plain-language summaries (from Phase 7).

**Testing:**
- `Integration: register with mismatched identity → no access, generic error (no enumeration)`
- `Security: portal token cannot read another defendant's case`
- `E2E (Playwright + axe): lookup page passes WCAG 2.2 AA automated checks`

#### 9.2 — Plea entry, payment & reschedule requests
**What:** Online plea (guilty/no-contest where permitted), payment via hosted checkout, and reschedule requests.

**Design:**
- `POST /portal/cases/{id}/plea` (allowed types per case_type config) → creates disposition for eligible offences or routes to clerk review.
- Payment reuses Phase 4 hosted checkout.
- `POST /portal/cases/{id}/reschedule-request` → clerk queue (not auto-approved).

**Testing:**
- `Integration: plea on ineligible case_type → routed to clerk, not auto-disposed`
- `E2E: defendant pays → balance reflected, receipt downloadable`
- `E2E (axe): plea + payment flows pass WCAG 2.2 AA`
- `Unit: multilingual — portal renders es/en strings correctly`

---

## Phase 10: Open API Surface, Webhooks, Integrations & Hardening

### Purpose
Finalise the day-one differentiator (published OpenAPI 3.1 + webhooks + SDKs), wire remaining external connectors, and complete security/compliance hardening for government procurement. After this phase the platform is integration-ready and audit-ready.

### Tasks

#### 10.1 — Published OpenAPI 3.1, API keys & outbound webhooks
**What:** Export the OpenAPI 3.1 spec as a CI artifact, add API-key auth for partner clients, and emit signed outbound webhooks.

**Design:**
- CI step exports `docs/openapi/openapi.json`; generate a Python + TypeScript SDK.
- Partner API keys (scoped, court-bound) alongside JWT; rate-limited via Redis.
- Outbound webhook subscriptions (case.filed, disposition.entered, payment.completed, warrant.issued) with HMAC signatures + retry/backoff.

**Testing:**
- `Integration: openapi.json validates against OpenAPI 3.1 schema in CI`
- `Integration: outbound webhook signed with HMAC; receiver verifies`
- `Integration: rate limit exceeded → 429`

#### 10.2 — LE RMS/CAD & ECF/NIEM exchange
**What:** Inbound RMS/CAD citation connector and ECF 5.01 (4.01-compat) message (de)serialisation with NIEM identifier mapping.

**Design:**
- `integrations/ecf/` (de)serialises FilingMessage/ReviewFilingMessage/DocumentMetadata to/from `cases`/`documents` using their `ecf_*`/`niem_*` fields.
- RMS connector maps inbound feed → import pipeline (Phase 2.3).

**Testing:**
- `Fixture: sample ECF 5.01 FilingMessage → case + document rows with ecf_filing_id`
- `Fixture: case → ECF message round-trips (semantic equality)`
- `Integration: ECF 4.01 input still parses (compat)`

#### 10.3 — Security & compliance hardening
**What:** OWASP ASVS/API-Top-10 pass, CJIS audit verification, PCI scope assertion, TLS, secrets, dependency scanning.

**Design:**
- `bandit` + dependency audit in CI; enforce TLS 1.3; verify every criminal-justice access path writes a CJIS-classified audit entry; document PCI SAQ-A scope (no card data); penetration-test checklist; rate limiting and authz tests for every router.
- Compliance docs under `docs/compliance/` mapping controls to NIST SP 800-53 AU/AC families.

**Testing:**
- `Security: automated authz matrix test — each endpoint × role → expected allow/deny`
- `Security: bandit + dependency scan clean (no high severity) in CI`
- `Compliance: every warrant/criminal-data read produces cjis_relevant audit (assertion test)`
- `Security: assert TLS-only, no card data persisted anywhere (schema + log scan)`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation, Tenancy & Audit Spine        ─── required by everything
    │
Phase 2: Defendants, Dedup & Case Intake          ─── requires Phase 1
    │
Phase 3: Docketing, Scheduling & Notices          ─── requires Phase 2
    │
Phase 4: Dispositions, Fines & Cashiering         ─── requires Phase 2 (uses Phase 3 notices for delinquency)
    │
Phase 5: Documents, Templates & Retention         ─── requires Phase 2; integrates Phase 4 (orders/judgements)
    │
    ├── Phase 6: Warrants, Workflow, Virtual Hearings ─ requires Phases 3,4,5; can parallel Phase 7
    ├── Phase 7: AI Engine                            ─ requires Phase 2 (data); can parallel Phase 6
    └── Phase 8: Reporting, DMV/State, Equity         ─ requires Phase 4; can parallel Phases 6,7
         │
Phase 9: Public Defendant Portal                  ─── requires Phases 2,3,4,5 (uses Phase 7 summaries if available)
    │
Phase 10: Open API, Webhooks, Integrations, Hardening ─ requires all prior; integrations 10.2 need Phase 5
```

**Parallelism opportunities:**
- After Phase 5, **Phases 6, 7, and 8 can be developed concurrently** by separate workstreams (warrants/workflow, AI engine, reporting) — they share Phase 2–5 foundations but touch largely disjoint modules.
- Within Phase 7, OCR (7.2), classification/summaries (7.3), and no-show/anomaly (7.5) can be parallelised once 7.1 (provider abstraction + suggestion store) lands.
- Frontend-staff and frontend-portal can be built incrementally alongside their backing API phases; the portal (Phase 9) only finalises once Phases 2–5 APIs are stable.

**MVP cut line:** Phases 1–5 plus Phase 9 (portal) and Phase 10.1 (published API) constitute the features.md "Must-have (MVP)" set. Phases 6–8 are "Should-have (v1.1)". Backlog items (jury management, STT transcripts, AI drafting beyond 7.x, predictive intervention engine) extend Phases 6–8.

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks in the phase are implemented.
2. All unit and integration tests for the phase pass; new code has meaningful coverage (target ≥ 85% for service layer).
3. `ruff` lint + format pass with no errors.
4. `mypy --strict` passes for new/changed modules.
5. `bandit` and dependency scan report no high-severity findings.
6. `docker compose up` builds and runs the full stack; `make migrate` applies cleanly and is reversible (`alembic downgrade -1`).
7. The feature works end-to-end against a real Postgres/Redis/MinIO (testcontainers or compose) — not only mocked.
8. New/changed endpoints appear in the auto-generated OpenAPI 3.1 spec; `openapi.json` artifact regenerated.
9. Alembic migration(s) created for any schema change, additive and non-destructive to prior phases.
10. Every state-changing operation writes an `audit_log` entry; criminal-justice/PII paths set CJIS/PII flags.
11. New config options are documented in `.env.example` and `docs/`.
12. Public-facing surfaces (Phase 9) pass automated WCAG 2.2 AA checks (axe) and manual keyboard/screen-reader smoke test.
13. No cardholder data persisted anywhere (asserted by test) — PCI SAQ-A scope preserved.
14. Phase compliance notes updated in `docs/compliance/` where the phase touches NIEM/ECF/CJIS/retention.
```
