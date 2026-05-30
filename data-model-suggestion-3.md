# Data Model Suggestion 3: Event-Sourced / Audit-First

> Project: Municipal Court Management · Created: 2026-05-26

## Philosophy

This model treats every state change in the court system as an immutable event in a single append-only store. The event store is the sole source of truth — citation filings, hearing scheduling, plea entries, disposition recordings, fine assessments, payments, warrant issuances, document filings, and notification deliveries are all recorded as typed events on domain streams. Materialised read models (CQRS projections) are rebuilt from these events to serve the operational queries that clerks, judges, and defendants need.

This architecture is particularly compelling for municipal courts because the domain has inherent audit, transparency, and due-process requirements that are difficult to retrofit onto a CRUD model. Every case action — from filing through disposition — must be traceable for appeals, public records requests, and judicial oversight. The event-sourced approach makes the audit trail the primary data structure rather than a secondary logging concern. When a defendant appeals a conviction, the complete event history provides an authoritative record of every action, hearing, and decision. When a journalist files a public records request, the event stream for a case can be replayed into a redacted timeline without querying across multiple tables.

The approach also enables the platform's key AI differentiators: no-show prediction models can train on the full hearing event history, disparity analytics can process disposition events across case types and demographics, and anomaly detection can flag irregular patterns in cashiering events — all from the same event stream.

**Best for:** Courts requiring complete audit trails, appeals-ready case histories, public records transparency, and rich event data for AI analytics and disparity monitoring.

**Trade-offs:**
- Pro: Complete, immutable audit trail satisfying due-process, appeals, and public records requirements
- Pro: Temporal queries ("what was the case status on March 15?") are trivial
- Pro: No separate audit logging system — the event store IS the court record
- Pro: AI training on full event history for no-show prediction and disparity analytics
- Pro: Public records requests answered by replaying and redacting a case event stream
- Con: Read models must be projected and maintained
- Con: Event schema evolution requires careful versioning
- Con: Eventually consistent read models may lag during bulk citation imports
- Con: Steeper learning curve for court IT staff unfamiliar with CQRS

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| NIEM Justice Domain | Case and person entities projected into `rm_case_docket` and `rm_defendant_account` with NIEM identifiers |
| OASIS LegalXML ECF 5.01 | `case.efiling_received` and `case.efiling_accepted` events model ECF message lifecycle |
| NCSC JTC Functional Standards | All NCSC functional areas covered by event types: initiation, calendaring, disposition, financials |
| CJIS Security Policy | Every event is an audit entry; `cjis_relevant` flag on events touching criminal justice information |
| NIST SP 800-53 | Immutable event store satisfies AU (Audit) control family by design |
| ISO 15489 Records Management | Document lifecycle events track retention and disposition of court records |
| PCI-DSS v4.0 | Payment events store processor references only; card data never enters the event store |
| WCAG 2.1 AA | Notification events carry language codes for accessible multilingual communication |
| CloudEvents 1.0 | Event envelope follows CloudEvents spec for interoperability |
| PDF/A (ISO 19005) | Document events track archival format designation |

---

## Event Store Infrastructure

