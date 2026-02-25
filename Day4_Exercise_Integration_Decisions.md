# Day 4 — Guided Exercise: DDD Patterns & Integration Mechanisms

**Duration:** 30 minutes total  
**Format:** Work individually for 20 minutes, then debrief in pairs for 10 minutes  
**Coverage:** LG 4-1 (DDD Strategic Design as integration decisions) + LG 4-2 (Technical mechanisms)

---

> **How to use this exercise**
>
> Each scenario gives you a real system with real constraints. Read the situation, then answer the questions. Don't skip to the next scenario without committing to an answer — the whole point is to make decisions under ambiguity, not to find the "safe" non-answer.
>
> Timing guide: Scenario 1 → 5 min · Scenario 2 → 6 min · Scenario 3 → 8 min · Scenario 4 → 11 min

---

## Scenario 1 — The Charging Station Problem

**Difficulty:** ⬜ Easy · **Time:** 5 minutes

---

VW Group operates a network of 3,200 charging stations across India through a subsidiary called **VW Charge IN**. The charging stations are manufactured by three third-party vendors — Tata Power, Exicom, and Servotech — each with their own proprietary management APIs.

A new **Charging Analytics service** needs to display real-time utilisation data (sessions started, energy dispensed, faults) for each station. The problem: Tata Power uses REST with a SOAP-era field naming convention (`<stationId>` in XML), Exicom uses a CSV polling endpoint, and Servotech has a modern JSON API but changes it with every firmware update.

The analytics team says: *"We want to just query one clean API and not care which vendor a station belongs to."*

Your job is to design the integration boundary.

---

**Q1.1 — Pattern identification** *(choose one)*

Which DDD pattern describes what you need to place between the vendor APIs and the Analytics service?

- [ ] A. Shared Kernel — both sides agree on a common charging station schema
- [ ] B. Anti-Corruption Layer — translate vendor models into your own clean domain model
- [ ] C. Conformist — Analytics service adopts whichever vendor model is most common
- [ ] D. Customer/Supplier — vendors publish events that Analytics consumes asynchronously

---

**Q1.2 — Consequence of the wrong choice** *(fill in the blank)*

If you had chosen Conformist instead, what breaks six months later when Servotech releases a firmware update that renames `energyKwh` to `totalEnergyDeliveredKwh`?

> Your answer: _______________

---

**Q1.3 — Mechanism selection** *(choose one)*

The analytics dashboard needs station utilisation data refreshed every 30 seconds. Vendors don't support webhooks — you must poll them. What integration mechanism fits between the ACL and the Analytics service?

- [ ] A. Database integration — ACL writes to a shared table, Analytics queries it
- [ ] B. REST API — ACL exposes a clean `/stations/{id}/metrics` endpoint; Analytics polls it
- [ ] C. Kafka messaging — ACL publishes `StationMetricEvent`; Analytics consumes it
- [ ] D. gRPC — ACL exposes a typed metrics service with Protobuf schema

---

**Q1.4 — Quick justification** *(one sentence)*

Why is database integration (option A above) a poor choice even though it's the simplest to implement?

> Your answer: _______________

---

## Scenario 2 — The Shared Catalogue Nobody Owns

**Difficulty:** ⬜⬜ Easy-Intermediate · **Time:** 6 minutes

---

Audi India and Škoda India both use a shared **Vehicle Configuration Catalogue** — a database of every model, trim, variant, colour, and option code available in the Indian market. Two separate teams maintain it:

- **Audi Config Team** (5 engineers in Pune) — manages Audi-specific trims, adds new variants every quarter
- **Škoda Config Team** (3 engineers in Mumbai) — manages Škoda lines, has a different release cadence

Both teams need to read and write to the same catalogue. They've been sharing a single PostgreSQL instance directly. Last month, Audi added a nullable column `launch_edition_flag` to the `variants` table. The Škoda team's Java service, which doesn't expect that column, started returning `NullPointerException` in production for 4 hours before anyone noticed.

Now they're asking you what to do.

---

**Q2.1 — Diagnose the current state** *(circle or mark)*

