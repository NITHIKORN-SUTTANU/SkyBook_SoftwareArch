# SkyBook — Airline Reservation System
## Software Architecture Design Document

**Case Study 10 | April 2026**

---

## 1. Background and Organisation

SkyBook is the reservation system for a regional airline. It covers **40 routes across 15 cities**, selling tickets through its own website, mobile app, and third-party travel agencies. The system handles search, booking, seat selection, check-in, and boarding — all digitally.

**Scale:** ~200 flights per day, up to 25,000 passengers per day.

### Stakeholders

| Who | What they do in the system |
|---|---|
| Passenger | Search flights, book tickets, select seats, check in online |
| Travel Agent | Book on behalf of customers via API |
| Ground Staff | Access passenger manifests, manage boarding gates |
| Flight Operations | Monitor schedules, handle delays and cancellations |
| Revenue Management | Update ticket pricing rules |

### Why we need a new system

The current system was built in the 1990s. These are the five specific problems it has:

| ID | Problem | What actually goes wrong |
|---|---|---|
| P-01 | Seat inventory not synced across channels | Two passengers can book the same seat at the same time |
| P-02 | Check-in load not managed | System becomes slow when all passengers check in near departure |
| P-03 | No automatic notifications | Passengers only find out about delays when they arrive at the airport |
| P-04 | Seat selection rules inconsistently enforced | Some channels let passengers bypass fare restrictions |
| P-05 | Travel agency confirmations take up to 10 minutes | Agents cannot confirm a booking in real time |

---

## 2. Functional Requirements

These are the six things the new system must be able to do, directly from the case:

| ID | Requirement | Description |
|---|---|---|
| FR-01 | Flight Search | Passengers search by route and date, see available seats and fares |
| FR-02 | Ticket Booking | Book tickets, select seats, add baggage options |
| FR-03 | Online Check-In | Opens 24 hours before departure, generates boarding pass |
| FR-04 | Ground Staff Access | View live passenger manifests and manage boarding |
| FR-05 | Automatic Notifications | System notifies all affected passengers when a flight is delayed or cancelled |
| FR-06 | Pricing Rule Updates | Revenue management updates pricing rules without shutting the system down |

---

## 3. Quality Attributes

Quality attributes describe *how well* the system must perform — not just what it does.

We focus on **five quality attributes** that are most critical for SkyBook's problems. Each one maps directly to a real problem in the case.

---

### QA-01: Availability
> *The system must be operational when it is needed.*

SkyBook runs 24/7. The highest-risk window is the 6 hours before departure when check-in peaks and ground staff need live manifests at the gate.

- **Source of concern:** P-02 (system slows down under load)
- **Stimulus:** All 200 flights concentrate check-in activity into one busy morning window
- **Response:** The system stays fully operational — no slowdown, no crash
- **Measure:** 99.9% uptime (the system can be down for at most ~9 hours per year)

---

### QA-02: Performance
> *How fast the system responds when an event happens. Lower latency = better.*

Passengers expect search results quickly. Travel agents need booking confirmation in seconds, not minutes.

- **Source of concern:** P-05 (10-minute confirmation delay)
- **Stimulus:** 10× normal traffic hits the system during a school holiday booking rush
- **Response:** Flight search returns results at normal speed
- **Measure:** Flight search responds in under 500ms at p95. Booking confirmation in under 3 seconds.

---

### QA-03: Modifiability
> *The cost of making a change — what can change, when, and who changes it.*

Revenue Management needs to change pricing rules frequently (peak hours, promotions, holidays) without the engineering team having to redeploy the whole system.

- **Source of concern:** P-04 (rules inconsistently enforced), plus FR-06
- **What can change:** Pricing rules — fare tiers, seat selection restrictions, surcharges
- **Who changes it:** Revenue Management team (non-technical users), without engineer help
- **When:** During live sales, with no downtime
- **Measure:** Rule change takes effect across all channels within 30 seconds, zero downtime

---

### QA-04: Security
> *The system resists unauthorised access while still serving legitimate users.*

