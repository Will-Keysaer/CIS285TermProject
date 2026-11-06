# SplitSmart: Verification Plan (Reconciled Candidate)

Status: **Reconciled candidate. Every test and command here is `PROPOSED`. No application code or test exists, so nothing has been run.** The only checks executed so far are the documentation and diagram checks recorded in [README.md §8](README.md#8-validation-record).

**Status values:**

| Status | Meaning |
|---|---|
| `PROPOSED` | Designed, not implemented |
| `UNVERIFIED` | Implemented but not yet passing, or depends on an unmeasured condition |
| `BLOCKED` | Cannot run until a decision is made |
| `PASSED` | Executed with a recorded result |

No item is `PASSED`.

## 1. Tooling and commands (PROPOSED; not run)

| Purpose | Command (Maven Wrapper recommended; plain `mvn` equivalent) |
|---|---|
| Full verification (CONST II, V) | `./mvnw verify` |
| Core only (before OQ-1) | `./mvnw verify -Dtest='splitsmart.domain.**,splitsmart.application.**'` |
| PostgreSQL integration and concurrency (requires Docker for Testcontainers, or a local PostgreSQL) | `./mvnw verify -Ppostgres` |
| Browser E2E [COND-A] | `./mvnw verify -Pe2e` |
| Performance (NFR-004) | `./mvnw verify -Pperf` |
| Diagram syntax | `java -jar plantuml.jar -checkonly architecture/final/diagrams/*.puml` |

**Build configuration requirements:**

- `maven.compiler.release=21` (CONST II).
- JUnit 5 through the `junit-bom`.
- Surefire for unit tests (`*Test`) and Failsafe for integration tests (`*IT`).

**Environment fact (F-11):** the authoring machine has **JDK 25.0.2** and **no Maven**. Verification is not reproducible there until OD-2 is decided.

## 2. Unit tests: domain (no Spring, no database, no GUI)

| ID | Test class | Expected result | Req / AC |
|---|---|---|---|
| VP-U01 | `MoneyParserTest` | Accepts `12.5`, `12.50`, `100`, `0.01`. Rejects `0`, `-1` (`AMOUNT_NOT_POSITIVE`), `12.505` (`AMOUNT_PRECISION_EXCEEDED`), `1E+2`, `1,000.00`, `$5` (`AMOUNT_FORMAT_INVALID`), `1000000.01` (`AMOUNT_TOO_LARGE`). JPY rejects `10.5`. Scale-3 currency accepts `1.234`. | FR-003, AC-005, AC-009 |
| VP-U02 | `MoneyTest` | Scale normalized. `equals` consistent with `compareTo`. Currency mismatch throws. Minor-unit round trip. Persisted excess scale throws. | NFR-003 |
| VP-U03 | `PercentageTest` | `33.33` → 3333 bp. Rejects `33.333`, `-1`, `100.01`, `abc`. | FR-005, AC-009 |
| VP-U04 | `AllocationCalculatorTest` | design §3.4 cases 1–7 exactly (e.g. $100.00 three ways → 33.34 / 33.33 / 33.33; $10.00 at 33.33/33.33/33.34 → 3.33 / 3.33 / 3.34). Tie-break by UUID string even where `UUID.compareTo` disagrees. One participant receives the full amount. | FR-004, FR-005, AC-007, AC-008, AC-021 |
| VP-U05 | `AllocationPropertyTest` (jqwik, optional) | For random valid inputs: Σ = amount; each share within 1 minor unit of exact; deterministic | NFR-003, AC-021, SC-002 |
| VP-U06 | `ExpenseValidatorTest` | Every code in design §4.2. Several errors returned together. Payer as non-participant valid. Payer as participant valid. Non-member participant rejected. Empty participants rejected. | FR-003, AC-009 |
| VP-U07 | `ExpenseStateMachineTest` | PENDING → APPROVED / REJECTED. Edit only by the submitter while PENDING. Any transition from APPROVED or REJECTED → `EXPENSE_NOT_PENDING`. Stale version → `CONCURRENT_MODIFICATION`. | FR-006, FR-007, AC-010–012 |
| VP-U08 | `ReviewPolicyTest` | Non-member → NOT_FOUND. Member → FORBIDDEN_ROLE. Manager → allowed. Manager-submitter → SELF_APPROVAL_NOT_ALLOWED unless enabled. `reviewBlocked` computed correctly. | FR-007, OQ-2 default |
| VP-U09 | `PairwiseBalanceCalculatorTest` | Payer's own share ignored. Netting in both directions. Payments reduce. Σ memberNet = 0. Quickstart S5–S7 figures reproduced. | FR-009, AC-014, NFR-003, SC-003 |
| VP-U10 | `PaymentRulesTest` | Amount = N allowed. N + 0.01 → `PAYMENT_EXCEEDS_OUTSTANDING`. N ≤ 0 → `NOTHING_OWED`. Self-payment, non-member recipient, and non-sender actor all rejected. | FR-010, AC-015, AC-016 |
| VP-U11 | `GroupMembershipRulesTest` | Creator protected. Last manager protected. Non-zero balance or pending involvement blocks removal. | FR-002, AC-004 |
| VP-U12 | `InvitationLifecycleTest` | Only the invitee accepts or declines. Expired → `INVITATION_NOT_PENDING`. Revoked → not actionable. A second accept → `INVITATION_NOT_PENDING`. | FR-002, AC-003, F-03 |
| VP-U13 | `DateRangeTest` | Inclusive boundaries. `to < from`, nulls, and > 5 years → `INVALID_DATE_RANGE`. | FR-012, AC-018, AC-019 |

## 3. Application tests (in-memory fakes for ports)

| ID | Test class | Expected result | Req / AC |
|---|---|---|---|
| VP-A01 | `AccountServiceTest` | Register, authenticate, generic `INVALID_CREDENTIALS`, change password | FR-001, AC-001 |
| VP-A02 | `PasswordResetFlowTest` | Unknown and known email get the same response. Token single use; expires at 30 min; old sessions invalid after reset. Recovery codes single use (B). | FR-001, AC-001, F-02 |
| VP-A03 | `GroupServiceTest` | Manager-only invite and remove (non-manager → `FORBIDDEN_ROLE`). Acceptance required before membership. History keeps account IDs after removal. | FR-002, AC-003, AC-004 |
| VP-A04 | `ExpenseServiceTest` | Submit creates PENDING with stored shares. Notifications to **other** active members only. Balances unchanged. Idempotent replay; `IDEMPOTENCY_KEY_REUSED`. | FR-003–FR-006, FR-008, AC-005, AC-010, AC-013 |
| VP-A05 | `ReviewServiceTest` | Approve → included in balances. Reject → excluded with the reason visible. Reviewed notifications sent. | FR-007, AC-011, AC-012 |
| VP-A06 | `PaymentServiceTest` | Records `recordedBy` = sender with a server timestamp. Balance drops immediately. Duplicate confirmation path. | FR-010, AC-015, AC-016 |
| VP-A07 | `SummaryServiceTest`, `ReportServiceTest` | Figures match the source records. Inclusive boundaries. Empty result valid. Scope limited to the caller's groups and party records. | FR-011, FR-012, AC-017–019 |
| VP-A08 | `AuthorizationMatrixTest` | One case per cell of design §9.2, including former members | NFR-002, AC-002, SC-005 |
| VP-A09 | `AuditTrailTest` | One audit event per write. Before/after on edit. Visibility per §9.2. Audit failure rolls back. | NFR-005, AC-004, AC-023 |

## 4. Integration, concurrency, and security tests

| ID | Test | Database | Expected result | Req / finding |
|---|---|---|---|---|
| VP-I01 | `MigrationIT` | H2 + PostgreSQL | Flyway applies cleanly. CHECK, UNIQUE, and FK constraints present (data-model §9). | Data integrity |
| VP-I02 | `BalanceQueryMatchesCalculatorIT` | H2 + PostgreSQL | SQL directed sums equal in-memory sums on randomized fixtures | NFR-003, SC-003 |
| VP-I03 | `ReportQueryIT` | H2 + PostgreSQL | Boundary-date rows included. Other groups never returned. | FR-012, NFR-002 |
| VP-C01 | `ConcurrentReviewIT` | **Selected production DB** (PostgreSQL for A) | Two threads approve or reject at once: exactly one succeeds; one `ReviewDecision` row | F-07, FR-007 |
| VP-C02 | `ConcurrentPaymentIT` | **Selected production DB** | Debt 50.00; two concurrent payments of 30.00: one succeeds, one `PAYMENT_EXCEEDS_OUTSTANDING`; final balance 20.00 | F-07, AC-015 |
| VP-C03 | `EditVsApproveIT` | **Selected production DB** | Edit commits first, so the approval with the stale version → `CONCURRENT_MODIFICATION` | F-07 |
| VP-C04 | `IdempotentRaceIT` | **Selected production DB** | Two identical keyed payments at once: one row; both responses identical | F-07 |
| VP-S01 [COND-A] | `NoLeakageApiTest` | H2 | A non-member gets 404 with no details on every group-scoped endpoint. Error bodies never include others' amounts. | NFR-002, AC-002 |
| VP-S02 [COND-A] | `CsrfAndSessionTest` | H2 | Missing CSRF token → 403. Session ID changes on login. Sessions rejected after a password change. | NFR-002 |
| VP-S03 [COND-A] | `ReceiptUploadTest` | H2 | PNG/JPEG accepted. A GIF renamed `.jpg` → 415. 6 MB → 413. EXIF removed. Non-member download → 404. | FR-003, AC-006, F-05 |
| VP-S04 [COND-A] | `RateLimitTest` | H2 | The 6th failed login within 15 min → 429 | NFR-002 |
| VP-K01 [COND-A] | `*ApiTest` (MockMvc) | H2 | Every endpoint in web-api.md: status codes, problem `code`s, money as plain strings, a JSON number for an amount → 400 `AMOUNT_FORMAT_INVALID`, paging shape | All FRs |
| VP-K02 [COND-B] | `*PresenterTest` | H2 | Error mapping; duplicate-payment confirmation; recovery codes shown once | All FRs (B) |
| VP-ARCH | `ArchitectureRulesTest` | none | Package dependency rules; no `double`/`float` money; Swing views without `java.math` | CONST III, IV |

## 5. End-to-end and acceptance tests

| ID | Scenario ([quickstart.md §4](quickstart.md#4-end-to-end-validation-scenarios)) | Platform | Expected result | AC |
|---|---|---|---|---|
| VP-E2E-1 … 9 | S1–S9 driven through the browser (Playwright for Java, optional; otherwise a manual script with recorded evidence) | [COND-A] | Each scenario's expected results | AC-001–AC-021, AC-023 |
| VP-E2E-10 | AC-020 web journey: S1–S9 completed using only a supported browser (QA-S2 list) | [COND-A] | All journeys complete | AC-020 / NFR-001. **BLOCKED** until OQ-1. Cannot pass under B. |
| VP-E2E-B | S1–S9 via the Swing UI | [COND-B] | Same functional results; **AC-020 recorded as not met** | — |
| VP-A11Y | Keyboard-only run of U1–U19 plus an automated accessibility scan | [COND-A] | No critical violations (suggested QA-S1, not required) | QA-S1 |

## 6. Performance protocol (NFR-004 / AC-022)

Status: **PROPOSED; UNVERIFIED. No measurement exists.** The values below define how NFR-004 *would* be measured. They are not results. SPEC treats the target as a planning assumption (AS-11). CR-03 proposes adding this protocol to SPEC.

| Element | Definition |
|---|---|
| Dataset | 1 group with **50 ACTIVE members**. **10,000 records**: 8,000 expenses (equal and percentage, 2–10 participants, about 85% APPROVED / 10% PENDING / 5% REJECTED, dates spread over 3 years) and 2,000 payments. Plus 4 other groups of 10 members with 500 records each, so that scope filtering has data to filter. Generated deterministically from a fixed seed. |
| Environment ("ordinary operating conditions", proposed) | One application instance with 2 vCPU and 4 GB RAM; PostgreSQL 16 on the same host or LAN; JVM warmed up; no other load. The actual hardware must be recorded with the results. |
| Load | 10 concurrent virtual users, each repeating the view mix with 1–3 s think time, for 10 minutes after a 2-minute warm-up |
| Views measured (all GET) | Account: `GET /me`. Expense: `GET /groups/{id}/expenses?page=0&size=50` and `GET /expenses/{id}`. Balance: `GET /groups/{id}/balances` and `GET /me/balances`. Report: `GET /me/summary` and `GET /me/reports` (1-year range, all types). |
| Timing boundary | From the client sending the request to receiving the last response byte, measured by the load client on the same LAN. Browser rendering is excluded and can be measured separately as an informational value. |
| Pass criterion | ≥ 95% of **all sampled requests** across the listed views complete in ≤ 3.0 s, **and** each individual view's p95 is ≤ 3.0 s (stricter reading, reported separately) |
| Tooling (proposed) | JUnit-driven Java HTTP client harness (`PerformanceIT`) or Gatling. Results exported as CSV with p50/p95/p99 per view. |
| If it fails | Profile; check indexes; then the materialized-balance fallback (D-18) with a reconciliation check. Re-run. |
| Recorded evidence | Hardware, versions, seed, raw CSV, and summary table, committed with the run date |

## 7. UML validation

| ID | Check | Status |
|---|---|---|
| VP-UML1 | Every `.puml` in `diagrams/` passes a PlantUML syntax check | See [README.md §8](README.md#8-validation-record) |
| VP-UML2 | Entity names, attributes, and cardinalities in `domain-model.puml` and `classes-detailed.puml` match data-model.md | Manual cross-check (README §8) |
| VP-UML3 | State names and guards in `states*.puml` match data-model §6 and design §5 | Manual cross-check |
| VP-UML4 | Sequence messages use service, method, and endpoint names from the contracts; error codes match application-services §7 | Manual cross-check |
| VP-UML5 | Use-case actors and privileges match the design §9.2 matrix | Manual cross-check |

## 8. Exit criteria per release

| Release | Exit criteria (all must be `PASSED` with evidence) |
|---|---|
| Core (pre-OQ-1) | VP-U01–U13, VP-A01–A09 (fakes), VP-ARCH, VP-I01–I02 on H2 |
| Release 1 [COND-A] | Core + VP-I03, VP-C01–C04 on PostgreSQL, VP-S01–S02, VP-S04, VP-K01, VP-E2E-1…8 (except receipt and report steps), VP-E2E-10 |
| Release 2 [COND-A] | Release 1 + VP-S03, VP-E2E-9 and the report steps, VP-A07 custom report, **VP-PERF** (§6) |
| Human review | CONST VII sign-off recorded |
