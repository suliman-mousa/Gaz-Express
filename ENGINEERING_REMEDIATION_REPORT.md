# Gas Express Mobile — Engineering Remediation Report

| Field | Value |
|---|---|
| **Document type** | Engineering Remediation Report |
| **Revision** | 1.1 (supersedes 1.0 of 18 October 2026) |
| **Report date** | 19 October 2026 |
| **Standard basis** | ISO/IEC 25010 (quality model) · OWASP MASVS v2 (mobile security verification) · Google Play policy requirements · SRE operational-readiness practice |
| **Classification** | **Internal — do not publish** (contains vulnerability and risk details) |
| **Product** | Gas Express (Flutter) |
| **Package** | `com.gazxpress24.app` |
| **Version** | `2.0.10+26` |
| **Platform** | Android (API 36) |
| **Roles in scope** | Customer / Driver / Admin |
| **Finding format** | Problem → Remediation → Result → Residual risk / next step |

---

## 1. Executive summary

This report records the compliance, security, performance, and quality remediation performed on the Gas Express Flutter client after a three-role engineering audit, and the additional best-practice gaps identified in revision 1.1.

**Change policy applied during remediation**

- No intentional change to product business logic or user-facing feature behavior.
- Emergency OTP fallback retained by explicit stakeholder decision (see SEC-004).
- API key rotation deferred pending backend/cloud coordination.

**Outcome**

| Metric | Value |
|---|---|
| Weighted overall score | **7.4 / 10** (method in §3) |
| Tracked findings | 30 |
| Closed on client | 11 |
| Accepted | 1 |
| Deferred (backend / ops / scheduled) | 7 |
| Open — newly identified in rev 1.1 | 11 |

**Engineering judgment**

- **Store compliance:** the client is ready for Google Play API 36 submission.
- **Security release posture:** server-side role-based authorization is in place (SEC-005). The security release is conditional on three items: authorization ownership verification, the OTP emergency-channel controls (SEC-004), and key governance (SEC-006).
- **Operational readiness:** gaps exist in observability, automated testing, and delivery pipeline (§4.5, §4.4). They should be closed before scaling beyond the current production base.

**Status vocabulary.** "Release-ready" in this report means *store-compliance ready*. It is not a statement that every security control has been verified.

---

## 2. Scope and methodology

| Element | Description |
|---|---|
| Quality model | ISO/IEC 25010: Performance Efficiency, Reliability, Security, Maintainability, Functional Suitability |
| Security practice | OWASP MASVS v2 mapping (§7) |
| Evidence | Static code review, targeted `flutter analyze` on touched files, build remediation for API 36 |
| In scope | Flutter Android client hardening, performance/quality fixes, and client-relevant backend dependencies |
| Out of scope | Backend implementation, penetration testing, device-farm benchmarks, uncoordinated key rotation |
| Evidence limitation | Performance results are assessed by code review and design reasoning. **No before/after measurements were collected** (see QUAL-004). |

### Severity scale

| Severity | Meaning |
|---|---|
| **Critical** | Blocks store readiness or enables high-impact privilege/data exposure |
| **High** | Material risk to security, stability, data integrity, or user-perceived performance |
| **Medium** | Meaningful defect with bounded blast radius |
| **Low** | Hygiene / maintainability debt |
| **Info** | Contextual observation |

### Disposition codes

| Disposition | Meaning |
|---|---|
| **Closed** | Remediation implemented |
| **Accepted** | Risk retained by product decision, with owner and conditions |
| **Deferred** | Requires backend/ops or scheduled follow-up |
| **Open** | Newly identified; remediation recommended, not yet started |
| **Verify** | Control may exist; evidence required before closure |

---

## 3. Post-remediation scorecard

### 3.1 Method

Overall score = weighted average of domain scores. Scores are expert judgment against the evidence in §2, not instrument measurements.