Travel agents call the booking API from outside the airline's network. Payment data must be protected. Ground staff manifests must only be seen by authorised staff.

- **Source of concern:** Multi-channel access (P-01, P-04) creates risk of unauthorised booking manipulation
- **Stimulus:** An unauthorised party tries to call the booking API directly
- **Response:** The request is blocked at the API Gateway before reaching any service
- **Measure:** 100% of API calls require a valid token. All access attempts are logged.

Security properties we care about (from lecture):
- **Confidentiality** — payment data encrypted in transit (TLS 1.3) and never stored raw
- **Integrity** — booking records cannot be modified outside authorised channels
- **Auditing** — every booking and change action is logged with who did it and when

---

### QA-05: Modifiability (UI)
> *Supporting usability changes without touching backend code.*

We also include **Usability** as a quality attribute because ground staff need to access manifests and scan boarding passes under time pressure. The system must be easy enough that they can do critical tasks in 2 steps, even if the network at the gate drops.

- **Stimulus:** Ground staff member needs to pull up the manifest during an unexpected gate change, 10 minutes before departure
- **Response:** Manifest is accessible in 2 taps; works offline from cached data if network is unavailable
- **Measure:** Critical tasks completable in ≤ 2 navigation steps; offline mode available

---

## 4. Architecture Model

### 4.1 Architecture Style: Microservices

We chose a **microservices architecture**. This means each major capability is a separate, independently deployable service with its own database.

**Why not a single system (monolith)?**
The five services in SkyBook have completely different load patterns:
- Check-In Service gets hammered for 2 hours before each departure, then goes quiet
- Pricing Engine changes rarely and handles low traffic
- Booking Service has unpredictable spikes during sales

If we built one big system, we would need to scale everything just to handle check-in peaks — which wastes resources. Worse, a bug in the Notification Service would crash the Booking Service too.

With microservices:
- Each service scales independently
- A crash in one service does not take down others
- Revenue Management can update the Pricing Engine without touching Booking

**How services communicate:**
- **Synchronous (REST/HTTP):** For things passengers are waiting for — search results, booking confirmation
- **Asynchronous (Apache Kafka):** For things that happen in the background — notifications, manifest updates, inventory sync

---

### 4.2 Use Case Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        SkyBook System                           │
│                                                                 │
│  Passenger ──────── Search Flights                              │
│  Passenger ──────── Book Ticket & Select Seat                   │
│  Passenger ──────── Online Check-In                             │
│  Passenger ──────── Receive Delay / Cancellation Notification   │
│                                                                 │
│  Travel Agent ───── Book Ticket via API                         │
│                                                                 │
│  Ground Staff ───── View Passenger Manifest                     │
│  Ground Staff ───── Manage Boarding                             │
│                                                                 │
│  Flight Operations ─ Trigger Delay / Cancellation              │
│                                                                 │
│  Revenue Management  Update Pricing Rules                       │
└─────────────────────────────────────────────────────────────────┘
```

---

### 4.3 Component Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│  CLIENTS                                                         │
│  [Passenger Web/Mobile]  [Travel Agent API]  [Ground Staff PWA]  │
└────────────────────────────┬─────────────────────────────────────┘
                             │ HTTPS
                    ┌────────▼────────┐
                    │   API Gateway   │  ← checks login, routes requests
                    └────────┬────────┘
           ┌─────────────────┼──────────────────┐
           │                 │                  │
  ┌────────▼───────┐  ┌──────▼──────┐  ┌────────▼────────┐
  │ Flight Search  │  │   Booking   │  │  Check-In       │
  │   Service      │  │   Service   │  │  Service        │
  │ (Elasticsearch)│  │ (Redis Lock)│  │  (SQS Queue)    │
  └────────────────┘  └──────┬──────┘  └────────┬────────┘
                             │                  │
                    ┌────────▼──────────────────▼────────┐
                    │         Inventory Service          │
                    │  (single source of truth for seats)│
                    └──────────────────┬─────────────────┘
                                       │
        ┌──────────────────────────────▼──────────────────────────┐
        │                  Apache Kafka (Event Bus)               │
        │  Topics: booking.confirmed · flight.cancelled ·         │
        │          flight.delayed · seat.released                 │
        └──────┬──────────────────────┬──────────────────┬────────┘
               │                      │                  │
     ┌─────────▼──────┐   ┌───────────▼──────┐  ┌───────▼──────────┐
     │ Notification   │   │  Flight Ops       │  │  Pricing Engine  │
     │ Service        │   │  Service          │  │  (Config DB,     │
     │ (email/SMS/    │   │                   │  │   hot-reload)    │
     │  push)         │   │                   │  │                  │
     └────────────────┘   └───────────────────┘  └──────────────────┘

  DATA STORES:
  PostgreSQL (bookings, manifests) · Redis (cache + seat locks) · Elasticsearch (search)
```

