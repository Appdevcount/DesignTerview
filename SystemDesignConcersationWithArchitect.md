Conversational System Design — Healthcare Prior Authorization (End‑to‑End)

> Format: A back‑and‑forth between a Senior Architect (SA) and a Senior Engineer (SE). We go from ideation → requirements → HLD → LLD → security → scaling → observability → data → APIs → events → deployment → testing → ADRs. Code snippets use C# (.NET). Standards: FHIR R4, X12 278/275, HIPAA.




---

0) Elevator Pitch

SA: Let’s build a Prior Authorization (PA) Platform for a payer that receives PA requests from providers, validates eligibility, applies policy rules, facilitates clinical review, and responds with determinations. We’ll integrate with EHRs via FHIR and clearinghouses via X12 278. Core tenets: compliance, automation, transparency, and resilience.

SE: We’ll aim for straight‑through processing (STP) where possible, target < 5 min decisioning for auto‑approved cases, and 99.9% availability.


---

1) Ideation & Problem Framing

SA: What problem are we solving?

SE: Providers face delays submitting PAs across multiple proprietary portals. Payers struggle with inconsistent data and manual reviews. We want a single, standards‑based entry point, automated checks, and auditable decisions.

SA: Who are our stakeholders?

SE:

Providers (clinics, hospitals) submitting PAs from EHR/portal

Payer Utilization Management (UM) nurses/MD reviewers

Members/patients (status visibility)

Clearinghouses

Payer back‑office systems (Eligibility, Benefits, Claims, Care Mgmt)

Compliance & Security (HIPAA)


SA: What are success metrics?

SE:

Cycle time from submission → determination (p50 under 5 min for STP)

Auto‑decision rate ≥ 30% in 6 months

First‑time pass rate (no data defects) ≥ 95%

Appeal overturn rate ≤ 5%

Uptime 99.9%; RTO 2h, RPO 15m



---

2) Scope & Assumptions

SA: MVP scope?

SE:

Inbound channels: FHIR PriorAuth (PA request via ServiceRequest, Coverage, Patient, Practitioner, Claim-like resources), X12 278 via clearinghouse, and a simple Provider Portal.

Outbound: status updates (FHIR Communication), X12 278 response (A13/A6), and notifications (email/SMS).

Clinical attachments: X12 275 or FHIR DocumentReference/Binary.

Rules: medical policy rules + benefits + clinical criteria (UM).


SA: Constraints?

SE: HIPAA/PHI, integrate with legacy eligibility (SOAP) and benefits (SQL), partner mTLS, and some states mandate FHIR prior auth APIs (per CMS interoperability final rules).


---

3) Non‑Functional Requirements

Security: OAuth2/OIDC, mTLS for B2B, at‑rest encryption (TDE/KMS), field‑level encryption for SSN.

Performance: 500 RPS burst intake; p95 API latency < 300ms (non‑decisioning calls).

Reliability: Idempotency, retries with backoff, DLQs, exactly‑once business semantics.

Observability: Traces (W3C), structured logs, SLOs, audit trail (immutable).

Compliance: Data retention 7 years, access logging, least privilege.



---

4) Target Product Capabilities (MVP → v2)

MVP: Intake, validation, eligibility/benefits checks, rules engine, manual review UI, determination, notifications, status API.

v2: NLP triage of notes, guideline service integration (InterQual/MCG), appeal workflow, analytics, provider scorecards.


---

5) High‑Level Architecture (HLA)

SA: Paint the macro picture.

SE:

+-------------------------+         +-------------------+
| Provider EHR (FHIR)     |<--FHIR->| PriorAuth API GW  |--+--> Auth (OIDC)
+-------------------------+         +-------------------+  |
                                      |                   |