| Domain | Weight | Score /10 | Weighted |
|---|---|---|---|
| Play / SDK compliance | 15% | 9.0 | 1.35 |
| Security | 30% | 7.0 | 2.10 |
| Performance efficiency | 20% | 8.0 | 1.60 |
| Quality / reliability | 25% | 7.0 | 1.75 |
| Operability (observability, delivery) | 10% | 5.5 | 0.55 |
| **Overall** | 100% | | **7.35 ≈ 7.4** |

### 3.2 Domain notes

| Domain | Status | Primary residual |
|---|---|---|
| Play / SDK compliance | Ready | Upload verification; Data Safety and account-deletion checks (OPS-003) |
| Security | Improved | Ownership verification, OTP channel controls, key governance, transport hardening |
| Performance efficiency | Improved (unmeasured) | Server pagination; measurement baseline |
| Quality / reliability | Improved | Automated tests, checkout idempotency, cache staleness policy |
| Operability | Gap | Crash reporting, CI/CD gates, dependency scanning |

### 3.3 Role view

| Role | Score /10 | Primary residual |
|---|---|---|
| Customer | 8.2 | Idempotent checkout; connectivity handling |
| Driver | 7.9 | Cache staleness on work queues; offline behavior; list scale |
| Admin | 7.6 | Pagination; fine-grained permissions; OTP-channel governance |

---

## 4. Findings register

---

### 4.1 Compliance

#### COMP-001 — Play / Platform Compliance

| | |
|---|---|
| **Severity** | Critical |
| **Disposition** | Closed |
| **Problem** | Android `targetSdk` / `compileSdk` below Google Play API level 36 requirement, blocking store submission and future updates. |
| **Remediation** | Raised `compileSdk` / `targetSdk` to 36; aligned NDK and build toolchain; resolved Kotlin incremental and dependency registrar conflicts (`intl` override, `shared_preferences` upgrade). |
| **Result** | Release pipeline builds against API 36. Version prepared as `2.0.10+26` for AAB generation. |
| **Residual risk / next step** | Final acceptance requires an internal/production upload verification on Play Console. |

---

### 4.2 Security

#### SEC-001 — Credential Storage

| | |
|---|---|
| **Severity** | High |
| **Disposition** | Closed |
| **Problem** | Auth bearer token persisted in plaintext `SharedPreferences`, exposing session credentials to local extraction on rooted/compromised devices. |
| **Remediation** | Migrated token to `flutter_secure_storage` (Android Keystore-backed); one-time migration; legacy plaintext key purged on init/set/clear. |
| **Result** | Session token stored in secure storage; plaintext leftovers removed. Synchronous token access preserved via in-memory cache. |
| **Residual risk / next step** | Device-level compromise remains inherent to mobile platforms. See SEC-009 for backup exposure. |

#### SEC-002 — Session Hygiene

| | |
|---|---|
| **Severity** | High |
| **Disposition** | Closed |
| **Problem** | Logout did not fully clear client-side network caches and session artifacts, enabling stale authenticated data retention after sign-out. |
| **Remediation** | Extended `SessionHelper` logout to clear secure session and Dio in-memory GET cache (and driver cache invalidation paths). |
| **Result** | Logout invalidates local session state and the short-lived GET cache consistently. |
| **Residual risk / next step** | Backend must enforce token revocation and TTL policy. |

#### SEC-003 — Observability / Secrets in Logs

| | |
|---|---|
| **Severity** | Medium |
| **Disposition** | Closed |
| **Problem** | Network helper debug logging could expose `Authorization` and `x-api-key` header values in developer logs. |
| **Remediation** | Redacted header views; verbose logging restricted to `kDebugMode`. |
| **Result** | Tokens and API keys are masked in debug output; release builds do not emit these logs. |
| **Residual risk / next step** | Remaining ad-hoc `print()` calls should be removed (tracked in QUAL-002). |

#### SEC-004 — OTP Emergency Channel

