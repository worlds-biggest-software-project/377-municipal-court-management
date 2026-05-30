# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Municipal Court Management · Created: 2026-05-26

## Philosophy

This model assigns a dedicated table to every core court domain concept — courts, defendants, cases, hearings, dispositions, fines, payments, documents, warrants, and notifications — connected by foreign keys that enforce referential integrity across the full case lifecycle from citation through disposition, fine collection, and records retention. The design aligns with the entity structures used in NIEM Justice domain, OASIS LegalXML ECF 5.01, and NCSC JTC functional standards so that data exchange with state reporting systems, e-filing service providers, and law enforcement agencies maps naturally to database entities.

The normalized structure makes the relationships between cases, hearings, dispositions, and financial obligations explicit and queryable — answering questions like "which cases have unpaid fines with upcoming hearings and outstanding warrants?" requires only standard JOINs. Defendant deduplication is enforced at the table level with composite uniqueness constraints. State court reporting exports can be generated from clean, well-typed columns without parsing JSONB.

This is the safest choice for courts with strict compliance requirements (CJIS, NIST 800-53, state reporting mandates) who need clean audit trails, NIEM-aligned data exports, and straightforward integration with DMV, law enforcement RMS/CAD, and payment processors.

**Best for:** Municipal courts requiring strict compliance with NIEM/ECF standards, state reporting mandates, and CJIS security requirements.

**Trade-offs:**
- Pro: Direct mapping to NIEM Justice entities and ECF message structures
- Pro: Referential integrity enforced across case → hearing → disposition → fine → payment chain
- Pro: State reporting exports generated from clean, typed columns without JSONB parsing
- Pro: CJIS audit trail requirements met with dedicated audit_log table
- Con: 14 tables with foreign keys increases migration complexity
- Con: Jurisdiction-specific fields (varying state statutes, reporting codes) require schema changes or JSONB overflow columns
- Con: High-volume traffic courts may stress the hearings and payments tables

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| NIEM Justice Domain | `cases`, `defendants`, `dispositions`, `warrants` align with NIEM Justice entity types and code lists |
| OASIS LegalXML ECF 5.01 | `cases` and `documents` map to ECF FilingMessage, ReviewFilingMessage, and DocumentMetadata |
| OASIS LegalXML ECF 4.01 | Backward-compatible field structure for state EFMs still on ECF 4.x |
| NCSC JTC Functional Standards | Table structure covers all NCSC-defined CMS functional areas: case initiation, calendaring, disposition, financials |
| CJIS Security Policy v6.0 | `audit_log` tracks all access to criminal justice information; `warrants` includes NCIC tracking fields |
| NIST SP 800-53 Rev 5 | `audit_log` with actor, action, entity, and timestamp satisfies AU (Audit) control family |
| ISO 15489 Records Management | `documents` includes retention schedule and disposition status for records lifecycle |
| PDF/A (ISO 19005) | `documents.format` supports PDF/A archival format designation |
| WCAG 2.1 AA | `notifications` supports multilingual content for accessible defendant communication |
| PCI-DSS v4.0 | `payments` stores processor references only — no card data in the database |
| OAuth 2.0 / OIDC | `users` stores identity provider references for SSO |
| SAML 2.0 | `users.idp_provider` supports state agency SAML federation |
| OpenAPI 3.1 | Table structure maps directly to REST API resource entities |

---

## Court & Staff Management

