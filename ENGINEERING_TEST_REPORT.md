# Gaz Express Mobile — Engineering Test Report

| Field | Value |
|---|---|
| **Document type** | Software Test Report / Test Architecture Description |
| **Revision** | 1.1 |
| **Standard alignment** | ISO/IEC/IEEE 29119 (software testing) · ISTQB test levels · ISO/IEC 25010 quality attributes |
| **Program** | Project 2025 · Gaz Express |
| **Product line** | v2 (production) · v3 (active engineering) |
| **System under test (SUT)** | Flutter multi-role client `gas_app` (`com.gazxpress24.app`) |
| **Client version** | `2.0.10+26` |
| **Report date** | 19 October 2026 |
| **Classification** | Internal / engineering |
| **Companion docs** | [`TESTING.md`](./TESTING.md) · [`ARCHITECTURE.md`](./ARCHITECTURE.md) · [`ENGINEERING_REMEDIATION_REPORT.md`](./ENGINEERING_REMEDIATION_REPORT.md) · [`DOCKER.md`](./DOCKER.md) |
 
---

## 1. Executive summary

Gaz Express is verified through two complementary layers:

1. **Automated test foundation:** Cubit state machines, repository failure modes, widget presentation against mocked Cubits, a visual golden baseline, and a customer journey path. All run without live network, Firebase, or device storage.
2. **Multi-scenario functional testing on the running application:** end-to-end scenarios executed on the app across different delivery zones, multiple certified agents, financial management, order status management, notifications, and registration and verification flows (§7).

| Metric | Result |
|---|---|
| Automated tests executed (`flutter test`) | **37 passed, 0 failed** |
| Of which on production code | 32 (the remaining 5 are a reference pattern for event-driven Bloc with debounce) |
| Functional scenarios on the running app | Executed across 6 functional areas (§7) |
| CI pipeline | GitHub Actions (`.github/workflows/ci.yml`) |
| Mocking strategy | `mocktail`, no real I/O in automated tests |
| State-test framework | `bloc_test` |
| Golden framework | `golden_toolkit` |

**Engineering judgment:** The automated suite establishes a maintainable, standards-aligned foundation for a Cubit-first Flutter application. Functional scenario testing on the running application confirms the core operational flows end to end. The expansion plan (§11) scales automated coverage toward the same flows validated functionally.

---

## 2. Scope and objectives

### 2.1 In scope

| Objective | Status |
|---|---|
| Test toolchain: `bloc_test`, `mocktail`, golden tooling, `integration_test` | Done |
| Shared helpers (`pumpApp`, mocks, fixtures); `test/` mirrors `lib/` | Done |
| Cubit tests (success / failure / verify / Equatable / seed) | Done |
| Event-driven Bloc and debounce pattern (`wait:`) | Done |
| Widget tests with `MockCubit` + `whenListen` + retry verification | Done |
| Repository tests (success / offline / server failure / JSON) | Done |
| Golden baseline for a visual surface | Done |
| Customer journey (widget + integration artifact) | Done |
| CI: format → analyze → test → coverage upload → threshold | Done |
| Multi-scenario functional testing on the running app | Done (§7) |
| Operator documentation (`TESTING.md`) and testing rules | Done |

### 2.2 Planned for next phases

| Item | Plan |
|---|---|
| Expanded Cubit coverage across remaining features | Incremental, by risk priority (§11) |
| Automated E2E against a staging environment | Staging flavor with dedicated test accounts |
| Performance and load testing | Separate discipline; measurement baseline in remediation QUAL-004 |
| Penetration / security testing | Separate engagement; see remediation report |

### 2.3 System under test characteristics

| Characteristic | Observation |
|---|---|
| Architecture | Feature-first Clean Architecture |
| Presentation state | Cubit-first (`flutter_bloc`) |
| DI | `get_it` |
| Equality | `Equatable` on states and entities |
| Networking | Dio via `DioNetworkHelper` |
| Routing | `go_router` |
| Localization | `easy_localization` |

---

## 3. Test strategy

### 3.1 Quality attributes addressed (ISO/IEC 25010)

| Attribute | How it is addressed |
|---|---|
| **Functional suitability** | Cubit success/failure paths; repository mapping; end-to-end scenarios across all roles |
| **Reliability** | Offline / server-failure branches; debounce coalescing; scenario runs of the full order lifecycle |
| **Maintainability** | Mirrored test tree, shared helpers, CI format/analyze gates |
| **Security** | No live credentials in automated tests; registration and verification flows exercised functionally |
| **Usability** | Widget state rendering, golden baseline, role-based flows exercised on the app |