| | |
|---|---|
| **Severity** | **High** (escalates to **Critical** if the OTP is returned to the requesting party rather than to an authorized operator) |
| **Disposition** | Accepted (conditional) |
| **Problem** | The emergency fallback returns the OTP in the API response and renders it on verify/reset screens. If the response reaches the requesting client, anyone who can request a code for a phone number can complete verification for that number, bypassing the second factor entirely. This is a verification-strength issue, not only a screen-leakage issue. |
| **Remediation** | No code change, by explicit product decision. Retained as an operational-continuity fallback. |
| **Result** | Behavior unchanged. Accepted as residual risk **conditional on the compensating controls below**. |
| **Compensating controls required** | (1) Fallback activation restricted to the Admin role. (2) Every activation and OTP issuance audit-logged. (3) Automatic expiry of the fallback mode. (4) Rate limiting and attempt limits per phone number and device. (5) Short OTP validity and single-use codes. (6) Alerting on fallback activation. |
| **Residual risk / next step** | Preferred end state: the API never returns the OTP to the requesting client; the emergency channel delivers it only to an authorized operator or alternate channel. Risk owner: product owner — record name and review date. |

#### SEC-005 — Authorization Model (RBAC)

| | |
|---|---|
| **Severity** | Medium (was Critical in rev 1.0) |
| **Disposition** | Partially closed (backend) — ownership verification open |
| **Problem** | Role separation (customer / driver / admin) must be enforced by the server; client routing is UX only. |
| **Remediation** | **Server-side role checks are enforced by the backend on privileged endpoints.** Client role routing remains a UX concern and is not part of the trust boundary. |
| **Result** | Privileged endpoints are protected by role verification at the API layer. Rev 1.0 incorrectly recorded this as absent; corrected here. |
| **Residual risk / next step** | Role checks do not by themselves prevent horizontal access (IDOR). **Verify** resource-level ownership on every order, user, and report endpoint (a driver reads or updates only orders assigned to them; a customer only their own). Add automated authorization tests covering role and ownership. Evaluate fine-grained permissions beyond three coarse roles. |

#### SEC-006 — Embedded API Keys

| | |
|---|---|
| **Severity** | Medium |
| **Disposition** | Deferred (Ops / Backend) |
| **Problem** | The Google Maps API key and `x-api-key` are compiled into the client binary and are extractable. A static client key cannot be treated as a secret. |
| **Remediation** | Left in place pending coordinated rotation with backend/cloud owners (explicit constraint). |
| **Result** | No rotation performed; risk acknowledged and deferred. |
| **Residual risk / next step** | (1) Restrict the Maps key by package name + SHA-1 and by API. (2) Rotate keys. (3) Ensure `x-api-key` is not the sole control for any sensitive operation; authorization must rely on user-bound tokens. (4) Set usage quotas and billing alerts. |

#### SEC-007 — Transport Security

| | |
|---|---|
| **Severity** | Medium |
| **Disposition** | Open (new) |
| **Problem** | No evidence of certificate pinning or a network-security configuration that forbids cleartext traffic (OWASP MASVS-NETWORK). On hostile networks, a user-installed CA can enable interception. |
| **Remediation** | Recommended: confirm `usesCleartextTraffic=false` and a network security config; evaluate certificate/SPKI pinning with a backup pin and rotation runbook. |
| **Result** | Not yet implemented. |
| **Residual risk / next step** | Pinning adds operational risk (certificate rotation). Adopt with a documented rotation process, or document the decision not to pin. |

#### SEC-008 — Release Hardening

| | |
|---|---|
| **Severity** | Low |
| **Disposition** | Open (new) |
| **Problem** | No evidence of code obfuscation or debug-symbol separation in release builds (MASVS-RESILIENCE); client logic is easier to reverse-engineer. |
| **Remediation** | Recommended: build releases with `--obfuscate --split-debug-info`, upload symbols to the crash-reporting service, keep R8/ProGuard enabled. Evaluate root/integrity detection (Play Integrity API) for sensitive roles. |
| **Result** | Not yet implemented. |
| **Residual risk / next step** | Obfuscation raises effort for attackers; it is not a security boundary. Server-side authorization remains the control. |

#### SEC-009 — Backup and Local Data Exposure