**What each component does:**

| Component | Responsibility |
|---|---|
| API Gateway | Checks authentication, rate-limits requests, routes to the right service |
| Flight Search Service | Returns flight results fast using a pre-built Elasticsearch index |
| Booking Service | Handles seat reservation with a lock to prevent double-booking |
| Inventory Service | The only place that knows how many seats are available — all channels read from here |
| Check-In Service | Processes check-in requests from a queue so the database is not overwhelmed |
| Notification Service | Listens for flight.cancelled and flight.delayed events, sends email/SMS/push |
| Pricing Engine | Applies current pricing rules; rules can be updated without restarting the service |
| Flight Ops Service | Used by Flight Operations to log delays and cancellations |
| Apache Kafka | The message bus — services publish events here, other services subscribe |

---

### 4.4 Key Flows

These are the three most important flows that directly address the case's design questions.

#### Flow A: How we prevent two passengers booking the same seat (solves P-01)

```
Passenger A ──► API Gateway ──► Booking Service
                                      │
                          Acquire Redis lock on seat 14A
                          (key: lock:flight:BK101:seat:14A, expires in 10s)
                                      │
                    ┌─────────────────┴──────────────────┐
               Lock acquired                        Lock NOT available
                    │                                     │
       Inventory confirms seat available       Return "seat no longer available"
       Charge payment                          to Passenger B immediately
       Confirm booking
       Publish booking.confirmed to Kafka
       Release lock
```

The **Redis lock** is the key idea here. Only one booking can hold the lock for seat 14A at a time. If Passenger B tries at the same moment, they get an immediate "unavailable" response — no race condition, no overbooking. The lock automatically expires after 10 seconds so it can never get stuck if the Booking Service crashes.

---

#### Flow B: What happens when a flight is cancelled (solves P-03)

```
Flight Operations ──► Flight Ops Service ──► Publish "flight.cancelled" to Kafka
                                                          │
                         ┌────────────────────────────────┼──────────────────┐
                         │                                │                  │
               Inventory Service                Notification Service    Rebooking Service
               Marks all seats unavailable   Queries all affected       Offers alternative
                                             passengers from database   flights, processes
                                             Sends email + SMS + push   refunds
                                             (retries on failure)
```

This is **event-driven architecture**. Flight Operations triggers one event. The Notification Service and Rebooking Service both react to it independently — they do not need to be called directly. If we add a new requirement (e.g., send alerts to the airport lounge system), we just add another subscriber to Kafka. Nothing else changes.

---

#### Flow C: How we handle peak check-in load (solves P-02)

```
Passengers check in ──► API Gateway ──► SQS Queue (buffer)
                                               │
                              Check-In workers drain the queue
                              at a controlled rate (e.g. 500/min)
                                               │
                              Generate boarding pass
                              Update manifest via Kafka event
                              Ground Staff PWA updates in real time (WebSocket)
```

Instead of all 3,000 passengers for one departure hitting the database at the same time, requests go into a queue first. Workers process them at a steady rate. The database never gets overwhelmed. This is the **Control Frequency of Events** performance tactic.

---

## 5. Tactics

A **tactic** is a specific design decision that directly improves one quality attribute. We pick the most important tactic for each quality attribute — the one that solves the actual problem in the case.

---

### 5.1 Availability Tactics — Fault Detection, Recovery, Prevention

