# SplitSmart: Detailed Design (Claude Code proposal)

| Field | Value |
|---|---|
| Status | **Draft proposal. Needs human review (Constitution VII).** Not implemented. |
| Date | 2026-11-05 |
| Relationship to other documents | [architecture.md](architecture.md) explains **what** the system is built from and **why**. This document explains **how** each component behaves, at the level a developer needs to implement and test it. Entities and columns: [data-model.md](data-model.md). Operation signatures, inputs, and outputs: [contracts/](contracts/README.md). Diagrams: [diagrams/](diagrams/). |
| Code in this document | Method signatures, pseudocode, and illustrative SQL describe contracts and algorithms. They are **not** application code and are not compiled. |

**Platform labels.** Most of this document describes the **shared core** (`splitsmart-domain`, `splitsmart-application`, `splitsmart-persistence`), which is the same whichever platform is chosen. Platform-specific sections are labeled:

- **[COND-A]**: only if the web platform is chosen (recommended).
- **[COND-B]**: only if the customer gives documented approval for Swing. Swing does **not** satisfy NFR-001.

The platform decision (OQ-1) is still **unresolved**.

---

## 1. Package and class structure

Base package `splitsmart` **[PROPOSAL]**. One package per domain concept. Classes are immutable unless noted.

### 1.1 `splitsmart-domain` (pure Java 21; JDK only)

| Package | Class / type | Kind | Responsibility |
|---|---|---|---|
| `domain.money` | `CurrencyCode` | record | Validated ISO-4217 code. Answers `scale()`. |
| | `CurrencyPolicy` | utility | Decides which currencies are supported (scale ∈ {0, 2, 3}) and returns their scale. |
| | `Money` | value object | A `BigDecimal` amount (always at currency scale) plus a `CurrencyCode`. Exact arithmetic only (§2). |
| | `Percentage` | value object | 0–100 with ≤ 2 decimals, held internally as **basis points** (`int` 0–10000). |
| `domain.allocation` | `SplitMethod` | enum | `EQUAL`, `PERCENTAGE` |
| | `ParticipantWeight` | record | `(AccountId, Percentage?)`, the input to allocation |
| | `ParticipantShare` | record | `(AccountId, Percentage?, Money share)` |
| | `Allocation` | value object | Ordered list of `ParticipantShare`. Guarantees Σ = amount. |
| | `AllocationCalculator` | stateless service | Largest-remainder allocation (§3) |
| `domain.expense` | `ExpenseStatus` | enum | `PENDING`, `APPROVED`, `REJECTED` |
| | `ExpenseDraft` | record | Already-parsed submission fields (amount, date, description, method, payer, weights) |
| | `ExpenseValidator` | stateless service | Collects **all** field errors for a draft against a `GroupContext` (§4.2) |
| | `Expense` | aggregate root (mutable, versioned) | Holds payer, participants, allocation, status. Enforces the state machine (§5.1). |
| `domain.review` | `ReviewPolicy` | stateless service | Decides whether an actor may decide a given expense |
| | `ReviewDecision` | entity | `(expenseId, reviewerId, decision, reason?, decidedAt, level = 1)` |
| `domain.group` | `GroupRole`, `MembershipStatus`, `InvitationStatus` | enums | As in data-model §1.5–1.6 |
| | `Group`, `Membership`, `Invitation` | entities | Group settings, membership state, invitation state |
| | `GroupContext` | record | A snapshot used for validation: `groupId`, `currency`, `selfApprovalAllowed`, `activeMemberIds`, `managerIds`, `creatorId` |
| | `GroupMembershipRules` | stateless service | Preconditions for invite, role change, remove, and leave (§5.5) |
| `domain.balance` | `DirectedSums` | value object | `Map<(debtor, creditor), Money>` for expense debts and `Map<(sender, recipient), Money>` for payments, in one currency |
| | `PairBalance` | record | `(debtor, creditor, Money amount > 0)` |
| | `PairwiseBalanceCalculator` | stateless service | Nets directed sums into pair balances and member nets (§6) |
| `domain.payment` | `PaymentDraft` | record | sender, recipient, amount, date, note |
| | `PaymentRules` | stateless service | Field validation and the outstanding-amount check (§7) |
| `domain.report` | `DateRange` | value object | Inclusive range; ≤ 5 years |
| | `ReportQuery` | record | Scope and filters (contracts §6) |
| `domain.error` | `ErrorCode` | enum | The single shared code list (contracts §7) |
| | `FieldError` | record | `(fieldPath, ErrorCode, messageKey, args)` |
| | `ValidationResult` | value object | Accumulates `FieldError`s. `throwIfInvalid()` raises `ValidationException`. |
| | `DomainException` (sealed) | exception hierarchy | §10.1 |
| `domain.common` | `AccountId`, `GroupId`, `ExpenseId`, `PaymentId`, … | records wrapping `UUID` | Type-safe IDs. `canonicalKey()` returns the lower-case string form, used for ordering (§3.3). |

Rules:

- The domain has no Spring, JPA, Swing, or servlet imports (ArchUnit-enforced).
- The domain never reads the system clock. Callers pass `today` (a `LocalDate`) or `now` (an `Instant`).

### 1.2 `splitsmart-application`

| Class | Responsibility |
|---|---|
| `ActorContext` (record `accountId`) | The authenticated caller. Built **only** by presentation from the session; never taken from request data. |
| `AccountService`, `GroupService`, `ExpenseService`, `ReviewService`, `BalanceService`, `PaymentService`, `NotificationService`, `ReportService` | The use-case services listed in [contracts/application-services.md](contracts/application-services.md). Each public method is one transaction. |
| `AuthorizationService` | Visibility and role checks (§9) |
| `MoneyParser` | Turns raw input strings into `Money` and `Percentage`, producing field errors (§2.2). Shared by A's API mappers and B's presenters, so both platforms validate identically. |
| `NotificationComposer` | Works out the recipients and builds the summary text for each notification type (§11) |
| `IdempotencyGuard` | Looks up earlier results by key and compares request hashes (§8.4) |
| `*Command`, `*Query`, `*View` records | Inputs and outputs. Views contain IDs, display names, and `BigDecimal` amounts at currency scale. |

**Ports**, interfaces implemented by infrastructure:

| Port | Key methods (contract) |
|---|---|
| `AccountRepository` | `Optional<Account> findById(AccountId)`; `Optional<Account> findByEmail(String normalized)`; `Account save(Account)` |
| `GroupRepository` | `Optional<Group> findById(GroupId)`; **`Group lockForUpdate(GroupId)`** (pessimistic row lock until commit); `Group save(Group)`; `boolean hasExpenses(GroupId)` |
| `MembershipRepository` | `Optional<Membership> find(GroupId, AccountId)`; `List<Membership> findActive(GroupId)`; `List<Membership> findByAccount(AccountId, Set<MembershipStatus>)`; `save` |
| `InvitationRepository` | `findById`; `findPending(GroupId, email)`; `findPendingForAccount(AccountId)`; `save` |
| `ExpenseRepository` | `findById`; `Page<ExpenseSummary> search(GroupId, status?, DateRange?, PageRequest)`; `insert(Expense)`; **`int updateIfPending(Expense, long expectedVersion)`** (conditional update, returns rows affected); `findByIdempotencyKey(GroupId, AccountId, UUID)`; `boolean hasPendingInvolving(GroupId, AccountId)` |
| `ReviewDecisionRepository` | `insert(ReviewDecision)`; `Optional<ReviewDecision> findByExpense(ExpenseId)` |
| `PaymentRepository` | `insert(Payment)`; `search(GroupId, DateRange?, partyFilter?, PageRequest)`; `findByIdempotencyKey(...)`; `boolean existsSimilarSince(GroupId, sender, recipient, Money, LocalDate, Instant since)` |
| `NotificationRepository` | `insertAll(List<Notification>)`; `page(AccountId, unreadOnly, PageRequest)`; `countUnread(AccountId)`; `markRead(AccountId, Set<UUID>)` |
| `BalanceQueryPort` | `DirectedSums directedSums(GroupId)`; `DirectedSums directedSumsForPair(GroupId, AccountId a, AccountId b)`; `List<GroupDirectedSums> directedSumsInvolving(AccountId)` |
| `ReportQueryPort` | `List<ReportRow> rows(ReportScope, ReportQuery)`; `SummaryFigures summary(ReportScope)` |
| `TransactionRunner` | `<T> T inTransaction(Supplier<T>)`; `<T> T readOnly(Supplier<T>)` |
| `Clock` | `java.time.Clock` (UTC), injected |
| `PasswordHasher` | `String hash(char[])`; `boolean matches(char[], String hash)` |
| `SecureTokenGenerator` | `String newToken()` (256-bit, base64url); `String sha256Hex(String)` |
| `RecoveryChannel` | [COND-A] `sendResetLink(email, token)`. [COND-B] not used; recovery codes are handled in `AccountService`. |
| `ReceiptStorage` [COND-A] | `StoredObject put(InputStream, contentType)`; `InputStream open(String key)`; `void delete(String key)` |
| `NotificationPublisher` | `void published(List<Notification>)`, called after commit. The MVP implementation does nothing; it is the hook for future email or push channels (OQ-3). |

### 1.3 `splitsmart-persistence`

- JPA entities mirror data-model §1 one-to-one. They live in `persistence.jpa` and are **separate** from domain classes. Mappers convert in both directions, which keeps JPA annotations out of the domain.
- `MoneyColumns`: a mapping helper. On read it applies `setScale(currency.scale(), RoundingMode.UNNECESSARY)`, so a stored value with excess precision fails loudly instead of being rounded (§2.4).
- Repository adapters implement the ports above. `lockForUpdate` uses `PESSIMISTIC_WRITE` (`SELECT … FOR UPDATE`) with a lock timeout of 5 s **[PROPOSAL]**.
- `JdbcBalanceQuery` runs the aggregate SQL in §6.2. `JdbcReportQuery` runs §12.
- Flyway scripts live in `db/migration` (shared) and `db/migration/postgresql` (the partial unique index).

### 1.4 Presentation modules

- [COND-A] `splitsmart-web`: §13.
- [COND-B] `splitsmart-desktop`: §14.

---

## 2. Money handling with `BigDecimal`

### 2.1 Currency scale

`CurrencyPolicy.scaleOf(code) = java.util.Currency.getInstance(code).getDefaultFractionDigits()`.

- Supported only if the scale is 0, 2, or 3. Otherwise `CURRENCY_UNSUPPORTED`.
- The scale of a group never changes after its first expense (`CURRENCY_LOCKED`).

### 2.2 Parsing input (`MoneyParser`)

Input arrives as text: JSON strings in A, text fields in B. Parsing steps:

1. Trim. Reject an empty value → `AMOUNT_FORMAT_INVALID`.
2. Match `^-?\d{1,9}(\.\d{1,9})?$`. This rejects exponents (`1E+2`), grouping separators, currency symbols, and `+`. A mismatch → `AMOUNT_FORMAT_INVALID`.
3. `new BigDecimal(text)`. Always from a string, **never** `BigDecimal.valueOf(double)` or `new BigDecimal(double)`.
4. If `value.signum() <= 0` → `AMOUNT_NOT_POSITIVE`.
5. If `value.stripTrailingZeros().scale() > s` → `AMOUNT_PRECISION_EXCEEDED`. `"12.50"` and `"12.5"` are fine for USD. `"12.505"` is not. `"100"` is fine: `stripTrailingZeros` gives `1E+2`, whose scale is −2, which is ≤ s.
6. If `value.compareTo(1_000_000) > 0` → `AMOUNT_TOO_LARGE`.
7. `Money.of(value.setScale(s, RoundingMode.UNNECESSARY), currency)`. This is exact because step 5 has passed.

**Percentages** follow the same steps with the pattern `^\d{1,3}(\.\d{1,2})?$`:

- Format errors → `PERCENTAGE_FORMAT_INVALID`.
- A leading `-` → `PERCENTAGE_NEGATIVE`. The regex check is done in an order that reports this code rather than a format error.
- > 100 → `PERCENTAGE_OUT_OF_RANGE`.
- More than 2 decimals → `PERCENTAGE_PRECISION_EXCEEDED`.
- Stored as `basisPoints = value.movePointRight(2).intValueExact()`.

[COND-A] JSON numbers in money or percentage fields are rejected with the same format codes. The DTO field type is `String`.

### 2.3 Arithmetic rules

| Operation | Rule |
|---|---|
| `plus` / `minus` | Same currency required (otherwise `IllegalArgumentException`, a programming error). The result stays at scale s because adding values of equal scale keeps that scale. |
| Comparison | `compareTo` only. `equals`/`hashCode` on `Money` compare `(currency, amount.compareTo)`, normalized to scale s at construction, so the two always agree. |
| Division | **Never** on `Money`. All division happens in `AllocationCalculator` on `long` minor units (§3). |
| Minor units | `toMinorUnits() = amount.movePointRight(s).longValueExact()`. `fromMinorUnits(long, currency)` is the inverse. |
| Overflow bounds | Max amount 1,000,000 × 10³ = 10⁹ minor units. Products in allocation are ≤ 10⁹ × 10⁴ = 10¹³, well below `Long.MAX_VALUE` (≈ 9.2 × 10¹⁸). `Math.multiplyExact` / `addExact` are used anyway, so an overflow fails loudly. Balance sums use `BigDecimal` and cannot overflow. |
| Rounding | No `RoundingMode` other than `UNNECESSARY` appears anywhere in the money code. Any rounding would be a bug and throws `ArithmeticException`. |

### 2.4 Persistence and display

- **Columns:** `NUMERIC(19,4)`. Written at scale s. Read back through `MoneyColumns` with `setScale(s, UNNECESSARY)`.
- **Display:**
  - [COND-A] the API returns `amount.toPlainString()` (never `toString()`, which can produce exponents). The browser formats strings for display without converting to a float for arithmetic.
  - [COND-B] views receive pre-formatted strings from presenters (`NumberFormat.getCurrencyInstance` with the currency set).

---

## 3. Split calculation (`AllocationCalculator`)

### 3.1 Contract

```
Allocation allocate(Money amount, SplitMethod method, List<ParticipantWeight> weights)
  pre:  amount > 0; weights non-empty, distinct accounts;
        EQUAL → no percentages; PERCENTAGE → all present, Σ basisPoints = 10000
        (callers run ExpenseValidator first; a violated precondition throws IllegalArgumentException = bug)
  post: Σ share = amount exactly; every share ≥ 0;
        each share differs from its exact proportional value by < 1 minor unit;
        result ordered by canonical account order (§3.3); deterministic for equal inputs
```

