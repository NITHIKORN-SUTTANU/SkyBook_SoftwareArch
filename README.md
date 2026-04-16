# SkyBook — Software Architecture Design Document

**Case Study 10 | Airline Reservation System**
**Author:** 
6731503011-DechawatWetprasit
6731503015-Teerapat Sukkasem
6731503018-Nithanthip Kulmong
6731503019-Nithikorn Suttanu
**Date:** April 2026

---

## 1. Overview

SkyBook is the reservation system for a regional airline running 40 routes across 15 cities, handling 200 flights per day and up to 25,000 passengers. Tickets are sold through the airline's own website, mobile app, and third-party travel agencies.

The current system was built in the 1990s and has five known problems:

| # | Problem |
|---|---------|
| 1 | Overbooking happens because seat counts are not synchronised across sales channels |
| 2 | The system slows down when many passengers check in close to departure time |
| 3 | Passengers are not notified when a flight is delayed or cancelled |
| 4 | Seat selection rules are applied inconsistently depending on which channel is used |
| 5 | Travel agency bookings take up to 10 minutes to confirm |

This document describes the new architecture that fixes each of these problems.

---

## 2. Functional Requirements

| ID | What the system must do |
|----|------------------------|
| FR1 | Passengers can search flights by route and date, and see available seats and fares |
| FR2 | Passengers can book a ticket, select a seat, and add baggage |
| FR3 | Online check-in opens exactly 24 hours before departure |
| FR4 | Ground staff can view the passenger list and manage boarding |
| FR5 | Passengers are automatically notified when their flight is delayed or cancelled |
| FR6 | Revenue management can change pricing rules while the system stays running |
| FR7 | Travel agents can book tickets through an API and receive confirmation immediately |

---

## 3. Quality Attributes

| ID | Quality Attribute | Requirement |
|----|-------------------|-------------|
| QA1 | **Consistency** | Two passengers must never be confirmed into the same seat |
| QA2 | **Availability** | The system must stay operational during the peak check-in window near departure |
| QA3 | **Performance** | Travel agent bookings must confirm within 3 seconds |
| QA4 | **Reliability** | Every affected passenger must receive a notification when a flight is cancelled |
| QA5 | **Modifiability** | Pricing rules must be changeable without restarting any service |
| QA6 | **Scalability** | The system must handle booking spikes during weekends and school holidays |

---

## 4. Architecture Model

### 4.1 Architecture Style

**Microservices with an Event-Driven backbone.**

Each part of the system is its own independent service. Services communicate by publishing and consuming events through **Apache Kafka** for background work, and call each other directly only when an immediate response is required. This means:

- One service failing does not bring down the others
- There is one single Inventory Service that all sales channels go through — eliminating the split-inventory overbooking problem
- Background work (notifications, baggage sync) does not slow down or block the booking response

### 4.2 Services

| Service | What it does |
|---------|-------------|
| **API Gateway** | Single entry point for the website and mobile app. Routes requests to the correct service |
| **Flight Search Service** | Returns available routes, dates, seat counts, and fares |
| **Inventory Service** | The only service that controls seat availability. Places and releases seat locks |
| **Booking Service** | Creates booking records, enforces all fare and seat selection rules, and handles rebooking |
| **Check-in Service** | Activates exactly 24 hours before departure. Issues boarding passes |
| **Notification Service** | Sends email notifications to passengers for delays, cancellations, and booking confirmations |
| **Boarding Service** | Gives ground staff access to the passenger manifest and boarding status |
| **Pricing Service** | Manages fare rules. Reloads updated rules at runtime without restarting |
| **Agent API Gateway** | Separate entry point for travel agents. Returns synchronous booking confirmation |

> **Why email only for notifications?**
> Every passenger provides an email address at booking time. SkyBook is a regional airline on 40 routes — adding SMS would require managing SMS gateway contracts across multiple regions for a system that already has reliable email contact for all passengers.

### 4.3 Component Diagram

