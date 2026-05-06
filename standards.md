# Standards & API Reference

> Project: Municipal Court Management · Generated: 2026-05-06

## Industry Standards & Specifications

### ISO Standards
- **ISO/IEC 27001:2022 — Information Security Management Systems** — https://www.iso.org/standard/27001 — Baseline ISMS controls expected for systems holding court records and PII.
- **ISO/IEC 27002:2022 — Code of practice for information security controls** — https://www.iso.org/standard/75652.html — Companion control set covering access control, cryptography, and logging suitable for court systems.
- **ISO/IEC 27018:2019 — Protection of PII in public clouds** — https://www.iso.org/standard/76559.html — Relevant for SaaS-hosted court data with multi-tenant or cloud storage.
- **ISO 15489-1:2016 — Records management** — https://www.iso.org/standard/62542.html — Framework for retention, classification, and disposal of court records.
- **ISO/IEC 25010:2023 — Systems and software quality models** — https://www.iso.org/standard/78176.html — Quality attributes (security, usability, reliability) used in government procurement.
- **ISO/IEC 40500:2012 — Web Content Accessibility Guidelines (WCAG 2.0)** — https://www.iso.org/standard/58625.html — Accessibility standard adopted by ISO for public-facing portals.

### W3C & IETF Standards
- **W3C WCAG 2.1 / 2.2 — Web Content Accessibility Guidelines** — https://www.w3.org/TR/WCAG22/ — Required for ADA-compliant defendant portals.
- **W3C XML 1.0** — https://www.w3.org/TR/xml/ — Foundation for ECF/NIEM exchange documents.
- **W3C XML Signature & XML Encryption** — https://www.w3.org/TR/xmldsig-core1/ — Used in court e-filing message signing.
- **RFC 7231 — HTTP/1.1 Semantics and Content** — https://datatracker.ietf.org/doc/html/rfc7231 — Baseline for any REST API surface.
- **RFC 9110 — HTTP Semantics** — https://datatracker.ietf.org/doc/html/rfc9110 — Modern HTTP semantics replacing 7231 series.
- **RFC 8259 — JSON** — https://datatracker.ietf.org/doc/html/rfc8259 — Data interchange for modern APIs and webhooks.
- **RFC 7519 — JSON Web Token (JWT)** — https://datatracker.ietf.org/doc/html/rfc7519 — Session and inter-service auth tokens.
- **RFC 6749 / 8252 — OAuth 2.0** — https://datatracker.ietf.org/doc/html/rfc6749 — Delegated authorisation for portal and API clients.
- **RFC 7515/7516/7517/7518 — JOSE (JWS, JWE, JWK, JWA)** — https://datatracker.ietf.org/doc/html/rfc7515 — Cryptographic envelopes for tokens, encrypted documents, and key distribution.
- **RFC 5246 / 8446 — TLS 1.2 / 1.3** — https://datatracker.ietf.org/doc/html/rfc8446 — Mandatory transport security.
- **RFC 5280 — Internet X.509 Public Key Infrastructure** — https://datatracker.ietf.org/doc/html/rfc5280 — Certificate handling for e-signatures.