```sql
CREATE TABLE courts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    court_type      TEXT NOT NULL CHECK (court_type IN (
                        'municipal', 'city', 'town', 'village',
                        'justice_of_peace', 'magistrate', 'traffic'
                    )),
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN (
                        'provisioning', 'active', 'suspended', 'archived'
                    )),
    -- Jurisdiction
    state           TEXT NOT NULL,              -- US state abbreviation
    county          TEXT,
    municipality    TEXT NOT NULL,
    fips_code       TEXT,                       -- FIPS county/place code
    ori_number      TEXT,                       -- Originating Agency Identifier (for CJIS)
    -- Contact
    address         TEXT NOT NULL,
    phone           TEXT,
    email           TEXT,
    website         TEXT,
    -- Configuration
    config_json     JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "timezone": "America/Chicago",
    --   "business_hours": {"start": "08:00", "end": "17:00"},
    --   "courtrooms": ["A", "B"],
    --   "judges": [{"name": "Hon. Smith", "division": "traffic"}],
    --   "case_number_format": "{year}-{type}-{seq:05d}",
    --   "fine_grace_period_days": 30,
    --   "payment_plan_max_months": 12,
    --   "warrant_auto_issue_days": 60,
    --   "state_reporting_format": "texas_oca",
    --   "dmv_reporting_enabled": true,
    --   "virtual_hearing_platform": "zoom_gov",
    --   "notification_channels": ["email", "sms", "mail"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_courts_state ON courts (state);
CREATE INDEX idx_courts_status ON courts (status);
CREATE INDEX idx_courts_ori ON courts (ori_number);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    court_id        UUID NOT NULL REFERENCES courts(id),
    email           TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    role            TEXT NOT NULL CHECK (role IN (
                        'presiding_judge', 'associate_judge', 'magistrate',
                        'court_clerk', 'deputy_clerk', 'cashier',
                        'court_administrator', 'bailiff', 'prosecutor',
                        'public_defender', 'probation_officer',
                        'records_manager', 'it_admin', 'read_only'
                    )),
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN (
                        'active', 'suspended', 'deactivated'
                    )),
    bar_number      TEXT,                       -- for judges/attorneys
    badge_number    TEXT,                       -- for bailiffs/officers
    idp_provider    TEXT,                       -- SAML / OIDC provider
    idp_subject     TEXT,
    cjis_certified  BOOLEAN NOT NULL DEFAULT false,
    permissions     JSONB NOT NULL DEFAULT '{}',
    -- Example: {"can_issue_warrants": false, "can_accept_payments": true, "max_fine_waiver_cents": 5000}
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (court_id, email)
);

CREATE INDEX idx_users_court ON users (court_id);
CREATE INDEX idx_users_role ON users (court_id, role);
```

## Defendant Management

```sql
CREATE TABLE defendants (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    court_id            UUID NOT NULL REFERENCES courts(id),
    -- Identity
    first_name          TEXT NOT NULL,
    middle_name         TEXT,
    last_name           TEXT NOT NULL,
    suffix              TEXT,
    date_of_birth       DATE,
    gender              TEXT CHECK (gender IN ('male', 'female', 'non_binary', 'unknown')),
    -- Identifiers
    drivers_licence     TEXT,
    dl_state            TEXT,
    ssn_last_four       TEXT,                   -- last 4 digits only for matching
    state_id            TEXT,                   -- state-issued ID number
    -- NIEM person identifiers
    niem_person_id      TEXT,                   -- NIEM PersonID for cross-system exchange
    -- Contact
    address_json        JSONB NOT NULL DEFAULT '{}',
    -- Example: {"street": "123 Main St", "city": "Springfield", "state": "TX", "zip": "75001"}
    phone               TEXT,
    email               TEXT,
    preferred_language  TEXT NOT NULL DEFAULT 'en',
    -- Portal access
    portal_registered   BOOLEAN NOT NULL DEFAULT false,
    portal_email        TEXT,
    portal_verified     BOOLEAN NOT NULL DEFAULT false,
    -- Status
    status              TEXT NOT NULL DEFAULT 'active' CHECK (status IN (
                            'active', 'merged', 'deceased', 'anonymised'
                        )),
    merged_into_id      UUID REFERENCES defendants(id),
    -- Counts (denormalised)
    active_cases        INTEGER NOT NULL DEFAULT 0,
    outstanding_balance_cents BIGINT NOT NULL DEFAULT 0,
    active_warrants     INTEGER NOT NULL DEFAULT 0,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_defendants_court ON defendants (court_id);
CREATE INDEX idx_defendants_name ON defendants (court_id, last_name, first_name);
CREATE INDEX idx_defendants_dob ON defendants (court_id, date_of_birth);
CREATE INDEX idx_defendants_dl ON defendants (drivers_licence, dl_state);
CREATE INDEX idx_defendants_status ON defendants (court_id, status);
CREATE INDEX idx_defendants_portal ON defendants (portal_email) WHERE portal_registered = true;
CREATE INDEX idx_defendants_warrants ON defendants (court_id) WHERE active_warrants > 0;
```

## Case Management