### 3.2 Test levels

```
                 /\
                /  \     Functional scenarios on the running app + integration journey
               /----\
              /      \   Widget + golden (MockCubit, golden_toolkit)
             /--------\
            /          \ Cubit / Bloc + repository / model (majority of automated tests)
           /------------\
```

| Level | Tools | Isolation |
|---|---|---|
| Unit (Cubit/Bloc) | `bloc_test`, `mocktail` | Usecases/repositories mocked |
| Unit (Repository/Model) | `flutter_test`, `mocktail` | Datasource + network info mocked |
| Widget | `flutter_test`, `whenListen` | Cubit mocked |
| Golden | `golden_toolkit` | Deterministic surface |
| Journey | `integration_test` + widget twin | No Firebase bootstrap |
| Functional scenarios | Running application, role-based accounts | End to end |

### 3.3 Guiding principles

1. Test observable behavior (emitted states and UI bindings), not private implementation.
2. No real network or storage in automated unit/widget tests.
3. One behavior per test; one `group()` per method or event.
4. Every Cubit method that calls a usecase has success and failure tests.
5. Cubits are tested without widgets; widgets with `MockCubit`.
6. `test/` mirrors production paths.

---

## 4. Environment and toolchain

| Component | Version / choice |
|---|---|
| Flutter / Dart | Project SDK `>=3.3.0 <4.0.0` |
| `bloc_test` | ^9.1.7 |
| `mocktail` | ^1.0.4 |
| `golden_toolkit` | ^0.15.0 |
| `stream_transform` | ^2.1.0 |
| `integration_test` | Flutter SDK |
| `flutter_lints` | ^3.0.0 |
| CI | `ubuntu-latest` · `subosito/flutter-action@v2` · cache enabled |

### 4.1 Tooling decisions

| Decision | Rationale |
|---|---|
| `mocktail` | Null-safe, lightweight, standard pairing with `bloc_test` |
| `golden_toolkit` | Stable golden workflow compatible with the current Flutter SDK |
| Reference `DebouncedSearchBloc` in `test/` | Documents the event→state and debounce testing pattern for future Blocs |
| Widget twin for the customer journey | Journey logic runs headlessly in CI via `flutter test`; the `integration_test` artifact is available for device runs |

---

## 5. Test architecture

```
test/
  helpers/                        # pumpApp, mocks, widget harnesses, reference Bloc
  core/                           # mirrors lib/core
  feature/                        # mirrors lib/feature
  golden/
  flutter_test_config.dart        # golden font loading
  widget_test.dart                # smoke
integration_test/
  customer_journey_test.dart
.github/workflows/ci.yml
.cursor/rules/flutter-testing.mdc
TESTING.md
dart_test.yaml
```

| Helper | Responsibility |
|---|---|
| `pumpApp` | Wraps the SUT in `MaterialApp` + optional `MultiBlocProvider`, localizations, theme |
| `mocks.dart` | `MockCubit` implementations, usecase/repository/datasource mocks, fallback values |
| `NotificationListView` | Loading / list / error+retry harness for widget tests |
| `flutter_test_config.dart` | Font loading for deterministic goldens |

---

## 6. Automated test inventory and results

### 6.1 Execution record

| Run | Command | Result |
|---|---|---|
| Unit + widget + golden | `flutter test` | **37 passed**, 0 failed |
| Analyze (test gate) | `flutter analyze --no-fatal-infos test integration_test lib/core/widgets/app_network_image.dart` | No errors |
| Customer journey (headless) | `flutter test test/feature/customer` | Passed (included in the 37) |
| Journey on device | `flutter test integration_test -d <deviceId>` | Artifact ready for device/emulator runs |

### 6.2 Suite breakdown