```sql
CREATE TABLE event_store (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type     TEXT NOT NULL CHECK (stream_type IN (
                        'court', 'defendant', 'case', 'hearing',
                        'financial', 'warrant', 'document',
                        'notification', 'ai', 'config'
                    )),
    stream_id       TEXT NOT NULL,              -- e.g. "case:2026-TR-00123" or "defendant:def-uuid"
    sequence_num    BIGINT NOT NULL,
    event_type      TEXT NOT NULL,
    event_data      JSONB NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- CloudEvents envelope
    ce_source       TEXT NOT NULL,              -- system component origin
    ce_specversion  TEXT NOT NULL DEFAULT '1.0',
    ce_type         TEXT NOT NULL,
    ce_time         TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- Court isolation
    court_id        UUID NOT NULL,
    -- Actor
    actor_type      TEXT NOT NULL CHECK (actor_type IN (
                        'user', 'defendant', 'system', 'ai',
                        'api_client', 'efsp', 'payment_processor',
                        'dmv_feed', 'rms_connector', 'scheduler',
                        'ocr_engine', 'notification_provider'
                    )),
    actor_id        TEXT NOT NULL,
    -- Tracing
    correlation_id  TEXT,
    causation_id    UUID,
    -- Compliance
    cjis_relevant   BOOLEAN NOT NULL DEFAULT false,
    pii_involved    BOOLEAN NOT NULL DEFAULT false,
    public_record   BOOLEAN NOT NULL DEFAULT true,  -- eligible for public records response
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, sequence_num)
) PARTITION BY RANGE (ce_time);

CREATE INDEX idx_events_stream ON event_store (stream_id, sequence_num);
CREATE INDEX idx_events_court ON event_store (court_id, ce_time DESC);
CREATE INDEX idx_events_type ON event_store (event_type, ce_time DESC);
CREATE INDEX idx_events_correlation ON event_store (correlation_id);
CREATE INDEX idx_events_causation ON event_store (causation_id);
CREATE INDEX idx_events_actor ON event_store (actor_type, actor_id);
CREATE INDEX idx_events_cjis ON event_store (court_id) WHERE cjis_relevant = true;
CREATE INDEX idx_events_public ON event_store (court_id, stream_type) WHERE public_record = true;

CREATE TABLE stream_snapshots (
    stream_id       TEXT NOT NULL,
    snapshot_at     BIGINT NOT NULL,
    state           JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_at)
);

CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_sequence   BIGINT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Event Taxonomy

### Court & Configuration Events
```
court.created                   — new court/municipality onboarded
court.configured                — jurisdiction, policies, or integrations updated
court.staff_added               — staff member added with role
court.staff_role_changed        — RBAC role changed
court.staff_deactivated         — staff account deactivated
court.fee_schedule_updated      — fine/fee schedule changed
court.judge_assigned            — judge added to court roster
config.integration_configured   — payment processor, RMS, DMV feed configured
```

### Defendant Events
```
defendant.created               — defendant record created
defendant.identity_updated      — name, address, contact changed
defendant.duplicate_detected    — potential duplicate flagged
defendant.merged                — duplicate records merged
defendant.portal_registered     — defendant created portal account
defendant.portal_verified       — portal identity verified
defendant.anonymised            — defendant data anonymised (privacy request)
```

### Case Lifecycle Events
```
case.filed                      — case/citation filed (manual, e-citation, EFSP, CSV import)
case.classified                 — case type and statute assigned (manual or AI)
case.assigned_judge             — judge assigned to case
case.assigned_prosecutor        — prosecutor assigned
case.efiling_received           — ECF e-filing message received from EFSP
case.efiling_accepted           — ECF filing accepted by court
case.efiling_rejected           — ECF filing rejected with reason
case.status_changed             — case status transition
case.transferred                — case transferred to another court
case.reopened                   — closed case reopened
case.sealed                     — case sealed by court order
case.expunged                   — case records expunged
```

### Hearing Events
```
hearing.scheduled               — hearing date/time/courtroom set
hearing.notice_sent             — notice of hearing sent to defendant
hearing.reminder_sent           — hearing reminder sent
hearing.confirmed               — defendant confirmed attendance
hearing.continued               — hearing continued (with reason and requester)
hearing.started                 — hearing in progress
hearing.completed               — hearing concluded with outcome
hearing.defendant_no_show       — defendant failed to appear
hearing.virtual_link_created    — Zoom/Teams link generated
hearing.recording_archived      — hearing recording saved
hearing.no_show_predicted       — AI predicted no-show probability
```

### Disposition Events
```
disposition.plea_entered        — plea entered (guilty, not guilty, nolo)
disposition.verdict_rendered    — judge rendered verdict
disposition.sentence_imposed    — sentence recorded (fines, community service, probation, etc.)
disposition.deferred            — deferred adjudication/disposition ordered
disposition.dismissed           — case dismissed
disposition.appealed            — defendant filed appeal
disposition.reversed            — disposition reversed on appeal
disposition.vacated             — disposition vacated
disposition.dmv_action_ordered  — DMV suspension/reinstatement ordered
```

### Financial Events
```
financial.fine_assessed         — fine or fee assessed on case
financial.surcharge_added       — state surcharge applied
financial.court_costs_assessed  — court costs assessed
financial.payment_received      — payment received (card, cash, check, ACH)
financial.payment_failed        — payment attempt failed
financial.payment_refunded      — refund issued
financial.fine_waived           — fine waived by judge/clerk
financial.payment_plan_created  — instalment payment plan established
financial.payment_plan_payment  — instalment payment received
financial.payment_plan_defaulted — defendant missed instalment
financial.late_fee_assessed     — late fee applied
financial.sent_to_collections   — account referred to collections
financial.collections_remittance — collections agency remitted payment
financial.revenue_posted        — payment posted to municipal ERP/GL
financial.drawer_reconciled     — cash drawer reconciled at end of day
```

### Warrant Events
```
warrant.issued                  — warrant issued (FTA, FTC, bench, arrest)
warrant.ncic_entered            — warrant entered in NCIC/state system
warrant.served                  — warrant served by law enforcement
warrant.recalled                — warrant recalled by judge
warrant.quashed                 — warrant quashed
warrant.dmv_hold_placed         — DMV licence hold placed
warrant.dmv_hold_released       — DMV hold released
```

### Document Events
```
document.filed                  — document filed on case
document.generated              — document generated from template
document.signed                 — document e-signed
document.served                 — document served on party
document.redacted               — document redacted for public access
document.retention_set          — retention schedule assigned
document.destruction_eligible   — document reached retention expiry
document.destroyed              — document destroyed per retention schedule
```

### Notification Events
```
notification.created            — notification queued
notification.sent               — notification dispatched
notification.delivered          — delivery confirmed
notification.failed             — delivery failed
notification.bounced            — email/SMS bounced
```

### AI Events
```
ai.citation_ocr_completed      — citation text extracted from image/PDF
ai.case_classified              — AI suggested case type and statute
ai.document_drafted             — AI drafted order, notice, or warrant
ai.plain_language_generated     — AI generated plain-language summary for defendant
ai.translation_completed        — AI translated notification to defendant's language
ai.no_show_predicted            — AI predicted no-show probability for hearing
ai.cashiering_anomaly_detected  — AI flagged irregular payment pattern
ai.missing_disposition_flagged  — AI flagged case without disposition past deadline
ai.disparity_detected           — AI flagged outcome disparity across demographics
ai.suggestion_accepted          — clerk/judge accepted AI suggestion
ai.suggestion_rejected          — clerk/judge rejected AI suggestion
```

---

## Read Models (CQRS Projections)

```sql
CREATE TABLE rm_case_docket (
    case_id             UUID PRIMARY KEY,
    court_id            UUID NOT NULL,
    case_number         TEXT NOT NULL,
    case_type           TEXT NOT NULL,
    status              TEXT NOT NULL,
    defendant_id        UUID NOT NULL,
    defendant_name      TEXT NOT NULL,
    charge              JSONB NOT NULL DEFAULT '{}',
    -- Example: {"citation": "T-2026-12345", "statute": "545.352", "offence": "Speeding 45/30", "date": "2026-05-01"}
    filing              JSONB NOT NULL DEFAULT '{}',
    assigned_judge      TEXT,
    prosecutor          TEXT,
    hearings            JSONB NOT NULL DEFAULT '[]',
    disposition         JSONB NOT NULL DEFAULT '{}',
    financial           JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "fines_assessed_cents": 40700, "paid_cents": 10700, "waived_cents": 0,
    --   "balance_cents": 30000, "payment_plan_active": false,
    --   "collections": false
    -- }
    warrants            JSONB NOT NULL DEFAULT '[]',
    documents           JSONB NOT NULL DEFAULT '[]',
    dmv                 JSONB NOT NULL DEFAULT '{}',
    retention           JSONB NOT NULL DEFAULT '{}',
    next_hearing_date   TIMESTAMPTZ,
    continuances        INTEGER NOT NULL DEFAULT 0,
    last_event_seq      BIGINT NOT NULL DEFAULT 0,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_docket_court ON rm_case_docket (court_id);
