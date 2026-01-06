
# TradeConnect Lab

A self-contained, production-inspired **FIX connectivity + adapter + mapping + Solace eventing + drop copy + migration + observability (OTEL)** lab platform.

This repo is designed as a **portfolio-grade project** to build credible hands-on experience in:
- FIX session management and troubleshooting
- Adapter-style onboarding across venues/asset classes
- Business data transformation and mapping across trading workflows
- Solace PubSub+ messaging integration
- Java microservices engineering
- CI/CD and deployment automation
- Observability with OpenTelemetry (logs, metrics, traces)
- Certification testing + migration (shadow/compare/cutover/rollback)
- Production support operational readiness (runbooks, dashboards, audit trails)

---

## What you get

- **FIX Edge**: multi-session initiator with message journaling, resend handling, and ops controls  
- **Adapter Packs**: config-driven “venue adapters” (metadata + session template + mapping/validation rules + runbook)  
- **Mapping Service**: transforms venue FIX into a **canonical trading model** and publishes events  
- **Drop Copy Service**: ingests drop copy flows (simulated) and reconciles “golden” executions  
- **Solace PubSub+**: canonical events are published to a topic taxonomy and consumed by downstream services  
- **Ops Console (React)**: dashboards + message explorer + migration center + reconciliation views  
- **OTEL Observability**: end-to-end tracing + metrics + structured logs with correlation IDs  
- **Certification Runner**: YAML-defined cert packs that produce onboarding evidence artifacts  
- **CI/CD Ready**: repeatable build + test + containerized integration test pipeline

---

## Architecture (high level)