What DDD anti-pattern does the current shared PostgreSQL setup represent?

- [ ] A. Shared Kernel used correctly — they're sharing by design, so this is expected
- [ ] B. Database integration posing as Shared Kernel — the DB schema is the de facto API, with no explicit contract
- [ ] C. Conformist — Škoda has implicitly conformed to Audi's data model
- [ ] D. Customer/Supplier — Audi is the supplier of schema changes, Škoda the customer

---

**Q2.2 — Pattern selection for the TO-BE state**

The teams want to keep sharing the vehicle catalogue data — it genuinely belongs to both. But they need protection from each other's changes. Match each option to its consequence:

| Option | Consequence |
|--------|-------------|
| A. True Shared Kernel — extract a versioned shared library (VehicleConfigCore.jar) with explicit schema types; both teams must approve changes to it | ___ |
| B. Published Language — one canonical Avro schema in a central registry; both services publish and consume events in that schema | ___ |
| C. Keep the shared DB, but add a view layer per team | ___ |

**Consequences (match the letter):**

1. Both teams retain independent deployment; schema changes flow as new event versions; consumers ignore unknown fields
2. Changes still require coordination, but the contract is now explicit and versioned rather than implicit DB coupling
3. Still coupled at the DB level — a DDL change on the underlying table can still break the views

---

**Q2.3 — Recommendation** *(one to two sentences)*

Which option would you recommend, and what's the one condition that must be true for it to work?

> Your answer: _______________

---

**Q2.4 — Mechanism question** *(choose one)*

If you go with Published Language (Option B above), what integration mechanism do you use to actually move the data?

- [ ] A. REST — one team exposes the catalogue, the other polls it
- [ ] B. Shared PostgreSQL — keep the DB, but enforce the Avro schema at application level
- [ ] C. Kafka with an Avro schema registry — both teams produce/consume `VehicleVariantUpdated` events; registry enforces the schema contract
- [ ] D. gRPC — strict Protobuf typing replaces the shared DB

---

## Scenario 3 — The Over-Eager Platform Team

**Difficulty:** ⬜⬜⬜ Intermediate · **Time:** 8 minutes

---

CARIAD is building an internal **Developer Platform** that all 5 CARIAD product teams must use. The Platform team has decided: *"Every service must integrate through our Platform API Gateway. All inter-service calls go through it. No direct service-to-service communication."*

The gateway is a REST-based HTTP proxy with added auth, logging, and rate limiting.

Here are three real integration use cases that are now routed through this gateway:

**Use case α:** The Ingestion SCS sends 40,000 telemetry events per second from vehicles to the Processing SCS.

**Use case β:** The Dashboard SCS fetches the latest battery health prediction for a VIN when a technician opens a vehicle record. Response needed in under 200ms. Happens ~500 times/day.

**Use case γ:** When a vehicle's battery falls below the critical health threshold, the Notification SCS must alert the assigned dealer within 5 minutes. The Processing SCS determines the threshold breach.

---

**Q3.1 — Spot the misfit** *(mark all that apply)*

Which use case(s) are a poor fit for the REST HTTP gateway mechanism, and why?

- [ ] A. Use case α — 40,000 events/second over REST/HTTP adds per-request overhead; TCP connection management and gateway processing become a bottleneck; Kafka is designed for exactly this volume
- [ ] B. Use case β — 500 requests/day over REST is entirely fine; the gateway overhead is negligible at this volume
- [ ] C. Use case γ — A threshold breach event requires reliable, durable delivery with retry; if the gateway is down when the breach occurs, the alert is lost; messaging (Kafka or queue) with at-least-once delivery is safer
- [ ] D. All three are fine — a well-scaled gateway can handle any volume

---

**Q3.2 — DDD pattern for the gateway relationship** *(choose one)*

Every product team must use the Platform Gateway exactly as the Platform team defines it, with no ability to negotiate the API contract. The Platform team ships new gateway versions unilaterally; product teams must adapt.

What DDD pattern describes the relationship between product teams and the Platform team?