CREATE INDEX idx_rm_docket_status ON rm_case_docket (court_id, status);
CREATE INDEX idx_rm_docket_type ON rm_case_docket (court_id, case_type);
CREATE INDEX idx_rm_docket_number ON rm_case_docket (court_id, case_number);
CREATE INDEX idx_rm_docket_defendant ON rm_case_docket (defendant_id);
CREATE INDEX idx_rm_docket_hearing ON rm_case_docket (next_hearing_date) WHERE status IN ('pending', 'set_for_hearing', 'continued');
CREATE INDEX idx_rm_docket_balance ON rm_case_docket (court_id) WHERE (financial->>'balance_cents')::BIGINT > 0;

CREATE TABLE rm_defendant_account (
    defendant_id        UUID PRIMARY KEY,
    court_id            UUID NOT NULL,
    name                TEXT NOT NULL,
    identity            JSONB NOT NULL DEFAULT '{}',
    status              TEXT NOT NULL,
    portal              JSONB NOT NULL DEFAULT '{}',
    active_cases        JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"case_id": "uuid", "number": "2026-TR-00123", "type": "traffic_moving", "status": "set_for_hearing", "balance_cents": 30000}]
    financial_summary   JSONB NOT NULL DEFAULT '{}',
    -- Example: {"outstanding_cents": 30000, "lifetime_paid_cents": 50000, "payment_plans": 0, "collections": false}
    active_warrants     JSONB NOT NULL DEFAULT '[]',
    upcoming_hearings   JSONB NOT NULL DEFAULT '[]',
    last_event_seq      BIGINT NOT NULL DEFAULT 0,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_def_court ON rm_defendant_account (court_id);
