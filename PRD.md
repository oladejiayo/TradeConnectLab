# TradeConnect Lab — Product Requirements Document (PRD)
**Document owner:** You (Connectivity Engineer)  
**Version:** 1.0  
**Status:** Draft (implementation-ready)  
**Target audience:** Engineering (FIX), Ops/Production Support, QA/Automation, Platform/Infra, BA/PM stakeholders

---

## 1. Executive summary
**TradeConnect Lab** is a self-contained platform that simulates a bank-grade FIX connectivity stack and the surrounding operational ecosystem: **venue adapters**, **session management**, **business mapping/transformation**, **CME-style drop copy processing**, **event distribution via Solace**, **migration tooling**, **certification testing**, **CI/CD**, and **observability (OTEL)**.

The end goal is not just “a demo that sends FIX,” but a platform that proves competence across:
- FIX connectivity workflows & troubleshooting
- Adapter-based onboarding/migration across venues/asset classes
- Data transformation & mapping in trading workflows
- Solace-based eventing patterns
- Java microservices engineering
- CI/CD + deployment automation
- Production support readiness + high availability design
- Observability (logs/metrics/traces with OTEL)

---

## 2. Problem statement
Many engineers struggle to gain credible, end-to-end experience in FIX connectivity roles because:
- Real venues/exchanges require certification environments and credentials.
- “Adapter platforms” (e.g., vendor solutions) are proprietary and not easily reproducible.
- Operational excellence (runbooks, monitoring, migration safety) is what differentiates senior engineers, but isn’t practiced in toy projects.

TradeConnect Lab provides a realistic lab environment to develop and demonstrate these skills without requiring real exchange access.

---

## 3. Goals and success criteria

### 3.1 Goals
1. **Multi-venue FIX connectivity** (initiator to multiple acceptors) with realistic session behaviors.
2. **Adapter Pack framework** that mimics vendor adapter onboarding patterns.
3. **Mapping/transformation engine** to normalize venue FIX into a canonical trading model.
4. **Solace messaging integration** for pub/sub event distribution to downstream services.
5. **Drop Copy ingestion** (simulated CME-style copies of executions and busts) + correlation.
6. **Migration tooling**: shadow, compare, cutover, rollback.
7. **Certification/onboarding test harness** producing artifacts and evidence.
8. **Observability + production ops**: OTEL-based tracing, metrics, logs; dashboards and runbooks.
9. **CI/CD + deployment automation**: repeatable builds, tests, containerization, deploy.

### 3.2 Success criteria (definition of done)
- You can run `docker compose up` and get a working environment:
  - FIX sessions establish and stay stable
  - Orders flow and map to canonical events
  - Events publish to Solace and consumers process them
  - Drop copy messages ingest and reconcile
  - Migration workflow executes with UI and audit trail
  - OTEL traces/metrics/logs visible in chosen tooling
  - CI pipeline runs unit + integration tests and publishes a test report artifact

---

## 4. Non-goals (explicitly out of scope for MVP)
- Connecting to real CME or live venues.
- Implementing proprietary vendor internals (Rapid Addition, etc.). We implement **equivalent workflows**, not proprietary features.
- Full OMS/EMS or full risk platform. Downstream services remain minimal.

---

## 5. Personas & primary user stories

### 5.1 Personas
1. **Connectivity Engineer (primary)**
   - Configures sessions, adapters, mapping rules, migrations
   - Diagnoses FIX issues, sequence gaps, rejects, resend storms

2. **Onboarding / Certification Analyst**
   - Runs certification packs for a new venue adapter
   - Produces onboarding evidence and sign-off artifacts

3. **Production Support (L1/L2)**
   - Monitors session health, investigates incidents
   - Executes runbooks (resend, reset, isolate bad flow)

4. **Downstream Consumer Engineer**
   - Subscribes to canonical topics for positions/risk/analytics

5. **BA/PM**
   - Reviews migration progress and acceptance evidence

### 5.2 High-level user journeys

#### Journey A — Onboard a new venue adapter
1. Create an Adapter Pack (metadata + session template + mapping bundle + validation rules)
2. Register it in the platform
3. Create environment-specific session configs (UAT/PROD)
4. Run certification pack
5. Produce onboarding report + operational runbook

#### Journey B — Operate FIX sessions in production-like mode
1. View live session health and throughput
2. Identify reject spikes and drill into message samples
3. Trigger resend request and verify resolution
4. Perform controlled sequence reset (if permitted by policy)
5. Record incident and resolution steps in audit log

#### Journey C — Migrate from legacy session to new adapter/session
1. Enable “shadow mode” for new path
2. Compare canonical output vs legacy output
3. Review divergences and fix mapping/rules
4. Execute controlled cutover
5. Validate stability; rollback if required

#### Journey D — Validate post-trade truth via drop copy
1. Ingest drop copy messages (fills/busts/corrections)
2. Correlate to internal order flow
3. Publish golden execution stream
4. Flag mismatches for ops review

---

## 6. Product scope

### 6.1 Repository structure (recommended)
- `/frontend` — React Ops Console
- `/services/fix-edge` — FIX session manager + Adapter Host (Java/Spring Boot)
- `/services/mapping-service` — canonicalization, validation, enrichment (Java/Spring Boot)
- `/services/dropcopy-service` — drop copy ingestion + reconciliation (Java/Spring Boot)
- `/services/consumer-positions` — simple consumer for canonical events (Java or Node)
- `/services/sim-venue-a` — FIX acceptor simulator
- `/services/sim-venue-b` — FIX acceptor simulator
- `/infra` — docker-compose, k8s manifests/Helm, OTEL collector config, CI scripts
- `/docs` — runbooks, onboarding guides, architecture notes
- `/cert-packs` — YAML-defined certification scenarios per adapter/venue

### 6.2 Environments
- **Local**: docker-compose with Solace container, Postgres, OTEL collector, services
- **Dev/UAT**: k8s namespace deployment (optional in MVP, required in Phase 2)

---

## 7. Architecture overview

### 7.1 Logical component diagram
```mermaid
flowchart LR
  TraderSim[Order Source / Test Harness] -->|FIX| FixEdge
  VenueA[Venue A Simulator<br/>FIX Acceptor] <-->|FIX| FixEdge
  VenueB[Venue B Simulator<br/>FIX Acceptor] <-->|FIX| FixEdge

  FixEdge -->|Raw FIX events| Mapping
  FixEdge -->|Session events| OpsAPI

  Mapping -->|Canonical events| Solace[(Solace PubSub+)]
  DropCopy[DropCopy Service] -->|Golden executions| Solace

  Solace --> Positions[Positions Consumer]
  Solace --> OpsStream[Ops Stream Consumer]

  FixEdge --> PG[(Postgres)]
  Mapping --> PG
  DropCopy --> PG

  Frontend[React Ops Console] --> OpsAPI[Ops API Gateway<br/>(or direct service endpoints)]

  subgraph Observability
    OTel[OTEL Collector]
  end

  FixEdge --> OTel
  Mapping --> OTel
  DropCopy --> OTel
  Positions --> OTel
  Frontend --> OTel