+-------------------------+         +-------------------+  |
| Clearinghouse (X12 278) |<--B2B-->| B2B Gateway (mTLS)|--+--> Rate Limit
+-------------------------+         +-------------------+
                                      |
                                      v
                              +---------------+
                              | Intake Svc    |---> Validation Svc
                              +---------------+      |      \
                                      |              |       \
                                      v              v        v
                               +-----------+   +-----------+  +--------------+
                               | Orchestr. |-->| Elig Svc |-> | Benefits Svc |
                               +-----------+   +-----------+  +--------------+
                                      |                \
                                      v                 \
                                +-----------------+      \
                                | Rules Engine    |<------+
                                +-----------------+
                                      |
                                      v
                          +------------------------+
                          | Clinical Review Svc    |<--> Reviewer UI
                          +------------------------+
                                      |
                                      v
                             +------------------+
                             | Determination Svc|
                             +------------------+
                                      |
                         +---------------------------+
                         | Event Bus (Kafka/ASB)     |
                         +---------------------------+
                          /     |         |        \
                         v      v         v         v
                    Status Svc  Notify   Docs     Audit/Trace
                                (SMS/EM)  (S3)      (WORM)

Downstream Integrations: Claims, Care Mgmt, Data Warehouse/Analytics

SA: Standards mapping?

SE:

FHIR resources: Patient, Coverage, Practitioner, Organization, ServiceRequest, Claim, ClaimResponse, Communication, DocumentReference, Binary.

X12 transactions: 278 (request/response), 275 (attachments). We’ll normalize inbound to an internal PA domain model.



---

6) Core Domain & Bounded Contexts

SA: Let’s define contexts.

SE:

Intake BC: Protocol adapters (FHIR, X12, Portal), validation, dedupe/idempotency.

Eligibility & Benefits BC: Checks coverage/benefit limits.

Rules BC: Medical policy & plan rules, guideline adapters.

Clinical Review BC: Queueing, assignment, notes, determinations.

Decisioning BC: Finalize approve/pend/deny, generate rationale.

Documents BC: Ingest/store attachments, virus scan, content type enforcement.

Status & Notifications BC: Status projection, provider/member comms.

Audit/Compliance BC: Immutable event log, access logs.



---

7) Key Interaction Flows (Sequences)

7.1 Submit PA (FHIR)

Client -> API GW -> Intake -> Validation -> Orchestrator ->
  (Eligibility, Benefits) -> Rules -> [Auto-Approve?]
   | yes -> Determination -> Status -> Notify
   | no  -> Clinical Review Queue -> Reviewer UI -> Determination

7.2 Attach Clinical Docs

Client uploads via signed URL → Documents Svc stores to object store → emits Document.Attached → Rules/Review re-evaluates if waiting.


7.3 Status Polling/Push

Provider polls Status API or subscribes to webhooks; for X12 route, we send 278 response A6/A13.



---

8) Data Model (Simplified)

PARequest(Id, MemberId, ProviderId, ServiceCode, DiagnosisCodes[], SiteOfService, Priority, ReceivedAt, Channel, Status)

EligibilityResult(PAId, EligibilityStatus, EffectiveFrom, EffectiveTo, PlanId)

BenefitAccumulation(PAId, Allowed, Used, Remaining, AuthRequired)

Determination(PAId, Outcome[Approved|Pended|Denied], Reasons[], ReviewerId, DecidedAt, SLATimer)

Document(DocId, PAId, Type, Uri, Hash, ReceivedAt)

Partitioning: by PayerTenantId + PAId; PII isolated; PHI minimized in events.


---

9) APIs (Representative)

Public (Provider-Facing)

POST /fhir/Claim/$submit-prior-auth (bundle)

POST /pa/{id}/attachments (pre-signed URL init)

GET /pa/{id}/status

POST /webhooks/register → events: pa.status.changed


B2B (X12)

AS2/EDI endpoint; we map 278/275 to domain model.


Internal

/eligibility/check (member, coverage)

/benefits/evaluate (plan, service)

/rules/evaluate (paId)

/review/assign (work queue)

/determinations (create/update)



---

10) Events (Topic Design)

pa.submitted.v1 { paId, memberRef, service, priority, channel }

eligibility.checked.v1 { paId, status }

benefits.evaluated.v1

rules.evaluated.v1 { outcome: autoApprove|needsReview|deny, reasons[] }

document.attached.v1 { paId, docId, type }

review.completed.v1 { paId, outcome, reviewer }

pa.determined.v1

pa.status.changed.v1