| | |
|---|---|
| **Severity** | Medium |
| **Disposition** | Verify (new) |
| **Problem** | If `android:allowBackup` is enabled without exclusion rules, app data may be extracted through backup channels (MASVS-STORAGE). |
| **Remediation** | Recommended: set `allowBackup=false` or define `dataExtractionRules` / `fullBackupContent` excluding preferences and secure-storage files; confirm no sensitive data in app logs or cache directories. |
| **Result** | Configuration not yet evidenced. |
| **Residual risk / next step** | Record the manifest configuration as evidence. |

#### SEC-010 — Realtime Channel Authorization

| | |
|---|---|
| **Severity** | Medium |
| **Disposition** | Verify (new) |
| **Problem** | Realtime notification channels must be private and authorized server-side; public or guessable channel names could expose order data across users. |
| **Remediation** | Recommended: verify private/presence channels with server-side channel authorization bound to the authenticated user and role. |
| **Result** | Not yet evidenced. |
| **Residual risk / next step** | Add an authorization test: a user cannot subscribe to another user's channel. |

---

### 4.3 Performance

> **Evidence note.** Results below describe design-level effects. No before/after measurements exist yet (QUAL-004).

#### PERF-001 — Image Decode / Memory

| | |
|---|---|
| **Severity** | High |
| **Disposition** | Closed |
| **Problem** | Network images decoded at full resolution in lists/cards, increasing RAM/GPU pressure and scroll jank. |
| **Remediation** | `AppNetworkImage` derives `memCacheWidth` / `memCacheHeight` from logical size × device pixel ratio (capped). |
| **Result** | Images decode near display size; reduced memory use is expected. *Not measured.* |
| **Residual risk / next step** | Server-side image compression and CDN delivery recommended. |

#### PERF-002 — Redundant Network GETs

| | |
|---|---|
| **Severity** | High |
| **Disposition** | Closed (with follow-up REL-001) |
| **Problem** | Repeated GET calls for the same resources on tab switches/navigation caused latency and backend load. |
| **Remediation** | Short-lived in-memory Dio GET cache (≈90 s TTL) keyed by URL/query/auth; cleared on logout; targeted invalidation for deliverymen refresh. |
| **Result** | Warm navigations reuse fresh responses. *Hit rate not measured.* |
| **Residual risk / next step** | See REL-001: cache policy for time-sensitive operational data. |

#### PERF-003 — Admin Shell / Tab Mounting

| | |
|---|---|
| **Severity** | Medium |
| **Disposition** | Closed |
| **Problem** | Admin/home tab stacks built all children eagerly, increasing startup cost and memory. |
| **Remediation** | `LazyIndexedStack`; tabs mount on first visit and preserve state. |
| **Result** | Initial load mounts only the active tab. |
| **Residual risk / next step** | None material. |

#### PERF-004 — Admin Drivers List

| | |
|---|---|
| **Severity** | High |
| **Disposition** | Closed |
| **Problem** | Drivers management was slow: repeated fetches, inefficient collections, heavy image paths. |
| **Remediation** | Shared `GetAllDriverCubit` as lazy singleton with TTL cache and prefetch from `AdminScreen`; List-based storage; `AppNetworkImage`; `forceRefresh` clears the Dio deliverymen cache. |
| **Result** | Drivers screen opens from warm cache; refresh path is explicit. |
| **Residual risk / next step** | Full dataset loaded until server pagination exists (PERF-008). |

#### PERF-005 — Financial / Order Records Filters

| | |
|---|---|
| **Severity** | High |
| **Disposition** | Closed |
| **Problem** | Reports screen rebuilt large subtrees on filter changes; rapid taps stampeded API calls; `Set` + `elementAt` access was O(n²). |
| **Remediation** | Debounced `getOrderRecordsDebounced`; stale-success UI while reloading; split `ValueListenableBuilder`; stable dropdown items; `RepaintBoundary` on cards; List semantics in `AppDropDown`. |
| **Result** | Fewer redundant rebuilds and network bursts; stable dropdown selection. |
| **Residual risk / next step** | Large report payloads remain unbounded without API pagination. |

#### PERF-006 — Cart Screen Rebuilds

