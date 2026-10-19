# Gaz Express — Architecture Diagrams

| Field | Value |
|---|---|
| **Document type** | Architecture diagram pack |
| **Notation** | Mermaid |
| **Viewpoint model** | C4-inspired (Context → Containers → Components) + runtime sequences + state + deployment |
| **Companion** | [`ARCHITECTURE.md`](./ARCHITECTURE.md) 
| **Program** | Project 2025 · Gaz Express |

> Render tip: open in GitHub, VS Code, or any Mermaid-compatible viewer.

---

## Diagram index

| ID | Diagram | Viewpoint |
|---|---|---|
| D-01 | System context | C4 Level 1 |
| D-02 | Container diagram | C4 Level 2 |
| D-03 | Zone dispatch engine | Logical capability |
| D-03b | Planning vs execution (assignment model) | Runtime |
| D-03c | Conflict-prevention invariants | Logical capability |
| D-04 | Order lifecycle state | Behavioral |
| D-05 | Customer checkout sequence | Runtime |
| D-06 | Agent fulfillment sequence | Runtime |
| D-07 | Admin operational control | Runtime |
| D-08 | Mobile clean architecture | C4 Level 3 (client) |
| D-09 | Mobile role shells | Application structure |
| D-10 | Client network & session | Component |
| D-11 | Deployment topology | Physical / ops |
| D-12 | Product evolution (v1 → v3) | Roadmap |

---

## D-01 — System context (C4 Level 1)

```mermaid
flowchart TB
  subgraph Actors
    C[Customer]
    A[Certified delivery agent]
    AD[Operations admin]
  end

  GE[Gaz Express Platform<br/>zone routing · order pipeline · ops control]

  VP[External verification / SMS provider]
  MAPS[Google Maps Platform]
  FCM[Firebase Cloud Messaging]
  RT[Realtime broadcast fabric]

  C --> GE
  A --> GE
  AD --> GE

  GE --> VP
  GE --> MAPS
  GE --> FCM
  GE --> RT

  RT --> C
  RT --> A
  RT --> AD
  FCM --> C
  FCM --> A
  FCM --> AD
```

**Intent:** System boundary, human actors, and external dependencies that affect continuity.

---

## D-02 — Container diagram (C4 Level 2)

```mermaid
flowchart LR
  subgraph Clients
    MC[Flutter multi-role client<br/>Customer · Agent · Admin]
  end

  subgraph GazExpressBackend
    API[API / application services<br/>authentication · authorization]
    ZE[Zone & dispatch engine]
    OP[Order pipeline]
    CFG[Admin configuration plane<br/>zones · assignments · continuity · pricing · hours]
    DB[(Persistence<br/>routing invariants)]
    EVT[Event / notification services]
  end

  EXT_VP[Verification provider]
  EXT_MAPS[Maps APIs]
  EXT_FCM[FCM]
  EXT_RT[Realtime gateway]

  MC -->|HTTPS REST| API
  API --> ZE
  API --> OP
  API --> CFG
  ZE --> DB
  OP --> DB
  CFG --> DB
  OP --> EVT
  EVT --> EXT_RT
  EVT --> EXT_FCM
  API --> EXT_VP
  MC --> EXT_MAPS
  API --> EXT_MAPS
  EXT_RT --> MC
  EXT_FCM --> MC
```

**Intent:** Persistence is the integrity boundary for routing; the API layer is the authorization boundary.

---

## D-03 — Zone dispatch engine (logical)

```mermaid
flowchart TD
  IN[Incoming order intent<br/>coordinates · products · schedule flag] --> VAL{Inside supported<br/>coverage?}
  VAL -->|No| REJ[Reject before order creation]
  VAL -->|Yes| ZONE[Resolve exactly one delivery zone]
  ZONE --> AGENTS[Load assigned, certified agents<br/>of that zone]
  AGENTS --> ELIG{Eligible agent<br/>available?}
  ELIG -->|No| REJ2[Reject before order creation]
  ELIG -->|Yes| RULES[Apply zone assignment rules]
  RULES --> ASSIGN[Resolve responsible agent]
  ASSIGN --> PERSIST[Atomic commit<br/>order + zone + agent snapshot]
  PERSIST --> EVENTS[Emit realtime state events]
```

**Design notes**

- Zones and assignments are admin-managed configuration, not client logic.
- Zone is resolved once; the result is stored with the order.
- Same zone configuration and agent state produce the same routing outcome.

---

## D-03b — Planning vs execution (assignment model)

```mermaid
sequenceDiagram
  autonumber
  actor Adm as Admin
  participant App as Flutter admin UI
  participant API as API + config plane
  participant DB as Persistence
  actor Cust as Customer
  participant ZE as Zone engine

  Note over Adm,DB: Planning phase (admin-managed configuration)
  Adm->>App: Assign agent to one or more zones
  App->>API: Submit assignment
  API->>API: Authorize admin role
  API->>DB: Persist agent–zone assignment
  Adm->>App: Expand / adjust zone coverage
  App->>API: Submit zone configuration
  API->>API: Authorize admin role
  API->>DB: Validate boundaries (zone exclusivity) and persist

  Note over Cust,DB: Execution phase (per order)
  Cust->>API: Order with selected coordinates
  API->>ZE: Resolve zone from coordinates
  ZE->>DB: Read zone's assigned agent(s)
  ZE-->>API: Zone + responsible agent
  API->>DB: Atomic commit (order + zone + agent)
  DB-->>API: Committed
  API-->>Cust: Order confirmed
```