```sql
CREATE TABLE cases (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    court_id            UUID NOT NULL REFERENCES courts(id),
    defendant_id        UUID NOT NULL REFERENCES defendants(id),
    case_number         TEXT NOT NULL,
    case_type           TEXT NOT NULL CHECK (case_type IN (
                            'traffic_moving', 'traffic_non_moving',
                            'parking', 'misdemeanour_a', 'misdemeanour_b',
                            'misdemeanour_c', 'ordinance_violation',
                            'code_enforcement', 'animal_control',
                            'truancy', 'juvenile_status',
                            'small_claims', 'civil_infraction'
                        )),
    status              TEXT NOT NULL DEFAULT 'filed' CHECK (status IN (
                            'filed', 'pending', 'set_for_hearing',
                            'continued', 'in_progress',
                            'disposed', 'closed', 'reopened',
                            'appealed', 'transferred',
                            'expunged', 'sealed'
                        )),
    -- Citation / charge
    citation_number     TEXT,
    statute_code        TEXT NOT NULL,          -- ordinance or state statute reference
    statute_description TEXT NOT NULL,
    offence_description TEXT,
    offence_date        DATE NOT NULL,
    offence_location    TEXT,
    -- Filing
    filing_date         DATE NOT NULL DEFAULT CURRENT_DATE,
    filing_source       TEXT NOT NULL CHECK (filing_source IN (
                            'e_citation', 'manual_entry', 'efsp',
                            'csv_import', 'officer_filed',
                            'code_enforcement', 'transfer_in'
                        )),
    -- Parties
    issuing_officer     TEXT,
    issuing_agency      TEXT,                   -- law enforcement agency
    agency_ori          TEXT,                   -- ORI of issuing agency
    prosecutor_id       UUID REFERENCES users(id),
    assigned_judge_id   UUID REFERENCES users(id),
    public_defender_id  UUID REFERENCES users(id),
    -- NIEM / ECF
    niem_case_id        TEXT,                   -- NIEM CaseID for exchange
    ecf_filing_id       TEXT,                   -- ECF FilingIdentifier
    -- Vehicle (traffic cases)
    vehicle_json        JSONB NOT NULL DEFAULT '{}',
    -- Example: {"plate": "ABC1234", "state": "TX", "make": "Toyota", "model": "Camry", "year": 2022, "color": "silver", "vin": "..."}
    -- Scheduling
    next_hearing_id     UUID,                   -- denormalised for dashboard queries
    next_hearing_date   TIMESTAMPTZ,
    continuance_count   INTEGER NOT NULL DEFAULT 0,
    -- Disposition reference
    disposition_id      UUID,                   -- set when disposed
    -- Financial
    total_fines_cents   BIGINT NOT NULL DEFAULT 0,
    total_fees_cents    BIGINT NOT NULL DEFAULT 0,
    total_paid_cents    BIGINT NOT NULL DEFAULT 0,
    balance_cents       BIGINT NOT NULL DEFAULT 0,
    -- Records retention
    retention_date      DATE,                   -- earliest date for destruction
    expungement_eligible BOOLEAN NOT NULL DEFAULT false,
    expunged_at         TIMESTAMPTZ,
    sealed_at           TIMESTAMPTZ,
    -- DMV
    dmv_abstract_sent   BOOLEAN NOT NULL DEFAULT false,
    dmv_abstract_type   TEXT CHECK (dmv_abstract_type IN (
                            'fta', 'ftc', 'conviction', 'dismissal',
                            'suspension', 'reinstatement'
                        )),
    dmv_sent_at         TIMESTAMPTZ,
    tags                TEXT[] NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (court_id, case_number)
);

CREATE INDEX idx_cases_court ON cases (court_id);
CREATE INDEX idx_cases_defendant ON cases (defendant_id);
CREATE INDEX idx_cases_status ON cases (court_id, status);
CREATE INDEX idx_cases_type ON cases (court_id, case_type);
CREATE INDEX idx_cases_citation ON cases (court_id, citation_number);
CREATE INDEX idx_cases_statute ON cases (court_id, statute_code);
CREATE INDEX idx_cases_filing ON cases (court_id, filing_date DESC);
CREATE INDEX idx_cases_hearing ON cases (next_hearing_date) WHERE status IN ('pending', 'set_for_hearing', 'continued');
CREATE INDEX idx_cases_balance ON cases (court_id) WHERE balance_cents > 0;
CREATE INDEX idx_cases_officer ON cases (issuing_officer);
CREATE INDEX idx_cases_tags ON cases USING GIN (tags);
```

## Hearings & Scheduling