| Suite ID | Area | Type | Outcome |
|---|---|---|---|
| T-NAV-001 | `BottomNavigatorBarCubit` | Cubit | Pass |
| T-AUTH-REPO-001 | `LoginRepositoriesImpl` + `LoginModel` | Repository / model | Pass |
| T-DRV-001 | `GetAllDriverCubit` (cache and force-refresh paths) | Cubit | Pass |
| T-ORD-CAN-001 | `CancelOrderCubit` | Cubit | Pass |
| T-ORD-REC-001 | `OrderRecordsCubit` (+ debounce) | Cubit | Pass |
| T-NOTIF-MODEL-001 | `NotificationModel` | Model | Pass |
| T-NOTIF-CUBIT-001 | `NotificationCubit` | Cubit | Pass |
| T-NOTIF-UI-001 | `NotificationListView` + MockCubit | Widget | Pass |
| T-GOLD-001 | Brand surface golden | Golden | Pass |
| T-BLOC-EX-001 | Event-driven Bloc + debounce (reference pattern) | Bloc | Pass |
| T-SMOKE-001 | Application shell smoke | Widget | Pass |
| T-JOURNEY-001 | Customer browse → cart → confirm | Journey | Pass |

### 6.3 Behavioral coverage matrix

| Behavior | Success | Failure | Verify call | Additional |
|---|---|---|---|---|
| Load notifications | Yes | Offline failure | Yes | Equatable, seeded multi-step |
| Cancel order | Yes | Server failure | Yes | — |
| Order records fetch | Yes | Server failure | Yes | Debounce (`wait:`) |
| Get all drivers | Yes | Offline failure | Yes | Cache hit without second call |
| Login repository | Right(model) | Offline + server exception | Yes | `fromJson` / `toJson` |
| Notification UI | List | Error + retry | Retry triggers reload | `whenListen` |

---

## 7. Functional scenario testing on the running application

Scenario testing was performed on the running application with role-based accounts (Customer, Delivery agent, Admin), covering the end-to-end flows below.

### 7.1 Coverage by functional area

| Area | What was exercised |
|---|---|
| **Zone-based ordering** | Orders placed from locations across different delivery zones; coverage validation for in-zone and out-of-zone locations; zone resolution from selected coordinates |
| **Agent assignment and dispatch** | Admin assignment of agents to one or more zones; routing of each order to the agent assigned to the customer's zone; multiple agents across multiple zones |
| **Financial management** | Pricing and delivery floor fees; order totals; admin order records and financial reporting with filters; report export |
| **Order status management** | Full lifecycle: creation, approval, scheduled and immediate orders, in-delivery progress, completion, cancellation and rejection; role-specific order queues |
| **Notifications** | Order events delivered to customer, agent, and admin; notification list, error and retry behavior |
| **Registration and verification** | Account registration, login, verification flow, session start and logout |

### 7.2 Representative scenarios

| ID | Scenario | Expected result | Outcome |
|---|---|---|---|
| S-ZONE-01 | Customer orders from a location inside zone A | Order accepted; zone A resolved; routed to the agent assigned to zone A | Validated |
| S-ZONE-02 | Customer orders from a location in a different zone B | Order routed to the agent assigned to zone B | Validated |
| S-ZONE-03 | Customer selects a location outside supported coverage | Request rejected before an order is created | Validated |
| S-ASSIGN-01 | Admin assigns an agent to an additional zone | New orders from that zone become eligible for that agent | Validated |
| S-ASSIGN-02 | Admin changes or expands zone assignments | Changes apply to subsequent orders without app redeployment | Validated |
| S-ORD-01 | Immediate order end to end | Created → approved → in delivery → completed; each role sees the correct state | Validated |
| S-ORD-02 | Scheduled order end to end | Scheduled queue for the agent; progresses to completion | Validated |
| S-ORD-03 | Order cancellation and admin rejection | Status updates reflected across roles | Validated |
| S-FIN-01 | Order with floor-based delivery fee | Fee and total computed per configured pricing | Validated |
| S-FIN-02 | Admin financial records with filters | Records filtered correctly; export produced | Validated |
| S-NOTIF-01 | Order created and status changes | Notifications received by the relevant roles | Validated |
| S-AUTH-01 | New user registration and verification | Account created; verification completes; session established | Validated |
| S-AUTH-02 | Login and logout | Session starts; logout clears session state | Validated |

---

## 8. Test design decisions

| ID | Decision | Outcome |
|---|---|---|
| TD-001 | Adopt the standard Flutter/bloc toolchain, shared helpers, mirrored tree, CI, and testing rules | 37 passing automated tests; documented operator guide |
| TD-002 | Keep production Cubit-only; keep a reference event-driven Bloc with debounce under `test/helpers/` | Bloc and debounce testing patterns available for future Blocs |
| TD-003 | Use `golden_toolkit` for visual baselines | Golden baseline under `test/golden/` |
| TD-004 | Isolate the smoke and journey tests from Firebase bootstrap | Suite runs headless without credentials |
| TD-005 | Provide a headless widget twin alongside the `integration_test` journey | Journey logic always verifiable in CI |
| TD-006 | Align expectations with actual Cubit emission order, with no product logic change | Stable, deterministic assertions |