### Data Model & API Specifications
- **OASIS LegalXML Electronic Court Filing (ECF) 5.01** — https://docs.oasis-open.org/legalxml-courtfiling/specs/ecf/v5.01/ecf-v5.01.html — Standard message set for court e-filing exchanged between EFSPs, EFMs, and CMSs.
- **OASIS LegalXML ECF 4.01** — https://docs.oasis-open.org/legalxml-courtfiling/specs/ecf/v4.01/ — Widely deployed prior version still used by many state EFM systems.
- **NIEM (National Information Exchange Model) — Justice domain** — https://www.niem.gov/ — Reference data model for court, law-enforcement, and corrections data exchange in the US.
- **GJXDM (Global Justice XML Data Model)** — https://it.ojp.gov/gjxdm — Predecessor to NIEM Justice; some legacy state systems still consume it.
- **NCSC JTC — Court Component Model and Function Standards** — https://www.ncsc.org/services-and-experts/areas-of-expertise/court-technology/joint-technology-committee — Joint Technology Committee standards from NCSC/COSCA/NACM defining CMS functional standards.
- **NCSC Next-Generation Court Technology Standards** — https://www.ncsc.org/services-and-experts/areas-of-expertise/court-technology — Reference architecture and integration patterns for modern CMS.
- **OpenAPI 3.1** — https://spec.openapis.org/oas/v3.1.0 — Recommended specification format for any REST API.
- **GraphQL (June 2018 spec)** — https://spec.graphql.org/June2018/ — Optional alternative for portal/back-office data access.
- **JSON Schema 2020-12** — https://json-schema.org/specification — Validation of API payloads and configuration documents.
- **HL7 FHIR (selected resources)** — https://hl7.org/fhir/ — Useful only where mental-health or substance-treatment court referrals are involved.
- **PDF/A (ISO 19005)** — https://www.iso.org/standard/63542.html — Archival PDF format for long-term court record storage.
- **CMIS (Content Management Interoperability Services) 1.1** — https://docs.oasis-open.org/cmis/CMIS/v1.1/CMIS-v1.1.html — Document-repository abstraction historically used by court ECMs.

### Security & Authentication Standards
- **NIST SP 800-53 Rev 5 — Security and Privacy Controls** — https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final — Baseline control catalogue for US public-sector systems.
- **NIST SP 800-63-3 / 4 (draft) — Digital Identity Guidelines** — https://pages.nist.gov/800-63-3/ — Identity proofing and authenticator assurance levels for portal users and staff.
- **NIST SP 800-171 Rev 3 — Protecting CUI in Nonfederal Systems** — https://csrc.nist.gov/pubs/sp/800/171/r3/final — Relevant when handling federally-supplied data such as NCIC checks.
- **CJIS Security Policy v6.0** — https://le.fbi.gov/cjis-division/cjis-security-policy-resource-center — Required for any system touching FBI Criminal Justice Information (warrants, criminal history, NCIC).
- **PCI-DSS v4.0** — https://www.pcisecuritystandards.org/document_library — Mandatory for any in-scope payment processing; usually scoped down via processor delegation.
- **OWASP ASVS 4.0** — https://owasp.org/www-project-application-security-verification-standard/ — Application security verification baseline.
- **OWASP API Security Top 10 (2023)** — https://owasp.org/API-Security/editions/2023/en/0x11-t10/ — API-specific risk catalogue.
- **OpenID Connect Core 1.0** — https://openid.net/specs/openid-connect-core-1_0.html — Identity layer over OAuth 2.0 for SSO.
- **SAML 2.0** — https://docs.oasis-open.org/security/saml/v2.0/ — Still common for inter-agency SSO with state identity providers.
- **FIPS 140-3** — https://csrc.nist.gov/projects/cryptographic-module-validation-program/standards — Cryptographic module validation expected in many state procurements.
- **GDPR (EU 2016/679)** — https://gdpr.eu/ — Applies if any EU-resident data subject is processed; informative for privacy-by-design.
- **CCPA / CPRA (California)** — https://oag.ca.gov/privacy/ccpa — Sub-national privacy regime with implications for California municipal courts.