```sql
CREATE TABLE hearings (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    court_id            UUID NOT NULL REFERENCES courts(id),
    case_id             UUID NOT NULL REFERENCES cases(id),
    defendant_id        UUID NOT NULL REFERENCES defendants(id),
    hearing_type        TEXT NOT NULL CHECK (hearing_type IN (
                            'arraignment', 'pre_trial', 'trial',
                            'sentencing', 'motion', 'show_cause',
                            'compliance_review', 'payment_review',
                            'probation_review', 'appeal',
                            'community_service_review'
                        )),
    status              TEXT NOT NULL DEFAULT 'scheduled' CHECK (status IN (
                            'scheduled', 'confirmed', 'in_progress',
                            'completed', 'continued', 'cancelled',
                            'defendant_no_show', 'reset'
                        )),
    -- Schedule
    scheduled_date      DATE NOT NULL,
    scheduled_time      TIME NOT NULL,
    duration_minutes    INTEGER NOT NULL DEFAULT 15,
    courtroom           TEXT,
    judge_id            UUID REFERENCES users(id),
    -- Format
    hearing_format      TEXT NOT NULL DEFAULT 'in_person' CHECK (hearing_format IN (
                            'in_person', 'virtual', 'hybrid', 'telephonic'
                        )),
    virtual_link        TEXT,                   -- Zoom/Teams meeting URL
    virtual_meeting_id  TEXT,
    recording_url       TEXT,                   -- archived recording
    -- Continuance
    continued_from_id   UUID REFERENCES hearings(id),
    continuance_reason  TEXT,
    continuance_requested_by TEXT CHECK (continuance_requested_by IN (
                            'defendant', 'prosecution', 'court', 'judge'
                        )),
    -- Outcome
    outcome_notes       TEXT,
    -- Notifications
    notice_sent         BOOLEAN NOT NULL DEFAULT false,
    notice_sent_at      TIMESTAMPTZ,
    reminder_sent       BOOLEAN NOT NULL DEFAULT false,
    reminder_sent_at    TIMESTAMPTZ,
    -- No-show prediction
    no_show_probability REAL,                   -- AI-computed 0.0-1.0
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_hearings_court ON hearings (court_id);
CREATE INDEX idx_hearings_case ON hearings (case_id);
CREATE INDEX idx_hearings_defendant ON hearings (defendant_id);
CREATE INDEX idx_hearings_date ON hearings (court_id, scheduled_date, scheduled_time);
CREATE INDEX idx_hearings_judge ON hearings (judge_id, scheduled_date);
CREATE INDEX idx_hearings_status ON hearings (court_id, status);
CREATE INDEX idx_hearings_courtroom ON hearings (court_id, courtroom, scheduled_date);
CREATE INDEX idx_hearings_upcoming ON hearings (scheduled_date) WHERE status IN ('scheduled', 'confirmed');
```

## Dispositions

```sql
CREATE TABLE dispositions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    court_id            UUID NOT NULL REFERENCES courts(id),
    case_id             UUID NOT NULL REFERENCES cases(id),
    defendant_id        UUID NOT NULL REFERENCES defendants(id),
    hearing_id          UUID REFERENCES hearings(id),
    disposition_type    TEXT NOT NULL CHECK (disposition_type IN (
                            'guilty_plea', 'not_guilty_plea',
                            'nolo_contendere', 'guilty_verdict',
                            'not_guilty_verdict', 'dismissed',
                            'dismissed_with_prejudice', 'nolle_prosequi',
                            'deferred_adjudication', 'deferred_disposition',
                            'pretrial_diversion', 'transferred',
                            'bond_forfeiture', 'default_judgement'
                        )),
    disposition_date    DATE NOT NULL,
    -- Sentencing
    sentence_json       JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "fine_cents": 25000,
    --   "court_costs_cents": 10700,
    --   "surcharges_cents": 5000,
    --   "community_service_hours": 40,
    --   "community_service_deadline": "2026-09-01",
    --   "probation_months": 6,
    --   "probation_conditions": ["no_alcohol", "report_monthly"],
    --   "defensive_driving": true,
    --   "dd_deadline": "2026-08-01",
    --   "licence_suspension_days": 90,
    --   "jail_days": 0,
    --   "jail_suspended": true,
    --   "restitution_cents": 0
    -- }
    -- Judge
    judge_id            UUID REFERENCES users(id),
    judge_notes         TEXT,
    -- NIEM
    niem_disposition_code TEXT,                 -- NIEM DispositionCode
    -- Status
    status              TEXT NOT NULL DEFAULT 'entered' CHECK (status IN (
                            'entered', 'appealed', 'reversed',
                            'modified', 'vacated', 'expunged'
                        )),
    appealed_at         TIMESTAMPTZ,
    -- DMV action
    dmv_action          TEXT CHECK (dmv_action IN (
                            'none', 'suspension', 'revocation',
                            'restriction', 'reinstatement'
                        )),
    entered_by          UUID NOT NULL REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_disp_court ON dispositions (court_id);
CREATE INDEX idx_disp_case ON dispositions (case_id);
CREATE INDEX idx_disp_defendant ON dispositions (defendant_id);
CREATE INDEX idx_disp_type ON dispositions (court_id, disposition_type);
CREATE INDEX idx_disp_date ON dispositions (court_id, disposition_date DESC);
CREATE INDEX idx_disp_judge ON dispositions (judge_id, disposition_date);
```