| | |
|---|---|
| **Severity** | Medium |
| **Disposition** | Closed |
| **Problem** | Cart `FutureBuilder` recreated futures/side-effects per build; `ScheduleNotifier` lifecycle incomplete; broad repaints. |
| **Remediation** | Stable `_cartFuture`; one-shot quantity init via post-frame callback; dispose `ScheduleNotifier`; `RepaintBoundary` on rows. |
| **Result** | Deterministic cart build path; fewer accidental reloads. |
| **Residual risk / next step** | Isolate price/total listeners if the cart grows more interactive. |

#### PERF-007 — Maps Rendering Cost

| | |
|---|---|
| **Severity** | Medium |
| **Disposition** | Closed |
| **Problem** | `MapType.hybrid` increased tile/GPU cost on location and order-detail surfaces. |
| **Remediation** | Switched primary surfaces to `MapType.normal`. |
| **Result** | Lower rendering cost with equivalent positioning UX. |
| **Residual risk / next step** | Hybrid can return as an opt-in. |

#### PERF-008 — Large List Pagination

| | |
|---|---|
| **Severity** | High |
| **Disposition** | Deferred (Backend + Mobile) |
| **Problem** | Orders/users/reports/drivers can grow without server-side page/cursor limits, capping client optimizations. |
| **Remediation** | Not implemented; requires backend contract changes. Client has pagination primitives for some admin lists. |
| **Result** | No end-to-end pagination for primary heavy endpoints. |
| **Residual risk / next step** | Highest remaining performance debt. Deliver cursor-based API + client page consumers. |

---

### 4.4 Quality and Reliability

#### QUAL-001 — Build Integrity

| | |
|---|---|
| **Severity** | High |
| **Disposition** | Closed |
| **Problem** | Release AAB failed due to dependency/toolchain conflicts (`intl`, `shared_preferences` v1 Registrar, Kotlin incremental path issues). |
| **Remediation** | Dependency overrides/upgrades, NDK alignment, `kotlin.incremental=false` where required; analyze verified on touched surfaces. |
| **Result** | Build blockers cleared; app compiles on the current toolchain. |
| **Residual risk / next step** | Review dependency pins on each Flutter upgrade. |

#### QUAL-002 — Static Analysis Debt

| | |
|---|---|
| **Severity** | Low |
| **Disposition** | Deferred |
| **Problem** | `flutter analyze` reports unused imports, deprecated `withOpacity`, async `BuildContext` infos, and residual `print()` calls. |
| **Remediation** | Deferred to avoid scope creep. |
| **Result** | No functional regressions attributed to these warnings. |
| **Residual risk / next step** | Lint cleanup sprint; CI gate on new warnings. |

#### QUAL-003 — Automated Test Coverage

| | |
|---|---|
| **Severity** | **High** (was Medium) |
| **Disposition** | Deferred |
| **Problem** | Auth, checkout/cart, session-clear, and admin-report paths lack automated coverage in a system that handles orders, dispatch, and money. |
| **Remediation** | Not in current scope; manual verification for release candidates. |
| **Result** | Confidence rests on code review and targeted manual scenarios. |
| **Residual risk / next step** | Add golden-path tests: login, checkout (including failure and retry), logout session clear, admin report filters. Track coverage on critical paths. Add backend concurrency and authorization tests (see REL-004, SEC-005). |

#### QUAL-004 — Performance Baseline Not Measured

| | |
|---|---|
| **Severity** | Medium |
| **Disposition** | Open (new) |
| **Problem** | Performance claims in PERF-001 to PERF-007 have no before/after data, so improvement cannot be demonstrated or protected against regression. |
| **Remediation** | Recommended measurement plan: Flutter DevTools / `flutter run --profile` for frame build/raster times (jank rate), memory on image-heavy lists, network call counts per screen flow, cold-start time, and cache hit rate. Record on at least one low-end and one mid-range device. |
| **Result** | No data yet. |
| **Residual risk / next step** | Store results in `ENGINEERING_TEST_REPORT.md`; add thresholds to CI where feasible. |