Guidelines: compact payloads; PHI minimization; PII tokens; schema registry; compat policy FORWARD+BACKWARD.


---

11) Security & Compliance

AuthN: OAuth2/OIDC; SMART-on-FHIR for EHR apps; client credentials for system-to-system.

AuthZ: Fine-grained (OPA/ABAC): provider can only see their submitted PAs; payer reviewers scoped by region/LOB.

Transport: TLS 1.2+, mTLS for B2B.

Data at Rest: AES-256, KMS-managed keys; field encryption for SSN/MemberId.

Audit: Immutable WORM store for access & decision logs; correlate with trace IDs.

PII/PHI Handling: Data minimization in logs/events; redaction filters.

Compliance: HIPAA BAAs, least privilege IAM, annual risk assessments.



---

12) Reliability Patterns

Idempotency keys on pa.clientRequestId.

Outbox/Inbox for event publication to ensure exactly-once business effects.

Circuit breakers on legacy services; bulkhead pools.

Retry with jittered exponential backoff; DLQs with quarantine UI.

Sagas in Orchestrator for cross-service steps.



---

13) Observability & SLOs

Tracing: W3C trace context across API GW → services → message bus.

Metrics: RPS, latency, error rate, auto-approval %, queue depth, reviewer SLA timers.

Logs: Structured JSON; request_id, memberRef (tokenized), paId.

SLOs: 99.9% availability; p95 < 300ms for intake; decision median < 5 min.

Runbooks: Playbooks for DLQ drain; rules hotfix; partner cert rotation.



---

14) Storage & Tech Choices (Example)

Operational DB: PostgreSQL (ACID, JSONB for FHIR Bundles) or Azure SQL.

Documents: S3/Azure Blob with immutable policy + AV scan.

Cache: Redis for eligibility/benefits memoization.

Search: OpenSearch for PA lookup by member/date/code.

Event Bus: Kafka or Azure Service Bus (ASB) for commands/events.

Rules: Drools/Rule Engine or custom DSL with versioning.



---

15) Low‑Level Design Highlights

15.1 Intake Service (CQRS + Outbox)

sequenceDiagram
participant C as Client
participant GW as API GW
participant I as IntakeSvc
participant DB as PA DB
participant OB as Outbox
participant BUS as EventBus

C->>GW: POST /fhir/Claim/$submit-prior-auth
GW->>I: request (OIDC)
I->>I: Validate bundle + idempotency
I->>DB: Save PARequest (status=RECEIVED)
I->>OB: Write outbox(pa.submitted.v1)
OB->>BUS: Publish (tx polled)

15.2 Orchestrator (Saga)

State machine: RECEIVED → VALIDATED → ELIG_CHECKED → BENEFITS_EVAL → RULES_EVAL → [AUTO|REVIEW] → DETERMINED.

Compensations: if external call fails persistently, move to PENDED with reason.


15.3 Reviewer UI

Queues cases by priority & skills; supports templated rationales; secure document viewer; audit on every action.



---

16) Example C# Snippets

16.1 Idempotent Intake Controller

[ApiController]
[Route("/fhir/Claim")]
public class PriorAuthController : ControllerBase
{
    private readonly IPASubmissionHandler _handler;

    public PriorAuthController(IPASubmissionHandler handler) => _handler = handler;

    [HttpPost("$submit-prior-auth")]
    public async Task<IActionResult> SubmitAsync([FromBody] FhirBundle bundle, [FromHeader(Name="Idempotency-Key")] string idemKey)
    {
        if (string.IsNullOrWhiteSpace(idemKey)) return BadRequest("Missing Idempotency-Key");
        var result = await _handler.HandleAsync(bundle, idemKey, HttpContext.TraceIdentifier);
        return Accepted(new { paId = result.PAId, status = result.Status });
    }
}

16.2 Handler with Outbox Pattern

public class PASubmissionHandler : IPASubmissionHandler
{
    private readonly PaDbContext _db;
    private readonly IOutbox _outbox;
    private readonly IBundleValidator _validator;

