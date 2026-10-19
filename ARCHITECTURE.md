# Gaz Express — Software Architecture Document

| Field | Value |
|---|---|
| **Document type** | Software Architecture Description (SAD) |
| **Standard alignment** | Inspired by ISO/IEC/IEEE 42010 · C4 viewpoints · ISO/IEC 25010 quality attributes |
| **Program** | Project 2025 · Gaz Express |
| **Product line** | v2 (production) · v3 (active engineering) |
| **Scope** | Platform architecture + Flutter multi-role client (`gas_app`) |
| **Package** | `com.gazxpress24.app` |
| **Client version** | `2.0.10+26` |
| **Classification** | Internal / stakeholder engineering |

**Companion artifacts:** [`README.md`](./README.md) · [`DIAGRAMS.md`](./DIAGRAMS.md) · [`ENGINEERING_REMEDIATION_REPORT.md`](./ENGINEERING_REMEDIATION_REPORT.md) 

---

## 1. Architecture drivers

### 1.1 Business problem and core constraint

Problem statement: see [`README.md`](./README.md) §1. In summary, every order must reach the certified agent responsible for the customer's delivery zone, through rule-based, reproducible dispatch decided at order time.

### 1.2 Non-negotiable architectural principle

> Critical dispatch invariants cannot rely on the UI or application layer alone.  
> The **persistence layer** is the final authority for routing consistency and conflict prevention.

Routing consistency is enforced through database-level constraints and transactional guarantees. Clients and admin tools are projections of that authority.

### 1.3 Quality attribute priorities (ISO/IEC 25010)

| Priority | Attribute | Architectural implication |
|---|---|---|
| P0 | **Functional correctness / integrity** | Zone validation before order creation; conflict-free routing invariants |
| P0 | **Reliability / operational continuity** | Continuity policy, admin messaging, configuration without redeploy |
| P1 | **Performance efficiency** | Client caching, lazy UI mounting; server pagination (v3) |
| P1 | **Security** | Secure session storage; server-side authorization as system of record |
| P2 | **Maintainability / evolvability** | Zone and assignment changes at configuration layer; clean architecture on mobile |
| P2 | **Compatibility / compliance** | Android API 36 for Google Play |

---

## 2. Context (system of interest)

| Subsystem | Role |
|---|---|
| **Zone & routing core** | Geospatial coverage, single zone resolution, agent–zone assignment |
| **Order pipeline** | Immediate + scheduled orders; status lifecycle; atomic routing commit |
| **Operational configuration** | Zones, assignments, continuity policy, messaging, pricing, service hours |
| **Real-time event layer** | Order and notification events to all roles |
| **Customer client** | Order placement, location, checkout, history |
| **Agent client** | Field queues, map navigation, delivery status |
| **Admin client** | Products, agents and assignments, users, zones, pricing, messaging, financial records |

**External dependencies:** verification/SMS providers (covered by the continuity policy), Google Maps Platform, Firebase Cloud Messaging.

---

## 3. Geographic distribution architecture

### 3.1 Two-phase model

| Phase | Actor | Activity |
|---|---|---|
| **Planning (configuration)** | Admin | Define zones; assign each certified agent to one or more zones; adjust and expand coverage |
| **Execution (per order)** | Backend | Validate coverage → resolve zone once → dispatch to the zone's assigned agent → atomic commit → emit events |

Separating planning from execution keeps the hot path simple and deterministic: at order time the system **reads configuration, it does not improvise**.

### 3.2 Dispatch flow

```
Customer location (lat/lng)
        │
        ▼
 Coverage validation (zone engine)
        │
        ├── outside → reject (no order created)
        │
        ▼
 Zone resolution (exactly one zone)
        │
        ▼
 Assigned, certified agents of that zone
        │
        ├── none eligible → reject (no order created)
        │
        ▼
 Assignment rules (zone configuration)
        │
        ▼
 Atomic commit: order + zone + agent (routing snapshot)
        │
        ▼
 Events → Customer / Agent / Admin clients
```

