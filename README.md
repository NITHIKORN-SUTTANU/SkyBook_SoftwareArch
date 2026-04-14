# SkyBook — Airline Reservation System
## Software Architecture Design Document
**Case Study 10 | Version 1.0 | April 2026**

---

## Table of Contents

1. [Background & Organisation Overview](#1-background--organisation-overview)
2. [Functional Requirements](#2-functional-requirements)
3. [Quality Attributes](#3-quality-attributes)
4. [Architecture Model](#4-architecture-model)
5. [Architecture Tactics](#5-architecture-tactics)
6. [Technical Decisions](#6-technical-decisions)
7. [Risks and Trade-Offs](#7-risks-and-trade-offs)
8. [Summary](#8-summary)

---

## 1. Background & Organisation Overview

SkyBook is the digital reservation platform for a regional airline operating **40 routes across 15 cities**. The system supports the full passenger journey — from flight search and booking through to check-in and boarding — across the airline's own website, mobile app, and third-party travel agency integrations.

### 1.1 Stakeholders

| Stakeholder | Role |
|---|---|
| **Passengers** | Search flights, book tickets, select seats, check in online, receive notifications |
| **Travel Agents** | Book on behalf of customers via REST API integration |
| **Ground Staff** | Access passenger manifests, manage check-in desks, boarding gates, and baggage |
| **Flight Operations** | Monitor schedules, manage delays, assign and swap aircraft |
| **Revenue Management** | Configure and update dynamic pricing rules based on demand |

### 1.2 Operational Context

- ~**200 flights per day** carrying up to **25,000 passengers**
- Continuous multi-channel ticket sales: website, mobile app, travel agent API
- Peak load periods: weekends, school holidays, promotional fare launches

### 1.3 Known Problems with Legacy System

| Problem | Impact |
|---|---|
| **Overbooking** | Seat counts not synchronised across channels — concurrent bookings cause conflicts |
| **Check-in performance** | System degrades severely during the 24-hour pre-departure rush |
| **No automatic notifications** | Passengers not informed of delays or cancellations |
| **Inconsistent seat rules** | Seat selection enforcement differs per channel and fare type |
| **Slow agency confirmations** | Travel agency bookings take up to 10 minutes to confirm |

---

## 2. Functional Requirements

| ID | Requirement | Detail |
|---|---|---|
| FR-01 | **Flight Search** | Search by route and date; display available seats, fare classes, and real-time prices |
| FR-02 | **Ticket Booking** | Select flight, choose seat, add baggage options, complete payment |
| FR-03 | **Online Check-In** | Check-in opens 24 hrs before departure; boarding pass generated and delivered |
| FR-04 | **Ground Staff Access** | Live passenger manifests, check-in status, and boarding gate operations |
| FR-05 | **Flight Notifications** | Automatic passenger notification on delay or cancellation via email, SMS, push |
| FR-06 | **Pricing Rule Updates** | Revenue team updates pricing rules without downtime or redeployment |
| FR-07 | **Travel Agent API** | Programmatic booking with confirmation returned in seconds |
| FR-08 | **Cancellation & Rebooking** | Automated notification and rebooking options for all affected passengers |

---

## 3. Quality Attributes

Seven quality attributes are identified as critical for SkyBook:

| # | Quality Attribute | Scenario | Target Metric |
|---|---|---|---|
| QA-01 | **Availability** | System stays up during peak departure window (200 flights in 6 hrs) | ≥ 99.9% uptime |
| QA-02 | **Performance** | Flight search under 10× normal load (holiday sale launch) | < 500ms p95 latency |
| QA-03 | **Consistency** | Two passengers simultaneously book the last seat on a flight | Zero duplicate seat assignments |
| QA-04 | **Scalability** | 500% traffic spike during promotional fare release | Auto-scale in < 90 seconds |
| QA-05 | **Modifiability** | Revenue team updates pricing rule during active sales | Zero downtime; < 30s propagation |
| QA-06 | **Reliability** | Flight cancelled — all passengers must be notified | 100% notification delivery |
| QA-07 | **Security** | Travel agents access booking API | OAuth2 + per-agency API key scoping |

### 3.1 Quality Attribute Scenarios

#### QA-01: Availability
- **Stimulus:** All 200 daily departures within a 6-hour peak window
- **Response:** Ground Staff Portal and Check-In Service remain fully operational
- **Measure:** No single point of failure; 99.9% uptime SLA enforced

#### QA-03: Consistency
- **Stimulus:** Two passengers on different channels book the same last seat simultaneously
- **Response:** Exactly one booking succeeds; the other receives a clear "seat unavailable" response
- **Measure:** Zero duplicate seat conflicts across all channels

#### QA-05: Modifiability
- **Stimulus:** Revenue manager changes the peak-hour surcharge rule at 08:00 on a Monday
- **Response:** New rule applied to all new searches and bookings within 30 seconds
- **Measure:** No redeployment required; no passenger-facing service interruption

#### QA-06: Reliability
- **Stimulus:** A flight is cancelled 2 hours before departure with 180 booked passengers
- **Response:** All 180 passengers notified within 5 minutes via their preferred channel
- **Measure:** At-least-once delivery guaranteed; failed notifications retried via dead-letter queue

---

## 4. Architecture Model

### 4.1 Architectural Style: Microservices

SkyBook adopts a **microservices architecture**. Each domain capability is encapsulated in an independently deployable service. Services communicate:

- **Asynchronously** via Apache Kafka for events (bookings, delays, cancellations)
- **Synchronously** via REST for user-facing requests

This enables independent scaling, fault isolation, and team autonomy — critical for an operation where check-in load, booking load, and pricing updates all have different scaling profiles.

### 4.2 Use Case Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                        SkyBook System                            │
│                                                                  │
│  Passenger ──────── Search Flights                               │
│  Passenger ──────── Book Ticket                                  │
│  Passenger ──────── Online Check-In                              │
│  Passenger ──────── Receive Notification ◄── Flight Ops          │
│                                                                  │
│  Travel Agent ───── API Booking                                  │
│                                                                  │
│  Ground Staff ───── Access Manifest                              │
│  Ground Staff ───── Manage Boarding                              │
│                                                                  │
│  Flight Ops ──────── Trigger Delay / Cancellation                │
│                                                                  │
│  Revenue Mgmt ───── Update Pricing Rules                         │
└──────────────────────────────────────────────────────────────────┘
```

### 4.3 Component Architecture

| Component | Responsibility | Technology |
|---|---|---|
| **API Gateway** | Auth, rate limiting, request routing across all channels | Kong / AWS API GW |
| **Flight Search Service** | Route/date search; availability queries via read replica | Node.js + Elasticsearch |
| **Booking Service** | Seat reservation with distributed lock; booking lifecycle | Java Spring Boot + Redis |
| **Inventory Service** | Single source of truth for seat counts; overbooking guard | PostgreSQL + Redis Pub/Sub |
| **Check-In Service** | Online check-in processing; boarding pass generation | Python FastAPI + SQS |
| **Notification Service** | Email/SMS/push on delay or cancellation events | Node.js + Kafka Consumer |
| **Pricing Engine** | Dynamic fare calculation; hot-reload of pricing rules | Python + Config DB |
| **Flight Ops Service** | Delay/cancel management; publishes Kafka events | Java + Kafka Producer |
| **Ground Staff Portal** | Live manifest, boarding scan; offline-capable | React PWA + WebSocket |
| **Event Bus** | Async messaging backbone; durable event log | Apache Kafka (3-broker) |
| **Distributed Lock** | Prevent concurrent same-seat booking | Redis Redlock |

### 4.4 System Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│                            CLIENTS                                   │
│   [Passenger Web/Mobile]   [Travel Agent API]   [Ground Staff PWA]   │
└──────────────────────────────┬───────────────────────────────────────┘
                               │ HTTPS
                      ┌────────▼────────┐
                      │   API Gateway   │  Auth · Rate-Limit · Routing
                      └────────┬────────┘
            ┌──────────────────┼────────────────────────┐
            │                  │                        │
   ┌────────▼────────┐  ┌──────▼───────┐  ┌────────────▼──────────┐
   │ Flight Search   │  │   Booking    │  │   Check-In Service    │
   │   Service       │  │   Service    │  │                       │
   └────────┬────────┘  └──────┬───────┘  └────────────┬──────────┘
            │                  │                        │
            │         ┌────────▼────────────────────────▼────────┐
            │         │           Inventory Service              │
            │         │   (Seat counts · overbooking guard)      │
            │         └──────────────────┬───────────────────────┘
            │                            │
   ┌────────▼────────────────────────────▼──────────────────────┐
   │                  Event Bus  (Apache Kafka)                 │
   │   flight.delayed | booking.confirmed | flight.cancelled    │
   └────────┬───────────────────────┬──────────────────┬────────┘
            │                       │                  │
   ┌────────▼──────┐   ┌────────────▼────┐  ┌─────────▼────────┐
   │ Notification  │   │  Flight Ops Svc │  │  Pricing Engine  │
   │   Service     │   │                 │  │  (Hot Reload)    │
   └───────────────┘   └─────────────────┘  └──────────────────┘

   ─────────────────────── DATA STORES ─────────────────────────────
   [PostgreSQL]    [Redis Cache/Locks]    [Kafka Log]    [Elastic]
```

### 4.5 Key Architectural Flows

#### Flow A — Concurrent Seat Booking (Consistency)

```
Client ──► API Gateway ──► Booking Service
                               │
                               ├── Acquire Redis Redlock (seat ID, 10s TTL)
                               │       └── Lock unavailable → "seat conflict" response
                               │
                               ├── Inventory Service: validate seat (strong read)
                               ├── Payment Service: charge card
                               ├── Confirm booking, publish booking.confirmed to Kafka
                               └── Release lock, Inventory decrements seat count
```

#### Flow B — Flight Cancellation (Reliability)

```
Flight Ops ──► Flight Ops Service ──► publish flight.cancelled (Kafka)
                                              │
                         ┌────────────────────┼────────────────────┐
                         │                    │                    │
               Inventory Service      Notification Service    Rebooking Service
               (mark seats N/A)   (query affected passengers   (offer alternatives,
                                   → email + SMS + push)        process refunds)
```

#### Flow C — Peak Check-In Load (Availability + Scalability)

```
Passenger ──► API Gateway ──► SQS Queue (buffer)
                                    │
                           Check-In Workers (drain at controlled rate)
                                    │
                           Generate boarding pass
                                    │
                           Update manifest (Inventory event)
                                    │
                           Ground Staff PWA ◄── WebSocket live update
```

---

## 5. Architecture Tactics

| Quality Attribute | Tactic | Implementation |
|---|---|---|
| Availability | **Redundancy / Failover** | Min 2 replicas per service; K8s liveness probes; PostgreSQL primary-replica failover via Patroni |
| Availability | **Circuit Breaker** | Resilience4j on Booking → Inventory calls; fallback returns "temporarily unavailable" to prevent cascade failure |
| Performance | **Read-Aside Cache** | Redis caches flight search results (30s TTL); seat snapshots updated via Inventory events |
| Performance | **Read Replicas** | Separate PostgreSQL read replicas for search queries; primary handles writes only |
| Consistency | **Distributed Lock** | Redis Redlock acquires seat-level lock during booking; TTL auto-releases stale locks on service crash |
| Consistency | **Saga Pattern** | Choreography saga: Book → Reserve → Charge → Confirm; compensating transactions on any failure |
| Scalability | **Horizontal Auto-Scaling** | Kubernetes HPA scales Booking and Check-In on CPU/RPS; stateless design enables instant scale-out |
| Scalability | **Queue-Based Load Levelling** | Check-in requests buffered in SQS during the 24-hr peak; workers drain at controlled rate |
| Modifiability | **Hot Configuration Reload** | Pricing rules stored in Redis/config DB; Pricing Engine subscribes to change events and reloads without restart |
| Reliability | **Event-Driven Notification** | Kafka fan-out to Notification Service; at-least-once delivery; dead-letter queue for failed notifications |
| Security | **Gateway Authentication** | JWT validation at API Gateway; per-agency OAuth2 API keys scoped to booking operations only |

### 5.1 Tactic Detail: Distributed Locking (Seat Consistency)

The most critical correctness problem in the legacy system is concurrent booking of the same seat. The **Redis Redlock** algorithm is applied as follows:

- Lock key constructed per seat: `lock:flight:{flightId}:seat:{seatId}`
- Booking Service acquires lock using `SETNX` with a 10-second TTL
- If lock acquired → booking transaction proceeds
- If lock unavailable → immediate "seat conflict" response to client; no DB write attempted
- TTL ensures stale locks (e.g. from a crashed instance) are auto-released
- Works uniformly across **all channels** — web, mobile, travel agent API — since all go through the same Booking Service

### 5.2 Tactic Detail: Saga Pattern (Booking Transaction)

Booking spans multiple services (seat reservation, payment, confirmation). A choreography-based Saga is used:

- Each step publishes a success or failure event to Kafka
- Compensating transactions automatically reverse completed steps on failure
  - Example: if payment fails after seat reserved → `seat.release` event published → Inventory restores count
- Avoids distributed ACID transactions while maintaining eventual consistency

### 5.3 Tactic Detail: Event-Driven Notification

The legacy system has no automatic notification. The new approach:

- Flight Ops Service publishes `flight.delayed` or `flight.cancelled` to Kafka
- Notification Service subscribes, queries booking DB for all affected passengers
- Messages dispatched via email (SendGrid), SMS (Twilio), and push notifications
- Kafka consumer with manual offset commit → at-least-once delivery guarantee
- Dead-letter queue (DLQ) captures failed notifications for retry and manual review

---

## 6. Technical Decisions

| Decision | Choice | Alternatives Considered | Rationale |
|---|---|---|---|
| **Architecture Style** | Microservices | Monolith, SOA | Independent scaling of Check-In/Booking during peak; fault isolation; team autonomy for Revenue Management |
| **Seat Locking Mechanism** | Redis Redlock | DB row locks, Optimistic concurrency | Sub-ms cross-service lock; handles multi-channel race conditions; TTL auto-releases stale locks |
| **Messaging Backbone** | Apache Kafka | RabbitMQ, AWS SQS | Durable replayable log; native fan-out; high throughput for 200 flights/day events |
| **Database** | PostgreSQL + Redis | MySQL, MongoDB, shared DB | ACID guarantees for booking/inventory; Redis for sub-ms availability reads; polyglot where justified |
| **Notifications** | Event-driven (Kafka) | Synchronous REST, DB polling | Decouples Flight Ops from Notification logic; retry on failure; scales independently |
| **Pricing Updates** | Hot-reload via config DB | Redeploy, feature flags | Zero downtime; Revenue team self-service; no engineer involvement for rule changes |
| **Check-In Load** | SQS queue buffer | Vertical scale, synchronous writes | Prevents DB saturation in 24-hr rush; predictable throughput; cost-effective |
| **Ground Staff App** | React PWA | Native mobile, thick client | Cross-device; offline via service worker; no app store deployment friction |
| **Container Orchestration** | Kubernetes + HPA | Docker Compose, bare VMs | Automated horizontal scaling; self-healing pods; industry standard for microservices |

### 6.1 Decision Detail: Microservices vs Monolith

A monolithic rebuild was considered for simplicity. Microservices were selected because:

- **Independent scaling** — Check-In Service scales during departure windows without affecting Booking Service
- **Fault isolation** — a crash in the Notification Service does not affect bookings or check-in
- **Team autonomy** — Revenue Management team owns, deploys, and updates the Pricing Engine independently
- **Trade-off accepted:** increased operational complexity, managed via Kubernetes, Istio service mesh, and centralised logging

### 6.2 Decision Detail: Apache Kafka vs Alternatives

- **Durable, replayable event log** — Notification Service can replay missed events after recovery
- **Native fan-out** — a single `flight.cancelled` event consumed simultaneously by Notification, Rebooking, and Reporting services
- **High throughput** — 200 flights/day with thousands of booking events is well within Kafka's capabilities
- **Offset management** — enables at-least-once delivery guarantee needed for notification correctness

### 6.3 Deployment Architecture

- All services containerised with Docker, orchestrated via **Kubernetes**
- **Kubernetes HPA** configured for Booking and Check-In (scale on CPU > 70% or RPS threshold)
- **PostgreSQL** deployed as primary + 2 read replicas with Patroni automatic failover
- **Kafka** deployed as 3-broker cluster with replication factor 3 for durability
- All services behind **API Gateway (Kong)**; mTLS between internal services
- **CI/CD** via GitHub Actions: dev → staging → production staged deployments

---

## 7. Risks and Trade-Offs

| Risk | Impact | Mitigation |
|---|---|---|
| **Microservices complexity** | Increased operational burden: distributed tracing, inter-service latency | Kubernetes + Istio service mesh; centralised ELK logging; Jaeger distributed tracing |
| **Kafka cluster failure** | Event loss could cause missed notifications or booking inconsistency | 3-broker cluster with replication factor 3; DLQ for failed events; alerting on consumer lag |
| **Redis lock TTL expiry** | Lock may expire before long-running payment transaction completes | Extended TTL (30s) for payment step; monitor lock acquisition latency in Grafana |
| **Read replica lag** | Stale seat availability displayed in search results | Acceptable eventual consistency for display; strong consistency enforced at lock acquisition time only |
| **Ground Staff PWA offline** | Cached manifest may be stale at departure gate if network is lost | Service worker caches last known manifest; "offline mode" warning banner shown to staff |

---

## 8. Summary

The SkyBook architecture directly addresses each legacy system failure through targeted architectural decisions:

| Legacy Problem | Architectural Solution |
|---|---|
| Overbooking | Redis Redlock distributed locking applied uniformly across all sales channels |
| Check-in performance | SQS queue-based load levelling + Kubernetes horizontal auto-scaling |
| No notifications | Event-driven Kafka pipeline with at-least-once delivery and dead-letter queue |
| Inconsistent seat rules | Centralised enforcement in Booking Service — no per-channel variation possible |
| Slow agency confirmations | Synchronous REST booking response in seconds; async Kafka confirmation |
| Pricing downtime | Hot-reload configuration mechanism — zero downtime, Revenue team self-service |

The microservices architecture, underpinned by **Kafka**, **Redis**, and **Kubernetes**, delivers the availability, consistency, scalability, and modifiability required to operate a modern airline reservation system at the stated operational scale of 200 flights and 25,000 passengers per day.

---

*End of Document*
