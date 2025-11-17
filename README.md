# Telemedicine Platform 📱💻🩺

**Goal:** Connect patients and clinicians for secure video visits, chat, triage, and prescriptions — reliable, private, and easy to use.

---

## 🔎 Overview
A scalable, secure telemedicine system supporting:
- Web & mobile patient/provider portals
- Real-time video consults (WebRTC via SFU/MCU)
- Chat & presence
- Electronic prescriptions (e‑Rx)
- Scheduling, audit logging, notifications

Non-functional priorities: confidentiality, auditability, resilient video across variable networks, and 24/7 availability.

---

## 🎯 Stakeholders
- **Patients** — book, join visits, receive prescriptions
- **Clinicians** — schedule, consult, write e‑Rx, access patient notes
- **Admins / Support** — manage providers, compliance, audit
- **Pharmacies** — receive e‑Rx (optional integration)
- **DevOps / SRE** — deploy, monitor, maintain SLAs

---

## 🧾 User Stories
- As a **patient**, I can book an appointment and join a video visit.
- As a **clinician**, I can start a secure video consult and create an e‑prescription.
- As an **admin**, I can view audit logs for compliance.
- As a **pharmacy**, I can receive machine-readable e‑Rx.
- As a **system**, I will retry transient failures and keep operations idempotent.

---

## 🧭 Context & Use-cases (ASCII flow)
```
Patient -> Booking UI -> Scheduling Service -> DB(appointments)
Patient -> Join -> Signaling + SFU -> Video Stream
Clinician -> Consultation UI -> Notes Service (doc-store)
Clinician -> Send eRx -> Prescription Service -> Pharmacy / Patient
Events -> Message Queue -> Audit/Notification/Analytics
```

---

## ⚖️ Constraints & Assumptions
- Must handle protected health information (PHI) → HIPAA/GDPR considerations
- Real-time video must survive network changes (adaptive bitrate)
- Target: tens of thousands of daily users (scale plan needed)
- External integrations (SMS, email, pharmacy) via secure APIs

---

## 🏛️ Architecture (high level)
- **Frontend:** Micro frontends — patient portal, clinician portal, admin console
- **API Layer:** Gateway (OIDC, rate limit, routing)
- **Services:** Auth, Scheduling, Video Signaling, Media (SFU/MCU), Chat, Prescription, Notes, Audit, Notification
- **Integrations:** SMS/Email providers, Pharmacy APIs, CDN for static content
- **Data stores:** Relational DB (PHI), Document DB (notes), Audit/event store (append-only), Object storage (attachments)
- **Infrastructure:** Kubernetes, message queue (Kafka/RabbitMQ), CDN, Load balancers

Diagram (simplified):
```
[Clients] -> [API Gateway] -> {Auth, Scheduling, Chat, Prescriptions, Notes}
                                  \-> [Signaling] -> [SFU / Media Service] -> CDN
Message Queue <-> Services -> Audit Store
Relational DB (transactions) + Doc DB (notes) + Object Storage
```

---

## 🧾 Architecture Trade-offs: Monolith vs Microservices vs Serverless
**Monolith**
- Pros: simpler dev/debug, easier local testing
- Cons: harder to scale individual media/chat components, slower release cycles

**Microservices (recommended)**
- Pros: scale media and signaling independently, fine-grained ownership, polyglot choices
- Cons: more complex ops (deployments, tracing), cross-service transactions

**Serverless**
- Pros: low ops for sporadic workloads, quick autoscaling for event-driven parts
- Cons: cold starts for latency-sensitive paths (video signaling), vendor lock-in

**Recommendation:** Hybrid: microservices for critical real-time & transactional parts; serverless for asynchronous tasks (notifications, thumbnails).

---

## 🔧 Component & Deployment Notes
- Deploy media/SFU on node pools with high network bandwidth & enabled SR-IOV if available.
- Use an ingress with sticky session or token-based WebRTC routing for signaling.
- Separate DB clusters for PHI with strong encryption and backups.

---