### 3.3 Conflict-free by construction

Conflicts are eliminated by invariants enforced at the persistence layer:

| ID | Invariant | Prevents |
|---|---|---|
| **INV-1** | **Zone exclusivity:** a coordinate resolves to at most one delivery zone; boundaries are managed to be mutually non-overlapping | Ambiguous zone resolution, double-routing |
| **INV-2** | **Assignment validity:** an order commits only if its zone has an eligible, certified, assigned agent, and an agent receives orders only from zones they are assigned to | Orders routed to unauthorized or unavailable agents |
| **INV-3** | **Atomic routing commit:** order, zone, and agent are committed together or not at all | Partial assignments, contradictory states under concurrent requests |
| **INV-4** | **Routing snapshot:** the committed order retains the zone and agent resolved at commit time | Retroactive reassignment when configuration changes later |

Because zone resolution happens **once**, on the backend, and commits **atomically**, concurrent orders, configuration edits, and client retries cannot yield two valid answers for the same order.

### 3.4 Zone and assignment lifecycle

| Operation | Handling |
|---|---|
| Assign / unassign agent | Admin configuration; takes effect for subsequent orders; existing orders keep their committed routing snapshot |
| Add zone | Configuration; validated against existing boundaries to preserve exclusivity |
| Expand zone | Configuration; validated against neighbors to preserve exclusivity |
| Split / merge | Controlled configuration operations validated against INV-1 and in-flight orders (extended in v3) |

### 3.5 Bounded capabilities

| Capability | Description | Evolution |
|---|---|---|
| **Coordinate-based zone engine** | ~30 zones; coverage check before pipeline entry | v1 |
| **Admin-managed agent–zone assignment** | One or more zones per agent; one or more agents per zone | v1 |
| **Zone lifecycle management** | Add / expand at configuration layer; split/merge as controlled operations | v1 → v3 |
| **Order pipeline** | Immediate + scheduled share one routing engine | v1 |
| **Real-time events** | State transitions to all roles | v1 |
| **Verification continuity** | Admin-controlled, server-side policy during provider incidents | v2 |
| **Emergency messaging** | Runtime system messages without deployment | v2 |
| **Floor-based pricing** | Configurable delivery floor fees | v2 |
| **Central admin configuration** | Operational parameters under one control plane | v2 |
| **Routing model extension** | Extend routing and operations under scale | **v3** |

### 3.6 Authority boundaries

| Concern | Authority | Client responsibility |
|---|---|---|
| Zone membership | Backend / persistence | Submit coordinates; display outcome |
| Agent assignment | Admin configuration, enforced by backend / persistence | Admin UI manages; others display queues |
| Dispatch decision | Backend / persistence | Display outcome |
| Price / floor fee | Backend + admin config | Display and checkout |
| OTP verification | Backend (+ continuity policy) | Capture OTP input |
| Authorization (role) | **Backend, on every privileged route** | Role-based navigation (UX only) |
| Authorization (ownership) | **Backend, resource-level checks** | None |

> Role-based access control is enforced server-side on all privileged endpoints. Client-side role routing is UX only and is not part of the trust boundary.

---

## 4. Mobile client architecture (this repository)

### 4.1 Purpose

A **multi-role operational client**: one binary, role-specific shells after authentication; consumes REST APIs and notifications; never owns dispatch truth.

### 4.2 Patterns

| Pattern | Application |
|---|---|
| **Clean Architecture** | `presentation` → `domain` → `data` per feature |
| **Repository pattern** | Domain contracts; data layer implements datasources |
| **Use cases** | Login, checkout, orders, profile, settings |
| **BLoC / Cubit** | Presentation state |
| **DI** | `get_it` (`injection_container.dart`) |
| **Either / Failure** | Functional error modeling (`dartz`) |

### 4.3 Layer responsibilities

