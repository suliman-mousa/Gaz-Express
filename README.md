# Gaz Express

**Full-stack logistics platform for residential gas cylinder delivery**  
Coordinate-based zone routing · Admin-managed agent assignment · Conflict-free dispatch

| | |
|---|---|
| **Program** | Project 2025 · Gaz Express |
| **Current product line** | v2 (production) · **v3 in active planning / delivery** |
| **This repository** | Flutter multi-role mobile client (`gas_app`) |
| **Android application ID** | `com.gazxpress24.app` |
| **Client version** | `2.0.10+26` |


> **Scope note.** This document describes the Gaz Express platform for context.
> This repository contains the **mobile client only**. The backend (zone engine,
> order pipeline, persistence, admin configuration) is maintained in a separate
> service. Architecture details live in [`ARCHITECTURE.md`](./ARCHITECTURE.md).

---

## 1. Problem statement

The delivery problem was simple. The routing problem was not.

Every order must reach the **certified agent responsible for the customer's delivery zone**: not by proximity guesswork, not through manual reassignment after the fact. Dispatch is **rule-based, reproducible, and decided at order time**.

That requirement shaped every architectural decision: geospatial validation before order creation, admin-managed agent–zone assignment, persistence-layer invariants for routing consistency, and role-specific experiences for customer, field agent, and operations.

---

## 2. Geographic distribution model

Gaz Express separates **planning** (done in advance by operations) from **execution** (done automatically per order).

### 2.1 Planning phase (admin-managed)

- Operations define **delivery zones** as geospatial boundaries.
- An admin **assigns each certified agent to one or more zones**. A zone may be served by one or more agents.
- Admins **adjust assignments and expand or reconfigure zone coverage** at any time through configuration, without changes to routing code.

### 2.2 Execution phase (per order)

1. The customer selects a delivery location; the client submits its coordinates.
2. The backend validates the coordinates against supported coverage. Out-of-coverage requests are **rejected before an order exists**.
3. The backend **resolves the delivery zone once** from the coordinates.
4. The order is dispatched to the agent assigned to that zone under the zone's configured assignment rules.
5. Zone, agent, and routing facts are **committed atomically** with the order, forming a routing record that does not change retroactively when configuration changes later.

### 2.3 Conflict-free by construction

Dispatch conflicts are prevented by three invariants enforced at the persistence layer, not by client behavior:

| Invariant | Guarantee |
|---|---|
| **Zone exclusivity** | A coordinate resolves to at most one delivery zone; zone boundaries are managed so they do not overlap. |
| **Assignment validity** | An order is accepted only for a zone that has an eligible, certified, assigned agent; an agent can only receive orders from zones they are assigned to. |
| **Atomic routing commit** | The order and its routing facts are committed together or not at all; concurrent requests cannot produce a partial or contradictory assignment. |

---

## 3. Core engineering constraint

> Critical dispatch invariants cannot rely on the UI or application layer alone.  
> The **persistence layer** is the final authority for routing consistency and conflict prevention.

Routing consistency is enforced through database-level constraints and transactional guarantees; clients and admin tooling consume and display that authority.

---

## 4. Product metrics (operational baseline)

| Metric | Value |
|---|---|
| Registered users | 4,000+ |
| Orders processed | 2,500+ |
| Active delivery zones | 30 |
| Engineering team (launch) | 4 engineers: system design, architecture, production launch |

---

## 5. Platform evolution

### 5.1 v1 — Routing model (validated in production)

| Capability | Description |
|---|---|
| **Coordinate-based zone engine** | 30 delivery zones defined by geospatial boundaries. Orders are validated against coverage before entering the pipeline; out-of-boundary requests are rejected before order creation. |
| **Admin-managed agent–zone assignment** | Admins assign agents to one or more zones; a zone may have one or more agents. Dispatch resolves the responsible agent from the zone's assignments at request time, never from hardcoded values. |
| **Zone lifecycle** | Zones are added and expanded at the configuration layer without modifying routing logic. Split and merge are controlled configuration operations (extended further in v3). |
| **Order pipeline** | Immediate and scheduled orders share one routing engine; routing consistency is enforced at persistence time. |
| **Real-time event layer** | Order state transitions emit events to customer, agent, and admin surfaces. |
| **Agent platform** | Map-integrated field environment: navigation, real-time order notifications, billing context. |
| **Admin platform** | Operational control: products, agents, assignments, working hours, zones, customer messaging, financial reporting. |

