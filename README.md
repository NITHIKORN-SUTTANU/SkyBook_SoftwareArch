# SkyBook — Airline Reservation System
## Software Architecture Design Document

**Case Study 10 | Version 2.0 | April 2026**

---

## Table of Contents

1. [Background and Organisation Overview](#1-background-and-organisation-overview)
2. [Functional Requirements and Quality Attributes](#2-functional-requirements-and-quality-attributes)
3. [Architecture Model](#3-architecture-model)
4. [Architecture Tactics](#4-architecture-tactics)
5. [Technical Decisions](#5-technical-decisions)
6. [Risks and Trade-Offs](#6-risks-and-trade-offs)
7. [Summary](#7-summary)

---

## 1. Background and Organisation Overview

SkyBook is the digital reservation platform for a regional airline operating **40 routes across 15 cities**. The system supports the complete passenger journey — from flight search and booking through to online check-in and gate boarding — across the airline's website, mobile application, and third-party travel agency integrations.

### 1.1 Organisation Stakeholders

| Stakeholder | Role |
|---|---|
| **Passengers** | Search flights, book tickets, select seats, check in online, receive notifications |
| **Travel Agents** | Book tickets on behalf of customers via REST API integration |
| **Ground Staff** | Access passenger manifests, manage check-in desks, boarding gates, and baggage |
| **Flight Operations** | Monitor schedules, manage delays and cancellations, assign aircraft |
| **Revenue Management** | Configure and update dynamic pricing rules based on demand |

### 1.2 Operational Context

- Approximately **200 flights per day** carrying up to **25,000 passengers**
- Continuous multi-channel ticket sales via website, mobile application, and travel agency API
- Peak load periods occur during weekends, school holidays, and promotional fare launches
- The system must operate **24 hours a day, 7 days a week** — flights depart at all hours

### 1.3 Known Problems with the Legacy System

The current system was built in the 1990s and has not been significantly updated. The following defects drive the requirements for a new architecture.

| ID | Problem | Operational Consequence |
|---|---|---|
| P-01 | Seat inventory not synchronised across sales channels | Overbooking occurs — multiple passengers confirmed for the same seat |
| P-02 | No load management during the 24-hour check-in window | System performance degrades severely as passengers check in near departure |
| P-03 | No automatic passenger notification mechanism | Passengers are unaware of delays or cancellations until they arrive at the airport |
| P-04 | Seat selection rules inconsistently enforced across channels | Passengers on some channels bypass fare class restrictions |
| P-05 | Travel agency booking confirmations take up to 10 minutes | Agents cannot confirm bookings to customers in real time |

---

## 2. Functional Requirements and Quality Attributes

### 2.1 Functional Requirements

| ID | Requirement | Description |
|---|---|---|
| FR-01 | **Flight Search** | Search by route and date; display available seats, fare classes, and real-time prices |
| FR-02 | **Ticket Booking** | Select flight, choose seat, add baggage options, and complete payment |
| FR-03 | **Online Check-In** | Opens 24 hours before departure; generates and delivers boarding pass |
| FR-04 | **Ground Staff Access** | Provides live passenger manifests, check-in status, and boarding gate operations |
| FR-05 | **Flight Notifications** | Automatically notifies passengers of delays or cancellations via email, SMS, and push |
| FR-06 | **Pricing Rule Updates** | Allows revenue team to update pricing rules without system downtime or redeployment |
| FR-07 | **Travel Agent API** | Supports programmatic ticket booking with confirmation returned within seconds |
| FR-08 | **Cancellation and Rebooking** | Automatically notifies and presents rebooking options to all affected passengers |

---

### 2.2 Quality Attributes

Quality attributes are classified into three categories: **System Quality Attributes**, **Business Quality Attributes**, and **Architecture Quality Attributes**.

#### 2.2.1 System Quality Attributes

System quality attributes describe the runtime behaviour of the system that stakeholders can directly observe or measure.

---

**QA-01: Availability**

Availability is the degree to which the system is operational and accessible when required. It is defined by the probability that the system will be functional at any given time, and is typically expressed as a percentage of uptime.

SkyBook must operate continuously. Ground staff require access to passenger manifests at all hours; passengers book tickets and check in around the clock; and flight disruptions can occur at any time. The 24-hour check-in window and the peak boarding period represent the highest-risk intervals for service failure.

| Element | Detail |
|---|---|
| **Stimulus** | All 200 daily departures concentrate check-in and boarding activity within a 6-hour window |
| **Response** | The Ground Staff Portal and Check-In Service remain fully operational throughout |
| **Measure** | ≥ 99.9% uptime; ≤ 0.1% probability the system is unavailable when needed |

---

**QA-02: Modifiability**

Modifiability is the cost of making a change to the system, encompassing what artefacts can be changed, when those changes can be made, and who is authorised to make them.

The most frequently required change in SkyBook is the adjustment of pricing rules by the Revenue Management team. These changes must be possible without engineering involvement, without redeployment, and without any interruption to booking activity.

| Element | Detail |
|---|---|
| **Artefact subject to change** | Dynamic pricing rules — fare tiers, surcharges, and promotional rates |
| **Agent of change** | Revenue Management team (non-technical business users) |
| **Timing of change** | During live sales periods, including peak booking windows |
| **Stimulus** | Revenue manager modifies the peak-hour surcharge rule during active sales |
| **Response** | New rule is applied across all channels within 30 seconds; no redeployment is required; no booking activity is interrupted |
| **Measure** | Zero system downtime; rule propagation latency < 30 seconds |

---

**QA-03: Performance**

Performance concerns how quickly the system generates a response following the arrival of an event. Latency — the time elapsed between event arrival and response generation — is the primary measure. Lower latency produces a better user experience and higher throughput.

Flight search is the highest-frequency operation in the system. Passengers and travel agents expect near-immediate results. Confirmation of a booking must also be returned promptly, particularly for travel agents serving customers in real time.

| Element | Detail |
|---|---|
| **Stimulus** | 10× normal load arrives at the Flight Search Service during a school holiday sale launch |
| **Response** | Search results are returned within the acceptable time window; no visible degradation |
| **Measure** | < 500ms at p95 latency for flight search; booking confirmation < 3 seconds across all channels |

---

**QA-04: Security**

Security is the system's ability to resist unauthorised access and usage while continuing to serve legitimate users. It encompasses confidentiality, integrity, availability to legitimate users, non-repudiation, and auditability.

Travel agents access the booking API from external networks. Passenger payment and personal data must be protected in transit and at rest. Ground staff access to full passenger manifests must be restricted to authenticated devices and personnel.

| Security Property | SkyBook Requirement |
|---|---|
| **Confidentiality** | Payment card data and passenger personal data protected from unauthorised access |
| **Integrity** | Booking records and seat inventory cannot be modified outside authorised service channels |
| **Availability** | The system must remain accessible to legitimate users even under attack conditions |
| **Auditing** | All booking, modification, and cancellation actions are logged with full attribution and timestamp |

| Element | Detail |
|---|---|
| **Stimulus** | An unauthenticated external party attempts to call the booking API |
| **Response** | The request is rejected at the API Gateway; no booking data is exposed |
| **Measure** | 100% of API calls require a valid authentication token; all access attempts are logged |

---

**QA-05: Testability**

Testability is the degree to which software components can be made to demonstrate their faults through testing. A testable system allows the internal state and inputs of each component to be controlled independently, and the resulting outputs to be observed.

Each microservice in SkyBook must be independently testable. The seat locking mechanism — the most critical correctness concern in the entire system — must be exercisable under simulated concurrent load. The notification pipeline must be verifiable without dispatching real communications to passengers.

| Element | Detail |
|---|---|
| **Stimulus** | A developer runs integration tests for the Booking Service in isolation |
| **Response** | A test harness injects simulated seat requests and observes booking outcomes without requiring live payment or notification services to be active |
| **Measure** | Each service exposes a dedicated test interface; ≥ 80% code coverage is achievable per service |

---

**QA-06: Usability**

Usability is the ease with which users can accomplish intended tasks within the system. It encompasses learning system features, operating the system efficiently, minimising the impact of errors, and increasing user confidence and satisfaction.

Ground staff must access passenger manifests and process boarding without re-learning procedures at each shift. Passengers must be able to complete a booking end-to-end without external assistance. Delay and cancellation notifications must be clear and actionable.

| Element | Detail |
|---|---|
| **Stimulus** | A ground staff member must retrieve the boarding manifest for an unexpected gate change with 10 minutes to departure |
| **Response** | The manifest is accessible within two navigation steps on the Progressive Web Application; an offline-cached copy is available if network connectivity is lost |
| **Measure** | Task completion rate ≥ 95% without instruction for trained staff; all critical operations reachable within two navigation steps |

---

#### 2.2.2 Business Quality Attributes

Business quality attributes describe requirements that originate from the organisation's operational and commercial context rather than from the system's runtime behaviour.

| Attribute | Relevance to SkyBook |
|---|---|
| **Time to Market** | The legacy system is actively causing overbooking incidents and real-time revenue loss. A phased rollout — booking core and inventory first, then check-in and notifications, then pricing engine and reporting — allows early value delivery while reducing deployment risk |
| **Cost and Benefit** | A microservices architecture increases initial build and operational cost relative to a monolith, but reduces long-term maintenance cost and allows services to scale independently. Each overbooking incident carries passenger compensation liability; the investment in a correct inventory system is justified |
| **Projected Lifetime** | The system is intended for long-term operation of ten or more years. This makes modifiability, scalability, and portability important design priorities from the outset |
| **Target Market** | The airline serves both direct consumers and travel agencies. Consistent multi-channel API behaviour and broad functionality are essential to maintaining both market segments |
| **Roll-out Schedule** | The phased delivery model means the architecture must support incremental deployment of capabilities without requiring re-architecture between phases |

#### 2.2.3 Architecture Quality Attributes

Architecture quality attributes describe properties of the architecture itself, independent of runtime behaviour.

| Attribute | Relevance to SkyBook |
|---|---|
| **Conceptual Integrity** | All services follow uniform patterns: REST interfaces with OpenAPI specifications, Kafka domain events with a shared schema registry, PostgreSQL data stores, and structured JSON payloads. Similar responsibilities are handled in the same way across all services |
| **Buildability** | Services are decomposed by domain — Booking, Inventory, Check-In, Notifications, Pricing, and Flight Operations — and assigned to separate development teams. This maximises parallel development: the Notification Service team has no dependency on the Booking Service team, and vice versa |

---

## 3. Architecture Model

### 3.1 Architectural Style

SkyBook adopts a **microservices architecture**. Each domain capability is encapsulated in an independently deployable service with its own data store. Services communicate in two modes:

- **Synchronous REST over HTTPS** — for user-facing request-response interactions (search, booking, check-in)
- **Asynchronous messaging via Apache Kafka** — for domain events (booking confirmed, flight cancelled, delay triggered, seat released)

This style was selected because the system's services have fundamentally different scaling profiles, different fault tolerance requirements, and different change frequencies. A monolithic architecture cannot satisfy these constraints simultaneously.

---

### 3.2 Use Case Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         SkyBook System                              │
│                                                                     │
│  Passenger ──────── Search Flights                                  │
│  Passenger ──────── Book Ticket and Select Seat                     │
│  Passenger ──────── Online Check-In                                 │
│  Passenger ──────── Receive Delay / Cancellation Notification       │
│                                                                     │
│  Travel Agent ───── Programmatic API Booking                        │
│                                                                     │
│  Ground Staff ───── View Passenger Manifest                         │
│  Ground Staff ───── Manage Boarding (scan boarding passes)          │
│                                                                     │
│  Flight Operations ─ Trigger Delay / Cancellation                   │
│  Flight Operations ─ Monitor Schedules and Aircraft Assignment      │
│                                                                     │
│  Revenue Management  Update Pricing Rules (zero downtime)           │
└─────────────────────────────────────────────────────────────────────┘
```

---

### 3.3 Component Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│                              CLIENTS                                 │
│   [Passenger Web / Mobile]  [Travel Agent API]  [Ground Staff PWA]   │
└──────────────────────────────┬───────────────────────────────────────┘
                               │ HTTPS / REST
                      ┌────────▼─────────┐
                      │   API Gateway    │
                      │  Authentication  │
                      │  Rate Limiting   │
                      │  Request Routing │
                      └────────┬─────────┘
            ┌──────────────────┼────────────────────────┐
            │                  │                        │
   ┌────────▼────────┐  ┌──────▼────────┐  ┌───────────▼───────────┐
   │  Flight Search  │  │    Booking    │  │   Check-In Service    │
   │    Service      │  │    Service    │  │   (SQS buffered)      │
   │ [Elasticsearch] │  │ [Redis Lock]  │  │                       │
   └────────┬────────┘  └──────┬────────┘  └───────────┬───────────┘
            │                  │                        │
            │         ┌────────▼────────────────────────▼──────────┐
            │         │              Inventory Service             │
            │         │   Single source of truth for seat counts   │
            │         │   Overbooking guard · Redis Pub/Sub         │
            │         └──────────────────┬─────────────────────────┘
            │                            │
   ┌────────▼────────────────────────────▼───────────────────────┐
   │                   Event Bus  (Apache Kafka)                 │
   │  Topics: flight.delayed  booking.confirmed  flight.cancelled │
   │          checkin.completed  seat.released  price.updated     │
   └────────┬───────────────────────┬─────────────────┬──────────┘
            │                       │                 │
   ┌────────▼──────┐   ┌────────────▼──────┐  ┌──────▼──────────┐
   │ Notification  │   │  Flight Ops       │  │ Pricing Engine  │
   │   Service     │   │    Service        │  │  (Hot Reload)   │
   │ email/SMS/    │   │                   │  │  [Config DB]    │
   │ push          │   │                   │  │                 │
   └───────────────┘   └───────────────────┘  └─────────────────┘

   ──────────────────────── DATA STORES ──────────────────────────────
   [PostgreSQL: primary + read replicas]    [Redis: cache and locks]
   [Apache Kafka: event log]                [Elasticsearch: search index]
```

---

### 3.4 Key Architectural Flows

#### Flow A — Concurrent Seat Booking (resolves P-01)

```
Client ──► API Gateway (authenticate and route) ──► Booking Service
                                                          │
                                       ┌──────────────────▼────────────────┐
                                       │  Acquire Redis Redlock            │
                                       │  Key: lock:flight:{id}:seat:{id}  │
                                       │  TTL: 10 seconds                  │
                                       └──────────┬────────────────────────┘
                                                  │
                                     ┌────────────┴──────────────┐
                                    YES                          NO
                                     │                            │
                         ┌───────────▼───────────┐   Return "seat unavailable"
                         │  Inventory: validate  │   to client immediately
                         │  Payment: charge card │
                         │  Confirm booking      │
                         │  Publish Kafka event  │
                         │  Release lock         │
                         └───────────────────────┘
```

#### Flow B — Flight Cancellation (resolves P-03)

```
Flight Operations ──► Flight Ops Service ──► Publish flight.cancelled to Kafka
                                                           │
                         ┌─────────────────────────────────┼─────────────────────┐
                         │                                 │                     │
               Inventory Service                  Notification Service     Rebooking Service
               Mark all seats unavailable     Query all affected passengers  Present alternatives
                                              Dispatch email + SMS + push    Process refunds
                                              Retry failed deliveries via DLQ
```

#### Flow C — Peak Check-In Load Management (resolves P-02)

```
Passenger ──► API Gateway ──► SQS Queue (buffer all check-in requests)
                                    │
                       Check-In workers drain queue at a controlled rate
                                    │
                       Generate and deliver boarding pass
                                    │
                       Inventory Service updates manifest via Kafka event
                                    │
                       Ground Staff PWA receives live manifest update via WebSocket
```

---

## 4. Architecture Tactics

A tactic is a design decision that directly influences the system's response to a quality attribute stimulus. A collection of related tactics forms an architectural strategy for a given quality attribute.

---

### 4.1 Availability Tactics

The goal of availability tactics is to achieve high system availability by managing faults before, during, and after they occur. Tactics are grouped into three categories: fault detection, fault recovery, and fault prevention.

#### Fault Detection

| Tactic | Application in SkyBook |
|---|---|
| **Ping / Echo** | Kubernetes liveness probes issue periodic pings to each service container. The monitoring component sends the ping; each service must echo a response within a defined timeout. Absence of a response triggers an automatic pod restart |
| **Heartbeat** | Each service emits a periodic heartbeat metric to Prometheus. If a service stops emitting its heartbeat, an alert is raised to the operations team. Unlike ping/echo, the monitored component initiates the signal |
| **Exception** | Services detect internal faults — such as seat lock timeouts or payment gateway failures — by catching exceptions within the same process. These are converted to structured error responses and propagated appropriately without crashing the service |

#### Fault Recovery

| Tactic | Application in SkyBook |
|---|---|
| **Active Redundancy** | The Booking Service and Check-In Service run with a minimum of two replicas at all times. All replicas receive and process requests in parallel. If one replica fails, the remaining replicas absorb the load immediately with no measurable downtime |
| **Passive Redundancy** | PostgreSQL is deployed as a primary instance with two read replicas kept in continuous synchronisation via streaming replication. On primary failure, Patroni promotes a secondary after verifying state consistency. Downtime is bounded by the state assurance and switching time |
| **Checkpoint and Rollback** | Apache Kafka topics serve as a durable, replayable checkpoint of all domain events. If the Notification Service crashes during message processing, it rolls back its consumer offset to the last committed position and re-processes events from that point, ensuring no notifications are permanently lost |

#### Fault Prevention

| Tactic | Application in SkyBook |
|---|---|
| **Process Monitor** | Kubernetes continuously monitors the health of all service pods. When a crashed or unresponsive pod is detected, Kubernetes deletes the faulty process and immediately creates a replacement instance, preventing prolonged service unavailability |
| **Transaction** | The booking workflow applies the Saga pattern, treating the sequence of steps — seat reservation, payment processing, booking confirmation — as a logical transaction. If any step fails, choreography-based compensating transactions automatically reverse all completed steps, preventing partial or inconsistent state from being persisted |
| **Removal from Service** | During rolling deployments, pods are gracefully removed from the load balancer before being shut down. In-flight requests are allowed to complete before the instance is terminated. This prevents service disruption during routine maintenance and deployment activity |

---

### 4.2 Modifiability Tactics

The goal of modifiability tactics is to reduce the cost of making changes by minimising the number of components affected by any given modification. Tactics are grouped into two categories: localising modification and preventing ripple effects.

#### Localising Modification

| Tactic | Application in SkyBook |
|---|---|
| **Abstract Common Services** | The Notification Service provides email, SMS, and push delivery as a shared service consumed by any component that must communicate with passengers — including Flight Operations, the Rebooking Service, and the Check-In Service. Changes to notification delivery logic are made once in a single location rather than across each consuming module |
| **Generalise the Module** | The Pricing Engine accepts a rule template and input data and produces a computed fare. Both standard fare calculation and promotional pricing use the same generalised engine; only the rule input differs. A new fare type requires a new rule, not a new module |
| **Limit Possible Options** | Pricing rules are expressed in a constrained domain-specific language with a defined and validated schema. Revenue Management can only modify values within the permitted schema — they cannot introduce arbitrary logic — which limits the scope of any rule change and reduces the risk of unintended side effects |

#### Preventing Ripple Effects

| Tactic | Application in SkyBook |
|---|---|
| **Hide Information** | Each service exposes only a well-defined REST interface as its public contract. Internal implementation details — database schema, caching strategy, locking mechanism — are entirely private. A change to how the Inventory Service stores seat data has no effect on the Booking Service, which interacts only with the public interface |
| **Maintain Existing Interface and Add Adaptor** | The Travel Agent API maintains a stable versioned interface (v1). When internal booking logic changes, an adaptor layer translates between the external contract and the updated internal implementation. External agents are not affected by internal changes |
| **Use Intermediary** | Apache Kafka acts as a messaging intermediary between event producers such as the Booking Service and Flight Operations Service, and event consumers such as the Notification Service and Reporting Service. Producers publish events without knowledge of which consumers exist. Adding a new consumer requires no modification to any producer, eliminating dependency-driven ripple effects |
| **Restrict Communication Paths** | Services are prohibited from reading another service's database directly. All inter-service communication passes through defined REST interfaces or Kafka topics. This limits the set of modules that must change when any single service's internal data model is modified |

---

### 4.3 Performance Tactics

The goal of performance tactics is to generate responses to events within acceptable time constraints. Latency is the time elapsed between the arrival of an event and the generation of a corresponding response. Tactics are grouped into three categories: resource demand, resource management, and resource arbitration.

#### Resource Demand

| Tactic | Application in SkyBook |
|---|---|
| **Increase Computational Efficiency** | Flight search results are pre-indexed in Elasticsearch using optimised inverted indexes. Seat availability counts are pre-computed and stored in Redis cache rather than calculated on each request, reducing the computational work required per search event |
| **Control Frequency of Events** | During the 24-hour check-in window, all check-in requests are enqueued in Amazon SQS rather than directed immediately to the database. This controls the rate of events reaching the persistence layer, preventing resource saturation during periods of concentrated demand |

#### Resource Management

| Tactic | Application in SkyBook |
|---|---|
| **Introduce Concurrency** | The Booking Service and Check-In Service are designed as stateless components and deployed across multiple replicas. Kubernetes Horizontal Pod Autoscaler provisions additional replicas as request load increases, enabling parallel processing of concurrent requests across available resources |
| **Maintain Multiple Copies of Data** | Redis caches flight search availability snapshots with a 30-second time-to-live, serving the majority of search requests from memory without reaching the database. Separate PostgreSQL read replicas handle search and reporting queries, preventing read contention on the primary write instance |
| **Increase Available Resources** | Kubernetes Horizontal Pod Autoscaler automatically provisions additional compute resources when CPU utilisation exceeds 70% or requests per second exceed a defined threshold. This increase in available resources occurs without manual intervention during peak periods |

#### Resource Arbitration

| Tactic | Application in SkyBook |
|---|---|
| **Fixed-Priority Scheduling** | The API Gateway applies differentiated rate limits by client type. Ground staff and Flight Operations requests are assigned higher processing priority than general passenger search requests during peak boarding windows, ensuring that operationally critical functions are not starved by high consumer search volume |
| **First-In First-Out Scheduling** | The SQS check-in queue processes requests in strict arrival order, treating all check-in submissions equally and providing predictable, fair throughput during concentrated demand periods |

---

### 4.4 Security Tactics

The goal of security tactics is to protect the system against unauthorised access while maintaining full availability to legitimate users. Tactics are grouped into three categories: resisting attacks, detecting attacks, and recovering from attacks.

#### Resisting Attacks

| Tactic | Application in SkyBook |
|---|---|
| **Authenticate Users** | All requests to the system are validated at the API Gateway. Travel agents authenticate using OAuth2 client credentials. Passengers authenticate using session tokens. Ground staff authenticate using device-bound certificates. No unauthenticated request proceeds beyond the gateway |
| **Authorise Users** | Access Control Lists are enforced per role. Passengers may access only their own booking records. Travel agents are scoped to booking creation and retrieval operations. Ground staff have read-only access to flight manifests. Flight Operations has write access to flight status only. Revenue Management has write access to pricing rules only |
| **Maintain Data Confidentiality** | All data in transit is encrypted using TLS 1.3. Payment card data is processed exclusively by a PCI-DSS compliant payment processor. SkyBook does not store raw card data at any point |
| **Maintain Integrity** | All Kafka messages include a checksum that is validated by consumers prior to processing. Booking records include a version hash to detect any unauthorised modification to persisted data |
| **Limit Exposure** | Services are deployed in isolated network zones. Only the API Gateway is accessible from the public internet. All internal services communicate exclusively over a private network. Database instances, Kafka brokers, and Redis clusters are not reachable from outside the service network |
| **Limit Access** | Firewall rules enforce strict traffic boundaries: external traffic may reach only the API Gateway; the API Gateway may reach only the service layer; the service layer may reach only the data layer. No direct external access to internal infrastructure is permitted |

#### Detecting Attacks

| Tactic | Application in SkyBook |
|---|---|
| **Intrusion Detection** | The API Gateway monitors all inbound traffic against known attack pattern signatures, including credential stuffing, reservation manipulation bots, and volumetric denial-of-service patterns. Detected anomalies are forwarded to a Security Information and Event Management system for analysis and alerting |

#### Recovering from Attacks

| Tactic | Application in SkyBook |
|---|---|
| **Restore State** | PostgreSQL supports point-in-time recovery, enabling the restoration of booking and inventory data to a confirmed correct state prior to a successful attack. Administrative data — user credentials, access control lists, and travel agent API keys — is treated as the highest-priority restoration target |
| **Maintain Audit Trail** | All API calls are logged with timestamp, source IP address, authenticated identity, and the action performed. Audit logs are written to an isolated, tamper-evident log store that accepts only append-write access from system services. The audit trail enables attribution of all actions to an authenticated identity and supports post-incident forensic analysis |

---

### 4.5 Testability Tactics

The goal of testability tactics is to enable efficient and thorough fault detection by making each component's internal state controllable and its outputs observable.

#### Input and Output Tactics

| Tactic | Application in SkyBook |
|---|---|
| **Record and Playback** | The Booking Service records all request and response pairs at its interface boundary during staging operation. These recordings are replayed by automated test harnesses to reproduce exact booking scenarios, including concurrent race conditions that previously caused overbooking in the legacy system |
| **Separate Interface from Implementation** | Each service is defined by a formal OpenAPI specification that constitutes its public interface contract. Test harnesses interact exclusively with this interface, injecting controlled inputs and observing outputs, without any dependency on or knowledge of internal implementation details |
| **Specialised Test Interfaces** | Each service exposes a dedicated administrative test endpoint that allows test harnesses to inject specific preconditions at any architectural level — for example, forcing seat count to one to test overbooking prevention, simulating payment gateway timeout, or triggering lock expiry — without requiring full end-to-end infrastructure |

#### Internal Monitoring

| Tactic | Application in SkyBook |
|---|---|
| **Built-in Monitor** | Each service exposes a Prometheus-compatible metrics endpoint that publishes operational state (healthy or degraded), performance measurements (request rate, p95 and p99 latency), resource capacity (active connections, queue depth), and security events (authentication failure rate). This monitoring interface is a first-class component of each service, not an optional addition |

---

### 4.6 Usability Tactics

The goal of usability tactics is to help users accomplish intended tasks efficiently and with minimal opportunity for error. Tactics are grouped into runtime tactics, which operate during system execution, and design-time tactics, which shape the system's structure to support future usability changes.

#### Runtime Tactics

| Category | Tactic | Application in SkyBook |
|---|---|---|
| **System Initiative** | Inform the user of system activity and expected duration | The booking payment flow displays labelled progress steps — "Reserving seat", "Processing payment", "Confirming booking" — so that passengers understand the system's current state and do not submit duplicate requests. The check-in queue displays estimated processing time |
| **System Initiative** | Communicate outcomes clearly | Booking completion presents a full confirmation summary including booking reference, seat assignment, and baggage details. Cancellation notifications state the reason, the new status, and include a direct link to the rebooking workflow |
| **User Initiative** | Support error correction | Passengers may cancel seat selection and return to the previous step without losing flight or passenger details already entered. Ground staff may reverse a boarding scan that was applied in error |

#### Design-Time Tactics

| Tactic | Application in SkyBook |
|---|---|
| **Separate the User Interface from Application Logic** | The Passenger Web Application, the Ground Staff Progressive Web Application, and the Revenue Management Portal are independent frontend applications that communicate with backend services exclusively through versioned REST APIs. User interface changes — visual redesign, language localisation, new device form factors — require no modification to any backend service. This separation is implemented using the Model-View-Controller pattern throughout all frontend applications |

---

## 5. Technical Decisions

The following table documents significant architectural decisions, the alternatives evaluated, and the rationale for each choice.

| ID | Decision | Choice | Alternatives Considered | Rationale |
|---|---|---|---|---|
| TD-01 | **Architecture Style** | Microservices | Monolith, Service-Oriented Architecture | Services have fundamentally different scaling profiles and change frequencies; fault isolation is required between booking and notification; supports parallel team development |
| TD-02 | **Seat Locking Mechanism** | Redis Redlock (distributed lock) | Database-level row locking, optimistic concurrency control | Provides sub-millisecond cross-service seat serialisation across all channels simultaneously; time-to-live ensures automatic release of stale locks if a service instance crashes; directly eliminates P-01 |
| TD-03 | **Asynchronous Messaging** | Apache Kafka | RabbitMQ, Amazon SQS | Provides a durable, replayable event log supporting checkpoint and rollback; enables native fan-out to multiple consumers without coupling producers to consumers; sustains high throughput for 200-flight-per-day event volume |
| TD-04 | **Persistence** | PostgreSQL with Redis cache | Shared database, MySQL, MongoDB | ACID guarantees for booking and inventory transactions; read replicas support query offloading; Redis provides sub-millisecond seat availability reads for the high-frequency search path |
| TD-05 | **Notification Delivery** | Event-driven via Kafka | Synchronous REST calls, database polling | Decouples Flight Operations from notification logic, preventing ripple effects; supports at-least-once delivery with retry; scales the Notification Service independently of event producers |
| TD-06 | **Pricing Rule Updates** | Hot-reload via configuration database and change event | Application redeployment on rule change, runtime feature flags | Satisfies the Modifiability quality attribute — Revenue Management makes rule changes without engineer involvement, redeployment, or service interruption |
| TD-07 | **Check-In Load Management** | Amazon SQS queue buffer with worker pool | Vertical scaling, synchronous database writes | Controls the frequency of events reaching the database during the concentrated 24-hour check-in window; resolves P-02 without requiring proportionally scaled infrastructure |
| TD-08 | **Ground Staff Interface** | React Progressive Web Application | Native mobile application, desktop thick client | Supports all device form factors from a single codebase; offline capability via service worker cache addresses the gate-area network reliability concern; eliminates the operational overhead of managing app store deployments |
| TD-09 | **Container Orchestration** | Kubernetes with Horizontal Pod Autoscaler | Docker Compose, bare virtual machines | Automates fault recovery via process monitoring and pod replacement; automates concurrency scaling via HPA; provides declarative infrastructure management with rollback capability |

### 5.1 Microservices Architecture — Detailed Rationale

A monolithic rebuild was evaluated as the lower-complexity alternative. The microservices architecture was ultimately selected on the following grounds:

- **Differential Scaling:** The Check-In Service experiences sharp load spikes during departure windows while the Pricing Engine operates at consistently low load. A monolithic deployment cannot allocate resources per-capability and would require over-provisioning the entire system to satisfy peak check-in demand
- **Fault Isolation:** A defect or crash in the Notification Service must not interrupt the Booking Service. In a monolith, a failure in any module can propagate and destabilise the entire application
- **Independent Deployability:** The Revenue Management team must be able to update pricing rules without coordinating with the Booking or Check-In Service teams. Microservices enable each domain to be owned, tested, and deployed independently
- **Accepted Trade-off:** Microservices introduce distributed systems complexity — network latency between services, distributed tracing requirements, and more complex deployment infrastructure. This complexity is managed through Kubernetes orchestration, Istio service mesh, centralised ELK-stack logging, and Jaeger distributed tracing

### 5.2 Apache Kafka as Messaging Intermediary — Detailed Rationale

The selection of Kafka over alternatives was driven by three requirements that simpler message brokers do not fully satisfy:

- **Durability and Replayability:** Kafka retains all messages for a configurable retention period. If the Notification Service is offline during a flight cancellation event, it can replay all missed events upon recovery. RabbitMQ messages are consumed and discarded; recovery from service downtime would result in permanent message loss
- **Fan-Out Without Coupling:** A single `flight.cancelled` event must be consumed by the Notification Service, the Rebooking Service, the Inventory Service, and the Reporting Service. Kafka's consumer group model allows any number of independent consumers to receive the same event without the producer knowing of their existence. This is the architectural embodiment of the Use Intermediary modifiability tactic
- **Throughput and Ordering:** Kafka partitions provide ordered, high-throughput delivery within a topic partition. Seat reservation events for a given flight are partitioned by flight identifier, ensuring that booking events for the same flight are processed in order

---

## 6. Risks and Trade-Offs

Quality attributes are not independent. Decisions made to strengthen one attribute frequently introduce tension with another. The following table identifies the principal risks and trade-offs in the SkyBook architecture.

| Risk | Attributes in Tension | Potential Impact | Mitigation Strategy |
|---|---|---|---|
| **Microservices operational complexity** | Modifiability vs. Availability | Distributed failure modes are harder to diagnose than monolithic failures; inter-service latency creates additional failure points | Kubernetes with Istio service mesh; centralised ELK-stack logging; Jaeger distributed tracing across all service calls |
| **Authentication and encryption overhead** | Security vs. Performance | JWT validation and TLS 1.3 encryption add measurable latency to every request | JWT token caching at the API Gateway (one validation per session); hardware TLS offloading at the network load balancer |
| **Redis lock TTL expiry under sustained load** | Availability vs. Consistency | A lock may expire before a payment transaction completes under extreme concurrent load, resulting in a potential duplicate booking | Extended TTL of 30 seconds for the payment step; Grafana alerting on lock contention rate; circuit breaker on the Booking-to-Inventory call path |
| **Read replica replication lag** | Performance vs. Consistency | Stale seat availability counts may be displayed to passengers searching during a period of high write volume | Eventual consistency is accepted for the search display path; strong consistency is enforced only at the point of Redlock acquisition during booking |
| **Kafka cluster unavailability** | Availability vs. Reliability | Kafka cluster failure would halt event delivery, causing notifications to be lost and inventory updates to be delayed | Three-broker Kafka cluster with replication factor three; dead-letter queue for failed event deliveries; off-site Kafka snapshot backups at 15-minute intervals |
| **Pricing rule misconfiguration** | Modifiability vs. Security / Integrity | A misconfigured pricing rule submitted by a non-technical user could result in incorrect fares being published to all channels | Schema and range validation enforced on rule save; staging environment preview before live promotion; immutable audit log of all rule changes with rollback capability |
| **PWA offline data staleness** | Usability vs. Consistency | The Ground Staff PWA may serve a stale offline-cached manifest if network connectivity is lost after a late check-in | Service worker displays a prominent "Offline Mode" banner when serving cached data; cache is refreshed on every successful network connection; manifest version timestamp is displayed to staff |

---

## 7. Summary

The SkyBook architecture is designed to directly resolve each identified legacy system defect through a combination of targeted quality attribute requirements and specific architectural tactics.

| Legacy Defect | Quality Attribute | Tactic Applied | Architectural Solution |
|---|---|---|---|
| P-01: Overbooking | Availability — Fault Prevention | Transaction; Distributed Lock | Redis Redlock serialises seat reservation across all channels simultaneously; Saga pattern rolls back on payment failure |
| P-02: Check-in slowdown | Performance — Resource Demand and Management | Control Frequency of Events; Introduce Concurrency | SQS queue buffers check-in requests at controlled rate; Kubernetes HPA auto-scales Check-In Service replicas |
| P-03: No passenger notifications | Availability — Fault Recovery; Modifiability | Checkpoint and Rollback; Use Intermediary | Kafka event-driven notification pipeline with at-least-once delivery guarantee and dead-letter queue for retry |
| P-04: Inconsistent seat rules | Modifiability — Localise Modification | Abstract Common Services; Hide Information | All seat selection rule enforcement is centralised exclusively within the Booking Service; no per-channel variation is architecturally possible |
| P-05: Slow agency confirmations | Performance — Resource Management | Maintain Multiple Copies of Data; Increase Available Resources | Stateless Booking Service with read-replica query offloading returns synchronous REST confirmation within three seconds |

The architecture achieves its quality attribute targets through the following strategies:

**Availability** is maintained through layered fault handling: Ping/Echo and Heartbeat for detection; Active and Passive Redundancy plus Checkpoint and Rollback for recovery; Process Monitor, Transaction (Saga), and Removal from Service for prevention.

**Modifiability** is achieved by localising the pricing rule artefact within the Pricing Engine and preventing ripple effects through information hiding behind service interfaces, the Use Intermediary pattern via Kafka, and the Adaptor pattern on the Travel Agent API boundary.

**Performance** is governed by controlling the frequency of check-in events via SQS, maintaining multiple copies of availability data via Redis and read replicas, and introducing concurrency through stateless service replicas managed by Kubernetes HPA.

**Security** is enforced through authentication at the API Gateway, role-based access control lists, data confidentiality via TLS and PCI-DSS compliance, integrity checksums on all Kafka messages, and a tamper-evident audit trail of all actions.

**Testability** is built into every service through formal OpenAPI interface contracts, record and playback test harnesses, specialised test injection endpoints, and first-class Prometheus monitoring interfaces.

**Usability** is supported at runtime through system-initiative progress feedback and outcome notification, user-initiative error correction flows, and at design time through complete separation of the user interface layer from application logic via the Model-View-Controller pattern.

---

*End of Document*
