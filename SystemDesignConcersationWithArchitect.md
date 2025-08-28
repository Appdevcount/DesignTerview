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



Conversational System Design — Healthcare Prior Authorization (End‑to‑End, Azure‑Native)

> Format: A back‑and‑forth between a Senior Architect (SA) and a Senior Engineer (SE). We go from ideation → requirements → HLD → LLD → security → scaling → observability → data → APIs → events → deployment → testing → ADRs. Code snippets use C# (.NET). Standards: FHIR R4, X12 278/275, HIPAA. Cloud stack: Microsoft Azure.




---

0) Elevator Pitch

SA: Let’s build a Prior Authorization (PA) Platform for a payer that receives PA requests from providers, validates eligibility, applies policy rules, facilitates clinical review, and responds with determinations. We’ll integrate with EHRs via FHIR and clearinghouses via X12 278. Core tenets: compliance, automation, transparency, resilience, and Azure‑native architecture.

SE: We’ll leverage Azure API Management, AKS, Functions, Service Bus, Event Hubs, App Services, Azure SQL/Cosmos DB, Blob Storage, Key Vault, and Azure Monitor. Target: < 5 min decisioning for auto‑approved cases, and 99.9% availability.


---

1) Ideation & Problem Framing

SA: What problem are we solving?

SE: Providers face delays submitting PAs across multiple proprietary portals. Payers struggle with inconsistent data and manual reviews. We want a single, standards‑based entry point, automated checks, and auditable decisions—all running on Azure for scalability and compliance.

SA: Who are our stakeholders?

SE:

Providers (EHR systems) → submit via FHIR APIs (APIM + AKS)

Clearinghouses → X12 278 via Azure B2B Gateway or Logic Apps

Payer UM reviewers → Reviewer UI hosted on Azure App Service

Members → status tracking via Azure API + Notifications

Payer back‑office → Eligibility/Benefits (legacy SOAP bridged with Logic Apps)

Compliance/Security → Azure Security Center + Key Vault



---

2) Scope & Assumptions

SA: MVP scope?

SE:

Inbound channels: FHIR API via APIM, X12 278 via Logic Apps + Integration Account, and Provider Portal (App Service).

Outbound: FHIR Communication APIs, X12 278 responses, Azure Communication Services (SMS/Email).

Attachments: FHIR DocumentReference, X12 275, stored in Blob Storage with AV scanning (Defender for Storage).

Rules: hosted in Azure Functions with versioned policies in Cosmos DB.


SA: Constraints?

SE: HIPAA compliance, BAA signed, multi‑region DR, encryption with Key Vault‑backed keys.


---

3) Non‑Functional Requirements (Azure Mapping)

Security: OAuth2/OIDC via Azure AD B2C (providers) and Azure AD (internal).

Performance: Ingestion via APIM with caching; async processing via Service Bus.

Reliability: Geo‑replicated SQL/Cosmos, ZRS Blob Storage; DLQs in Service Bus.

Observability: Azure Monitor, App Insights, Log Analytics, Azure Sentinel.

Compliance: HIPAA, PHI masked in logs, immutable Blob WORM.



---

4) Target Product Capabilities

MVP: Intake (FHIR/X12/Portal), eligibility, benefits, rules, reviewer UI, determination, notifications, audit.

v2: ML triage (Azure ML), guideline service integration, appeal workflow, Power BI dashboards, Cosmos DB analytical store.


---

5) High‑Level Architecture (HLA)

SA: Azure architecture sketch?

SE:

Providers/EHR ----> Azure API Management ----> AKS (FHIR Intake, Validation, Orchestrator)
Clearinghouse ---> Logic Apps + Integration Account (X12 278/275) ---> Service Bus (normalized events)
Provider Portal -> App Service (React/Blazor) ----> APIs (AKS)

                  +------------------------------------+
                  | Azure Service Bus (Pub/Sub + DLQ)  |
                  +------------------------------------+
                           |        |          |   
                           v        v          v   
             Azure Function (Rules)  Azure App Service (Reviewer UI) 
                           |        v
                           v   Azure SQL / Cosmos DB
                     Determination Svc in AKS
                           |
                           v
                Azure Event Hub --> Analytics / Power BI