#### REL-001 — Cache Staleness on Time-Sensitive Data

| | |
|---|---|
| **Severity** | Medium |
| **Disposition** | Open (new) |
| **Problem** | A 90-second GET cache on order queues and status endpoints can show a driver or admin a stale order state in a time-sensitive logistics workflow. |
| **Remediation** | Recommended: cache by allowlist, not by default. Exclude order queues, order detail/status, and notifications; keep caching for catalog, zones, settings, and driver directory. Invalidate on every successful write and on push/realtime events. Provide pull-to-refresh bypass. |
| **Result** | Not yet implemented. |
| **Residual risk / next step** | Document the allowlist as a cache policy in `ARCHITECTURE.md`. |

#### REL-002 — Checkout Idempotency

| | |
|---|---|
| **Severity** | High |
| **Disposition** | Open (new) |
| **Problem** | If the network drops after a checkout request is sent, a retry or double-tap can create a duplicate order. Dispatch invariants guard routing consistency but do not by themselves prevent duplicate intent. |
| **Remediation** | Recommended: client sends an idempotency key (UUID per checkout attempt); backend stores key + result and returns the original response on replay; client disables the submit control while a request is in flight. |
| **Result** | Not yet implemented. |
| **Residual risk / next step** | Backend contract change plus client change; add a test for replayed checkout. |

#### REL-003 — Connectivity Handling for Field Agents

| | |
|---|---|
| **Severity** | Medium |
| **Disposition** | Open (new) |
| **Problem** | Drivers work in variable network conditions. Behavior on timeouts, offline state, and status updates made while disconnected is undefined in the documentation. |
| **Remediation** | Recommended: explicit offline/timeout states, retry with backoff for idempotent reads, queued status updates with conflict-safe replay, and a visible connectivity indicator. |
| **Result** | Not yet specified. |
| **Residual risk / next step** | Define target behavior first; status transitions must stay safe under replay (server-side state-machine validation). |

#### REL-004 — Dispatch Concurrency Verification

| | |
|---|---|
| **Severity** | Medium |
| **Disposition** | Verify (new) |
| **Problem** | The platform states that dispatch is conflict-free by persistence-layer invariants. That claim needs executable evidence. |
| **Remediation** | Recommended backend tests: concurrent checkouts for the same zone, assignment changes during checkout, zone boundary edits against existing zones, and replayed requests. Record results. |
| **Result** | Evidence not yet recorded. |
| **Residual risk / next step** | Link results in the architecture document next to the invariants. |

---

### 4.5 Operability and Delivery

#### OPS-001 — Crash Reporting and Observability

| | |
|---|---|
| **Severity** | Medium |
| **Disposition** | Open (new) |
| **Problem** | No evidence of crash reporting, error aggregation, or client-side telemetry. Production failures in field use would be invisible until users report them. |
| **Remediation** | Recommended: crash reporting (for example Firebase Crashlytics, with the obfuscation symbols from SEC-008); breadcrumbs for network errors; key service indicators such as crash-free users/sessions and checkout success rate; no personal data or tokens in telemetry. |
| **Result** | Not yet implemented. |
| **Residual risk / next step** | Define target thresholds (for example crash-free sessions) and alert owners. |

#### OPS-002 — CI/CD Pipeline and Quality Gates

| | |
|---|---|
| **Severity** | Medium |
| **Disposition** | Open (new) |
| **Problem** | No evidence of an automated pipeline running analysis, tests, build, and dependency checks before release. |
| **Remediation** | Recommended pipeline: `flutter analyze` and tests on every change; dependency vulnerability scanning; secret scanning to keep keystores and `key.properties` out of the repository; signed release build; staged Play rollout (internal → closed → production with percentage rollout). |
| **Result** | Not yet implemented. |
| **Residual risk / next step** | Add a release checklist and a documented rollback path. |

#### OPS-003 — Play Policy and Privacy Compliance