```
  Website / Mobile App     Travel Agents       Flight Operations
         │                      │                      │
         ▼                      ▼                      ▼
  ┌─────────────┐    ┌──────────────────┐    publishes flight
  │ API Gateway │    │ Agent API Gateway│    status events
  └──────┬──────┘    └────────┬─────────┘              │
         │                    │                        │
    ┌────┼──────────┐          │                        │
    ▼    ▼          ▼          │                        │
  ┌────┐ ┌────────┐ ┌────────┐ │                        │
  │Flt.│ │Booking │ │Check-in│ │                        │
  │Srch│ │Service │ │Service │ │                        │
  └────┘ └───┬────┘ └────┬───┘ │                        │
             │           │     │                        │
             └─────┬─────┘     │                        │
                   │◄──────────┘                        │
                   ▼                                    │
          ┌─────────────────┐                           │
          │ Inventory Svc   │  ← Single source of       │
          │ (Redis locking) │    truth for seats         │
          └────────┬────────┘                           │
                   │                                    │
                   └──────────────┬─────────────────────┘
                                  │
                                  ▼
                         ┌─────────────────┐
                         │  Apache Kafka   │  ← All background events
                         └──┬──────────┬───┘
                            │          │
                            ▼          ▼
               ┌──────────────┐  ┌──────────────────┐
               │Notification  │  │Booking Svc       │
               │  Service     │  │(rebooking only)  │
               └──────────────┘  │Boarding Svc      │
                                 │Pricing Svc       │
                                 └──────────────────┘
```

> Both gateways route to the **same Booking Service and Inventory Service** — there is no separate seat count per channel.

### 4.4 Use Case Summary

**Passenger**
- Search flights → Flight Search Service
- Book + select seat → Booking Service → Inventory Service (seat lock applied)
- Check in → Check-in Service (available exactly 24h before departure)
- Receive cancellation or delay email → Notification Service

**Travel Agent**
- Book via API → Agent API Gateway → Booking Service → Inventory Service → confirmation returned under 3 seconds

**Ground Staff**
- View passenger manifest → Boarding Service
- Mark passenger as boarded → Boarding Service → Kafka event

**Revenue Management**
- Edit fare rules → Pricing Service admin panel → rules reload within 60 seconds, no restart needed

**Flight Operations**
- Mark flight as delayed or cancelled → publishes `flight.cancelled` or `flight.delayed` event directly to Kafka → Notification Service and Booking Service each consume the event independently

---

## 5. Tactics

### QA1 — Consistency: No Double Booking

**Tactic: Pessimistic Seat Locking with Redis TTL**

When a passenger selects a seat, the Inventory Service writes a lock for that seat in **Redis** with a 10-minute expiry (TTL). While the lock exists, no other booking from any channel can claim that seat.

- Payment completed within 10 min → lock converts to a confirmed booking in the database
- Payment not completed within 10 min → Redis automatically deletes the lock, seat becomes available again
- All channels (website, mobile app, travel agent API) call the same Inventory Service — there is no separate seat count per channel to go out of sync

**Why Redis?** It supports key expiry natively (TTL), operates in memory so lock checks take under 1 millisecond, and handles concurrent write requests from multiple service instances safely.

---

### QA2 — Availability: Surviving the Departure Rush

**Tactic: Horizontal Scaling via Kubernetes HPA**

The case study states the system becomes slow as passengers check in near departure time. The Check-in Service is the service under heaviest load during this window. It is deployed on **Kubernetes** with a Horizontal Pod Autoscaler (HPA) that adds instances automatically when CPU exceeds 70%.

- Normal hours: 2 instances of the Check-in Service running
- Peak window (final 1–2 hours before departure, when most passengers check in last-minute): scales up to 8 instances automatically

The **Check-in Service database** also uses a read replica. Passengers viewing their boarding pass go to the replica. Only write operations (confirming check-in, issuing a boarding pass) go to the primary database. This prevents read traffic from competing with writes during the rush.

---

### QA3 — Performance: Fast Travel Agent Confirmation

**Tactic: Dedicated Synchronous Agent API Gateway**

The current system is slow for travel agents because their requests compete with all other traffic on the same path. The new design gives travel agents their own **Agent API Gateway** that connects directly to the Booking Service and waits for a synchronous response — no queue in between.

- Target response time: **under 3 seconds**
- Rate limit: 100 requests/second per agency account
- Authentication: API key issued per travel agency

Agent traffic is completely isolated from passenger traffic. A surge in passenger bookings during school holidays does not affect agent confirmation speed.

---

### QA4 — Reliability: Guaranteed Cancellation Notifications

**Tactic: Event-Driven Fan-out with Retry Queue**

When Flight Operations marks a flight as cancelled, they publish a `flight.cancelled` event directly to Kafka. This event includes the list of affected passenger email addresses, so downstream services do not need to query another service's database.

Two services consume this event independently:

- **Notification Service** — sends a cancellation email to every affected passenger in parallel. If an email fails, it retries up to **3 times** with a 30-second wait between attempts. If all 3 retries fail, the failure is logged for manual review by the operations team.
- **Booking Service** — finds the next available flight on the same route, places a 2-hour seat hold per affected passenger, and provides the rebooking link that is included in the cancellation email.

---

### QA5 — Modifiability: Pricing Rule Updates Without Downtime

**Tactic: Externalized Configuration with Hot Reload**