Documents -> Blob Storage (immutable, AV scan)
Audit Logs -> Azure Data Lake + Sentinel
Notifications -> Azure Communication Services


---

6) Core Domain & Contexts (Azure Deployment)

Intake (AKS Pods): FHIR & X12 adapters, validation, dedupe.

Eligibility/Benefits: Logic Apps + SOAP wrappers; results cached in Redis (Azure Cache for Redis).

Rules Engine: Azure Functions + Cosmos DB policy store.

Clinical Review: Reviewer UI (App Service), queue from Service Bus.

Decisioning: Stateful orchestrator in AKS.

Docs: Blob Storage + Event Grid for processing.

Audit/Compliance: Data Lake Gen2 + immutable storage.



---

7) Events (Service Bus + Event Grid)

pa.submitted.v1 (Service Bus Topic)

eligibility.checked.v1

benefits.evaluated.v1

rules.evaluated.v1

document.attached.v1 (Event Grid triggers re‑evaluation)

review.completed.v1

pa.determined.v1

pa.status.changed.v1


SA: Why Service Bus vs Event Hub?

SE: Service Bus for transactional workflows (DLQ, sessions). Event Hub for telemetry/analytics (to Data Lake/Power BI).


---

8) Security (Azure Services)

AuthN: Azure AD B2C (providers), Azure AD (internal staff).

AuthZ: OPA sidecar in AKS pods; Rego policies in GitHub + deployed via Azure DevOps.

Data Protection: Always Encrypted (SQL), TDE, Blob SSE with CMK in Key Vault.

Secrets: Managed identities + Key Vault.

Audit: Diagnostic logs shipped to Azure Sentinel.



---

9) Reliability Patterns (Azure)

Service Bus DLQ + Logic App remediation flows.

Event Grid retries with exponential backoff.

Azure Traffic Manager for multi‑region failover.

AKS Horizontal Pod Autoscaler (HPA) based on RPS.

Cosmos DB multi‑region write.



---

10) Observability (Azure)

Tracing: App Insights distributed tracing.

Metrics: APIM metrics, Service Bus queue depth, Cosmos RU usage.

Logging: App Insights + Log Analytics workspace.

Alerts: Azure Monitor action groups.



---

11) Deployment Strategy

IaC: Bicep/ARM or Terraform.

CI/CD: Azure DevOps Pipelines → AKS, Functions, Logic Apps.

Blue/Green: via AKS Ingress + Traffic Split.

Secrets: Managed Identities; Key Vault integration in AKS.

DR: Active‑passive with geo‑replication (SQL, Cosmos, Blob).



---

12) Example ADRs (Azure Specific)

ADR‑001: Event Bus — Azure Service Bus vs Event Hub

Decision: Service Bus for workflows, Event Hub for analytics.


ADR‑002: Database — SQL vs Cosmos DB

Decision: SQL for strong consistency (eligibility, determinations), Cosmos DB for rules, document metadata, and flexible querying.


ADR‑003: API Management

Decision: Use APIM as unified entry point (FHIR APIs, OAuth2, rate limiting, CORS).


ADR‑004: Secrets Management

Decision: All secrets in Azure Key Vault, rotated by policy.


ADR‑005: Observability

Decision: Azure Monitor + App Insights as standard telemetry stack.



---

13) Epilogue

SA: This Azure‑native architecture leverages managed services for reliability and compliance while giving us flexibility with AKS for domain services.

SE: Exactly—cloud‑native where possible (Functions, Logic Apps, Service Bus), custom where domain complexity requires (AKS orchestration, reviewer UI). It’s scalable, secure, and future‑proof.

SA: Let’s align with security and compliance teams, then start PoCing intake and orchestration in AKS.


---

25) Azure‑Native Architecture Mapping & Enhancements

> This augments every major decision with Azure services, deployment topology, policies, and actionable snippets.