CREATE INDEX idx_rm_def_status ON rm_defendant_account (court_id, status);
CREATE INDEX idx_rm_def_identity ON rm_defendant_account USING GIN (identity);

CREATE TABLE rm_daily_calendar (
    court_id            UUID NOT NULL,
    calendar_date       DATE NOT NULL,
    courtroom           TEXT NOT NULL,
    hearings            JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {"hearing_id": "uuid", "time": "09:00", "duration_min": 15,
    --    "case_number": "2026-TR-00123", "case_type": "traffic_moving",
    --    "defendant": "John Doe", "charge": "Speeding 45/30",
    --    "hearing_type": "arraignment", "format": "in_person",
    --    "no_show_probability": 0.12, "status": "scheduled"}
    -- ]
    judge               TEXT,
    total_hearings      INTEGER NOT NULL DEFAULT 0,
    completed           INTEGER NOT NULL DEFAULT 0,
    no_shows            INTEGER NOT NULL DEFAULT 0,
    continued           INTEGER NOT NULL DEFAULT 0,
    last_event_seq      BIGINT NOT NULL DEFAULT 0,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (court_id, calendar_date, courtroom)
);

CREATE TABLE rm_revenue_dashboard (
    court_id            UUID NOT NULL,
    period              DATE NOT NULL,          -- daily revenue summary
    revenue             JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "assessed_cents": 120000, "collected_cents": 85000,
    --   "waived_cents": 5000, "refunded_cents": 2000,
    --   "by_method": {"card_online": 45000, "card_in_person": 25000, "cash": 10000, "check": 5000},
    --   "by_charge_type": {"fine": 60000, "court_costs": 20000, "surcharge": 5000},
    --   "payment_plans_active": 15,
    --   "sent_to_collections_cents": 30000,
    --   "processor_fees_cents": 1800,
    --   "reconciled": true
    -- }
    caseload            JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "filed": 12, "disposed": 8, "clearance_rate_pct": 66.7,
    --   "by_type": {"traffic_moving": 7, "parking": 3, "ordinance": 2},
    --   "warrants_issued": 2, "warrants_recalled": 1,
    --   "hearings_held": 25, "no_shows": 3
    -- }
    last_event_seq      BIGINT NOT NULL DEFAULT 0,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (court_id, period)
);

CREATE TABLE rm_disparity_monitor (
    court_id            UUID NOT NULL,
    period              DATE NOT NULL,          -- monthly analysis
    analysis            JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "dispositions_analysed": 450,
    --   "by_case_type": {
    --     "traffic_moving": {
    --       "total": 300, "guilty": 200, "dismissed": 50, "deferred": 50,
    --       "avg_fine_cents": 22000, "median_fine_cents": 20000,
    --       "payment_plan_pct": 35.0
    --     }
    --   },
    --   "by_zip_code": {...},
    --   "disparity_flags": [
    --     {"metric": "dismissal_rate", "group_a": "zip_75001", "group_b": "zip_75002",
    --      "rate_a": 0.08, "rate_b": 0.22, "significance": 0.03}
    --   ]
    -- }
    last_event_seq      BIGINT NOT NULL DEFAULT 0,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (court_id, period)
);
```

---

## Event-Driven Query Examples

### Complete case history for appeal

```sql
-- Full event timeline for a case, providing authoritative appeals record
SELECT ce_time, event_type, actor_type, actor_id,
       event_data, public_record