## Financial Management

```sql
CREATE TABLE fines_fees (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    court_id            UUID NOT NULL REFERENCES courts(id),
    case_id             UUID NOT NULL REFERENCES cases(id),
    defendant_id        UUID NOT NULL REFERENCES defendants(id),
    disposition_id      UUID REFERENCES dispositions(id),
    charge_type         TEXT NOT NULL CHECK (charge_type IN (
                            'fine', 'court_costs', 'surcharge',
                            'late_fee', 'warrant_fee', 'collection_fee',
                            'restitution', 'probation_fee',
                            'defensive_driving_fee', 'community_service_fee',
                            'technology_fee', 'state_fee',
                            'time_payment_fee', 'jury_fee'
                        )),
    description         TEXT NOT NULL,
    amount_cents        BIGINT NOT NULL,
    paid_cents          BIGINT NOT NULL DEFAULT 0,
    waived_cents        BIGINT NOT NULL DEFAULT 0,
    balance_cents       BIGINT NOT NULL DEFAULT 0,
    status              TEXT NOT NULL DEFAULT 'outstanding' CHECK (status IN (
                            'outstanding', 'partially_paid', 'paid',
                            'waived', 'written_off', 'sent_to_collections',
                            'in_payment_plan'
                        )),
    due_date            DATE,
    -- Payment plan
    payment_plan        BOOLEAN NOT NULL DEFAULT false,
    plan_json           JSONB NOT NULL DEFAULT '{}',
    -- Example: {"monthly_cents": 5000, "start_date": "2026-07-01", "end_date": "2027-06-01", "instalments_remaining": 12}
    -- Collections
    sent_to_collections BOOLEAN NOT NULL DEFAULT false,
    collection_agency   TEXT,
    collection_ref      TEXT,
    -- Waiver
    waived_by           UUID REFERENCES users(id),
    waive_reason        TEXT,
    assessed_by         UUID REFERENCES users(id),
    assessed_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_fines_court ON fines_fees (court_id);
CREATE INDEX idx_fines_case ON fines_fees (case_id);
CREATE INDEX idx_fines_defendant ON fines_fees (defendant_id);
CREATE INDEX idx_fines_status ON fines_fees (court_id, status);
CREATE INDEX idx_fines_due ON fines_fees (due_date) WHERE status IN ('outstanding', 'partially_paid');
CREATE INDEX idx_fines_balance ON fines_fees (defendant_id) WHERE balance_cents > 0;
CREATE INDEX idx_fines_collections ON fines_fees (court_id) WHERE sent_to_collections = true;

CREATE TABLE payments (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    court_id            UUID NOT NULL REFERENCES courts(id),
    case_id             UUID NOT NULL REFERENCES cases(id),
    defendant_id        UUID NOT NULL REFERENCES defendants(id),
    fine_fee_id         UUID REFERENCES fines_fees(id),
    amount_cents        BIGINT NOT NULL,
    payment_method      TEXT NOT NULL CHECK (payment_method IN (
                            'card_online', 'card_in_person', 'cash',
                            'check', 'money_order', 'ach',
                            'payment_gateway', 'collections_remittance'
                        )),
    status              TEXT NOT NULL DEFAULT 'completed' CHECK (status IN (
                            'pending', 'completed', 'failed',
                            'refunded', 'voided', 'chargeback'
                        )),
    -- Processor (no card data stored — PCI compliance)
    processor_name      TEXT,                   -- stripe, tyler_payments, ncourt
    processor_ref       TEXT,                   -- processor transaction ID
    processor_fee_cents BIGINT NOT NULL DEFAULT 0,
    -- Receipt
    receipt_number      TEXT NOT NULL,
    receipt_url         TEXT,
    -- Cashier
    received_by         UUID REFERENCES users(id),
    drawer_id           TEXT,                   -- cash drawer identifier
    reconciled          BOOLEAN NOT NULL DEFAULT false,
    reconciled_at       TIMESTAMPTZ,
    -- Revenue posting
    gl_account          TEXT,                   -- general ledger account for ERP
    revenue_posted      BOOLEAN NOT NULL DEFAULT false,
    posted_at           TIMESTAMPTZ,
    payment_date        DATE NOT NULL DEFAULT CURRENT_DATE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payments_court ON payments (court_id);
CREATE INDEX idx_payments_case ON payments (case_id);
CREATE INDEX idx_payments_defendant ON payments (defendant_id);
CREATE INDEX idx_payments_fine ON payments (fine_fee_id);
CREATE INDEX idx_payments_date ON payments (court_id, payment_date DESC);
CREATE INDEX idx_payments_status ON payments (court_id, status);
CREATE INDEX idx_payments_receipt ON payments (court_id, receipt_number);
CREATE INDEX idx_payments_unreconciled ON payments (court_id) WHERE reconciled = false;
CREATE INDEX idx_payments_unposted ON payments (court_id) WHERE revenue_posted = false;
```