### 5.2 v2 — Operational control (production)

v1 validated the routing model. v2 addressed risks **outside** the routing core.

**Principle:** *If we cannot control it, we build a layer that does.*

| Change | Intent |
|---|---|
| **Verification continuity** | External verification providers can fail. An admin-controlled, server-side continuity policy keeps the order pipeline operating during provider incidents. |
| **Emergency messaging layer** | Admins publish notices and system-wide notifications in real time, without redeployment. |
| **Dynamic floor-based pricing** | Delivery effort differs by floor level; pricing is configurable from the admin panel. |
| **Centralized admin configuration** | Continuity policy, messaging, pricing, and order constraints live under one admin-controlled configuration layer. |

### 5.3 v3 — Current focus

**Status: in planning / active engineering**

- Extend the routing and operational model under scale
- Authorization audit and fine-grained permissions
- Pagination for heavy lists
- Client performance, session hygiene, Google Play / Android API 36 compliance

Remediation history: [`ENGINEERING_REMEDIATION_REPORT.md`](./ENGINEERING_REMEDIATION_REPORT.md).

---

## 6. System architecture (logical)

```
                    ┌─────────────────────────┐
                    │   Admin configuration   │
                    │ zones · assignments     │
                    │ continuity · pricing    │
                    └───────────┬─────────────┘
                                │
┌──────────┐    order     ┌─────▼──────┐    resolve    ┌──────────────┐
│ Customer │─────────────►│ Order +    │──────────────►│ Zone engine  │
│  client  │  + location  │ pipeline   │  zone/agent   │ + assignments│
└──────────┘              └─────┬──────┘               └──────────────┘
                                │
                     persistence invariants
                                │
              ┌─────────────────┼─────────────────┐
              ▼                 ▼                 ▼
        ┌──────────┐     ┌──────────┐      ┌──────────┐
        │ Customer │     │  Agent   │      │  Admin   │
        │  events  │     │  events  │      │  events  │
        └──────────┘     └──────────┘      └──────────┘
```

| Layer | Responsibility |
|---|---|
| **Geospatial / zone engine** | Coverage validation; single zone resolution; agent eligibility |
| **Order pipeline** | Immediate + scheduled orders; atomic routing commit |
| **Persistence** | Final authority for routing invariants and conflict prevention |
| **Real-time events** | State propagation to all roles |
| **Clients** | Role-specific UX; never the authority for dispatch rules |

---

## 7. This repository — mobile client

| Role | Responsibilities |
|---|---|
| **Customer** | Authentication, catalog, cart, location selection, checkout (immediate/scheduled), order history, notifications |
| **Delivery agent** | Approved / scheduled / in-delivery / completed orders, map navigation, order detail and billing context, profile |
| **Admin** | Products, agents and zone assignments, users, messaging, working hours, delivery pricing, order records and financial reporting, PDF export |

### 7.1 Client architecture

```
lib/
├── core/                 # network, storage, DI, shared widgets, routing helpers
├── feature/
│   ├── auth/             # login, signup, verification, profile
│   ├── home/             # customer / driver / admin domains
│   ├── notification/
│   ├── setting/
│   └── splash/
├── injection_container.dart
└── main.dart
```

```
feature/<name>/
├── data/          # models, remote datasource, repository impl
├── domain/        # entities, repository contracts, usecases
└── presentation/  # screens, cubits/notifiers, UI state
```

### 7.2 Technology stack