### 3.2 Algorithm (all integer arithmetic)

```
A  ← amount.toMinorUnits()                       // long
EQUAL:      w_i ← 1          ; W ← n
PERCENTAGE: w_i ← bp_i       ; W ← 10000         // basis points
for each i:
    num_i   ← A × w_i                            // multiplyExact
    floor_i ← num_i ÷ W                          // integer division (all values ≥ 0)
    rem_i   ← num_i mod W                        // integer remainder, 0 ≤ rem_i < W
L ← A − Σ floor_i                                // 0 ≤ L < n  (proof: Σ rem_i / W = L and each rem_i/W < 1)
rank participants by (rem_i descending, canonicalKey ascending)
share_i ← floor_i + (1 if i is among the first L ranked else 0)
assert Σ share_i = A                             // invariant check; failure throws IllegalStateException
return shares converted with fromMinorUnits, ordered by canonicalKey
```

Remainders are integers, so comparing them is exact. No fractional `BigDecimal` division is needed. This is the same as the formula in [architecture.md §6.1](architecture.md#61-monetary-precision-and-allocation-decision) (`raw_i = A × p_i / 100`), written in basis points.

### 3.3 Deterministic tie-breaking

- **Rule:** among equal remainders, the participant whose `AccountId.canonicalKey()` (the lower-case 36-character UUID string) sorts **first** lexicographically gets the extra minor unit.
- **Do not use `java.util.UUID.compareTo`.** It compares signed 64-bit halves, which gives a **different order** from the string form for UUIDs whose high bit is set. Using the string form keeps Java, SQL (`ORDER BY account_id::text`), and test fixtures consistent.
- **Why not submission order:** that would make the result depend on UI checkbox order and would require storing the order. The account ID is stable and already stored.
- **Shares are stored** (`expense_participant.share_amount`), so a later change to the algorithm cannot change historical expenses.

### 3.4 Worked cases (become `AllocationCalculatorTest` cases; values checked with a throwaway script during authoring)

| # | Input | Expected |
|---|---|---|
| 1 | $100.00, EQUAL, 3 (ids k1<k2<k3) | 33.34, 33.33, 33.33 |
| 2 | $10.00, PCT 33.33/33.33/33.34 | bp 3333/3333/3334 → num 3,333,000/3,333,000/3,334,000; floors 333/333/333; rems 3000/3000/4000; L=1 → k3: 3.33, 3.33, **3.34** |
| 3 | $0.05, PCT 50/50 | floors 2/2, rems 5000/5000, L=1 → k1: 0.03, 0.02 |
| 4 | ¥1000, EQUAL, 3 | 334, 333, 333 |
| 5 | $20.00, PCT 100/0 | 20.00, 0.00 |
| 6 | $0.01, EQUAL, 3 | 0.01, 0.00, 0.00 |
| 7 | $1,000,000.00, PCT 33.33/33.33/33.34 | 333,300.00 / 333,300.00 / 333,400.00 (no overflow) |
| 8 | Property: random A ∈ [1, 10⁸], n ∈ [1, 8], random bp summing to 10000 | Σ = A; every \|share − exact\| < 1 minor unit; same input → same output |

---

## 4. Validation design

### 4.1 Order of checks (every command)

1. **Authenticated?** Otherwise `NOT_AUTHENTICATED`. Enforced by presentation.
2. **Resource visible to the actor?** Otherwise `NOT_FOUND`.
3. **Actor has the required role or relationship?** Otherwise `FORBIDDEN_ROLE`, `PAYMENT_NOT_SENDER`, or `SELF_APPROVAL_NOT_ALLOWED`.
4. **Field validation.** Collect **every** `FieldError`, then throw `ValidationException(VALIDATION_FAILED, errors)`.
5. **Business and state rules** (need current state, usually under lock). For example `EXPENSE_NOT_PENDING`, `PAYMENT_EXCEEDS_OUTSTANDING`, `MEMBER_HAS_OUTSTANDING_BALANCE`.
6. **Persist.** Optimistic or conditional failures → `CONCURRENT_MODIFICATION`.

Steps 2–3 run before step 4, so a non-member never learns anything from validation messages.

### 4.2 `ExpenseValidator.validate(draft, GroupContext ctx, LocalDate today) → ValidationResult`

| Field path | Check | Code |
|---|---|---|
| `amount` | §2.2 steps 1–6, at `ctx.currency` scale | `AMOUNT_FORMAT_INVALID`, `AMOUNT_NOT_POSITIVE`, `AMOUNT_PRECISION_EXCEEDED`, `AMOUNT_TOO_LARGE` |
| `expenseDate` | present; ≤ `today + 1` | `VALIDATION_FAILED`, `DATE_IN_FUTURE` |
| `description` | trimmed length 1–200 | `DESCRIPTION_INVALID` |
| `splitMethod` | present | `VALIDATION_FAILED` |
| `payerId` | ∈ `ctx.activeMemberIds` | `PAYER_NOT_MEMBER` |
| `participants` | non-empty | `PARTICIPANTS_REQUIRED` |
| `participants[i].accountId` | no duplicates | `PARTICIPANT_DUPLICATE` |
| `participants[i].accountId` | ∈ `ctx.activeMemberIds` | `PARTICIPANT_NOT_MEMBER` |
| `participants[i].percentage` | EQUAL → absent; PERCENTAGE → present and valid (§2.2) | `PERCENTAGE_NOT_ALLOWED`, `PERCENTAGE_REQUIRED`, `PERCENTAGE_FORMAT_INVALID`, `PERCENTAGE_NEGATIVE`, `PERCENTAGE_OUT_OF_RANGE`, `PERCENTAGE_PRECISION_EXCEEDED` |
| `participants` | PERCENTAGE and all individual percentages valid → Σ bp = 10000 | `PERCENTAGES_NOT_100` (message includes the actual total) |

Notes:

- The payer is **not** required to be in `participants`, and is **not** forbidden from it (FR-003, assumption AS-12).
- Non-member account IDs and existing-but-unrelated IDs produce the same code, so the validator reveals nothing about other accounts.

### 4.3 Other validators

| Validator | Rules (codes from contracts) |
|---|---|
| `PaymentRules.validateFields(draft, ctx, actor, today)` | `actor == sender` → else `PAYMENT_NOT_SENDER` (checked at step 3). `sender ≠ recipient` → `PAYMENT_SELF`. recipient ∈ active members → `RECIPIENT_NOT_MEMBER`. Amount per §2.2. Date ≤ today+1 → `DATE_IN_FUTURE`. Note ≤ 200 characters. |
| `AccountValidator` | Email: lower-cased, trimmed, ≤ 254, one `@`, non-empty local and domain parts (deliberately not a full RFC 5322 parse). Password: 12–128 characters, no other composition rules **[PROPOSAL]**. Display name 1–80. |
| `GroupValidator` | Name 1–80; description ≤ 500; currency supported. |
| `DateRange.of(from, to)` | Both present; `to ≥ from`; span ≤ 1827 days (5 years) → else `INVALID_DATE_RANGE`. |

---

## 5. Domain state machines and the operations that drive them

### 5.1 `Expense` aggregate

| Method | Guard (in this order) | Effect |
|---|---|---|
| `static Expense submit(ExpenseId, GroupContext, AccountId submitter, ExpenseDraft, Allocation, Instant now)` | Validated draft; allocation matches draft | status PENDING, version 0, `submittedAt = now` |
| `void edit(AccountId actor, long expectedVersion, ExpenseDraft, Allocation, Instant now)` | `actor == submittedBy` else `FORBIDDEN_ROLE`; `status == PENDING` else `EXPENSE_NOT_PENDING`; `version == expectedVersion` else `CONCURRENT_MODIFICATION` | Replaces fields and allocation; `lastEditedAt = now` |
| `ReviewDecision approve(AccountId reviewer, long expectedVersion, String reason?, Instant now)` | `status == PENDING` else `EXPENSE_NOT_PENDING`; version matches else `CONCURRENT_MODIFICATION` | status APPROVED; returns a level-1 decision |
| `ReviewDecision reject(…)` | same | status REJECTED |
| `boolean involves(AccountId)` | none | true if the account is the payer, a participant, or the submitter |

`ReviewPolicy` is checked by `ReviewService` **before** `approve`/`reject` is called (role before state, §4.1). The repository's conditional update (§8.2) repeats the status and version guard in the database, so the in-memory check is not the only protection.

### 5.2 `ReviewPolicy.check(actorId, Expense e, GroupContext ctx) → Decision`

```
if actorId ∉ ctx.activeMemberIds                       → DENY NOT_FOUND
if actorId ∉ ctx.managerIds                            → DENY FORBIDDEN_ROLE
if actorId == e.submittedBy and !ctx.selfApprovalAllowed → DENY SELF_APPROVAL_NOT_ALLOWED
otherwise                                               → ALLOW
```

`listPendingReviewsForMe` uses the same policy, so the UI only offers actions that will succeed.

**Deadlock check:** when an expense is submitted and its submitter is the **only** manager and self-approval is off, `ExpenseView` carries `reviewBlocked = true`. The UI then explains the situation (architecture §4.4).

### 5.3 Invitation

| Method | Guard | Effect |
|---|---|---|
| `Invitation.create(group, email, invitedBy, inviteeAccount?, now)` | inviter is a manager (via `GroupMembershipRules`) | PENDING; `expiresAt = now + 14 days` |
| `effectiveStatus(now)` | none | Returns `EXPIRED` if PENDING and `now ≥ expiresAt`. Expiry is **lazy**: no scheduler. The stored value is updated the next time the invitation is touched. |
| `accept(actor, now)` | `actor == inviteeAccountId` else `NOT_FOUND`; effective status PENDING else `INVITATION_NOT_PENDING` | ACCEPTED |
| `decline(actor, now)` | same | DECLINED |
| `revoke(actorIsManager, now)` | manager; PENDING | REVOKED |

### 5.4 Membership

`Membership.activate(role = MEMBER, now)` creates a new membership or reactivates a REMOVED or LEFT row. Role and status changes follow data-model §6.3.

### 5.5 `GroupMembershipRules`

| Check | Inputs | Failure code |
|---|---|---|
| `requireCanManage(actorMembership)` | role | `FORBIDDEN_ROLE` |
| `requireCanDemote(target, ctx)` | target ≠ creator; managers after the change ≥ 1 | `CANNOT_DEMOTE_CREATOR`, `LAST_MANAGER` |
| `requireCanEnd(target, ctx, hasNonZeroBalance, hasPendingInvolvement)` | target ≠ creator; if target is a manager, managers after the change ≥ 1; balance zero; no pending expense where the target is payer or participant | `CANNOT_REMOVE_CREATOR`, `LAST_MANAGER`, `MEMBER_HAS_OUTSTANDING_BALANCE`, `MEMBER_HAS_PENDING_EXPENSES` |

---

## 6. Balance calculation

### 6.1 `PairwiseBalanceCalculator`

```
PairNet net(DirectedSums s, AccountId a, AccountId b)
    N = s.debt(a→b) − s.debt(b→a) − s.paid(a→b) + s.paid(b→a)        // Money arithmetic, exact
    return N > 0 ? owes(a, b, N) : N < 0 ? owes(b, a, −N) : settled(a, b)

GroupBalances all(DirectedSums s, Set<AccountId> people)
    for each unordered pair {a,b} appearing in s (not all n² pairs): net(...)
    keep non-zero pairs; sort by debtor display order then creditor
    memberNet(x) = Σ over pairs (owed to x) − Σ (owed by x)
    assert Σ memberNet = 0                                            // invariant I-15
```

- **Input size:** at most one directed entry per ordered pair that ever had activity. That is ≤ 50 × 49 = 2,450 entries at NFR-004 scale, so netting is trivial.
- **`people`** includes former members who still have activity, so historical pairs are never dropped. Removal requires a zero balance anyway (D-2).

### 6.2 Aggregate query (illustrative SQL for `JdbcBalanceQuery.directedSums`)

```sql
-- expense debts: participant owes payer, approved only, payer's own share excluded
SELECT ep.account_id AS debtor, e.payer_id AS creditor, SUM(ep.share_amount) AS amount
FROM expense e JOIN expense_participant ep ON ep.expense_id = e.id
WHERE e.group_id = :g AND e.status = 'APPROVED' AND ep.account_id <> e.payer_id
GROUP BY ep.account_id, e.payer_id;

-- payments
SELECT sender_id, recipient_id, SUM(amount) FROM payment WHERE group_id = :g
GROUP BY sender_id, recipient_id;
```

- `directedSumsForPair` adds `AND {debtor, creditor} = {:a, :b}` predicates. It is used inside the payment transaction to keep the locked section short.
- **Supporting indexes:** `expense(group_id, status, …)`, `expense_participant(expense_id)` (the primary key), `payment(group_id, sender_id, recipient_id)` (data-model §2).
- **Consistency test:** `BalanceQueryMatchesCalculatorIT` loads random fixtures, computes directed sums both in SQL and in memory from the raw rows, and asserts they are equal.

### 6.3 When balances change

Balances are not stored. They change exactly when one of these commits:

1. An expense goes PENDING → APPROVED.
2. A payment is inserted.

Nothing else affects them: not submission, edit, rejection, or membership changes. This is the whole basis of FR-006, FR-009, and FR-010, and of the concurrency design in §8.

### 6.4 Cross-group view (`getMyBalances`)

1. `directedSumsInvolving(me)` returns directed sums per group, limited to rows where I am debtor or creditor (or sender or recipient), across all groups where I am ACTIVE or was a PARTY.
2. The per-group calculation runs on each.
3. `byCounterparty` sums pair nets per `(counterparty, currency)`. It never adds amounts in different currencies.

---

## 7. Payment recording design

`PaymentService.recordPayment(actor, cmd)`:

```
1  idem ← IdempotencyGuard.check(PAYMENT, cmd.groupId, actor, cmd.idempotencyKey, hash(cmd))
     → same key & hash: return stored PaymentView (no new row)       → different hash: IDEMPOTENCY_KEY_REUSED
2  inTransaction:
3    group ← groupRepo.lockForUpdate(cmd.groupId)                    // waits ≤ 5 s; timeout → CONCURRENT_MODIFICATION
4    authz.requireActiveMember(actor, group)                         // NOT_FOUND
5    if actor ≠ cmd.senderId → PAYMENT_NOT_SENDER
6    PaymentRules.validateFields(...).throwIfInvalid()               // all field errors
7    sums ← balanceQuery.directedSumsForPair(group, sender, recipient)
8    N    ← calculator.net(sums, sender, recipient) as "amount sender owes recipient" (≤ 0 if none)
9    if N ≤ 0              → NOTHING_OWED
10   if amount > N         → PAYMENT_EXCEEDS_OUTSTANDING {outstanding: N}
11   if !cmd.confirmPossibleDuplicate and paymentRepo.existsSimilarSince(group, s, r, amount, date, now − 24h)
                           → POSSIBLE_DUPLICATE_PAYMENT
12   payment ← Payment(id, group, s, r, amount, date, note, recordedBy = actor, recordedAt = clock.now(),
                       idempotencyKey, requestHash)
13   paymentRepo.insert(payment)                                     // unique-key violation → step 1 replay
14   return PaymentView + PairBalance(N − amount)
```

**Why the lock is taken before validation:** steps 4–6 need the group context (members, currency). Taking the lock first means one consistent read covers both validation and the outstanding check.

**Why the group lock, not a pair lock:** at the group sizes involved, contention is negligible. One lock rule ("balance-affecting writes lock the group row") is easy to review. Approval takes the same lock (§8.2).

**Immediate effect (AC-016):** the payment commits in this transaction, and balances are derived on read, so the next read (including the step-14 response) reflects it.

---

## 8. Transactions and concurrency

### 8.1 Transaction boundaries

| Use case | Transaction contents | Lock(s) |
|---|---|---|
| `submitExpense` | expense row + participant rows + notification rows | none (no balance effect) |
| `editPendingExpense` | conditional update of the expense (`status = PENDING AND version = ?`) + replace participants | optimistic only |
| `approveExpense` / `rejectExpense` | group lock → conditional update → decision row → notification rows | group row |
| `recordPayment` | §7 steps 3–13 | group row |
| `removeMember` / `leaveGroup` | group lock → balance and pending checks → membership update | group row |
| `setManager` | group lock → manager-count check → membership update | group row |
| `respondToInvitation` (accept) | invitation update + membership insert or reactivate | invitation row (optimistic version) |
| `inviteMember` | invitation insert (+ notification) | the unique pending index stops concurrent duplicates → `INVITATION_ALREADY_PENDING` |
| Reads (balances, reports, lists) | read-only transaction, `READ COMMITTED` | none |

**After commit:** `NotificationPublisher.published(...)` and receipt-file cleanup run only after a successful commit, through a `TransactionRunner` after-commit hook.

### 8.2 Approval and edit races

```
approveExpense(actor, id, expectedVersion):
  inTransaction:
    e     ← expenseRepo.findById(id)            → NOT_FOUND
    group ← groupRepo.lockForUpdate(e.groupId)   // lock order: group FIRST, then expense
    ctx   ← groupContext(group)
    ReviewPolicy.check(actor, e, ctx)            → NOT_FOUND / FORBIDDEN_ROLE / SELF_APPROVAL_NOT_ALLOWED
    decision ← e.approve(actor, expectedVersion, reason, now)   → EXPENSE_NOT_PENDING / CONCURRENT_MODIFICATION
    rows  ← expenseRepo.updateIfPending(e, expectedVersion)
            -- UPDATE expense SET status=:s, version=version+1 WHERE id=:id AND status='PENDING' AND version=:v
    if rows = 0 → reload: status ≠ PENDING ? EXPENSE_NOT_PENDING : CONCURRENT_MODIFICATION
    reviewRepo.insert(decision); notificationRepo.insertAll(composer.reviewed(e, decision))
```

| Race | Outcome |
|---|---|
| Two managers approve at the same moment | Serialized by the group lock. The second sees status APPROVED → `EXPENSE_NOT_PENDING`. |
| Approve vs. reject | Same. Exactly one decision row (also guaranteed by `UNIQUE(expense_id, level)`). |
| Submitter edits while a manager approves the **old** version | The edit takes no group lock. Whichever conditional update commits first wins. If the edit wins, the approval's `expectedVersion` is stale → `CONCURRENT_MODIFICATION`, so a manager **can never approve content they have not seen**. If the approval wins, the edit gets `EXPENSE_NOT_PENDING`. |
| Approval during a payment validation | Serialized by the group lock. The payment check always sees a committed set of approvals. |

**Deadlock avoidance:** every transaction that locks rows takes the group row **first** and touches at most one group. There is no cross-group locking anywhere.

### 8.3 Isolation level

PostgreSQL `READ COMMITTED` (the default) is enough, because every check-then-write sequence is protected by either:

- the group row lock, or
- a conditional `UPDATE … WHERE` that re-checks the precondition in the database.

H2 (COND-B) supports `SELECT … FOR UPDATE` with the same semantics. `SERIALIZABLE` was considered and rejected because every caller would need retry loops ([research R-11](research.md#r-11-concurrency-control)).

### 8.4 Idempotency (`IdempotencyGuard`)

- **Scope:** `submitExpense` and `recordPayment`.
- **Key:** client-supplied UUID ([COND-A] the `Idempotency-Key` header; [COND-B] generated by the presenter for each dialog open).
- **Stored:** the key and `idempotency_request_hash` = SHA-256 of a canonical serialization of the command (fields in fixed order, amounts as plain strings).
- **Check flow:**
  1. Look up `(group, actor, key)`. If found with the same hash → return the stored result. If found with a different hash → `IDEMPOTENCY_KEY_REUSED`.
  2. If not found → proceed. If two identical requests race, the second insert hits the unique constraint. The service catches it, rolls back, and **replays step 1**, returning the first result.
- Without a key there is no deduplication. For payments, the possible-duplicate check (§7 step 11) still applies.

---

## 9. Authorization design

### 9.1 `AuthorizationService`

| Method | Returns / throws |
|---|---|
| `Membership requireActiveMember(AccountId actor, GroupId g)` | ACTIVE membership, else `NOT_FOUND` (also when the group does not exist) |
| `Membership requireManager(AccountId actor, GroupId g)` | `requireActiveMember`, then role MANAGER, else `FORBIDDEN_ROLE` |
| `void requireExpenseVisible(AccountId actor, Expense e)` | ACTIVE member of e's group, **or** a former member with `e.involves(actor)`; else `NOT_FOUND` |
| `void requirePaymentVisible(AccountId actor, Payment p)` | ACTIVE member, or a former member who is sender or recipient; else `NOT_FOUND` |
| `ReportScope scopeFor(AccountId actor, GroupId? g)` | The set of group IDs where the actor is ACTIVE, plus a party-only filter for former groups. If `g` is given and the actor is not ACTIVE in it → `NOT_FOUND`. |

Every repository query that returns group data takes a `GroupId` or `ReportScope` that **came from this service**, never directly from request parameters.

### 9.2 Operation × relationship matrix

✅ allowed · 404 → `NOT_FOUND` · 403 → `FORBIDDEN_ROLE` (or the specific permission code)

| Operation | Non-member | Former member (party) | Member | Manager | Specific relationship required |
|---|---|---|---|---|---|
| View group / members | 404 | 404 | ✅ | ✅ | none |
| Update group settings | 404 | 404 | 403 | ✅ | none |
| Invite / revoke invitation / list invitations | 404 | 404 | 403 | ✅ | none |
| Accept / decline invitation | 404 | n/a | n/a | n/a | invitee only (others 404) |
| Set manager / remove member | 404 | 404 | 403 | ✅ | rules in §5.5 |
| Leave group | 404 | 404 | ✅ | ✅ | self; §5.5 rules |
| Submit / preview expense | 404 | 404 | ✅ | ✅ | none |
| Edit pending expense / attach receipt | 404 | 404 | 403 unless submitter | 403 unless submitter | submitter |
| View expense / receipt | 404 | ✅ if party, else 404 | ✅ | ✅ | none |
| Approve / reject | 404 | 404 | 403 | ✅ (not own unless `selfApprovalAllowed` → `SELF_APPROVAL_NOT_ALLOWED`) | §5.2 |
| View group balances | 404 | 404 | ✅ | ✅ | none |
| My balances / my summary | n/a | own pairs only | ✅ | ✅ | self |
| Record payment | 404 | 404 | ✅ as sender | ✅ as sender | sender (`PAYMENT_NOT_SENDER`) |
| List payments | 404 | party rows only | ✅ | ✅ | none |
| Notifications | n/a | n/a | own only | own only | recipient |
| Custom report | groups silently limited to `scopeFor` | party rows only | ✅ | ✅ | none |

`AuthorizationMatrixTest` is generated from this table: one test per cell.

---

## 10. Error handling design

### 10.1 Exception hierarchy (`domain.error`)

```
sealed DomainException(ErrorCode code, Map<String,Object> details) extends RuntimeException
 ├── ValidationException        (code = VALIDATION_FAILED, List<FieldError> errors)
 ├── NotFoundException          (code = NOT_FOUND)                       -- no details ever
 ├── PermissionException        (FORBIDDEN_ROLE | PAYMENT_NOT_SENDER | SELF_APPROVAL_NOT_ALLOWED)
 ├── BusinessRuleException      (422-category codes; details e.g. {outstanding: "25.00"})
 ├── ConflictException          (409-category codes: EXPENSE_NOT_PENDING, CONCURRENT_MODIFICATION, …)
 └── AuthenticationException    (INVALID_CREDENTIALS, RESET_TOKEN_INVALID, RATE_LIMITED, NOT_AUTHENTICATED)
```

### 10.2 Rules

- **Messages:** domain code uses message **keys** (for example `error.percentages.not100`) with arguments. Presentation localizes them. English only for now.
- **Details:** only values the actor is allowed to see. `NotFoundException` never carries details.
- **Unexpected exceptions** (`IllegalStateException` from an invariant check, `ArithmeticException`, DB errors) are logged at ERROR with a correlation ID and the actor ID. The actor receives `INTERNAL_ERROR` with the correlation ID only.
- **Logging rules:** never log passwords, reset tokens, recovery codes, session IDs, or receipt bytes. Amounts may be logged at DEBUG.
- **Mapping to the platform:**
  - [COND-A] `ProblemDetailsAdvice` maps each exception type to the HTTP status in [contracts/application-services.md §7](contracts/application-services.md#7-error-codes).
  - [COND-B] `ErrorPresenter` maps codes to field messages or dialogs.

---

## 11. Notification design

| Trigger (in the same transaction) | Type | Recipients | Summary template (built from data the recipient may see) |
|---|---|---|---|
| `submitExpense` | `EXPENSE_SUBMITTED` | All ACTIVE members except the submitter (FR-008) | "{submitter} submitted '{description}' ({amount} {currency}), pending review" |
| `approveExpense` / `rejectExpense` | `EXPENSE_REVIEWED` | Submitter ∪ payer ∪ participants, minus the reviewer, limited to ACTIVE members | "{reviewer} approved/rejected '{description}'" (+ reason for rejection) |
| `inviteMember` (account exists) | `INVITATION_RECEIVED` | Invitee | "{inviter} invited you to {group}" |

Notes:

- Editing a pending expense sends no notification in the MVP. That is open under OQ-6.
- **Unread count:** `countUnread` uses the index `notification(recipient_id, read_at, created_at DESC)`.
- [COND-A] the client polls `/me/notifications/unread-count` every 60 s.
- [COND-B] the count refreshes on sign-in and on each screen change.
- **Future channels:** `NotificationPublisher` receives the committed notifications after commit, so adding email later (OQ-3) needs no change to the services.

---

## 12. Reporting design

### 12.1 Summary (`getSummary`), in the candidate MVP

For each currency in scope:

| Figure | Definition |
|---|---|
| `totalPaidByMe` | Σ amount of APPROVED expenses where payer = me |
| `myTotalShare` | Σ my share_amount over APPROVED expenses |
| `pendingCount` | Count of PENDING expenses in scope that I submitted, pay, or participate in |
| `owedToMe` / `iOwe` | Sum of positive / negative pair nets involving me (§6.4) |
| `paymentsSent` / `paymentsReceived` | Σ payments where I am sender / recipient |

### 12.2 Custom report (`runReport`), full scope and deferred in the candidate MVP

1. `DateRange.of(from, to)` → `INVALID_DATE_RANGE` if invalid (AC-019).
2. `scope ← authz.scopeFor(actor, groupId?)`.
3. Rows (illustrative):
   ```sql
   SELECT 'EXPENSE' AS type, e.expense_date AS d, … FROM expense e
   WHERE e.group_id IN (:activeGroups) AND e.status IN (:statuses)
     AND e.expense_date BETWEEN :from AND :to                       -- inclusive both ends
     [AND (:onlyMine = false OR e.payer_id = :me OR EXISTS (participant = :me) OR e.submitted_by = :me)]
   UNION ALL
   SELECT 'PAYMENT', p.payment_date, … FROM payment p
   WHERE p.group_id IN (:activeGroups) AND p.payment_date BETWEEN :from AND :to
     [AND (:onlyMine = false OR :me IN (p.sender_id, p.recipient_id))]
   -- plus the same two selects over :formerGroups restricted to rows where :me is a party
   ORDER BY d, type, id
   ```
4. Totals are computed per currency in Java (`Money.plus`).
5. **Balance movement** per counterparty: run the §6.1 netting over only the in-range APPROVED expenses and payments that involve me. This shows how much each relationship changed during the period. It is not the balance at a point in time.
6. An empty result returns a valid report with zero totals.

**Business dates, not timestamps:** filtering on `expense_date` / `payment_date` matches what users entered. A record dated on a boundary is included (AC-018 edge case). A record's approval time does not affect which period it falls in.

---

## 13. Web presentation design [COND-A]

### 13.1 Components

| Class | Responsibility |
|---|---|
| `AuthController`, `MeController`, `GroupController`, `InvitationController`, `ExpenseController`, `ReviewController`, `BalanceController`, `PaymentController`, `NotificationController`, `ReportController` | One per resource area in [contracts/rest-api.md](contracts/rest-api.md). Each method: resolve `ActorContext` → map DTO to command (raw strings stay strings) → call **one** service method → map view to DTO. No business logic. |
| `CurrentActorResolver` | Turns the Spring Security principal into `ActorContext` |
| `DtoMapper`s | View ↔ JSON. Money via `toPlainString()`. Timestamps as ISO-8601 UTC. |
| `ProblemDetailsAdvice` | `@RestControllerAdvice` that maps exceptions to RFC 9457 responses (§10) |
| `SecurityConfig` | Filter chain; form/JSON login endpoint; session-fixation protection (new session ID on login); `CookieCsrfTokenRepository`; CSP and HSTS headers; `/api/**` needs authentication except `/auth/**` |
| `SessionValidityFilter` | Rejects a session whose issue time is earlier than `account.password_changed_at`. This invalidates all sessions after a reset or password change. |
| `LoginRateLimiter` | In-memory buckets keyed by IP and by email: 5 failures per 15 min **[PROPOSAL]**. This is enough for a single instance. Multi-instance hosting would need a shared store. |
| `EmailRecoveryChannel` / `ConsoleRecoveryChannel` | SMTP in `prod`; logs the link in `local` |
| `ReceiptUploadHandler` | §13.3 |

### 13.2 Account recovery flow

1. `POST /auth/password-reset-requests {email}` → always 202.
   - If an account exists and is not rate-limited: `token ← SecureTokenGenerator.newToken()`.
   - Store `sha256Hex(token)` with `expiresAt = now + 30 min`. Earlier unused tokens for that account are invalidated.
   - Email the link `https://<host>/reset#token=<token>`. The token sits in the URL **fragment**, so it is not sent to the server in request lines, logs, or Referer headers.
2. The SPA reads the fragment and calls `POST /auth/password-resets {token, newPassword}`.
3. The server looks up the hash.
   - Unknown, expired, or used → `RESET_TOKEN_INVALID`.
   - Otherwise, in one transaction: set the new hash, `password_changed_at = now`, and `used_at = now`.
   - Existing sessions then fail `SessionValidityFilter`.

### 13.3 Receipt upload pipeline (full scope)

1. The multipart limit (5 MB) is enforced by the servlet container. Exceeding it → 413 `RECEIPT_TOO_LARGE`.
2. `ExpenseService.attachReceipt` checks visibility, that the actor is the submitter, and that the expense is PENDING.
3. **Magic bytes:** JPEG starts with `FF D8 FF`; PNG starts with `89 50 4E 47 0D 0A 1A 0A`. Anything else → 415 `RECEIPT_TYPE_UNSUPPORTED`. The declared `Content-Type` is ignored.
4. **Re-encode** through `javax.imageio`, with no metadata copied. This strips EXIF/GPS and neutralizes polyglot files. Images whose pixel dimensions exceed 50 megapixels are rejected (protection against decompression bombs).
5. `ReceiptStorage.put` writes to `<root>/<uuid>`: a temp file first, then an atomic move.
6. In the transaction, insert or replace the `receipt_attachment` row.
   - **On rollback:** an after-rollback hook deletes the new file.
   - **On commit, if a receipt was replaced:** an after-commit hook deletes the old file.
7. **Download:** visibility check, then stream with `Content-Disposition: attachment`, `nosniff`, and `Cache-Control: private, no-store`.

---

## 14. Swing presentation design [COND-B]

Valid only after documented customer approval. Does not satisfy NFR-001. Screen inventory: [contracts/swing-ui.md](contracts/swing-ui.md).

| Class | Responsibility |
|---|---|
| `DesktopApplication` | Composition root. Starts a non-web Spring context (for transactions and JPA) against `jdbc:h2:file:<profile>/SplitSmart/splitsmart`. Runs Flyway. Builds presenters. Shows `SignInView` on the EDT. |
| `SessionHolder` | Holds `ActorContext`. Cleared on sign-out or after 30 min idle (Swing `Timer`). |
| `AbstractPresenter<V>` | `run(Supplier<T> call, Consumer<T> onSuccess)`. Runs the call in a `SwingWorker`, disables the view while busy, and on completion runs `onSuccess` or `ErrorPresenter.show(view, DomainException)` on the EDT. |
| `ExpenseFormPresenter` | Collects raw strings from the view and calls `ExpenseService.previewAllocation` while typing (debounced 300 ms). Calls `submitExpense` with an idempotency key fixed for each dialog instance. Maps `FieldError.fieldPath` to widgets. |
| `PaymentDialogPresenter` | Pre-fills the outstanding amount from `getGroupBalances`. On `POSSIBLE_DUPLICATE_PAYMENT` it shows a confirmation, then resubmits with `confirmPossibleDuplicate = true` and the **same** idempotency key. |
| `RegisterPresenter` | After `register`, shows the 8 recovery codes once in a modal that requires acknowledgement. They cannot be shown again. |
| Views | Only layout, event forwarding, and rendering of pre-formatted strings. No `BigDecimal` arithmetic (ArchUnit rule: `..desktop.view..` must not depend on `java.math`). |

**Single instance:** H2's file lock prevents two processes from opening the DB. A second launch shows "SplitSmart is already running."

---

## 15. Workflow walk-throughs (end-to-end)

Each walk-through lists the calls in order, the transaction boundary (⟦ … ⟧), and the state changes. Diagrams: [seq-expense-submit-approve.puml](diagrams/seq-expense-submit-approve.puml), [seq-payment-record.puml](diagrams/seq-payment-record.puml), [expense-states.puml](diagrams/expense-states.puml).

### 15.1 Expense submission

```
UI → ExpenseService.submitExpense(actor, cmd)
  IdempotencyGuard.check(EXPENSE, …)                     replay if same key+hash
  ⟦ authz.requireActiveMember(actor, g)                  → NOT_FOUND
    ctx    ← groupContext(g)                             currency, active members, managers
    draft  ← MoneyParser + field mapping                 field errors collected
    ExpenseValidator.validate(draft, ctx, today).throwIfInvalid()
    alloc  ← AllocationCalculator.allocate(draft.amount, draft.method, draft.weights)
    e      ← Expense.submit(newId, ctx, actor, draft, alloc, now)        state: ∅ → PENDING
    expenseRepo.insert(e)                                expense + participants
    notificationRepo.insertAll(composer.submitted(e, ctx))  every other ACTIVE member
  ⟧ after commit: publisher.published(...)
  return ExpenseView(PENDING, shares, reviewBlocked?)
Balances: unchanged (§6.3).
```

### 15.2 Approval / rejection

See §8.2. The state changes from PENDING to APPROVED or REJECTED. On APPROVED, the expense's allocations enter the balance query from the next read onward. On REJECTED, they never do. A notification goes to the submitter, payer, and participants.

### 15.3 Split calculation

This runs inside submission and edit (and `previewAllocation`, which has no transaction and no persistence). See §3.

### 15.4 Balance update and read

There is no write path for balances (§6.3). A read is: `authz` → `balanceQuery.directedSums(g)` → `calculator.all(...)` → view.

### 15.5 Payment recording

See §7. The new state is a new immutable `Payment` row. The pair balance drops by the payment amount immediately.

### 15.6 Invitation round trip

```
Manager → GroupService.inviteMember(actor, g, email)
  ⟦ authz.requireManager(actor, g)                       → NOT_FOUND / FORBIDDEN_ROLE
    email ← normalize(email); validate
    if email belongs to an ACTIVE member of g            → ALREADY_MEMBER (visible to managers anyway)
    if pending invitation exists (effective status)      → INVITATION_ALREADY_PENDING
    invitee ← accountRepo.findByEmail(email)             may be empty
    inv ← Invitation.create(g, email, actor, invitee?, now)               state: ∅ → PENDING
    invitationRepo.save(inv)
    if invitee present: notificationRepo.insertAll([INVITATION_RECEIVED])
  ⟧ return uniform InvitationAck                         same response whether or not invitee exists

Invitee → GroupService.respondToInvitation(actor, invId, accept=true)
  ⟦ inv ← invitationRepo.findById(invId); if inv.inviteeAccountId ≠ actor → NOT_FOUND
    inv.accept(actor, now)                               → INVITATION_NOT_PENDING (incl. expired)
    membership ← find(g, actor) ?: new; membership.activate(MEMBER, now)  state: → ACTIVE
    save both (optimistic version on invitation guards double-accept)
  ⟧ return MembershipView
```

An invitation addressed to an email with no account stays PENDING and unactionable until it expires. If that person registers later, the invitation does **not** attach automatically **[DECISION]**; the manager re-invites. This avoids silently binding invitations to newly registered accounts.

### 15.7 Member removal

```
⟦ authz.requireManager(actor, g); group ← lockForUpdate(g)
  target ← membershipRepo.find(g, memberId) must be ACTIVE → else NOT_FOUND
  nonZero ← calculator.all(balanceQuery.directedSums(g)).involves(memberId)
  pending ← expenseRepo.hasPendingInvolving(g, memberId)
  GroupMembershipRules.requireCanEnd(target, ctx, nonZero, pending)
  target.end(REMOVED, endedBy = actor, now)
⟧
```

The member's history keeps their account ID (I-18). They keep read access to party records only (§9).

### 15.8 Reporting

See §12. Everything is read-only. Scope always comes from `authz.scopeFor`.

---

## 16. Test design

These are proposed JUnit 5 test classes. All run under `mvn verify` unless a profile is noted. Scenario IDs S# refer to [quickstart.md §4](quickstart.md#4-end-to-end-validation-scenarios).

| Test class | Layer | Key cases |
|---|---|---|
| `MoneyParserTest` | domain/app | Accepts `12.5`, `12.50`, `100`, `0.01`. Rejects `0`, `-1`, `12.505`, `1E+2`, `1,000.00`, `$5`, `1000000.01`. JPY rejects `10.5`. KWD (scale 3) accepts `1.234`. |
| `MoneyTest` | domain | Scale normalization; `equals` consistent with `compareTo`; currency mismatch throws; `toMinorUnits` round-trip; persisted value with excess scale throws. |
| `PercentageTest` | domain | `33.33` → 3333 bp; rejects `33.333`, `-1`, `100.01`, `abc`. |
| `AllocationCalculatorTest` | domain | Every row of §3.4; tie-break uses the string order (fixture: two UUIDs whose `UUID.compareTo` and string order disagree). |
| `AllocationPropertyTest` | domain (jqwik, optional) | §3.4 case 8 |
| `ExpenseValidatorTest` | domain | Each code in §4.2; several errors returned together; payer as non-participant valid; payer as participant valid. |
| `ExpenseStateMachineTest` | domain | Every transition and guard in §5.1, including edit by non-submitter and approve after reject. |
| `ReviewPolicyTest` | domain | Non-member, member, manager, manager-submitter with self-approval on and off. |
| `PairwiseBalanceCalculatorTest` | domain | Payer's own share ignored; netting both directions; payment reduces; Σ memberNet = 0; quickstart S5–S7 numbers. |
| `PaymentRulesTest` | domain | Each code in contracts §5; amount = N allowed; N + 0.01 rejected. |
| `GroupMembershipRulesTest` | domain | Creator protection; last manager; balance or pending blocks removal. |
| `DateRangeTest` | domain | Inclusive boundaries; `to < from`; null; > 5 years. |
| `ExpenseServiceTest`, `ReviewServiceTest`, `PaymentServiceTest`, `GroupServiceTest`, `AccountServiceTest`, `ReportServiceTest` | application (in-memory fakes) | Authorization denials per §9.2; notifications created with the correct recipients; idempotent replay and key reuse; summary figures. |
| `AuthorizationMatrixTest` | application | One case per cell of §9.2. |
| `BalanceQueryMatchesCalculatorIT` | persistence (H2; `-Ppostgres`) | SQL directed sums = in-memory sums on randomized fixtures. |
| `MigrationIT` | persistence | Flyway applies cleanly on H2 and PostgreSQL; constraints present. |
| `ConcurrentReviewIT` | persistence | Two threads approve/reject the same expense, released together by a `CountDownLatch` → exactly one succeeds and one `ReviewDecision` row exists. |
| `ConcurrentPaymentIT` | persistence (PostgreSQL profile) | Debt 50.00; two threads each pay 30.00 → one succeeds, one `PAYMENT_EXCEEDS_OUTSTANDING`; final balance 20.00. |
| `EditVsApproveIT` | persistence | Edit commits first → approval with stale version gets `CONCURRENT_MODIFICATION`. |
| `ArchitectureRulesTest` | all (ArchUnit) | No `double`/`float` fields or parameters whose names match `amount\|share\|balance\|price\|total` in any module; domain has no framework imports; presentation does not depend on `..persistence..`; Swing views do not depend on `java.math`. |
| [COND-A] `*ApiTest` | web (MockMvc) | Status codes and problem codes per rest-api.md; 404-not-403; CSRF missing → 403; JSON number for amount → 400 `AMOUNT_FORMAT_INVALID`; money serialized as plain strings. |
| [COND-A] `PasswordResetFlowTest` | web | Uniform 202; token single-use and expiring; old sessions rejected afterward. |
| [COND-A] `ReceiptUploadTest` | web | PNG/JPEG accepted; GIF renamed `.jpg` rejected; 6 MB rejected; EXIF removed; non-member download 404. |
| [COND-B] `*PresenterTest` | desktop (no GUI) | Field-error mapping; duplicate-payment confirm path; recovery codes shown once. |
| `PerformanceIT` (`-Pperf`) | web or desktop | NFR-004 dataset; p95 per view; results recorded, not asserted against invented numbers until measured. |

---

## 17. Design decisions introduced in this document

These refine, and do not contradict, decisions in [architecture.md](architecture.md) and [research.md](research.md). They need human review like everything else.

| # | Decision | Rationale |
|---|---|---|
| DD-1 | Percentages are held as integer basis points; allocation uses only `long` arithmetic | Exact remainders; no `BigDecimal` division |
| DD-2 | Tie-break uses the **string** order of UUIDs, not `UUID.compareTo` | Same order in Java, SQL, and tests |
| DD-3 | Money input is accepted only as plain decimal strings (regex-checked) | Rejects exponents and locale formats; avoids float parsing in clients |
| DD-4 | `RoundingMode.UNNECESSARY` is the only rounding mode allowed in money code | Any hidden rounding becomes a visible failure |
| DD-5 | Lock order: group row first, one group per transaction | Deadlock-free by construction |
| DD-6 | Approval requires the version the reviewer saw | A reviewer can never approve edited content they have not seen |
| DD-7 | Idempotency stores a request hash; a concurrent duplicate replays via the unique constraint | Correct retry semantics without a separate table |
| DD-8 | Invitations expire lazily (no scheduler) | No background jobs in the MVP |
| DD-9 | Invitations to unregistered emails do not auto-attach on later registration | Avoids silent binding; the manager re-invites |
| DD-10 | Reset token carried in the URL fragment | Keeps the token out of server logs and Referer headers |
| DD-11 | Receipts are re-encoded on upload | Strips metadata; neutralizes polyglot files |
| DD-12 | `reviewBlocked` flag on `ExpenseView` | Makes the self-approval deadlock visible (architecture §4.4) |