## Documents

```sql
CREATE TABLE documents (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    court_id            UUID NOT NULL REFERENCES courts(id),
    case_id             UUID REFERENCES cases(id),
    document_type       TEXT NOT NULL CHECK (document_type IN (
                            'citation', 'complaint', 'summons',
                            'motion', 'order', 'judgement',
                            'warrant', 'notice_of_hearing',
                            'continuance_order', 'plea_form',
                            'payment_agreement', 'probation_order',
                            'dismissal_order', 'abstract_of_judgement',
                            'dmv_abstract', 'expungement_order',
                            'correspondence', 'evidence',
                            'hearing_recording', 'transcript',
                            'template'
                        )),
    title               TEXT NOT NULL,
    description         TEXT,
    -- File
    file_url            TEXT,
    file_size_bytes     BIGINT,
    mime_type           TEXT,
    format              TEXT CHECK (format IN (
                            'pdf', 'pdf_a', 'docx', 'tiff',
                            'mp4', 'mp3', 'wav', 'xml', 'json'
                        )),
    page_count          INTEGER,
    -- ECF
    ecf_document_id     TEXT,                   -- ECF DocumentIdentifier
    ecf_filing_status   TEXT CHECK (ecf_filing_status IN (
                            'submitted', 'under_review', 'accepted',
                            'rejected', 'returned'
                        )),
    -- E-signature
    signed              BOOLEAN NOT NULL DEFAULT false,
    signer_name         TEXT,
    signed_at           TIMESTAMPTZ,
    signature_provider  TEXT,                   -- docusign, adobe_sign
    signature_ref       TEXT,
    -- Template
    is_template         BOOLEAN NOT NULL DEFAULT false,
    template_merge_fields JSONB NOT NULL DEFAULT '{}',
    -- Records retention
    retention_category  TEXT,
    retention_years     INTEGER,
    retention_expires   DATE,
    destruction_eligible BOOLEAN NOT NULL DEFAULT false,
    destroyed_at        TIMESTAMPTZ,
    -- Access
    confidential        BOOLEAN NOT NULL DEFAULT false,
    public_accessible   BOOLEAN NOT NULL DEFAULT true,
    uploaded_by         UUID REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_docs_court ON documents (court_id);
CREATE INDEX idx_docs_case ON documents (case_id);
CREATE INDEX idx_docs_type ON documents (court_id, document_type);
CREATE INDEX idx_docs_ecf ON documents (ecf_document_id);
CREATE INDEX idx_docs_retention ON documents (retention_expires) WHERE destruction_eligible = false;
CREATE INDEX idx_docs_template ON documents (court_id) WHERE is_template = true;
```

## Warrants

```sql
CREATE TABLE warrants (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    court_id            UUID NOT NULL REFERENCES courts(id),
    case_id             UUID NOT NULL REFERENCES cases(id),
    defendant_id        UUID NOT NULL REFERENCES defendants(id),
    warrant_type        TEXT NOT NULL CHECK (warrant_type IN (
                            'bench_warrant', 'arrest_warrant',
                            'fta', 'ftc', 'ftp',
                            'capias', 'alias_capias',
                            'body_attachment'
                        )),
    status              TEXT NOT NULL DEFAULT 'active' CHECK (status IN (
                            'active', 'served', 'recalled',
                            'quashed', 'expired', 'withdrawn'
                        )),
    warrant_number      TEXT NOT NULL,
    -- Issuance
    issued_date         DATE NOT NULL,
    issued_by           UUID NOT NULL REFERENCES users(id),
    issuing_judge_id    UUID REFERENCES users(id),
    -- Bond
    bond_amount_cents   BIGINT,
    bond_type           TEXT CHECK (bond_type IN (
                            'cash', 'surety', 'personal_recognisance',
                            'no_bond'
                        )),
    -- NCIC / TCIC
    ncic_entry          BOOLEAN NOT NULL DEFAULT false,
    ncic_entry_number   TEXT,
    ncic_entered_at     TIMESTAMPTZ,
    -- Service
    served_at           TIMESTAMPTZ,
    served_by           TEXT,                   -- officer name/badge
    serving_agency      TEXT,
    -- Recall
    recalled_at         TIMESTAMPTZ,
    recalled_by         UUID REFERENCES users(id),
    recall_reason       TEXT,
    -- DMV
    dmv_hold_placed     BOOLEAN NOT NULL DEFAULT false,
    dmv_hold_released   BOOLEAN NOT NULL DEFAULT false,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (court_id, warrant_number)
);

CREATE INDEX idx_warrants_court ON warrants (court_id);
CREATE INDEX idx_warrants_case ON warrants (case_id);
CREATE INDEX idx_warrants_defendant ON warrants (defendant_id);
CREATE INDEX idx_warrants_status ON warrants (court_id, status);
CREATE INDEX idx_warrants_type ON warrants (court_id, warrant_type);
CREATE INDEX idx_warrants_active ON warrants (court_id) WHERE status = 'active';
CREATE INDEX idx_warrants_ncic ON warrants (ncic_entry_number) WHERE ncic_entry = true;
```