## 🧩 Design Patterns Used
- **Proxy (media proxy)** — isolate clients from media servers; centralize access control.
- **Builder (prescription)** — construct e‑Rx objects step-by-step; validation hooks.
- **CQRS (Command & Query Responsibility Segregation)** — separate read-optimized stores for scheduling & audit queries.
- **Retry & Idempotency** — commands are idempotent; retry policies with exponential backoff.
- **Circuit Breaker** — protect downstream services (pharmacy, SMS) from cascading failures.

**Anti-patterns avoided**
- Chat-in-db: don’t store live messages synchronously in the relational DB; use event streams.
- Shared DB across services: avoid tight coupling and single point of schema change.

---

## 🗄️ Database Design (summary)
### Stores
- **Relational DB (primary PHI):** users, providers, appointments, prescriptions (normalized)
- **Document DB:** consultation notes, visit transcripts (flexible schema)
- **Audit/Event store (append-only):** all meaningful events with immutability
- **Object storage:** attachments (consent forms, images)

### ERD (core tables, simplified)
- `users` (id, name, email, role, contact, encrypted_meta)
- `providers` (id, user_id, specialty, license_info)
- `appointments` (id, patient_id, provider_id, start, end, status, timezone)
- `prescriptions` (id, appointment_id, prescriber_id, created_at, payload_encrypted)
- `notes` (id, appointment_id, author_id, doc_json)
- `audit_events` (id, actor_id, action, resource, timestamp, details_hash)

### Normalization & Rationale
- Normalize patient/provider core info to avoid duplication and ensure consistency.
- Denormalize read-heavy query views (e.g., provider schedule snapshots) into materialized views for performance.

### Indexing
- Index `appointments` on (provider_id, start), `users` on email (unique), `prescriptions` on appointment_id.
- Use partial indexes for active appointments.

### Partitioning & Sharding
- Partition appointments and audit_events by date (monthly) for faster range queries and archival.
- Shard by geographic region for scale and data locality if user base is globally distributed.

---

## 🚀 Performance & Scale
- **Caching:** in-memory cache (Redis) for sessions, token blacklists, read-heavy provider schedules. CDN for static assets and media fallback segments.
- **Load balancing:** L7 LB for HTTP; UDP-friendly LB for WebRTC where supported. Use autoscaling node pools per service.
- **Replication:** read replicas for relational DB; multi-AZ deployments for high availability.
- **Backpressure:** use queues and rate limiting for spikes (e.g., signing into mass telehealth events).
- **Adaptive bitrate & retransmission:** SFU should support simulcast/SVC and congestion control.

---

## 🔒 Security & Reliability
### Threat model (brief)
- **Threats:** data exfiltration, MITM on media, unauthorized access, forged prescriptions, DDoS.
- **Mitigations:** strong auth, mutual TLS between services, DLP checks, least privilege RBAC.

### AuthN / AuthZ
- Use **OIDC** + MFA for users and staff.
- Fine-grained **RBAC** (clinician vs admin vs support) and attribute-based access for patient records.

### Encryption & Data Protection
- Encryption in transit (TLS 1.3), SRTP for media.
- Encryption at rest for PHI and e‑Rx payloads (customer-managed keys preferred).
- DLP hooks before exporting attachments or prescriptions.

### OWASP Top 10 Defenses
- Input validation & output encoding, prepared statements, CSRF protections, secure cookies, CSP for frontends.

### Reliability Patterns
- Circuit breakers, retries with jitter, health checks, graceful shutdowns.
- Disaster Recovery: daily backups, point-in-time recovery for DBs, runbook & failover playbooks.

---

## 🛠️ API Spec (high-level)
> Versioning: `/v1/` and `/v2/` paths; use header `Accept: application/vnd.telemed.v2+json` for content negotiation.

### Auth
- `POST /v1/auth/login` — 200 OK / 401
- `POST /v1/auth/refresh` — idempotent token refresh

### Scheduling
- `POST /v1/appointments` — create (idempotent via client-generated `request_id`) — 201 / 409
- `GET /v1/appointments?patient_id=...` — list — 200
- `PATCH /v1/appointments/{id}` — update (idempotent on request_id)