---

## D-03c — Conflict-prevention invariants

```mermaid
flowchart LR
  subgraph Persistence["Persistence layer (final authority)"]
    I1["INV-1 Zone exclusivity<br/>one coordinate → at most one zone"]
    I2["INV-2 Assignment validity<br/>agent only receives orders<br/>from assigned zones"]
    I3["INV-3 Atomic routing commit<br/>order + zone + agent<br/>all or nothing"]
    I4["INV-4 Routing snapshot<br/>stored with the order"]
  end

  REQ[Concurrent order requests<br/>configuration edits · client retries] --> I1
  I1 --> I2
  I2 --> I3
  I3 --> I4
  I4 --> OUT[Single, consistent,<br/>reproducible routing outcome]
```

**Intent:** Show that conflict prevention is a property of the persistence layer, independent of client behavior.

---

## D-04 — Order lifecycle (simplified state)

```mermaid
stateDiagram-v2
  [*] --> DraftIntent: Customer builds cart
  DraftIntent --> RejectedCoverage: Outside zone / no eligible agent
  DraftIntent --> Created: Checkout accepted
  Created --> Approved: Ops / pipeline approval
  Approved --> Scheduled: Scheduled delivery window
  Approved --> InDelivery: Agent progresses
  Scheduled --> InDelivery: Window reached / agent starts
  InDelivery --> Completed: Delivered
  Created --> Cancelled: Customer / policy cancel
  Approved --> Rejected: Admin reject
  Completed --> [*]
  Cancelled --> [*]
  Rejected --> [*]
  RejectedCoverage --> [*]
```

**Note:** Status vocabulary is owned by backend enums; clients map API states into role queues.

---

## D-05 — Customer checkout sequence

```mermaid
sequenceDiagram
  autonumber
  actor Cust as Customer
  participant App as Flutter client
  participant API as API services
  participant ZE as Zone engine
  participant DB as Persistence
  participant EVT as Event layer
  participant Agent as Agent client

  Cust->>App: Select products + location
  App->>API: Checkout (coordinates, cart, schedule)
  API->>API: Authenticate session + authorize role
  API->>ZE: Validate coverage + resolve zone
  alt Outside coverage or no eligible agent
    ZE-->>API: Reject
    API-->>App: Error (no order created)
  else Zone resolved with eligible agent
    API->>DB: Atomic commit (order + zone + agent)
    alt Invariant violation
      DB-->>API: Rejected
      API-->>App: Retryable error (no partial order)
    else Committed
      DB-->>API: Order persisted
      API->>EVT: Order created / assigned
      EVT-->>Agent: Realtime / push notification
      API-->>App: Success + order reference
    end
  end
```

---

## D-06 — Agent fulfillment sequence

```mermaid
sequenceDiagram
  autonumber
  actor Ag as Agent
  participant App as Flutter agent UI
  participant API as API services
  participant DB as Persistence
  participant EVT as Event layer
  participant Cust as Customer client
  participant Maps as Google Maps

  Ag->>App: Open approved / scheduled queue
  App->>API: Request role order lists
  API->>API: Authorize role + resource ownership
  API->>DB: Query orders assigned to this agent
  DB-->>API: Order set
  API-->>App: Queue payload
  Ag->>App: Open order detail / navigate
  App->>Maps: Directions / map render
  Ag->>App: Update delivery status
  App->>API: Submit status transition
  API->>API: Authorize role + ownership
  API->>DB: Persist transition
  API->>EVT: Broadcast status change
  EVT-->>Cust: Notification / status update
```

---

## D-07 — Admin operational control

```mermaid
sequenceDiagram
  autonumber
  actor Adm as Admin
  participant App as Flutter admin UI
  participant API as API + config plane
  participant DB as Persistence
  participant Cust as Customer clients

  Note over Adm,Cust: Operational control outside pure routing

  Adm->>App: Update delivery pricing / service hours
  App->>API: Submit configuration change
  API->>API: Authorize admin role
  API->>DB: Persist configuration
  API-->>App: OK

  Adm->>App: Publish emergency system message
  App->>API: Submit message
  API->>API: Authorize admin role
  API->>DB: Persist message
  API-->>Cust: Surface on next session / push

  Note over Adm,API: Provider incident
  Adm->>App: Activate continuity policy
  App->>API: Submit policy change
  API->>API: Authorize admin role
  API->>DB: Persist policy
  API-->>App: Pipeline remains available
```

---

## D-08 — Mobile clean architecture (C4 Level 3)