**Goal:** The system must stay up during peak departure windows. We apply all three groups from the lecture.

#### Fault Detection — Heartbeat

Each service sends a heartbeat signal to our monitoring system (Prometheus) every 30 seconds. If the signal stops, an alert fires and the team knows which service is down.

> Why Heartbeat and not Ping/Echo? Because in Heartbeat, *the service itself* initiates the signal. This means even if a service is partially working (can receive pings but is internally broken), the heartbeat will fail because the service can no longer emit it. It catches deeper failures.

#### Fault Recovery — Active Redundancy

The Booking Service and Check-In Service each run with **at least 2 replicas at all times**. All replicas handle requests simultaneously. If one crashes, the other takes all the traffic instantly — zero downtime for users.

For the PostgreSQL database we use **Passive Redundancy**: one primary + one standby that is kept in sync. If the primary fails, the standby is promoted. There is a small delay (seconds) for the promotion to complete, but no data is lost.

#### Fault Prevention — Transaction (Saga Pattern)

Booking involves three steps: reserve seat → charge payment → confirm booking. If the payment fails halfway through, the seat must be released. We use the **Saga pattern**: each step publishes an event. If any step fails, a compensating event automatically reverses the previous steps. No partial bookings are ever saved to the database.

---

### 5.2 Modifiability Tactics — Localise Modification and Prevent Ripple Effect

**Goal:** Revenue Management can change pricing rules without the engineering team, and changing one service does not break other services.

#### Localise Modification — Abstract Common Services

The Notification Service is a shared service. Instead of each service (Booking, Flight Ops, Check-In) having its own notification logic, they all call one Notification Service. If we ever need to change from email to WhatsApp, we change it in **one place only**.

#### Prevent Ripple Effect — Use Intermediary (Kafka)

This is the most important modifiability tactic in our architecture. Kafka sits between services that produce events and services that consume them.

**Without Kafka:** Flight Ops Service would need to directly call Notification Service, Rebooking Service, and any future service — every new consumer requires changing Flight Ops code.

**With Kafka:** Flight Ops publishes one event. Any service can subscribe without Flight Ops knowing about it. Adding a new consumer (e.g., a lounge notification system) requires **zero changes** to any existing service.

---

### 5.3 Performance Tactics

**Goal:** Flight search stays fast under 10× load. Check-in does not crash the database.

#### Resource Demand — Control Frequency of Events

During the 24-hour check-in window, check-in requests go into an **SQS queue** before reaching the database. Workers drain the queue at a controlled rate. This directly prevents the P-02 problem — the database never sees all 3,000 passengers at once.

#### Resource Management — Maintain Multiple Copies of Data (Caching)

Flight availability is cached in **Redis** with a 30-second expiry. The Flight Search Service reads from Redis, not from the main database. This means 10× traffic does not increase database load at all — it just hits Redis, which is extremely fast (sub-millisecond).

---

### 5.4 Security Tactics

**Goal:** Unauthorised access is blocked. Payment data is protected. All actions are traceable.

#### Resisting Attack — Authenticate and Authorise Users

Every request to the API Gateway must carry a valid **JWT token**. The gateway checks this before routing to any service. Access control is role-based:
- Passengers: can only see their own bookings
- Travel Agents: can only create and retrieve bookings
- Ground Staff: read-only access to manifests
- Flight Operations: can only write to flight status

#### Recovering from Attack — Maintain Audit Trail

Every action (booking, cancellation, pricing rule change) is logged to a **separate, append-only log store**. Services can write to it but cannot delete or modify entries. If an attack happens, we can trace exactly what changed, who did it, and when.

---

## 6. Technical Decisions

These are the four most important decisions we made and why.

---

### Decision 1: Microservices (not a monolith)

We chose microservices because the five services in SkyBook have completely different scaling and change needs. Check-In needs to scale out during departure windows; Pricing Engine barely changes. If we built a monolith, a crash in Notifications would take down Booking.

**Trade-off we accept:** More operational complexity. We manage this with Kubernetes (automatic restarts, scaling) and centralised logging (ELK stack).

---