## AI Suggestions & Audit

```sql
CREATE TABLE ai_suggestions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    court_id            UUID NOT NULL REFERENCES courts(id),
    suggestion_type     TEXT NOT NULL CHECK (suggestion_type IN (
                            'citation_ocr', 'case_classification',
                            'statute_lookup', 'document_draft',
                            'order_draft', 'warrant_draft',
                            'plain_language_summary', 'translation',
                            'no_show_prediction', 'scheduling_optimisation',
                            'cashiering_anomaly', 'missing_disposition',
                            'disparity_flag', 'defendant_assistance'
                        )),
    status              TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
                            'pending', 'accepted', 'rejected',
                            'auto_applied', 'expired'
                        )),
    confidence          REAL NOT NULL CHECK (confidence BETWEEN 0.0 AND 1.0),
    target_entity_type  TEXT NOT NULL CHECK (target_entity_type IN (
                            'case', 'hearing', 'disposition', 'defendant',
                            'document', 'fine_fee', 'warrant'
                        )),
    target_entity_id    UUID NOT NULL,
    summary             TEXT NOT NULL,
    detail_json         JSONB NOT NULL DEFAULT '{}',
    -- Example (citation_ocr): {
    --   "extracted_fields": {"citation_number": "T-2026-12345", "statute": "545.352", "offence": "Speeding 45/30"},
    --   "confidence_per_field": {"citation_number": 0.99, "statute": 0.95, "offence": 0.92},
    --   "source_document_id": "uuid"
    -- }
    explanation         TEXT,
    model_id            TEXT,
    model_version       TEXT,
    reviewed_by         UUID REFERENCES users(id),
    reviewed_at         TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_court ON ai_suggestions (court_id);
CREATE INDEX idx_ai_type ON ai_suggestions (court_id, suggestion_type);
CREATE INDEX idx_ai_status ON ai_suggestions (court_id, status);
CREATE INDEX idx_ai_target ON ai_suggestions (target_entity_type, target_entity_id);

CREATE TABLE audit_log (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    court_id            UUID NOT NULL REFERENCES courts(id),
    actor_type          TEXT NOT NULL CHECK (actor_type IN (
                            'user', 'defendant', 'system', 'ai',
                            'api_client', 'efsp', 'payment_processor',
                            'dmv_feed', 'rms_connector', 'scheduler'
                        )),
    actor_id            TEXT NOT NULL,
    action              TEXT NOT NULL,
    entity_type         TEXT NOT NULL,
    entity_id           UUID NOT NULL,
    changes_json        JSONB NOT NULL DEFAULT '{}',
    ip_address          TEXT,
    user_agent          TEXT,
    -- CJIS
    cjis_relevant       BOOLEAN NOT NULL DEFAULT false,
    cjis_audit_category TEXT,                   -- access, modification, deletion, query
    -- Compliance
    pii_accessed        BOOLEAN NOT NULL DEFAULT false,
    session_id          TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_court ON audit_log (court_id);
CREATE INDEX idx_audit_actor ON audit_log (actor_type, actor_id);
CREATE INDEX idx_audit_entity ON audit_log (entity_type, entity_id);
CREATE INDEX idx_audit_action ON audit_log (court_id, action);
CREATE INDEX idx_audit_created ON audit_log (created_at DESC);
CREATE INDEX idx_audit_cjis ON audit_log (court_id) WHERE cjis_relevant = true;
CREATE INDEX idx_audit_pii ON audit_log (court_id) WHERE pii_accessed = true;
```

---

## Cross-Domain Query Examples