- [ ] A. Customer/Supplier — product teams can raise requirements; Platform team considers them in their roadmap
- [ ] B. Conformist — product teams adopt the Platform's model entirely; no negotiation
- [ ] C. Published Language — all teams agreed on the gateway API schema at the start
- [ ] D. Anti-Corruption Layer — product teams wrap the gateway in their own façade

---

**Q3.3 — The hidden cost of Conformist** *(short answer)*

The Platform team ships Gateway v2, which changes the auth header format from `X-CARIAD-Token` to `Authorization: Bearer`. All 5 product teams must update their services simultaneously or their services break.

What architectural property have you lost, and what does that cost you in practice?

> Your answer: _______________

---

**Q3.4 — Redesign** *(fill in the table)*

Propose a better integration mechanism for each use case. You cannot remove the gateway entirely — it handles auth and rate limiting at the perimeter. But inter-service calls don't need to go through it.

| Use case | Current mechanism | Better mechanism | DDD pattern at boundary |
|----------|------------------|-----------------|------------------------|
| α — 40K telemetry events/sec | REST via gateway | | |
| β — 500 health prediction requests/day | REST via gateway | | |
| γ — Critical battery alert | REST via gateway | | |

---

**Q3.5 — Pushback question**

A senior engineer on the Platform team argues: *"Kafka for telemetry is fine, but you're fragmenting the platform. Having some services use Kafka and some use REST means we need to operate and monitor two completely different infrastructure stacks."*

Is this a valid concern? How would you respond in two sentences?

> Your answer: _______________

---

## Scenario 4 — The Merger That Nobody Planned For

**Difficulty:** ⬜⬜⬜⬜ Hard · **Time:** 11 minutes

---

VW Group India has acquired **AutoBridge**, a Mumbai-based SaaS company that runs the dealer management system (DMS) used by 4,200 dealers across India. AutoBridge has 8 engineers, a monolithic Rails application backed by MySQL, and a deeply coupled data model where sales, service, inventory, finance, and customer CRM all live in one schema.