| Concern | Choice |
|---|---|
| Framework | Flutter (Dart SDK `>=3.3.0 <4.0.0`) |
| State management | `flutter_bloc` + notifiers |
| DI | `get_it` |
| Networking | `dio` |
| Auth session | `flutter_secure_storage` (token) |
| Maps / location | Google Maps, `geolocator`, `geocoding`, `location` |
| Push | Firebase Messaging + local notifications |
| Localization | `easy_localization` |
| Images | `cached_network_image` / `AppNetworkImage` |
| Reporting | PDF generation / printing |

### 7.3 API surface (client)

| Domain | Scope |
|---|---|
| Authentication | Login, verification, session |
| Catalog & orders | Products, checkout, order history |
| Geospatial | Zones and places |
| Agent operations | Work queues by order state |
| Operational configuration | Assignments, pricing, service hours, messaging (admin-authorized) |

The full API reference is maintained with the backend service.

---

## 8. Engineering standards applied on the client

Informed by ISO/IEC 25010 quality dimensions and OWASP MASVS session/secrets hygiene:

| Area | Practice |
|---|---|
| **Security** | Bearer token in secure storage; session and GET-cache cleared on logout; redacted debug headers; authorization enforced server-side (client role routing is UX only) |
| **Performance** | Display-sized image decode cache; short-lived GET cache; lazy tab mounting; admin report optimizations; lighter map type |
| **Compliance** | Android `targetSdk` / `compileSdk` **36** |
| **Maintainability** | Feature-first clean architecture; shared network/storage abstractions |

---

## 9. Getting started

### Prerequisites

- Flutter SDK compatible with Dart `>=3.3.0 <4.0.0`
- Android SDK with API 36 platform components
- Valid `android/key.properties` for release signing (not committed)
- Firebase / Google Maps configuration for your environment
- **Optional (Docker path):** Docker Engine / Docker Desktop with ≥ 8 GB RAM (see [`DOCKER.md`](./DOCKER.md))

### Commands

```bash
flutter pub get
flutter run
flutter analyze
flutter build appbundle --release
```

Release output: `build/app/outputs/bundle/release/`.

### Docker (analyze / test / APK)

Use Docker when you want a reproducible toolchain without installing the full Android SDK locally. Full requirements and signing notes: [`DOCKER.md`](./DOCKER.md).

```bash
docker compose build
docker compose run --rm analyze
docker compose run --rm test
# Release APK — requires host android/key.properties + keystore (mounted, never baked into the image)
docker compose run --rm apk
```

APK output: `build/app/outputs/flutter-apk/app-release.apk`.

---

## 10. Configuration notes

| Item | Guidance |
|---|---|
| API base URL | `ApisWords.BASEURL` in `lib/core/strings/apis.dart` |
| Keys & secrets | Only publishable, restricted keys are used client-side. Server secrets remain server-side. Keystores and `key.properties` are never committed. |
| Verification continuity | Admin-controlled, server-side policy (see ADR-003 in `ARCHITECTURE.md`) |

---

## 11. Related documentation

| Document | Purpose |
|---|---|
| [`ARCHITECTURE.md`](./ARCHITECTURE.md) | Software Architecture Document (views, quality attributes, ADRs) |
| [`DIAGRAMS.md`](./DIAGRAMS.md) | Mermaid diagram pack |
| [`ENGINEERING_REMEDIATION_REPORT.md`](./ENGINEERING_REMEDIATION_REPORT.md) | Problem → Remediation → Result register |
| [`ENGINEERING_TEST_REPORT.md`](./ENGINEERING_TEST_REPORT.md) | Test strategy, inventory, results, backlog |


---

## 12. Roadmap snapshot

| Version | Focus | Status |
|---|---|---|
| **v1** | Zone routing, admin-managed assignment, order pipeline, real-time roles | Validated in production |
| **v2** | Operational control outside routing | Production |
| **v3** | Extend routing model; authorization audit, pagination, scale hardening | **Active** |

---

## 13. License & ownership

Proprietary — Gaz Express / project stakeholders.  
Not licensed for redistribution without authorization.

---

*Gaz Express — logistics engineered around zones, agents, and operational truth.*