### Video & Signaling
- `POST /v1/signaling/session` — create signaling session (returns token & SFU info)
- WebRTC ICE/TURN endpoints handled by media infra

### Chat
- `POST /v1/chat/{room}/messages` — 201 (write to event queue; ack = message id)
- `GET /v1/chat/{room}/messages?since=...` — 200

### Prescriptions
- `POST /v1/prescriptions` — create e‑Rx (Builder pattern used; validate prescriber credentials) — 201 / 400 / 403
- `GET /v1/prescriptions/{id}` — 200 (encrypted payload)

### Errors & Idempotency
- Standard error schema: `{ "code": "ERR_CODE", "message": "human friendly", "details": {}}`
- Use `Idempotency-Key` header for POSTs that create resources.

---

## 🔍 Observability
- **Logs:** structured JSON logs, sensitive fields redacted; central log store (e.g., ELK or Loki)
- **Metrics:** request latencies, error rates, CPU/memory, active WebRTC sessions, packet loss
- **Tracing:** distributed tracing (OpenTelemetry) across gateway → services → SFU
- **SLOs / SLIs:**
  - Video join success rate ≥ 99% (SLO)
  - 95th percentile API latency < 300ms
  - Error rate < 0.1% (per minute)
- **Alerting / Runbook:** paging on SLO breaks, degraded SFU capacity, DB replication lag; include runbooks for failover.

---

## 🧪 Testing & QA
- Unit & integration tests per service, contract tests for API boundaries.
- End-to-end tests (UI -> Signaling -> SFU -> Media) in staging.
- Chaos testing for network partitions and SFU failures.

---

## 🧰 Tech Stack (comparisons & trade-offs)
### Frontend
- Option A: React micro frontends — rich UX, mature ecosystem.
- Option B: SvelteKit — faster, smaller bundles but smaller ecosystem.

### Real-time Media
- Option A: Janus / Jitsi (open-source SFU) — cost-effective, flexible.
- Option B: Commercial SFU (Twilio, Agora) — easier integration, built-in global infra, cost per minute.

### Messaging / Queue
- Option A: Kafka — high throughput, durable event streams.
- Option B: RabbitMQ — simpler for traditional messaging, easier at small scale.

### Databases
- Relational: Postgres (ACID, extensions) vs MySQL (familiar). Postgres preferred for advanced features.
- Document store: MongoDB vs Couchbase. MongoDB common choice for flexible notes storage.

**Choice rationale:** Use open-source core (Postgres + Redis + Kafka + Janus) to reduce vendor lock-in; evaluate commercial SFU if global low-latency SLA is required.

---

## ✅ Deployment Checklist
- Harden API Gateway (WAF + rate limiting)
- Configure OIDC + MFA
- Provision TURN servers and SFU clusters in each region
- Enable backups and point-in-time restore for DBs
- Define retention & archival for audit logs

---

## 📜 Compliance & Legal Notes
- Treat PHI carefully: regional compliance (HIPAA, GDPR) depending on audience.
- Data residency: host PHI in-region where required.

---

## 📦 Deliverables (what to include in repo)
- `README.md` (this file)
- `arch/` diagrams & deployment manifests
- `api/` OpenAPI spec (YAML/JSON)
- `infra/` Terraform & k8s manifests
- `services/*` service skeletons and tests
- `runbook/` SRE ops playbooks

---

## 📝 Quick Example: Idempotent Appointment Create (pseudo-HTTP)
```http
POST /v1/appointments
Headers: { Idempotency-Key: "abc-123" }
Body: { patient_id: 1, provider_id: 2, start: "2025-12-01T10:00:00Z" }

201 Created
Location: /v1/appointments/42
```

---

## 📚 Further Reading & Next Steps
- Prepare OpenAPI spec for all endpoints
- Build a PoC: WebRTC signaling + Janus SFU + simple React client
- Run compliance review & pen-test before pilot

---

*Made with ❤️ — tweak and simplify as needed for presentations or proposals.*