VW Group India's own systems include:
- **Dealer Inventory service** (the modular service you've been working with — PostgreSQL, team of 3)
- **VW India CRM** (Salesforce, managed by a vendor, no code access, REST API with rate limit of 600 req/min)
- **Warranty service** (SOAP-based legacy, Java EE, team of 2 who refuse to change anything)
- **Financial Reporting service** (reads from a data warehouse, Redshift, scheduled jobs at 00:00 IST)

The business requirement: **Within 6 months**, dealer staff must be able to open AutoBridge DMS and see live VW India inventory, create warranty claims that flow to the Warranty service, and have all sales data flow into Financial Reporting by end of day.

The AutoBridge team will keep running their Rails monolith. They are not migrating to microservices. They do not have Kafka. They do not have a schema registry.

You have 6 months, a team of 4 senior engineers, and the authority to add infrastructure to the VW India side — but not to the AutoBridge side.

---

**Q4.1 — Map the integration boundaries**

For each integration point below, identify the DDD pattern AND the integration mechanism. There is no single correct answer — but your choices must be internally consistent and justifiable given the constraints.

| Integration point | From → To | Constraints | DDD pattern | Integration mechanism | Your justification |
|-------------------|-----------|-------------|------------|----------------------|-------------------|
| Dealer views VW inventory in AutoBridge DMS | AutoBridge Rails → Dealer Inventory service | AutoBridge cannot add Kafka; can make HTTP calls | | | |
| Dealer creates warranty claim in AutoBridge | AutoBridge Rails → Warranty service (SOAP) | Warranty team won't change anything; SOAP only | | | |
| Sales data flows to Financial Reporting | AutoBridge MySQL → Redshift data warehouse | Reporting runs at 00:00; AutoBridge can add a webhook or scheduled job; no real-time needed | | | |
| VW CRM customer records visible in AutoBridge | Salesforce CRM → AutoBridge Rails | Salesforce: 600 req/min rate limit; REST only; no webhooks available | | | |

---

**Q4.2 — The dangerous assumption**

The product manager says: *"For the CRM integration, just have AutoBridge call Salesforce directly using the VW India API key. It's simpler — one less moving part."*

Identify **two** specific architectural risks this creates, in the context of the given constraints.

> Risk 1: _______________

> Risk 2: _______________

---

**Q4.3 — Separate Ways or not?**

The Financial Reporting team says: *"We already get sales data from our own ETL jobs that read AutoBridge MySQL directly. We don't need a formal integration — just give us read access to their DB."*

Should you grant this? Answer yes or no, then defend your position in 3 sentences.

> Your answer (Yes / No): ___

> Defence: _______________

---

**Q4.4 — The 6-month constraint changes everything**

You've been told 6 months. Your proposed integration for the warranty claim boundary involves building an ACL on the VW India side that translates AutoBridge's REST call into a SOAP envelope and forwards it to the Warranty service.

A colleague proposes a shortcut: *"Just have AutoBridge call the SOAP Warranty service directly. Give them the WSDL. They're smart engineers — they can implement SOAP."*

Which DDD pattern does your colleague's proposal result in, and what problem does it create for VW Group India specifically?

> DDD pattern: _______________

> Problem for VW Group India: _______________

---

**Q4.5 — Hardest question: the thing nobody wants to talk about**

Three months in, the AutoBridge team tells you: *"Our MySQL schema for the `service_orders` table is being restructured as part of an internal refactor. The `dealer_code` column will be split into `dealer_region_code` and `dealer_branch_code` next month."*

Financial Reporting currently reads this column from Redshift (which gets it from the ETL of AutoBridge MySQL). The ETL is a scheduled SQL job — no transformation layer.

What has gone wrong architecturally, and what would have prevented it? Be specific about the pattern and mechanism that should have been in place.

> What went wrong: _______________

> What would have prevented it: _______________

---

## Answer Key

*Reveal after 20 minutes of individual work. Use the answers to drive the pair debrief.*

---

### Scenario 1

**Q1.1 — B. Anti-Corruption Layer**

The ACL is specifically designed for this: you have multiple external systems with messy, unstable, or incompatible models, and you need to protect your domain from them. Three vendors with three different data models is the textbook ACL use case.

Why not D (Customer/Supplier)? Because the vendors don't publish events — they expose polling APIs. You're the one initiating the call. Customer/Supplier implies the upstream consciously publishes something for downstream consumers.

**Q1.2 — Conformist consequence**

The Analytics service would need to be updated to handle the renamed field. If you have three vendors and each changes their schema independently on their own release cycle, you're constantly patching the Analytics service for reasons that have nothing to do with analytics. The ACL absorbs these changes; Conformist propagates them.

**Q1.3 — C. Kafka messaging** is the best fit here, with B (REST) as a defensible second choice.

Kafka fits because: the ACL is already polling vendors on a schedule; it makes sense to publish normalised `StationMetricEvent` messages to a topic; Analytics consumes at its own pace; the ACL and Analytics can be deployed independently. If you chose REST, it's acceptable — at this volume (30-second polling, 3,200 stations) REST is fine. The key is that the ACL exposes a *clean* endpoint, not the vendor's raw format.

**Q1.4 — Database integration problem**

Database integration means the ACL's internal schema becomes an API contract. When the ACL needs to restructure how it stores vendor data (normalisation, adding vendor metadata, etc.), it breaks the Analytics service. You've coupled two services at the storage layer rather than at a well-defined interface.

---

### Scenario 2

**Q2.1 — B. Database integration posing as Shared Kernel**

This is the core confusion to resolve. A Shared Kernel requires an *explicit*, *jointly-owned*, *versioned* contract. What they have is an implicit contract defined by a PostgreSQL schema that either team can modify unilaterally. The production incident proves it: a nullable column addition (a safe DDL change by most standards) broke the other team's service.

**Q2.2 — Match**

- Option A (Shared Kernel / versioned JAR) → Consequence **2**: explicit and versioned, but changes still need coordination
- Option B (Published Language / Avro + Kafka) → Consequence **1**: independent deployment, schema evolution without breaking consumers
- Option C (Views) → Consequence **3**: still coupled at DB level

**Q2.3 — Recommendation**

Option B (Published Language with Kafka + Avro schema registry) is the strongest long-term choice, but it requires both teams to actually publish and consume events rather than directly reading a shared DB — which means the teams must shift their thinking from "shared data store" to "events as the source of truth for each team's local read model." If the teams aren't ready for that operational shift in the near term, Option A (explicit Shared Kernel as a versioned library) is a safer migration step.

**Q2.4 — C. Kafka with Avro schema registry**

REST (A) just recreates the polling problem. Shared PostgreSQL (B) defeats the purpose — you've changed nothing architecturally. gRPC (D) is viable but Protobuf schema evolution is stricter and adds operational complexity the teams don't have today.

---

### Scenario 3

**Q3.1 — A and C**

Use case α (40K events/sec through REST gateway) is a throughput mismatch. REST is synchronous and stateless; each of 40,000 events per second creates an HTTP request through the gateway, which adds latency, connection overhead, and a single point of failure. Kafka is built for exactly this: high-throughput, durable, partitioned, with consumers reading at their own pace.

Use case γ (critical alert over REST) is a reliability mismatch. If the gateway is unavailable for 30 seconds when a battery hits the critical threshold, the alert is lost. Kafka retains the event; the Notification SCS can process it when it recovers.

Use case β (500 requests/day) is fine over REST. Low volume, synchronous, low latency required — REST is the correct tool.

**Q3.2 — B. Conformist**

Product teams have no negotiating power. They adopt the Platform team's model entirely. This isn't inherently wrong — Platform teams often operate this way — but you should name it correctly, because recognising it as Conformist tells you what your exposure is (see Q3.3).

**Q3.3 — What's lost**

You've lost **independent deployability** at the product team level. When the Platform team ships a breaking change, all 5 teams must coordinate a simultaneous update — you've recreated the monolith deployment problem across service boundaries. In practice this means Platform releases require 5-team coordination windows, which slows the entire organisation's delivery cadence.

**Q3.4 — Redesign**

| Use case | Better mechanism | DDD pattern at boundary |
|----------|-----------------|------------------------|
| α — 40K telemetry events/sec | Kafka — Ingestion SCS publishes `TelemetryEvent` to a topic; Processing SCS consumes | Published Language (shared Avro schema enforced via registry) |
| β — 500 health prediction requests/day | REST direct service-to-service (no gateway) with mTLS | Customer/Supplier — Dashboard is the customer; ML Prediction is the supplier with a versioned API |
| γ — Critical battery alert | Kafka — Processing SCS publishes `CriticalHealthEvent`; Notification SCS consumes with at-least-once delivery | Customer/Supplier via async event |

**Q3.5 — The Platform team pushback**

The concern is valid — operating two infrastructure stacks (HTTP gateway + Kafka) does add complexity. But the right response is: *the cost of misusing a tool at 40,000 events/second exceeds the cost of operating Kafka; the alternative is either an overengineered gateway or degraded reliability.* The framing should shift from "one stack vs two" to "right tool for each interaction pattern" — and Kafka for streaming + REST for request-response is a standard, well-understood combination that any platform team with real streaming requirements ends up running anyway.

---

### Scenario 4

**Q4.1 — Integration boundaries**

These answers involve genuine trade-offs. Grade yourself on the quality of the justification, not just the pattern name.

| Integration point | DDD pattern | Integration mechanism | Key justification |
|-------------------|------------|----------------------|-------------------|
| AutoBridge → Dealer Inventory (view inventory) | Customer/Supplier | REST — AutoBridge calls VW Dealer Inventory's published REST API | AutoBridge is the customer; Dealer Inventory publishes a versioned endpoint. AutoBridge cannot use Kafka, so synchronous REST is the only viable option. The Dealer Inventory team owns the contract. |
| AutoBridge → Warranty service (create claim) | Anti-Corruption Layer (on VW India side) | ACL translates AutoBridge's REST call → SOAP envelope → Warranty service | You cannot change AutoBridge or the Warranty service. The ACL sits on VW India infrastructure, translates between the two models, and shields AutoBridge from the SOAP horror. |
| AutoBridge MySQL → Financial Reporting (Redshift) | Anti-Corruption Layer (ETL with transformation) | Scheduled batch ETL job (nightly) — extract from AutoBridge MySQL, transform to VW reporting schema, load into Redshift | No real-time requirement; batch is appropriate. But the ETL must include a transformation layer, not a direct column copy — this is where Q4.5 becomes relevant. |
| Salesforce CRM → AutoBridge | Anti-Corruption Layer (on VW India side, as a cache/proxy) | REST proxy/cache service on VW India side — periodically syncs Salesforce data (respecting 600 req/min limit) to a local store; AutoBridge queries the proxy, not Salesforce directly | AutoBridge calling Salesforce directly exposes the rate limit and API key to a third party. The ACL/proxy absorbs rate-limit management and insulates AutoBridge from Salesforce API changes. |

**Q4.2 — Risks of direct Salesforce access from AutoBridge**

Risk 1: **Rate limit exhaustion.** AutoBridge has 4,200 dealers using the DMS. If even a small percentage of dealer sessions trigger Salesforce lookups simultaneously, 600 req/min is exhausted in seconds. VW India's own Salesforce integrations (internal reporting, ops) get starved. There's no way to prioritise or throttle once AutoBridge has the API key.

Risk 2: **Security and credential leakage.** The VW India Salesforce API key is now embedded in AutoBridge's Rails codebase and configuration — a third-party system VW Group India does not fully control, whose security posture and deployment practices are inherited from a startup. A breach of AutoBridge's config leaks VW India's Salesforce credentials, granting an attacker access to CRM data for the entire VW India operation.

**Q4.3 — Separate Ways / direct DB access**

**No.** You should not grant direct read access to AutoBridge MySQL.

The Financial Reporting team calling it "Separate Ways" is a misuse of the pattern. Separate Ways means each context solves a problem independently with *no integration* — it doesn't mean bypassing integration boundaries to read another system's DB directly. Direct MySQL access makes the Reporting team's ETL a conformist consumer of AutoBridge's internal schema, which is exactly what Q4.5 demonstrates breaks immediately. When AutoBridge restructures `service_orders`, Reporting breaks — and AutoBridge had no obligation to tell anyone, because the Reporting team was never a formal consumer.

**Q4.4 — Colleague's shortcut**

Your colleague's proposal results in **Conformist** — AutoBridge engineers adopt VW India's Warranty service SOAP model entirely, implementing SOAP client code in Rails.

The problem for VW Group India: AutoBridge now has a direct dependency on the Warranty service's WSDL. Any future changes to the Warranty service interface (even minor ones) require AutoBridge engineering effort — from a team you don't manage, with their own roadmap, who have every incentive to de-prioritise VW India's Warranty schema changes in favour of features their paying customers want. More critically, you've permanently exported knowledge of VW India's internal Warranty service model to an externally-acquired company's codebase. If the acquisition ever unwinds, or if AutoBridge loses engineers who understand that SOAP integration, VW India has no control over the dependency.

**Q4.5 — The ETL schema coupling failure**

**What went wrong:** The ETL job was implemented as a direct column-copy — `SELECT dealer_code FROM service_orders` — with no transformation layer between AutoBridge's internal schema and the Redshift reporting schema. This is exactly database integration: AutoBridge's internal column name became an implicit API contract that Redshift depended on. When AutoBridge refactored their internal schema (a legitimate internal change they had no obligation to announce), it silently broke a downstream system that had no formal integration boundary with them.

**What would have prevented it:** An Anti-Corruption Layer in the ETL — a transformation step that maps AutoBridge's raw columns to VW India's reporting domain model (`dealer_code → dealer_region_code + dealer_branch_code` mapping defined in the ETL, not assumed from source). When AutoBridge restructures their schema, the ETL's transformation layer breaks loudly and explicitly, in a place VW India controls and can fix without touching AutoBridge or Redshift. The rule is: any ETL that reads from a system you don't own must include a translation step. Direct column projection is Conformist by accident.

---

*End of Exercise*