```mermaid
flowchart TB
  subgraph Presentation
    UI[Screens · Widgets]
    CUB[Cubits / Blocs / Notifiers]
  end

  subgraph Domain
    UC[Use cases]
    ENT[Entities]
    REPO_I[Repository interfaces]
  end

  subgraph Data
    REPO_X[Repository implementations]
    DS[Remote datasources]
    MOD[DTOs / models]
  end

  subgraph Core
    DIO[DioNetworkHelper]
    SEC[StorageHandler<br/>secure token]
    DI[get_it DI]
    RW[Shared widgets · router]
  end

  UI --> CUB
  CUB --> UC
  UC --> REPO_I
  REPO_X -.implements.-> REPO_I
  REPO_X --> DS
  DS --> DIO
  CUB --> DI
  UC --> DI
  DIO --> SEC
```

**Dependency rule:** Presentation → Domain ← Data; Core is shared infrastructure.

---

## D-09 — Mobile role shells

```mermaid
flowchart TB
  SPLASH[Splash / session restore] --> AUTH{Authenticated?}
  AUTH -->|No| LOGIN[Login / Signup / Verification]
  AUTH -->|Yes| ROLE{Role}
  LOGIN --> ROLE

  ROLE -->|Customer| CH[Customer shell<br/>products · cart · orders · profile]
  ROLE -->|Delivery| DH[Agent shell<br/>queues · map · status]
  ROLE -->|Admin| AH[Admin shell<br/>LazyIndexedStack tabs]

  AH --> P[Products]
  AH --> D[Drivers and zone assignments]
  AH --> U[Users]
  AH --> R[Order records / finance]
  AH --> S[Settings · hours · pricing · messages]
```

**Note:** Role routing on the client is UX only. Authorization is enforced by the backend on every privileged request.

---

## D-10 — Client network & session cross-cutting

```mermaid
flowchart LR
  CUB[Cubit / Usecase] --> REPO[Repository]
  REPO --> DS[Remote datasource]
  DS --> DIO[DioNetworkHelper]

  DIO --> CACHE{GET cache hit<br/>short TTL?}
  CACHE -->|Yes| RES[Cached AppResponse]
  CACHE -->|No| NET[HTTPS API]
  NET --> STORE[Write short-lived cache]
  STORE --> RES

  DIO --> HDR[Authorization Bearer<br/>+ API headers]
  HDR --> SEC[(flutter_secure_storage)]

  LOGOUT[SessionHelper.logout] --> CLR[Clear secure session]
  LOGOUT --> CC[Clear GET cache]
  LOGOUT --> DC[Invalidate driver list cache]
```

---

## D-11 — Deployment topology

```mermaid
flowchart TB
  subgraph Devices
    AND[Android clients<br/>targetSdk 36]
    IOS[iOS clients<br/>shared Flutter codebase]
  end

  subgraph CloudEdge
    API[API host]
    CDN[Media / static assets]
  end

  subgraph DataPlane
    DB[(Primary datastore)]
    RT[Realtime gateway]
  end

  subgraph SaaS
    FCM[FCM]
    MAPS[Google Maps]
    VP[Verification / SMS provider]
  end

  AND --> API
  IOS --> API
  AND --> MAPS
  IOS --> MAPS
  AND --> FCM
  IOS --> FCM
  API --> DB
  API --> RT
  API --> VP
  API --> MAPS
  RT --> AND
  RT --> IOS
  API --> CDN
```

> **Build tooling (not a runtime container):** the Flutter client may be **analyzed, tested, and APK-built** via Docker (`Dockerfile` / `docker-compose.yml`). That path does not appear above because it is a developer/CI concern, not a production deployment node. See [`DOCKER.md`](./DOCKER.md).

---

## D-12 — Product evolution (v1 → v3)

```mermaid
timeline
  title Gaz Express platform evolution
  section v1 — Routing model
    Zone engine : Geospatial coverage validation, single zone resolution
    Agent assignment : Admin-managed, one or more zones per agent
    Order pipeline : Immediate + scheduled, atomic routing commit
    Role platforms : Customer · Agent · Admin
  section v2 — Operational control
    Verification continuity : Provider independence
    Emergency messaging : Runtime communications
    Floor pricing : Configurable delivery effort
    Admin config plane : No-deploy field adaptation
  section v3 — Active
    Routing extensions : Deeper dispatch model, split/merge hardening
    Scale hardening : Pagination · authorization audit
    Client excellence : Performance · Play API 36
```

---

## Quality checklist for diagram consumers

| Check | Expected |
|---|---|
| Who defines agent–zone assignment? | Admin, through configuration |
| Where is dispatch truth? | Persistence / zone engine, not the Flutter UI |
| Can two zones claim one coordinate? | No (INV-1) |
| Can an order be partially assigned? | No (INV-3) |
| Does a config change alter existing orders? | No (INV-4) |
| Where is authorization enforced? | Backend API layer (role + ownership) |
| What is this repo? | Multi-role Flutter container, not the zone engine |

---

## Document control

| Item | Value |
|---|---|
| Source of truth for visuals | This file |
| Narrative architecture | [`ARCHITECTURE.md`](./ARCHITECTURE.md) |
| Update triggers | New containers, trust-boundary changes, v3 routing extensions |

---

*End of diagram pack.*