25.1 Azure Service Mapping (By Capability)

API Ingress & Governance: Azure API Management (APIM) (external + internal), Front Door (WAF) for global routing & WAF, App Gateway (WAF) for regional L7 + mTLS to AKS/ACA.

B2B EDI/X12/AS2: Logic Apps (Standard) with Integration Account (schemas/maps), AS2/EDI connectors; private endpoints to storage & Key Vault.

Compute (Microservices): AKS (mission‑critical, sidecars like OPA/Envoy) or Azure Container Apps (ACA) (serverless scale, Dapr for pub/sub & bindings). Reviewer UI on App Service or ACA.

Events & Messaging: Azure Service Bus (ASB) (queues, topics, sessions, DLQ) for commands/workflows; Event Hubs (Kafka‑compatible) for streaming/analytics; optional Storage Queues for simple fan‑out.

Data Stores: Azure SQL (OLTP, strong relational), Cosmos DB (SQL API) for high‑scale JSON (FHIR bundles, projections), Azure Blob Storage with Immutable (WORM) legal hold for docs/audit, Azure Files only if SMB is required.

Caching: Azure Cache for Redis (Enterprise) for token, eligibility memoization, and rate‑limit counters.

Search: Azure AI Search for PA lookups with role‑aware indexers.

Security & Secrets: Entra ID (Azure AD) (OIDC/OAuth2), Managed Identities, Key Vault (HSM‑backed keys), Defender for Cloud for posture.

Observability: Application Insights (traces, metrics), Log Analytics (central logs), Azure Monitor Alerts, Workbooks for SLOs.

Data & Analytics: Azure Data Lake Storage Gen2 → Synapse/Microsoft Fabric for warehouse & BI; PHI governance via Purview.


25.2 Azure Reference Topology (Hub‑Spoke with Private Endpoints)

Internet/Partners
   │
   ├── Azure Front Door (WAF) ──► APIM (External)
   │                               │
   │                               ├── Public products (FHIR intake, Status)
   │                               └── B2B route ► Logic Apps (AS2/X12) via VNet PE
   │
   └── Partner VPN/ExpressRoute ► Hub VNet (Firewall, Bastion)
                                   │
                                   └─► Spoke VNet(s):
                                         - AKS/ACA Subnet (private)
                                         - APIM Internal Subnet (for internal APIs)
                                         - Data Subnet: SQL/Cosmos/Redis via Private Endpoints
                                         - Integration Subnet: Service Bus/Event Hubs

Private Link/Endpoints for SQL, Cosmos, Storage, Service Bus, Event Hubs, Key Vault.

NSGs + Azure Firewall with FQDN tags; Private DNS Zones for PEs.


25.3 Identity, Access & API Policies (APIM)

Auth: Entra ID for OAuth2; Client‑credential flow for system‑to‑system; SMART‑on‑FHIR profile for EHR apps.

mTLS: APIM policies to require client cert for B2B; certs stored in Key Vault; rotate via automation.

Rate Limiting/Quota: per‑tenant products; spike arrest; retry‑after headers.

Idempotency: enforce Idempotency-Key header and cache handshake with Redis.

CORS: allow least‑privilege origins; dynamic origin lists via APIM named values.


APIM Sample Policy (extract):

<policies>
  <inbound>
    <validate-jwt header-name="Authorization" failed-validation-httpcode="401" require-scheme="Bearer">
      <openid-config url="https://login.microsoftonline.com/<tenant>/v2.0/.well-known/openid-configuration" />
      <required-claims>
        <claim name="aud">
          <value>api://prior-auth</value>
        </claim>
      </required-claims>
    </validate-jwt>
    <check-header name="Idempotency-Key" failed-check-httpcode="400"/>
    <rate-limit calls="100" renewal-period="60" />
    <set-header name="x-trace-id" exists-action="override">
      <value>@(context.Variables.GetValueOrDefault("request-id"))</value>
    </set-header>
  </inbound>
  <backend>
    <forward-request />
  </backend>
  <outbound>
    <set-header name="Strict-Transport-Security" exists-action="override">
      <value>max-age=31536000</value>
    </set-header>
  </outbound>