| | |
|---|---|
| **Severity** | Medium |
| **Disposition** | Verify (new) |
| **Problem** | Beyond target API level, Play requires accurate Data Safety disclosures, a privacy policy, justified location permissions, and an in-app path to request account deletion for apps that allow account creation. |
| **Remediation** | Recommended: reconcile the Data Safety form with actual collection (location, phone number, order data, push token); confirm the privacy policy URL; confirm the account-deletion flow and web link; confirm foreground-only location use or provide the required background-location justification. |
| **Result** | Not yet evidenced. |
| **Residual risk / next step** | Complete before the production upload. |

---

## 5. Traceability matrix

| ID | Severity | Area | Disposition |
|---|---|---|---|
| COMP-001 | Critical | Play / platform compliance | Closed |
| SEC-001 | High | Credential storage | Closed |
| SEC-002 | High | Session hygiene | Closed |
| SEC-003 | Medium | Secrets in logs | Closed |
| SEC-004 | High (Critical if OTP reaches requester) | OTP emergency channel | Accepted (conditional) |
| SEC-005 | Medium | Authorization (role enforced; ownership to verify) | Partially closed |
| SEC-006 | Medium | Embedded API keys | Deferred |
| SEC-007 | Medium | Transport security | Open |
| SEC-008 | Low | Release hardening | Open |
| SEC-009 | Medium | Backup / local data | Verify |
| SEC-010 | Medium | Realtime channel authorization | Verify |
| PERF-001 | High | Image decode / memory | Closed |
| PERF-002 | High | Redundant GETs | Closed |
| PERF-003 | Medium | Tab mounting | Closed |
| PERF-004 | High | Admin drivers list | Closed |
| PERF-005 | High | Order records filters | Closed |
| PERF-006 | Medium | Cart rebuilds | Closed |
| PERF-007 | Medium | Map rendering | Closed |
| PERF-008 | High | List pagination | Deferred |
| QUAL-001 | High | Build integrity | Closed |
| QUAL-002 | Low | Static analysis debt | Deferred |
| QUAL-003 | High | Automated tests | Deferred |
| QUAL-004 | Medium | Performance baseline | Open |
| REL-001 | Medium | Cache staleness policy | Open |
| REL-002 | High | Checkout idempotency | Open |
| REL-003 | Medium | Connectivity handling | Open |
| REL-004 | Medium | Dispatch concurrency evidence | Verify |
| OPS-001 | Medium | Crash reporting / observability | Open |
| OPS-002 | Medium | CI/CD and gates | Open |
| OPS-003 | Medium | Play policy / privacy | Verify |

---

## 6. Open actions (priority backlog)

| Priority | Action | Owner | Related IDs |
|---|---|---|---|
| **P0** | Verify resource-level ownership checks on all order/user/report endpoints; add automated authorization tests | Backend | SEC-005 |
| **P0** | Apply compensating controls to the OTP emergency channel (admin-only activation, audit log, expiry, rate limits); plan removal of OTP from API responses | Product + Backend | SEC-004 |
| **P0** | Checkout idempotency (key + in-flight lock) | Backend + Mobile | REL-002 |
| **P1** | Pagination for orders, reports, users, drivers | Backend + Mobile | PERF-008 |
| **P1** | Restrict and rotate Maps / `x-api-key`; quotas and alerts | Backend + Cloud | SEC-006 |
| **P1** | Cache allowlist for non-operational data only | Mobile | REL-001 |
| **P1** | Golden-path tests (auth, checkout, logout, reports) and dispatch concurrency tests | Mobile + Backend | QUAL-003, REL-004 |
| **P1** | Crash reporting; CI pipeline with analysis, tests, dependency and secret scanning | Mobile + DevOps | OPS-001, OPS-002 |
| **P1** | Verify Play Data Safety, privacy policy, account deletion, location permissions | Product + Mobile | OPS-003 |
| **P2** | Verify `allowBackup`, cleartext policy, realtime channel authorization | Mobile + Backend | SEC-009, SEC-007, SEC-010 |
| **P2** | Obfuscation and symbol upload; evaluate pinning and integrity checks | Mobile | SEC-007, SEC-008 |
| **P2** | Performance measurement baseline | Mobile | QUAL-004 |
| **P2** | Connectivity behavior specification and implementation | Product + Mobile | REL-003 |
| **P3** | Lint cleanup | Mobile | QUAL-002 |