---

## 9. Continuous integration

| Stage | Action | Gate |
|---|---|---|
| 1 | Checkout; Flutter setup with cache | — |
| 2 | `flutter pub get` | Resolve must succeed |
| 3 | `dart format --set-exit-if-changed test integration_test` | Formatting consistent |
| 4 | `flutter analyze --no-fatal-infos` (test tree + `AppNetworkImage`) | No errors |
| 5 | `flutter test --coverage` | All tests pass |
| 6 | Upload `coverage/lcov.info` | Artifact |
| 7 | Line-coverage threshold ≥ 5% | Ratchet upward as suites grow |

CI gates the test stack first so that quality gates tighten incrementally. Coverage targets follow the roadmap in §11.

---

## 10. Traceability to roles and architecture

| Role / capability | Confidence sources |
|---|---|
| Customer registration and verification | Login repository + model tests; scenarios S-AUTH-01/02 |
| Customer ordering across zones | Journey tests; scenarios S-ZONE-01..03 |
| Customer order cancel | `CancelOrderCubit`; S-ORD-03 |
| Notifications | Cubit + widget tests; S-NOTIF-01 |
| Agent dispatch and fulfillment | S-ASSIGN-01/02, S-ORD-01/02 |
| Admin drivers and assignments | `GetAllDriverCubit`; S-ASSIGN-01/02 |
| Admin financial records | `OrderRecordsCubit` debounce; S-FIN-01/02 |
| Shared navigation | `BottomNavigatorBarCubit` |
| Zone engine and persistence invariants | Backend-owned; validated functionally through zone and dispatch scenarios (§7) |

---

## 11. Roadmap and priorities

| Priority | Phase | Action | Owner |
|---|---|---|---|
| **P0** | Phase 1 | Automated Cubit tests for checkout, order status, send-OTP, and session logout, mirroring the validated scenarios | Mobile |
| **P0** | Phase 1 | Injectable seam for the push-token dependency in the login path, enabling `LoginCubit` tests | Mobile |
| **P1** | Phase 2 | Raise the CI coverage gate: 20% → 40% → 60% on critical domains | Mobile / DevOps |
| **P1** | Phase 2 | Add CI steps: release build, `lib/` analyze baseline, dependency and secret scanning | DevOps |
| **P1** | Phase 2 | Contract tests for checkout and order DTOs against staging fixtures | Mobile + Backend |
| **P1** | Phase 2 | Android emulator job for `integration_test/` | DevOps |
| **P2** | Phase 3 | Backend dispatch concurrency test report (parallel checkouts, assignment changes) | Backend |
| **P2** | Phase 3 | Golden tests for notification error/retry and admin filter surfaces; generate goldens in the CI environment | Mobile |
| **P2** | Phase 3 | Performance measurement baseline (remediation QUAL-004) | Mobile |

---

## 12. Commands (operator quick reference)

```bash
flutter test
flutter test --coverage
flutter test --update-goldens
flutter test test/feature/customer
flutter test integration_test -d <deviceId>
dart format --set-exit-if-changed test integration_test
flutter analyze --no-fatal-infos test integration_test lib/core/widgets/app_network_image.dart
```

Full operator guide: [`TESTING.md`](./TESTING.md).

---

## 13. Conclusion

The Gaz Express client is verified at two levels: a standards-aligned automated foundation (`bloc_test`, `mocktail`, `MockCubit`, `golden_toolkit`, journey tests, CI gates) and multi-scenario functional testing on the running application covering zones, agent assignment, financial management, order status, notifications, and registration and verification.

The roadmap converts the validated scenarios into automated regression coverage, in priority order, so quality is protected as v3 evolves.

---

## 14. Document control

| Item | Value |
|---|---|
| Revision history | 1.0 (19 Oct 2026): initial. 1.1: functional scenario testing section added; test inventory restated against production code; phased roadmap with owners |
| Prepared as | Engineering test report |
| Not a substitute for | Formal certification, penetration testing, or load/performance reports |
| Re-validation triggers | New Cubit methods; CI failure; Flutter major upgrade; golden baseline changes; changes to zone, assignment, or order-status logic |

---
*End of Engineering Test Report.*