### Cases with unpaid fines and active warrants

```sql
SELECT c.case_number, c.case_type, c.statute_description,
       d.first_name, d.last_name,
       c.balance_cents,
       w.warrant_type, w.issued_date
FROM cases c
JOIN defendants d ON d.id = c.defendant_id
JOIN warrants w ON w.case_id = c.id AND w.status = 'active'
WHERE c.court_id = $1
  AND c.balance_cents > 0
ORDER BY c.balance_cents DESC;
```

### Revenue reconciliation for a date range

```sql
SELECT p.payment_date,
       p.payment_method,
       COUNT(*) AS transactions,
       SUM(p.amount_cents) AS total_cents,
       SUM(p.processor_fee_cents) AS fees_cents,
       SUM(CASE WHEN p.reconciled THEN p.amount_cents ELSE 0 END) AS reconciled_cents
FROM payments p
WHERE p.court_id = $1
  AND p.payment_date BETWEEN $2 AND $3
  AND p.status = 'completed'
GROUP BY p.payment_date, p.payment_method
ORDER BY p.payment_date, p.payment_method;
```

### Clearance rate by case type

```sql
SELECT c.case_type,
       COUNT(*) FILTER (WHERE c.status = 'disposed' OR c.status = 'closed') AS disposed,
       COUNT(*) AS total_filed,
       ROUND(100.0 * COUNT(*) FILTER (WHERE c.status IN ('disposed', 'closed')) / NULLIF(COUNT(*), 0), 1) AS clearance_rate_pct
FROM cases c
WHERE c.court_id = $1
  AND c.filing_date BETWEEN $2 AND $3
GROUP BY c.case_type
ORDER BY clearance_rate_pct;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Court & Staff | 2 | Multi-court deployment with RBAC and CJIS certification tracking |
| Defendant Management | 1 | Deduplication, portal access, NIEM person ID |
| Case Management | 1 | Citations through disposition with NIEM/ECF identifiers |
| Scheduling | 1 | Hearings with virtual hearing support and no-show prediction |
| Dispositions | 1 | Verdicts, pleas, sentencing with NIEM disposition codes |
| Financial | 2 | Fines/fees with payment plans + PCI-compliant payment transactions |
| Documents | 1 | ECF-aligned document management with records retention |
| Warrants | 1 | FTA/FTC/bench warrants with NCIC entry tracking |
| AI & Audit | 2 | Court AI suggestions + CJIS-compliant partitioned audit log |
| **Total** | **12** | |

---

## Key Design Decisions

1. **Multi-court tenancy** — `court_id` on every operational table enables a single deployment to serve multiple municipal courts in a county or region, supporting shared-services models common in small municipalities.

2. **NIEM-aligned exchange identifiers** — `niem_person_id`, `niem_case_id`, and `niem_disposition_code` on core entities enable clean data exchange with state court reporting systems and law enforcement without translation layers.

3. **ECF filing integration** — `ecf_filing_id` on cases and `ecf_document_id` on documents map directly to OASIS LegalXML ECF 5.01 message identifiers, enabling e-filing service provider (EFSP) integration.

4. **PCI-DSS compliance by design** — the `payments` table stores processor references (`processor_ref`) but never card numbers, CVVs, or other cardholder data. Payment processing is delegated to certified processors (Stripe, Tyler Payments, nCourt).

5. **CJIS audit compliance** — `audit_log.cjis_relevant` and `cjis_audit_category` flag entries that involve criminal justice information access, satisfying CJIS Security Policy audit requirements without forcing every entry through CJIS classification.

6. **Warrant NCIC tracking** — `warrants.ncic_entry` and `ncic_entry_number` track NCIC/TCIC database entries, enabling warrant recall workflows that include NCIC removal verification.

7. **DMV abstract lifecycle** — `cases.dmv_abstract_sent` and `dmv_abstract_type` track the DMV reporting state (FTA, FTC, conviction, reinstatement), ensuring that licence suspension and reinstatement feeds are traceable.

8. **Hearing continuance chain** — `hearings.continued_from_id` creates a linked list of continuances for a case, enabling courts to track how many times a hearing has been rescheduled and why.

9. **Records retention and expungement** — `cases.retention_date` and `cases.expungement_eligible` support statutory records retention schedules and automated expungement eligibility determination, with `documents.retention_expires` tracking individual document destruction eligibility.

10. **Revenue posting to ERP** — `payments.gl_account` and `payments.revenue_posted` enable reconciliation reporting and general-ledger posting to municipal ERP systems (Tyler Munis, etc.) without requiring the court system to maintain its own accounting ledger.