### Decision 2: Redis distributed lock for seat booking

To prevent two passengers booking the same seat (P-01), we use **Redis Redlock**.

When Passenger A starts booking seat 14A, the Booking Service sets a key in Redis: `lock:flight:BK101:seat:14A`. This key expires in 10 seconds. If Passenger B tries to book the same seat at the same time, they cannot get the lock and receive an immediate "unavailable" response.

This works across all channels simultaneously — website, mobile app, and travel agency API all go through the same Booking Service.

**Why not a database row lock?** A database lock ties up a database connection for the whole transaction. Redis is in-memory (1,000× faster) and the TTL ensures the lock is released even if the Booking Service crashes mid-transaction.

---

### Decision 3: Apache Kafka as the event bus

Kafka is the backbone of our async communication. We chose it for three specific reasons:

1. **Replayability:** If the Notification Service crashes during a flight cancellation, when it restarts it replays the `flight.cancelled` event from Kafka and sends all the notifications. With RabbitMQ or SQS, the message would be lost.
2. **Fan-out:** One `flight.cancelled` event can be consumed by the Notification Service, Rebooking Service, and Inventory Service simultaneously. No producer needs to know who is consuming.
3. **Order guarantee:** Booking events for the same flight are processed in order (Kafka partitions by flight ID). This matters for seat inventory — you cannot process "seat released" before "seat booked" on the same flight.

---

### Decision 4: Hot-reload configuration for pricing rules

Revenue Management updates pricing rules frequently. We store rules in a **configuration database**. The Pricing Engine watches for changes and reloads the new rules without restarting.

This means: Revenue Management changes a rule → it is live in 30 seconds → no engineer is involved → no downtime.

This directly solves FR-06 and the inconsistent rule enforcement in P-04 (rules are now enforced by one centralised engine, not applied differently per channel).

---

## 7. Risks and Trade-Offs

Every architectural decision involves a trade-off. These are the ones we are accepting.

| Risk | The tension | What we do about it |
|---|---|---|
| Microservices are complex to operate | Modifiability vs Availability — fixing one service can be hard to trace across the network | We use Kubernetes for automatic restarts and Jaeger for distributed tracing |
| JWT + TLS adds latency to every request | Security vs Performance — every request is slower because of auth checks | JWT tokens are cached after the first validation. TLS termination is offloaded to the load balancer |
| Redis lock expires before payment finishes | Availability vs Consistency — under very high load, a 10-second lock might expire mid-payment | We extend the lock TTL to 30 seconds for the payment step |
| Read replica is slightly behind the primary | Performance vs Consistency — search might show a seat as available when it was just booked | This is acceptable for search display. The Redis lock enforces strict consistency at the actual booking step |

---

## 8. Summary

Each problem in the case maps to a specific quality attribute and a specific tactic we apply.

| Problem from case | Quality Attribute | Tactic we apply | How it works |
|---|---|---|---|
| P-01: Double booking | Availability (Fault Prevention) | Transaction + Redis Distributed Lock | Lock on each seat ID prevents concurrent booking; Saga rolls back on failure |
| P-02: Check-in slowdown | Performance (Resource Demand) | Control Frequency of Events | SQS queue buffers requests; workers drain at controlled rate |
| P-03: No notifications | Availability (Fault Recovery) + Modifiability | Checkpoint/Rollback + Use Intermediary (Kafka) | Kafka fan-out triggers Notification Service; missed events replayed on recovery |
| P-04: Inconsistent seat rules | Modifiability (Localise Modification) | Abstract Common Services | All seat rule logic in one Booking Service; no per-channel variation possible |
| P-05: Slow agency confirmation | Performance (Resource Management) | Maintain Multiple Copies of Data | Redis cache + read replicas keep Booking Service fast; confirmation under 3 seconds |

The architecture we designed uses **microservices** connected by **Apache Kafka**, with **Redis** handling both seat locking and caching, and **Kubernetes** managing scaling and recovery. Together, these four technical choices address all five problems in the case while meeting the six functional requirements.

---

*SkyBook Architecture Design — Case Study 10*
