# SplitSmart: Requirements Traceability (Claude Code proposal)

Status: **draft, needs human review.** Maps every customer-baseline requirement to design elements and proposed verification. Test class names are **proposed**; no tests exist yet. "S#" refers to scenarios in [quickstart.md §4](quickstart.md#4-end-to-end-validation-scenarios).

MVP columns:

- **Cand. MVP:** in the spec's candidate MVP slice?
- **A MVP / B MVP:** is the requirement satisfied by the MVP under each alternative?
- Legend: ✅ satisfied · ◐ partially · ⏸ deferred (retained in full scope) · ❌ not satisfied.

## 0. Requirement → detailed design section

| Requirement | [design.md](design.md) sections |
|---|---|
| FR-001 | §4.3 (AccountValidator), §13.2 [COND-A], §14 RegisterPresenter [COND-B] |
| FR-002 | §5.3–5.5, §8.1, §15.6, §15.7 |
| FR-003 | §2.2, §4.2, §13.3 (receipt, full scope), §15.1 |
| FR-004 / FR-005 | §3 (algorithm, tie-break, worked cases) |
| FR-006 / FR-007 | §5.1, §5.2, §6.3, §8.2, §15.2 |
| FR-008 | §11 |
| FR-009 | §6 |
| FR-010 | §7, §8.4 |
| FR-011 / FR-012 | §12, §15.8 |
| NFR-001 | §13 [COND-A]. Not met by §14 [COND-B]. |
| NFR-002 | §9 (authorization matrix), §10.2 |
| NFR-003 | §2, §3, §6.1 (Σ memberNet = 0) |
| NFR-004 | §6.1 (input size), §6.2 (indexes), §16 PerformanceIT |
| NFR-005 | §5.1, §11, §15.2 |
| Tests | §16 |

## 1. Functional requirements

| Req | Summary | Design elements | Key decisions | Proposed verification | Cand. MVP | A MVP | B MVP [COND] |
|---|---|---|---|---|---|---|---|
| **FR-001** | Accounts (email + password), sign-in, recovery | `AccountService`; `Account`; `PasswordHasher` (BCrypt); A: Spring Security session, `PasswordResetToken`, SMTP `RecoveryChannel`; B: `RecoveryCode` | Uniform recovery responses; tokens hashed, 30 min, single use; sessions invalidated on reset (arch §4.1, §5) | `AccountServiceTest`, `PasswordResetFlowTest`, `AuthApiTest` (A), `RecoveryCodeTest` (B); S1 | Yes | ✅ | ✅ (recovery by codes, not email) |
| **FR-002** | Groups, membership, manager-only invite/remove, acceptance required | `GroupService`; `Group`, `Membership`, `Invitation`; `GroupMembershipRules`; `AuthorizationService.requireManager` | Creator always manager; ≥ 1 manager; removal blocked with balance or pending involvement (D-2); history keeps account IDs | `GroupMembershipRulesTest`, `GroupServiceTest` (incl. non-manager denial), `InvitationLifecycleTest`; S2, S9 | Yes | ✅ | ◐ invitees must use the same installation |
| **FR-003** | Expense fields, payer (member, separate, may be non-participant), participants, optional receipt | `ExpenseService.submitExpense`; `Expense`, `ExpenseParticipant`; `Money`/`CurrencyPolicy`; A: `ReceiptAttachment`, `ReceiptStorage` | Excess precision rejected, never rounded; payer chosen independently; shares stored; receipt uploaded separately, JPEG/PNG ≤ 5 MB | `MoneyTest`, `SubmitExpenseValidationTest`, `ExpenseApiTest`, `ReceiptUploadTest` (A); S3, S4, S6 | Yes, **except receipt (deferred)** | ◐ receipt ⏸ | ◐ receipt ⏸ |
| **FR-004** | Equal split | `AllocationCalculator` (EQUAL) | Integer minor units; largest remainder; tie-break by ascending accountId | `AllocationCalculatorTest` (table in arch §6.1), property test Σ = amount; S3 | Yes | ✅ | ✅ |
| **FR-005** | Percentage split, ≥ 0, Σ = 100, largest remainder | `AllocationCalculator` (PERCENTAGE); `Percentage` | Scale ≤ 2 (AS-2); exact Σ = 100.00 | `AllocationCalculatorTest`, `PercentageValidationTest`; S4, S5 | Yes | ✅ | ✅ |
| **FR-006** | Pending expense excluded from balances | `Expense` state machine; `PairwiseBalanceCalculator` filters APPROVED | Balances derived only from APPROVED (I-8) | `BalanceExcludesPendingTest`; S3 step 4 | Yes | ✅ | ✅ |
| **FR-007** | Review/approval; record approve or reject | `ReviewService`; `ReviewPolicy`; `ReviewDecision` | Managers approve; no self-approval by default (OQ-2); single level; conditional update + version (arch §6.3) | `ReviewPolicyTest`, `ReviewServiceTest`, `ConcurrentReviewIT`; S5 | Yes (single step) | ✅ single-level; multi-level ⏸ | ✅ single-level; multi-level ⏸ |
| **FR-008** | Notify other members on submission | `NotificationService`; `Notification`; `NotificationPublisher` port | Inserted in the same transaction as the expense; in-app only (OQ-3) | `SubmitNotifiesOtherMembersTest`; S3 step 3 | Yes (in-app) | ✅ in-app; other channels ⏸ | ◐ visible only on that machine |
| **FR-009** | Pairwise balances from approved expenses and payments | `BalanceService`; `PairwiseBalanceCalculator`; balance aggregate query | **Derived, not materialized** (data-model §1.13, §4); per group; per-currency totals | `PairwiseBalanceCalculatorTest`, `BalanceQueryMatchesCalculatorIT`, `NetSumsToZeroTest`; S5, S6 | Yes | ✅ | ✅ (single installation) |
| **FR-010** | Sender records payment; immediate effect; recorder + timestamp | `PaymentService`; `Payment`; `PaymentRules` | Actor must be sender; group row lock; reject overpayment (D-1); idempotency key; duplicate confirmation | `PaymentRulesTest`, `PaymentServiceTest`, `ConcurrentPaymentIT`; S7 | Yes | ✅ | ✅ |
| **FR-011** | Summaries of expenses, balances, payments (authorized) | `ReportService.getSummary` | Scoped to the actor's groups and PARTY records; per currency | `SummaryServiceTest`, `SummaryAuthorizationTest`; S8 step 1 | Yes (basic) | ✅ | ✅ |
| **FR-012** | Custom date-range / parameter reports | `ReportService.runReport`; `DateRange`; `ReportQuery` | Inclusive business-date range; listed filters; no export (OQ-5) | `DateRangeTest`, `ReportServiceTest` (boundaries, empty, invalid); S8 steps 2–3 | **No (deferred)** | ⏸ | ⏸ |

## 2. Non-functional requirements

| Req | Summary | Design elements | Proposed verification | A MVP | B MVP [COND] |
|---|---|---|---|---|---|
| **NFR-001** | Primarily web-based | **A:** `splitsmart-web` (REST API + browser client). **B:** none. A Swing UI cannot satisfy it (constitution VIII). | Browser E2E smoke tests (S1–S9 via browser); AC-020 | ✅ | ❌ **NOT SATISFIED.** Stays an outstanding customer requirement. Requires a later web delivery. |
| **NFR-002** | Privacy and authorization | `AuthorizationService`; 404-not-403 rule; server-computed report scope; UUID IDs; session/CSRF (A); file-permission caveat (B) | `AuthorizationMatrixTest` (every operation × non-member/member/manager/former member), `NoLeakageApiTest` (A); S1, S2 step 4, S8; SC-005 | ✅ | ◐ app-level enforcement only; DB file readable by OS users unless encrypted |
| **NFR-003** | Financial integrity / reconciliation | `Money`, `AllocationCalculator`, derived balances, `NUMERIC(19,4)` at currency scale; invariants I-1 – I-4, I-13, I-15 | Rounding table tests; property tests; `NetSumsToZeroTest`; `BalanceQueryMatchesCalculatorIT`; ArchUnit "no double/float money"; AC-021, SC-002, SC-003 | ✅ | ✅ |
| **NFR-004** | 95% of views ≤ 3 s at 50 members / 10,000 records | Indexes (data-model §2); paging; single aggregate balance query; materialized fallback if needed | `-Pperf` harness; AC-022, SC-004. **Unmeasured.** | Target (to measure) | Target (to measure; single user) |
| **NFR-005** | History distinguishes pending/approved/rejected/settled | `Expense.status`, `ReviewDecision`, immutable `Payment` with recordedBy/recordedAt; status badges in UI | `ExpenseHistoryApiTest`, presenter tests (B); S5 step 3, S7 step 1; AC-023 | ✅ | ✅ |

## 3. Acceptance criteria → verification

| AC | Requirement | Primary verification |
|---|---|---|
| AC-001 | FR-001 | S1; `AccountServiceTest`, `PasswordResetFlowTest` / `RecoveryCodeTest` |
| AC-002 | FR-001, NFR-002 | S1, S2.4; `AuthorizationMatrixTest` |
| AC-003 | FR-002 | S2.1–2.2; `InvitationLifecycleTest` |
| AC-004 | FR-002 | S2.3, S9; `GroupServiceTest` (denial, history attribution) |
| AC-005 | FR-003 | S3; `SubmitExpenseValidationTest` |
| AC-006 | FR-003 | `ReceiptUploadTest` (A, full scope). **Deferred in candidate MVP.** |
| AC-007 | FR-004 | S3; `AllocationCalculatorTest` |
| AC-008 | FR-005 | S5.3; `AllocationCalculatorTest` |
| AC-009 | FR-003, FR-005 | S4; `SubmitExpenseValidationTest` |
| AC-010 | FR-006 | S3.4; `BalanceExcludesPendingTest` |
| AC-011 | FR-007 | S5.2; `ReviewServiceTest` |
| AC-012 | FR-007 | S5.3; `ReviewServiceTest` |
| AC-013 | FR-008 | S3.3; `SubmitNotifiesOtherMembersTest` |
| AC-014 | FR-009 | S5.2, S6; `PairwiseBalanceCalculatorTest` |
| AC-015 | FR-009, FR-010 | S7.1–7.3; `PaymentRulesTest` (overpayment rejected) |
| AC-016 | FR-010 | S7.1, S7.4; `PaymentServiceTest` |
| AC-017 | FR-011 | S8.1; `SummaryServiceTest` |
| AC-018 | FR-012 | S8.2; `ReportServiceTest`. **Deferred in candidate MVP.** |
| AC-019 | FR-012 | S8.3; `DateRangeTest`. **Deferred in candidate MVP.** |
| AC-020 | NFR-001 | Browser E2E (A). **Cannot pass under B.** |
| AC-021 | NFR-003 | `AllocationCalculatorTest`, property tests |
| AC-022 | NFR-004 | S10 (`-Pperf`) |
| AC-023 | NFR-005 | S5.3, S7.1; `ExpenseHistoryApiTest` |

## 4. Requirements not fully addressed by the candidate MVP

These stay in the **full customer scope**. Deferral is a delivery-sequencing proposal, not a change to customer requirements (constitution I). It needs no baseline change, but the customer should be told.

| Requirement | Gap in candidate MVP | Why it remains in scope | Design readiness |
|---|---|---|---|
| FR-003 (receipt image) | Optional receipt not delivered | Explicit customer feature ("optional receipt image") | Fully designed for A (data-model §1.9, rest-api §3.3); B would add a local file copy |
| FR-012 (custom/date-range reports) | Only basic summaries | Customer asks for "custom reports with specified date ranges and other parameters" | Fully designed (application-services §6) |
| FR-007 (multiple approval levels) | Single review step only | Customer says approval "could involve multiple levels … depending on the needs of the group" | `ReviewDecision.level` reserved; needs OQ-2 |
| FR-008 (other channels) | In-app only | Customer did not specify a channel; OQ-3 | `NotificationPublisher` port |
| **NFR-001** | **Not met under B** | Customer baseline: primarily web-based | Met by A; under B it requires a separate web delivery |
| AC-020 | Fails under B | Tied to NFR-001 | Same as above |

## 5. Constitution traceability

| Principle | Where addressed |
|---|---|
| I | This document; [architecture.md](architecture.md) labels (FACT, DECISION, PROPOSAL, ASSUMPTION, OPEN); [design.md §17](design.md#17-design-decisions-introduced-in-this-document) |
| II | Parent POM (`release 21`), [quickstart.md](quickstart.md) |
| III | Module dependency rule + ArchUnit ([architecture.md §3.1](architecture.md#31-layers-and-maven-modules-proposal)) |
| IV | [data-model.md §3](data-model.md#3-money-rules), [architecture.md §6.1](architecture.md#61-monetary-precision-and-allocation-decision) |
| V | [architecture.md §8](architecture.md#8-testing-and-verification-strategy) |
| VI | [diagrams/](diagrams/) |
| VII | Status banner on every document; human review pending |
| VIII | §1 and §2 above (NFR-001 ❌ under B); [architecture.md §2](architecture.md#2-platform-alternatives-and-decision-analysis) |