Fare rules (e.g., which fare types allow seat selection, dynamic pricing multipliers) are stored in a **dedicated PostgreSQL configuration table**, not in the Pricing Service code. Revenue management edits rules through an admin panel that writes directly to this table.

The Pricing Service polls the table every **60 seconds**. When a change is detected, it reloads the rules into memory — no restart, no redeployment, no impact on bookings in progress.

**Why PostgreSQL?** It is already used as the primary relational database across services. A separate config server would add unnecessary infrastructure for rules that change at most a few times per day.

---

### QA6 — Scalability: Handling Weekend and Holiday Booking Spikes

**Tactic: Async Processing for Non-Critical Work via Kafka**

The case study states the heaviest booking periods are weekends and school holidays. When a passenger completes a booking, the Booking Service immediately returns a confirmation response. Everything that happens after — sending the booking confirmation email, syncing baggage options to the check-in record — runs **asynchronously** as Kafka consumers in the background.

The Booking Service does not wait for any of these tasks before responding. This keeps booking response time fast regardless of how many passengers are booking at the same time.

---

## 6. Technology Decisions

| Decision | Choice | Reason |
|----------|--------|--------|
| Seat lock store | **Redis** | Native TTL; sub-millisecond reads; handles concurrent writes safely |
| Message broker | **Apache Kafka** | Durable event log; one event consumed independently by multiple services |
| Container orchestration | **Kubernetes** | Built-in HPA for per-service auto-scaling |
| Notification channel | **Email** | All passengers provide email at booking; no SMS provider contracts needed |
| Agent API style | **REST synchronous** | Agents require instant confirmation; async would reintroduce the 10-minute delay |
| Pricing config store | **PostgreSQL table** | Already in use; no extra infrastructure needed for infrequent rule changes |
| Database model | **One database per service** | Prevents one service's schema change from breaking another |

---

## 7. Trade-offs

### Consistency over Availability — Seat Booking

When the Inventory Service is under heavy load or has a brief network issue, it will **reject new seat lock requests** rather than risk confirming two passengers into the same seat. A failed lock attempt is recoverable — the passenger retries in seconds. A double-booked seat requires staff intervention at the gate. Consistency is the correct priority for a booking system.

### Synchronous Agent Path — Less Scalable, but Required

The Agent API Gateway holds a connection open while waiting for a booking response. This is slightly less efficient than async processing. However, the case study explicitly identifies slow confirmation as a problem that must be fixed. Making agent bookings async would reintroduce the same problem. The dedicated gateway isolates agent traffic from passenger traffic, so this efficiency cost has no impact on passenger-facing performance.

---

## 8. Flight Cancellation — Full Flow

> Example: Flight SK-110 (Bangkok → Chiang Mai) is cancelled due to weather.

```
Flight Operations marks SK-110 as CANCELLED
      │
      ▼
Flight Operations publishes event directly to Kafka:
  {
    event: "flight.cancelled",
    flightId: "SK-110",
    date: "2026-04-20",
    route: "Bangkok → Chiang Mai",
    passengers: [ { name: "...", email: "..." }, ... ]
  }
      │
      ├─────────────────────────────────────────────┐
      ▼                                             ▼
Notification Service (consumer 1)          Booking Service (consumer 2)
  │                                                 │
  │ Reads passenger list from event                 │ Finds next available
  │ Sends cancellation email to each                │ Bangkok → Chiang Mai flight
  │ passenger in parallel                           │ Places 2-hour seat hold
  │ (all emails sent within 30 seconds)             │ per affected passenger
  │                                                 │
  │ If email fails → retry x3 (30s gaps)            │
  │ If all retries fail → log for ops team          │
  │                                                 │
  └──────── Email includes rebooking link ──────────┘

Passenger confirms rebooking within 2 hours:
  └── Seat hold → Confirmed booking

Passenger does not respond within 2 hours:
  └── Seat hold expires → seat released back to Inventory Service
                        → refund triggered automatically
```

---

## 9. Problem-to-Solution Summary

| Original Problem | Solution |
|-----------------|----------|
| Overbooking across channels | Single Inventory Service; Redis seat lock with 10-min TTL; all channels use the same service |
| System slowdown during check-in rush | Kubernetes HPA scales Check-in Service up to 8 instances; read replica on Check-in DB |
| No passenger notifications | Flight Operations publishes to Kafka → Notification Service emails all affected passengers in parallel |
| Inconsistent seat selection rules | All bookings enforced through one Booking Service regardless of channel |
| Slow travel agent confirmation | Dedicated Agent API Gateway; synchronous REST response under 3 seconds |
| Pricing updates cause downtime | Fare rules in PostgreSQL config table; Pricing Service hot-reloads every 60 seconds |