| Layer | Contains | Allowed dependencies |
|---|---|---|
| **Presentation** | Screens, Cubits/Blocs, notifiers | Domain (+ UI kits) |
| **Domain** | Entities, repository interfaces, usecases | None on data/Flutter widgets |
| **Data** | Models, remote datasources, repository impl | Domain + `core/network` |
| **Core** | Dio helper, secure storage, routing, shared widgets, pagination helpers | Platform packages |

### 4.4 Role shells

| Role | Primary surfaces |
|---|---|
| **Customer** | Products, cart, choose location, checkout, order history, notifications |
| **Driver** | Approved / scheduled / in-delivery / completed; order detail + map |
| **Admin** | Products, drivers and assignments, users, messaging, hours, pricing, order records / finance, PDF export |

Navigation uses `go_router` with role-aware home composition and lazy tab mounting (`LazyIndexedStack`).

### 4.5 Cross-cutting concerns

| Concern | Design |
|---|---|
| **HTTP** | `DioNetworkHelper`: timeouts, auth header injection, debug redaction, short-lived GET cache |
| **Session** | Token in `flutter_secure_storage`; `SessionHelper` clears session and caches on logout |
| **Images** | `AppNetworkImage`: disk cache + memCache sizing |
| **Location / maps** | Geolocator + Google Map widgets |
| **Push** | FCM + local notifications |
| **Realtime** | Per-role notification channels; channel access follows backend authorization policy |
| **i18n** | `easy_localization` |

---

## 5. Runtime views

### 5.1 Customer order placement

1. Authenticate and establish a session
2. Select product(s) → cart
3. Select delivery location (coordinates)
4. Checkout; backend validates coverage, resolves the zone, and dispatches to the zone's assigned agent
5. Order, zone, and agent are committed atomically
6. Events notify agent and admin; customer sees status

### 5.2 Agent fulfillment

1. Agent receives their approved/scheduled queue
2. Agent progresses delivery status
3. Map/navigation assists routing
4. Completion updates the pipeline and emits events

### 5.3 Admin operational control

1. **Assign agents to zones; adjust and expand zone coverage**
2. Configure hours and delivery pricing
3. Operate continuity controls and runtime messaging during provider incidents
4. Monitor and export financial/order records

Sequence diagrams: [`DIAGRAMS.md`](./DIAGRAMS.md).

---

## 6. Integration contracts (client-relevant)

| Domain | Scope |
|---|---|
| Authentication | Login, verification, session |
| Catalog & orders | Products, checkout, order history |
| Geospatial | Zones and places |
| Agent operations | Work queues by order state |
| Operational configuration | Assignments, pricing, service hours, messaging (admin-authorized) |

**Integration rules**

- Clients send coordinates and order intents; they never assign agents.
- Only publishable, restricted keys are used client-side; server secrets remain server-side.
- Authorization is enforced server-side on every privileged route; the client is never trusted to self-declare role or ownership.

---

## 7. Deployment view

| Node | Technology | Notes |
|---|---|---|
| Mobile clients | Flutter Android (API 36), iOS-capable codebase | Package `com.gazxpress24.app` |
| API / domain services | Backend at production base URL | Zone engine + order pipeline + admin config |
| Persistence | Backend datastore | Routing invariants INV-1..4 |
| Realtime | Broadcast/WebSocket-compatible channels | Notifications |
| Maps | Google Maps Platform | Directions / distance / tiles |
| Push | Firebase Cloud Messaging | Device messaging |
| Verification | External provider + continuity policy | Operational continuity |

---

## 8. Evolution strategy

| Version | Theme | Status |
|---|---|---|
| **v1** | Zone routing, admin-managed assignment, role platforms | Validated in production (1,700+ orders, 30 zones) |
| **v2** | Control operational dependencies outside routing | Production (3,000+ users) |
| **v3** | Extend routing model; scale hardening | **Active** |

### 8.1 v3 architecture backlog