---

## 7. OWASP MASVS v2 mapping

| MASVS group | Coverage in this report | Status |
|---|---|---|
| **MASVS-STORAGE** | Token in secure storage (SEC-001); logout clears session/cache (SEC-002); backup exposure (SEC-009) | Mostly addressed; backup to verify |
| **MASVS-CRYPTO** | Keystore-backed storage via `flutter_secure_storage`; no custom cryptography identified | Not separately assessed |
| **MASVS-AUTH** | Server-side role enforcement (SEC-005); OTP emergency channel (SEC-004) | Partial; ownership to verify; OTP conditional |
| **MASVS-NETWORK** | Header redaction (SEC-003); transport hardening and pinning (SEC-007) | Open |
| **MASVS-PLATFORM** | Permissions, push payload handling, deep links | Not assessed; add to next review |
| **MASVS-CODE** | Dependency upkeep (QUAL-001); static analysis (QUAL-002); dependency scanning (OPS-002) | Partial |
| **MASVS-RESILIENCE** | Obfuscation, integrity checks (SEC-008) | Open |
| **MASVS-PRIVACY** | Data Safety, privacy policy, account deletion (OPS-003) | Verify |

---

## 8. Role impact summary

### 8.1 Customer

| Dimension | Score | Summary |
|---|---|---|
| Performance | 8.5 | Lighter cart, product images, maps; GET cache (unmeasured) |
| Quality | 8.0 | Deterministic cart initialization; checkout idempotency open |
| Security | 7.0 | Secure session; server-side role enforcement; ownership to verify |
| UX | 8.0 | Suitable for daily ordering flows |

### 8.2 Driver

| Dimension | Score | Summary |
|---|---|---|
| Performance | 8.0 | Normal maps and cached images; list scale API-bound |
| Quality | 7.5 | Order detail stable; cache staleness and connectivity handling open |
| Security | 7.0 | Ownership checks to verify (own orders only) |
| UX | 7.8 | Adequate for field operations |

### 8.3 Admin

| Dimension | Score | Summary |
|---|---|---|
| Performance | 7.5 | Clear gain on drivers/reports; ceiling is pagination |
| Quality | 8.0 | Lazy tabs, fewer report rebuilds |
| Security | 6.8 | Privileged surfaces: role enforced; audit trail for sensitive actions recommended |
| UX | 7.5 | Faster after client fixes; large datasets still heavy |

---

## 9. Conclusion

The client meets **Google Play API 36 readiness** and shows design-level improvements in credential storage, session hygiene, log safety, and performance for admin lists, cart, images, and maps.

Revision 1.1 corrects the authorization finding: **server-side role enforcement exists**, so the remaining authorization work is verification of resource-level ownership and automated authorization testing.

From a global professional engineering standard, the release candidate is **acceptable for store submission**. Full **security and operational readiness** depends on closing the following before scaling:

1. Ownership verification and authorization tests (SEC-005)
2. Compensating controls on the OTP emergency channel (SEC-004)
3. Checkout idempotency (REL-002)
4. Key governance (SEC-006)
5. Pagination (PERF-008)
6. Automated tests, crash reporting, and CI gates (QUAL-003, OPS-001, OPS-002)

---

## 10. Document control

| Item | Value |
|---|---|
| Prepared as | Expert engineering remediation record |
| Revision history | 1.0 (18 Oct 2026): initial. 1.1 (19 Oct 2026): RBAC finding corrected; severity changes (SEC-004 conditional escalation, SEC-005 downgraded, QUAL-003 raised); 12 findings added; scoring method published; MASVS mapping added |
| Not a substitute for | Formal certification audit, penetration-test report, or legal compliance attestation |
| Re-validation triggers | Backend P0 delivery; Play Console upload; major Flutter/SDK upgrade; any change to the OTP emergency channel |
| Handling | Internal only; contains vulnerability and risk detail |

---

*End of report.*