### Domain & Regulatory Frameworks
- **ADA Title II — Effective communication & web accessibility** — https://www.ada.gov/resources/web-guidance/ — DOJ web accessibility rule referencing WCAG 2.1 AA for state and local government.
- **NCSC/COSCA/NACM Joint Technology Committee Resource Bulletins** — https://www.ncsc.org/jtc — Position papers on case management, e-filing, virtual hearings, AI in courts.
- **State Court Administrator data-reporting standards** — Examples: Texas OCA monthly reporting (https://www.txcourts.gov/oca/), California JBSIS (https://www.courts.ca.gov/programs-jbsis.htm) — Each state mandates its own statistical and case-level reporting format.

### MCP Server Specifications
- **Model Context Protocol** — https://modelcontextprotocol.io/ — Useful for exposing case-data tools (lookup, schedule, draft) to AI assistants used by clerks or defendants.
- **MCP specification (current draft)** — https://spec.modelcontextprotocol.io/ — Reference for tool, resource, and prompt schemas; relevant to any AI-clerk integration layer.

---

## Similar Products — Developer Documentation & APIs

### Tyler Technologies — Odyssey & Tyler API Platform
- **Description:** The dominant US court CMS; Tyler exposes integration via the Odyssey Web Services and the Tyler API Platform (limited to certified partners).
- **API Documentation:** Restricted partner portal — overview at https://www.tylertech.com/products/enterprise-justice/odyssey-case-manager
- **SDKs/Libraries:** Provided to certified partners; SOAP/REST profiles; no public SDK.
- **Developer Guide:** Tyler Partner Program — https://www.tylertech.com/partners
- **Standards:** ECF 4.01/5.0 (e-filing), NIEM Justice, SOAP & REST/JSON
- **Authentication:** OAuth 2.0 / API key, depending on product tier; SAML for staff SSO

### Tyler — Odyssey File & Serve (EFSP/EFM)
- **Description:** Tyler's e-filing service-provider and manager used in many states.
- **API Documentation:** EFSP integration documentation provided to certified EFSPs; public marketing at https://odysseyfileandserve.tylertech.cloud/
- **SDKs/Libraries:** ECF 4.01-conformant message libraries; partner-only.
- **Developer Guide:** Partner-only.
- **Standards:** OASIS LegalXML ECF 4.01 / 5.0, MTOM/XOP attachments, WS-Security
- **Authentication:** WS-Security with X.509 certificates; SAML

### Journal Technologies — eCourt / eFileIT
- **Description:** Browser-based CMS and e-filing used heavily in California and other state superior courts.
- **API Documentation:** Vendor-managed; product overview at https://www.journaltech.com/ecourt
- **SDKs/Libraries:** Web services (SOAP/REST); ECF-conformant adapters.
- **Developer Guide:** Provided under customer/partner agreement.
- **Standards:** ECF 4.01, NIEM, REST/JSON for newer interfaces
- **Authentication:** SAML, OAuth 2.0

### Thomson Reuters C-Track
- **Description:** Court CMS focused on appellate and trial courts with integrated e-filing.
- **API Documentation:** Customer-only; product page https://legal.thomsonreuters.com/en/products/c-track
- **SDKs/Libraries:** Web services API set.
- **Developer Guide:** Customer documentation portal.
- **Standards:** REST/JSON, ECF support, OpenAPI for newer modules
- **Authentication:** OAuth 2.0, SAML SSO

### Equivant — Court Manager
- **Description:** Court CMS with cross-justice ecosystem (supervision, jail).
- **API Documentation:** Customer/partner-only; product page https://www.equivant.com/court/
- **SDKs/Libraries:** Web service connectors to Equivant supervision and jail products.
- **Developer Guide:** Provided under partner agreement.
- **Standards:** NIEM Justice, ECF, REST/SOAP
- **Authentication:** SAML, OAuth 2.0

### GovPilot
- **Description:** Cloud government-operations platform with a court module.
- **API Documentation:** REST API for departmental modules; overview at https://www.govpilot.com/api
- **SDKs/Libraries:** None publicly listed; HTTP clients suffice.
- **Developer Guide:** Customer support portal.
- **Standards:** REST/JSON, OpenAPI-style endpoints
- **Authentication:** API key + OAuth 2.0

### ImageSoft — TrueFiling
- **Description:** E-filing service provider used across multiple state EFM deployments.
- **API Documentation:** Filer API overview at https://www.truefiling.com/ — partner specs provided under agreement.
- **SDKs/Libraries:** ECF 4.01/5.0 client libraries provided to integrators.
- **Developer Guide:** Filer integration docs through TrueFiling support.
- **Standards:** OASIS LegalXML ECF 4.01/5.0, NIEM Justice
- **Authentication:** API key, mTLS, signed messages

### Pioneer Technology Group — Benchmark
- **Description:** County and municipal court CMS with strong Florida footprint.
- **API Documentation:** Customer-only; product page https://pioneertechnologygroup.com/court-software/
- **SDKs/Libraries:** Web services for state reporting.
- **Developer Guide:** Customer portal.
- **Standards:** Florida CCIS reporting formats, NIEM-aligned
- **Authentication:** OAuth 2.0 / API key

### Stripe (representative payment processor)
- **Description:** Payment processor commonly used for court online payment integrations.
- **API Documentation:** https://stripe.com/docs/api
- **SDKs/Libraries:** Node, Python, Java, Go, Ruby, .NET, PHP — https://stripe.com/docs/libraries
- **Developer Guide:** https://stripe.com/docs
- **Standards:** REST/JSON, OpenAPI, PCI-DSS SAQ A scope when using hosted checkout
- **Authentication:** API keys + OAuth (Connect), webhook signatures

### NIC / Tyler Payments / nCourt (gov payment gateways)
- **Description:** Government-specialised payment gateways frequently integrated by court CMSs.
- **API Documentation:** nCourt — https://www.ncourt.com/ ; Tyler Payments — https://www.tylertech.com/products/payments
- **SDKs/Libraries:** REST clients; partner-issued credentials.
- **Developer Guide:** Provided under integration agreement.
- **Standards:** REST/JSON, PCI-DSS Level 1 service providers
- **Authentication:** API key + HMAC-signed callbacks

### Zoom for Government
- **Description:** Virtual hearing platform widely used by US courts; FedRAMP Moderate authorised.
- **API Documentation:** https://developers.zoom.us/docs/api/
- **SDKs/Libraries:** Meeting/Video SDKs for web, iOS, Android, Windows, macOS — https://developers.zoom.us/docs/sdk/
- **Developer Guide:** https://developers.zoom.us/docs/
- **Standards:** REST/JSON, OAuth 2.0, webhooks
- **Authentication:** OAuth 2.0, Server-to-Server OAuth, JWT (legacy)

### DocuSign eSignature
- **Description:** Common e-signature provider for orders, judgements, and consent forms.
- **API Documentation:** https://developers.docusign.com/docs/esign-rest-api/
- **SDKs/Libraries:** C#, Java, Node, PHP, Python, Ruby — https://github.com/docusign
- **Developer Guide:** https://developers.docusign.com/docs/esign-rest-api/quickstart/
- **Standards:** REST/JSON, OpenAPI, XAdES/PAdES e-signatures
- **Authentication:** OAuth 2.0 (Authorization Code, JWT Grant)

### Twilio (notifications)
- **Description:** SMS/voice notifications for hearing reminders and FTA reduction.
- **API Documentation:** https://www.twilio.com/docs/api
- **SDKs/Libraries:** Multiple languages — https://www.twilio.com/docs/libraries
- **Developer Guide:** https://www.twilio.com/docs
- **Standards:** REST/JSON, webhook signatures, TCPA compliance considerations
- **Authentication:** API key (Account SID + Auth Token), API Keys/SigV1

---

## Notes

- The most strategically important standards for an open-source municipal CMS are **OASIS LegalXML ECF 5.01**, **NIEM Justice**, **NCSC JTC functional standards**, **WCAG 2.2**, **CJIS Security Policy** (where criminal-history or NCIC data is involved), and **PCI-DSS** scoping rules for payments.
- ECF 5.0 modernises the message layer toward JSON and REST while preserving NIEM semantics; new builds should target 5.x but maintain ECF 4.01 compatibility for state EFMs that have not yet upgraded.
- Most incumbents do not publish open API documentation. An open-source project differentiates by publishing an OpenAPI 3.1 spec from day one and offering language SDKs.
- State-specific reporting formats (Texas OCA, California JBSIS, Florida CCIS, etc.) are jurisdiction-specific and must be configurable rather than hard-coded.
- AI integrations (chat assistants, drafting, classification) are an emerging area without established standards; using **MCP** to expose case-data tools to LLMs is a forward-looking choice.
- E-recording and land-records standards (PRIA) are out of scope unless the municipality combines court and recorder functions.