</policies>

25.4 Orchestration on Azure

Saga/Workflow: Keep orchestration in code within AKS/ACA (e.g., MassTransit + ASB sessions) for portability; optionally use Logic Apps for partner‑facing EDI orchestrations and human approvals (non‑PHI where possible).

Rules Engine: Host rules svc in AKS/ACA, rules in Blob (versioned) or Cosmos; hot‑reload; protect with mTLS + APIM.


25.5 Storage Decisions (ADR‑Expanded)

ADR‑004A: Use Azure SQL for PA core (ACID + relational joins, auditing). Store FHIR bundle snapshots in Cosmos DB (JSONB‑like) or as Blob with hash; reference by PAId.

ADR‑004B: Blob Storage with immutable policies and legal holds for clinical docs & audit (meets retention).


25.6 Messaging Design on Azure

Commands/Work queues: Service Bus queues with sessions for PAId ordering; DLQs monitored by runbooks.

Domain events: Service Bus Topics for internal fan‑out; Event Grid for external webhooks (low PHI) with validation handshake.

Stream analytics: emit de‑identified events to Event Hubs → Stream Analytics/Synapse.


25.7 Observability & Audit (Azure Monitor Stack)

App Insights: Distributed tracing across APIM, AKS/ACA, Logic Apps.

Log Analytics: Central workspace; diagnostic settings on Service Bus, SQL, Storage, Key Vault.

Dashboards/Workbooks: SLOs—intake p95, auto‑approval %, DLQ depth, reviewer SLA.

Audit: Write decision/audit events to Blob (WORM) and Event Hubs (for immutable ledgering in DW). Consider Azure Confidential Ledger for tamper‑evident metadata.


25.8 Security Posture

Defender for Cloud policies + regulatory compliance (HIPAA/HITRUST blueprints).

Azure Policy to enforce private endpoints, TLS1.2+, approved SKUs, geo restrictions.

Key Vault + Managed HSM for signing (e.g., X12, JWT), CMKs for SQL TDE & Storage.

Confidential Computing (optional) for sensitive rules eval using CCI/SEV‑SNP on AKS node pools.


25.9 Deployment & CI/CD

IaC: Bicep or Terraform for all resources; separate stacks per environment; PR‑driven.

Pipelines: GitHub Actions or Azure DevOps; container builds → image scan → deploy to AKS/ACA with blue/green; APIM revisions + auto‑tests.


Bicep (excerpt):

resource sb 'Microsoft.ServiceBus/namespaces@2022-10-01' = {
  name: '${prefix}-sb'
  location: location
  sku: { name: 'Premium' }
}
resource topic 'Microsoft.ServiceBus/namespaces/topics@2022-10-01' = {
  name: '${sb.name}/pa-events'
  properties: { enableBatchedOperations: true }
}

GitHub Actions (deploy ACA excerpt):

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: azure/login@v2
        with: { creds: ${{ secrets.AZURE_CREDENTIALS }} }
      - uses: azure/container-apps-deploy-action@v2
        with:
          resourceGroup: rg-priorauth-prod
          name: pa-intake
          imageToDeploy: ghcr.io/org/pa-intake:${{ github.sha }}
          ingress: internal
          envVars: |
            SERVICEBUS_CONN=${{ secrets.SB_CONN }}
            KEYVAULT_URI=${{ vars.KV_URI }}

25.10 Enhanced Failure Handling on Azure

Resilience: Polly policies; Service Bus auto‑forward to DLQ on poison; Scheduled retries using SB scheduled messages.

Chaos: Azure Chaos Studio to inject dependency faults; verify p95 and SLA compliance.

Scale: AKS HPA on CPU/RPS/queue length; ACA KEDA scale rules on Service Bus.


25.11 Cost & Ops Guardrails

Premium SKUs only where needed (APIM, Service Bus) in prod; dev/test on Standard.