| Item | Why it is architectural |
|---|---|
| Authorization audit & fine-grained permissions | Verify role + ownership checks across privileged routes; permission-based policies; automated authorization tests |
| Pagination / cursor APIs | Protects performance under growth |
| Zone split/merge with in-flight orders | Formalizes lifecycle operations against INV-1 and routing snapshots |
| Verification policy governance | Ownership, review cadence, audit coverage |
| Key governance | Rotation and restriction policy for publishable keys |

---

## 9. Architectural decisions (ADR summary)

### ADR-001 — Persistence as dispatch authority
- **Status:** Accepted
- **Context:** UI-level assignment creates race conditions and an error surface.
- **Decision:** Enforce routing invariants (INV-1..4) at the persistence layer.
- **Alternatives considered:** Application-level locking; client-side validation only.
- **Consequences:** Clients stay thin; the backend owns conflict prevention.

### ADR-002 — Admin-managed, configuration-driven zones and assignments
- **Status:** Accepted
- **Context:** Operational geography and staffing change frequently.
- **Decision:** Zones and agent assignments are configuration managed by admins, not code.
- **Alternatives considered:** Hardcoded zone logic per region; proximity-only dispatch.
- **Consequences:** Fast expansion without redeployment; requires validation of boundaries and assignments.

### ADR-003 — Admin-controlled continuity layer
- **Status:** Accepted
- **Context:** External provider failures can halt commerce even when routing is correct.
- **Decision:** Server-side continuity policy and runtime messaging, managed by the Admin role.
- **Alternatives considered:** Secondary provider; queue-and-retry.
- **Consequences:** Higher operational resilience; policy governed under admin authorization.

### ADR-004 — Multi-role single mobile binary
- **Status:** Accepted
- **Context:** One field product serves customer, agent, and admin.
- **Decision:** Role-based shells in one Flutter app.
- **Alternatives considered:** Separate apps per role (higher maintenance and release overhead).
- **Consequences:** Shared code and one release pipeline; authorization is server-side, client role routing is UX only.

### ADR-005 — Resolve-once, snapshot-on-commit routing
- **Status:** Accepted
- **Context:** Configuration can change while orders are in flight.
- **Decision:** Resolve zone and agent once at commit and store them with the order.
- **Alternatives considered:** Re-resolving on every read (non-reproducible, can contradict history).
- **Consequences:** Reproducible, auditable routing; later configuration changes affect only new orders.

---

## 10. Client build tooling (Docker)

The Flutter client supports an **optional Docker path** for reproducible analysis, tests, and Android APK builds (`Dockerfile`, `docker-compose.yml`). This does **not** replace the backend service and does not run the mobile UI inside a container.

| Concern | Decision |
|---|---|
| Base image | `ghcr.io/cirruslabs/flutter` (Android SDK capable) |
| Secrets | `key.properties` / keystores are bind-mounted at APK build time; excluded via `.dockerignore` |
| CI relationship | GitHub Actions may keep `flutter-action`; Docker is a parallel local/CI option |

---

## 11. Compliance and assurance

| Topic | Posture |
|---|---|
| Google Play Android target | API 36 |
| Client session storage | Secure storage for bearer token |
| Logging | Auth/API headers redacted in debug network logs |
| Authorization | Enforced server-side; audit and fine-grained permissions planned for v3 |
| Formal assurance | Engineering remediation report; independent penetration testing recommended as a future milestone |
| Build secrets hygiene | Release signing materials never committed or baked into Docker images |

---

## 12. Document control

| Item | Value |
|---|---|
| Maintainers | Gaz Express engineering |
| Update triggers | Major version capabilities; trust-boundary changes; Play/SDK policy shifts |
| Diagram source of truth | [`DIAGRAMS.md`](./DIAGRAMS.md) |
| Single source of truth | Problem statement: `README.md`; architecture decisions: this document |

---

*End of Software Architecture Document.*