FROM event_store
WHERE stream_id = 'case:2026-TR-00123'
  AND court_id = $1
ORDER BY sequence_num ASC;
```

### Public records request (redacted timeline)

```sql
-- Events eligible for public disclosure on a specific case
SELECT ce_time, event_type,
       CASE WHEN pii_involved THEN '{"redacted": true}'::JSONB ELSE event_data END AS event_data
FROM event_store
WHERE stream_id = 'case:2026-TR-00123'
  AND court_id = $1
  AND public_record = true
ORDER BY sequence_num ASC;
```

### Cashiering anomaly detection (event pattern analysis)

```sql
-- All financial events by a specific cashier in a time window
SELECT ce_time, event_type, event_data
FROM event_store
WHERE court_id = $1
  AND stream_type = 'financial'
  AND actor_id = $2
  AND ce_time BETWEEN $3 AND $4
ORDER BY ce_time;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Infrastructure | 3 | Partitioned event store + snapshots + projection checkpoints |
| Read Models | 5 | Case docket, defendant account, daily calendar, revenue dashboard, disparity monitor |
| **Total** | **8** | |

---

## Key Design Decisions

1. **Single unified event store for all court operations** — case lifecycle, hearings, financials, warrants, and documents share one partitioned event store. This means the complete history of any case is a single-stream query, providing the authoritative record for appeals and public records requests.

2. **CloudEvents envelope** — `ce_source`, `ce_type`, `ce_specversion`, and `ce_time` follow the CloudEvents 1.0 spec, enabling events to be published to state reporting systems and partner agencies in a standard format.

3. **Public records flag on every event** — `public_record` marks events eligible for disclosure in public records requests. Events involving sealed cases, juvenile records, or CJIS-protected information are flagged `false`. This enables automated redacted timeline generation.

4. **10 stream types covering full court lifecycle** — court, defendant, case, hearing, financial, warrant, document, notification, ai, config. Each has its own event taxonomy but shares the same storage.

5. **CJIS compliance by design** — `cjis_relevant` flag on events touching criminal justice information (warrant NCIC entries, criminal history queries) enables CJIS audit reporting without separate logging infrastructure.

6. **Five read models for five user personas** — `rm_case_docket` (clerks), `rm_defendant_account` (defendant portal), `rm_daily_calendar` (judges and courtroom staff), `rm_revenue_dashboard` (court administrator and finance), `rm_disparity_monitor` (presiding judge and oversight). Each is optimised for its primary use case.

7. **Disparity monitor as a projected aggregate** — `rm_disparity_monitor` aggregates disposition events by case type, geography, and other dimensions to surface outcome disparities. Because it's a projection, the analysis methodology can be updated and replayed without modifying stored data.

8. **Financial events as a complete ledger** — the financial stream captures every monetary event (assessment, payment, waiver, refund, collection referral, ERP posting) in sequence. The `rm_revenue_dashboard` projects this into daily summaries for the court administrator, while the raw events support cashiering reconciliation and anomaly detection.

9. **Hearing no-show as a predictive event** — `hearing.no_show_predicted` events carry the AI-computed probability and are linked via `causation_id` to the `hearing.scheduled` event. This enables evaluation of prediction accuracy by comparing predictions to `hearing.defendant_no_show` outcomes.

10. **ECF filing lifecycle as events** — `case.efiling_received` → `case.efiling_accepted` / `case.efiling_rejected` models the OASIS LegalXML ECF message exchange as events, providing a complete audit trail of the e-filing workflow for both the court and the filing service provider.

11. **Document lifecycle events** — rather than tracking documents as mutable records, the event stream captures `document.filed` → `document.signed` → `document.served` → `document.retention_set` → `document.destruction_eligible` → `document.destroyed`, providing a complete chain of custody for records management compliance (ISO 15489).

12. **PII flagging for privacy compliance** — `pii_involved` on events enables GDPR/CCPA-style data subject access requests: querying all events that touched a defendant's PII gives a complete processing record.