Autoscale floors/ceilings; stop non‑prod overnight.

Centralize logs to a single Log Analytics workspace with retention policies (e.g., 30–90 days hot, archive beyond).


25.12 Concrete Flow with Azure Components

1. Front Door (WAF) receives FHIR submission → routes to APIM (external).


2. APIM enforces JWT + idempotency, forwards to AKS/ACA Intake over private network.


3. Intake writes PA to Azure SQL, snapshots bundle to Cosmos/Blob, emits pa.submitted.v1 to Service Bus.


4. Orchestrator consumes; calls Eligibility/Benefits (legacy via Private Endpoint/APIM internal); caches in Redis.


5. Rules Service evaluates; if auto‑approve, write Determination and publish events; else enqueue to Clinical Review.


6. Reviewer UI (App Service/ACA) pulls work via Service Bus; docs come from Blob (SAS, limited scope, malware‑scanned).


7. Status Service projects state to Cosmos for fast reads; APIM exposes /status; for B2B, Logic Apps sends X12 278 response.


8. Notifications via Event Grid + Functions/Communication Services; Audit to Blob (WORM).



25.13 Azure‑Specific ADRs

ADR‑006: AKS vs ACA — Decision: Start with ACA for simpler ops and event‑driven scale; move hot paths to AKS if sidecars/custom mesh needed. Consequence: Two runtimes, but faster MVP.

ADR‑007: Service Bus vs Event Hubs for domain events — Decision: Service Bus Topics for internal domain events (ordering, sessions); Event Hubs only for analytics/telemetry. Consequence: Clear separation; avoid mixing.

ADR‑008: APIM Front Door vs App Gateway — Decision: Front Door for global routing + WAF; App Gateway (private) for regional L7 to AKS. Consequence: Dual layers; improved resilience.

ADR‑009: SQL vs Cosmos — Decision: SQL for PA core transactions; Cosmos for read models/status projections. Consequence: Polyglot persistence; ETL complexity mitigated with change feed.



---

26) What to Implement Next (Azure‑Ready Backlog)

1. APIM products/policies (JWT, mTLS, rate limits, CORS, idempotency)


2. ACA environment + PA Intake/Orchestrator services with KEDA (Service Bus scaler)


3. Service Bus namespace (Premium), queues/topics, topic subscriptions per BC


4. Azure SQL + Cosmos + Blob (WORM) with Private Endpoints & CMKs


5. Logic Apps (AS2/X12) + Integration Account; X12 278/275 schemas & maps


6. App Insights + central Log Analytics, DCRs for diagnostics across resources


7. GitHub Actions pipelines + Bicep for baseline platform


8. Defender policy set + Azure Policy (PE required, TLS1.2+, private DNS)




---

27) Conversation — Azure Enhancements (Selected Excerpts)

SA: For MVP, ACA or AKS?

SE: ACA gets us faster with built‑in Dapr and KEDA‑style scaling on Service Bus. If we need OPA sidecars, service mesh, or node‑level confidential compute, we’ll carve out AKS for those workloads per ADR‑006.

SA: How do we keep PHI off the wire?

SE: Private endpoints for all data planes, APIM internal for east‑west traffic, Front Door only for public north‑south. Logs scrubbed via Diagnostic Settings → Log Analytics with data collection rules; blobs set to WORM with legal holds.

SA: X12 partners can be… quirky. What’s our plan?

SE: Terminate AS2 at Logic Apps (Standard) with Integration Account; use maps to our canonical model. We’ll version MAPs/SCHEMAs in source control and promote via IaC.

SA: What if eligibility is slow?

SE: Circuit breaker + cache results in Redis with a short TTL; on repeated failure, pend with a “system unavailable” reason and alert SREs via Azure Monitor.

SA: How do we expose status to providers securely?

SE: APIM product with per‑client quotas, fine‑grained scopes; Status Service projects to Cosmos for low‑latency GETs. For webhooks, we use Event Grid + validation, and store only de‑identified payloads.

SA: Good. Let’s execute this backlog and schedule a threat model session next.