```mermaid
flowchart LR
  VenueA[Venue A Simulator<br/>FIX Acceptor] <-->|FIX| FixEdge
  VenueB[Venue B Simulator<br/>FIX Acceptor] <-->|FIX| FixEdge

  FixEdge -->|Raw FIX| PG[(Postgres)]
  FixEdge -->|Raw FIX events| Mapping
  FixEdge -->|Session telemetry| OpsAPI

  Mapping -->|Canonical events| Solace[(Solace PubSub+)]
  DropCopy[DropCopy Service] -->|Golden executions| Solace

  Solace --> Positions[Positions Consumer]
  Solace --> OpsStream[Ops Stream Consumer]

  Frontend[React Ops Console] --> OpsAPI

  subgraph Observability
    OTel[OTEL Collector]
  end
  FixEdge --> OTel
  Mapping --> OTel
  DropCopy --> OTel
  Positions --> OTel
````

---

## Repository layout

> This is the recommended structure. Adjust if you’ve already created folders.

```
.
├─ frontend/                    # React Ops Console
├─ services/
│  ├─ fix-edge/                 # FIX initiator + adapter host + journaling
│  ├─ mapping-service/          # canonicalization + validation + enrichment
│  ├─ dropcopy-service/         # drop copy ingestion + reconciliation
│  ├─ consumer-positions/       # example downstream consumer
│  ├─ sim-venue-a/              # FIX acceptor simulator
│  └─ sim-venue-b/              # FIX acceptor simulator
├─ infra/
│  ├─ docker-compose.yml        # local lab environment
│  ├─ otel-collector.yaml       # telemetry pipeline config
│  └─ k8s/                      # (optional) manifests/helm for Phase 2
├─ cert-packs/                  # YAML-defined certification scenarios
├─ docs/
│  ├─ runbooks/                 # prod-support runbooks
│  └─ architecture/             # notes/diagrams
└─ README.md
```

---

## Quick start (local)

### Prerequisites

* Docker + Docker Compose
* Java 17+ (if running services outside containers)
* Node 18+ (if running the frontend locally)

### 1) Start the full stack

```bash
docker compose -f infra/docker-compose.yml up -d
```

### 2) Verify services

* **Ops Console**: [http://localhost:3000](http://localhost:3000)
* **Ops API**: [http://localhost:8080](http://localhost:8080) (example; adjust per your config)
* **Solace UI**: (depends on docker-compose ports; see `infra/docker-compose.yml`)
* **Postgres**: `localhost:5432` (if exposed)

> If you don’t see expected ports, open `infra/docker-compose.yml` and confirm mappings.

### 3) Create activity (simulate venue traffic)

You have two options:

* Use built-in simulator scripts (recommended)
* Use the certification runner to generate predictable flows

Example (placeholder):

```bash
# Example: trigger Venue A simulator scenario "basic-order-fill"
curl -X POST http://localhost:8081/sim/scenarios/basic-order-fill
```

---

## Core concepts

### Adapter Packs

Adapters are versioned “venue integration bundles”:

* `adapter.json` — metadata
* `session-template.yaml` — recommended FIX session config
* `mapping-rules/*.yaml` — mapping DSL rules
* `validation-rules.yaml` — required tags, enums, constraints
* `runbook.md` — known rejects and standard ops actions

Adapters follow a lifecycle:
**Register → Validate → Enable → Certify → Promote**

### Canonical event topics (Solace)

Initial taxonomy:

* `canon.order.<env>.<venue>.<assetClass>`
* `canon.exec.<env>.<venue>.<assetClass>`
* `canon.session.<env>.<venue>`
* `canon.migration.<env>`
* `canon.dropcopy.<env>.<venue>`

---

## Ops Console (features)

* **Sessions Dashboard**

  * session state, seq nums, heartbeat freshness, throughput, reject rate
  * actions: start/stop, resend request, quarantine
* **Message Explorer**

  * filter by session/msgType/time, inspect raw FIX + canonical output
* **Mapping Health**

  * rule errors, missing tags, validation failures
* **Migration Center**

  * shadow/compare/cutover/rollback with divergence drill-down
* **Drop Copy Reconciliation**

  * matched vs unmatched, mismatches, bust/corrections

---

## Observability (OpenTelemetry)

Every service emits:

* **Traces**: fix receive → journal → mapping → publish → consume
* **Metrics**: throughput, reject rates, resend depth, mapping latency/errors
* **Logs**: structured logs with `traceId`, `correlationId`, `sessionId`, `adapterId`

Telemetry is routed via `infra/otel-collector.yaml`.

---

## Certification packs

Certification packs are YAML scenario definitions stored under `cert-packs/`.

A cert run produces:

* FIX transcript (in/out)
* Canonical event snapshots
* pass/fail summary
* reconciliation evidence (for drop copy scenarios)

Example (placeholder):

```bash
# Example: run cert pack "venue-a-basic"
./gradlew :services:cert-runner:run --args="--adapter VENUE_A --pack venue-a-basic --env DEV"
```

---

## CI/CD

This repo is intended to support:

* unit tests
* contract tests (FIX fixtures → canonical snapshots)
* containerized integration tests (docker-compose in CI)
* build artifacts (reports/evidence)
* image publishing (optional)

See `.github/workflows/` (or your CI system) once added.

---

## Roadmap

### Phase 1 — MVP (“credible connectivity lab”)

* multi-session FIX edge + journaling
* adapter packs + mapping service + Solace publishing
* drop copy ingestion + reconciliation
* basic migration (shadow + compare)
* core ops UI
* OTEL instrumentation
* CI pipeline with integration tests

### Phase 2 — Production realism

* durable queues + replay
* HA strategy for sessions (active/passive or partitioned)
* stronger authn/z and audit controls
* cutover/rollback policy gates
* richer simulators + failure injection

---

## Contributing

This is a learning/portfolio repo, but contributions are welcome:

1. Create a branch: `feature/<short-name>`
2. Add/modify adapter packs under `adapters/` (or your chosen path)
3. Add cert packs under `cert-packs/`
4. Ensure tests pass locally and in CI
5. Open a PR with a clear description and evidence

---

## License

Choose one:

* MIT (recommended for public learning repos)
* Apache-2.0

Add `LICENSE` once decided.

---

## FAQ

### “Is this a real Rapid Addition adapter implementation?”

No. This project implements **adapter-style workflows** (packaging, config, validation, onboarding, certification, ops controls, migration), not proprietary vendor internals.

### “Is this using real CME Drop Copy?”

No. Drop copy is simulated, but the reconciliation and processing patterns are realistic and interview-relevant.

---