    public async Task<SubmitResult> HandleAsync(FhirBundle bundle, string idemKey, string traceId)
    {
        var existing = await _db.PaRequests.FirstOrDefaultAsync(x => x.IdempotencyKey == idemKey);
        if (existing != null)
            return new SubmitResult(existing.PAId, existing.Status);

        var pa = _validator.MapToDomain(bundle);
        pa.IdempotencyKey = idemKey;
        pa.TraceId = traceId;
        _db.PaRequests.Add(pa);
        await _db.SaveChangesAsync();

        await _outbox.EnqueueAsync("pa.submitted.v1", new { paId = pa.Id, channel = "FHIR" });
        return new SubmitResult(pa.Id, pa.Status);
    }
}

16.3 Event Consumer (Orchestrator)

public class PaSubmittedConsumer : IConsumer<PaSubmitted>
{
    public async Task Consume(ConsumeContext<PaSubmitted> ctx)
    {
        // call eligibility + benefits with resilience policies (Polly)
        // then publish rules.evaluate command
    }
}


---

17) Integration Adapters

FHIR Adapter: Validate against profiles; handle bundles and references; translate codes (LOINC/SNOMED/CPT/HCPCS).

X12 Adapter: AS2 transport, 278/275 parsing; ACK handling; TA1/999; map to domain.

Legacy SOAP: Wrap with façade + circuit breaker; translate to canonical model.



---

18) Deployment & Environments

Envs: dev → test → perf → prod; isolated PHI data.

Blue/Green for intake & status; canary for rules.

Secrets: Managed identities, KMS; rotate certs quarterly.

DR: Multi‑AZ; warm standby region; cross‑region replicated object store.



---

19) Testing Strategy

Contract tests (PACT) for FHIR & X12 adapters.

Golden path e2e: auto‑approval & manual review.

Chaos tests: dependency latency, message loss.

Security: static analysis, SCA, DAST, fuzzing for FHIR payloads.



---

20) Sample ADRs (Concise)

ADR‑001: Event Bus — Kafka vs Azure Service Bus

Context: Need both pub/sub and ordered processing.

Decision: ASB for commands/queues (DLQ, sessions); Kafka for analytics streams where needed.

Status: Accepted. Consequences: Dual ops overhead; clearer separation of concerns.


ADR‑002: FHIR‑Native vs X12‑First

Decision: FHIR‑Native canonical model; X12 mapped at edges. Rationale: future‑proofing, CMS rules.


ADR‑003: Rules Engine

Decision: Start with configurable rules engine + externalized policy store; evolve to guideline integrations (InterQual/MCG) in v2.


ADR‑004: Data Store

Decision: PostgreSQL primary; JSONB for FHIR artifacts; Blob for documents.


ADR‑005: AuthZ

Decision: OPA sidecar for ABAC decisions; Rego policies versioned, code‑reviewed.



---

21) Backlog (MVP Cut)

1. FHIR intake endpoint with idempotency, validation


2. X12 278 gateway adapter (receive & respond)


3. Orchestrator saga with eligibility & benefits calls


4. Rules evaluation + decision outcomes


5. Reviewer UI (basic) + work queue


6. Document upload (pre‑signed) + AV scan


7. Status API + webhooks + X12 A6/A13


8. Audit trail (WORM) + observability baseline




---

22) Risks & Mitigations

Partner variability (EHR/coding): strict profiles + conformance reports; sandbox program.

Legacy flakiness: circuit breakers, caches, fallbacks to pend.

PII leakage: log scrubbing, egress scanners.

Rule drift: versioned rules with A/B shadow evaluation.



---

23) What “Good” Looks Like at Go‑Live

95%+ FHIR validation pass rate on first attempt

30%+ auto‑approval

10k/day submissions with p95 < 300ms intake

Zero P1 security incidents; audit completeness ≥ 99.99%



---

24) Epilogue — Conversation Wrap

SA: This design gives us a compliant, standards‑aligned PA platform with clear separation of concerns and strong reliability.

SE: And it’s pragmatic: adapters at the edges, a consistent domain model, automation where possible, and human-in‑the‑loop when necessary. We can iterate toward more AI‑assisted triage and guideline integrations.

SA: Great. Let’s turn this into tickets and wireframes next.

