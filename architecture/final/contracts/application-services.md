# Application-Service Contracts (shared core; both platform alternatives)

Status: **Reconciled candidate. Pending human review.** Package: `splitsmart.application`. Derived from Claude Code `contracts/application-services.md`; Spec Kit has no service-level contract, and its web and UI contracts were reconciled into [web-api.md](web-api.md) and [ui-workflows.md](ui-workflows.md). These are Java interface **contracts written in prose and tables**, not implementation code.

## 1. Conventions

- **Signature:** every method takes `ActorContext actor` (authenticated `accountId`) plus a command or query object, and returns an immutable result.
  - Exceptions: `register`, `authenticate`, and the recovery methods take no actor.
- **Transactions:** each command method runs in **one transaction** (all-or-nothing).
- **Order of checks:** (1) authentication present, (2) resource visibility (not visible → `NOT_FOUND`), (3) role/permission (`FORBIDDEN_ROLE`), (4) validation (all field errors collected → `VALIDATION_FAILED` with details), (5) business rules / state (`EXPENSE_NOT_PENDING`, `PAYMENT_EXCEEDS_OUTSTANDING`, …), (6) concurrency (`CONCURRENT_MODIFICATION`).
- **Types:** money is `BigDecimal`; IDs are `UUID`; business dates are `LocalDate`; timestamps are `Instant`.
- **Idempotency:** commands marked *idempotent* accept an optional `idempotencyKey` (UUID). The same key with the same command returns the first result. The same key with a different command → `IDEMPOTENCY_KEY_REUSED`.
- **Visibility abbreviations:** **AM** = actor is an ACTIVE member of the group. **MGR** = actor is an ACTIVE MANAGER. **PARTY** = actor is payer, participant, sender, or recipient of the record (former members keep read access to their own PARTY records).

## 2. AccountService (FR-001)

| Operation | Input | Output | Validation | Authorization | Errors |
|---|---|---|---|---|---|
| `register` | email, displayName, password | AccountView (id, email, displayName); **B:** also `recoveryCodes[8]` (shown once) | email syntax, ≤ 254; displayName 1–80; password ≥ 12 chars, ≤ 128 | public | `VALIDATION_FAILED`, `EMAIL_UNAVAILABLE` |
| `authenticate` | email, password | AccountView | non-blank | public; rate-limited | `INVALID_CREDENTIALS` (same for unknown email and wrong password), `RATE_LIMITED` |
| `requestPasswordReset` **[A]** | email | none (always succeeds) | email syntax | public; rate-limited | `RATE_LIMITED` only |
| `completePasswordReset` **[A]** | token, newPassword | none | password rules | token holder | `RESET_TOKEN_INVALID` (unknown, expired, or used) |
| `recoverWithCode` **[B]** | email, recoveryCode, newPassword | none | password rules | code holder | `INVALID_CREDENTIALS` |
| `changePassword` | currentPassword, newPassword | none | password rules | self | `INVALID_CREDENTIALS`, `VALIDATION_FAILED` |
| `getMyProfile` | none | AccountView | none | self | none |

State effects: a successful reset or password change sets `password_changed_at`, which invalidates other sessions (A) and consumes the token or code.

## 3. GroupService (FR-002)

| Operation | Input | Output | Validation | Authorization | Errors |
|---|---|---|---|---|---|
| `createGroup` | name, description?, currencyCode (default USD), selfApprovalAllowed (default false) | GroupView | name 1–80; description ≤ 500; supported currency (scale 0/2/3) | any authenticated | `VALIDATION_FAILED`, `CURRENCY_UNSUPPORTED` |
| `updateGroup` | groupId, version, name?, description?, currencyCode?, selfApprovalAllowed? | GroupView | as above; currency change only if no expenses exist | MGR | `NOT_FOUND`, `FORBIDDEN_ROLE`, `CURRENCY_LOCKED`, `CONCURRENT_MODIFICATION` |
| `listMyGroups` | none | List<GroupSummary> (id, name, currency, myRole, memberCount) | none | self (ACTIVE memberships) | none |
| `getGroup` | groupId | GroupView + active members (id, displayName, email, role) | none | AM | `NOT_FOUND` |
| `inviteMember` | groupId, inviteeEmail | InvitationAck (uniform; reveals nothing about account existence) | email syntax | MGR | `NOT_FOUND`, `FORBIDDEN_ROLE`, `ALREADY_MEMBER` (only if the email belongs to a current **active** member, already visible to MGR), `INVITATION_ALREADY_PENDING` |
| `revokeInvitation` | invitationId | none | none | MGR of the group | `NOT_FOUND`, `FORBIDDEN_ROLE`, `INVITATION_NOT_PENDING` |
| `listMyInvitations` | none | List<InvitationView> (group name, inviter name, createdAt) | none | invitee | none |
| `respondToInvitation` | invitationId, accept: boolean | MembershipView (if accepted) | none | invitee only | `NOT_FOUND`, `INVITATION_NOT_PENDING` (incl. expired) |
| `setManager` | groupId, memberId, manager: boolean | MembershipView | target ACTIVE | MGR | `FORBIDDEN_ROLE`, `CANNOT_DEMOTE_CREATOR`, `LAST_MANAGER` |
| `removeMember` | groupId, memberId | none | target ACTIVE, not creator | MGR | `FORBIDDEN_ROLE`, `CANNOT_REMOVE_CREATOR`, `LAST_MANAGER`, `MEMBER_HAS_OUTSTANDING_BALANCE`, `MEMBER_HAS_PENDING_EXPENSES` |
| `leaveGroup` | groupId | none | not creator | AM (self) | `CANNOT_REMOVE_CREATOR`, `LAST_MANAGER`, `MEMBER_HAS_OUTSTANDING_BALANCE`, `MEMBER_HAS_PENDING_EXPENSES` |

State effects:

- `inviteMember` creates an `INVITATION_RECEIVED` notification only when the email matches an account.
- `removeMember`, `leaveGroup`, and `setManager` take the group row lock.

## 4. ExpenseService and ReviewService (FR-003 – FR-008)

### 4.1 Commands and queries

| Operation | Input | Output | Authorization | Errors |
|---|---|---|---|---|
| `submitExpense` *(idempotent)* | SubmitExpenseCommand (below) | ExpenseView (status PENDING, shares) | AM | `NOT_FOUND`, validation codes (§4.2), `IDEMPOTENCY_KEY_REUSED` |
| `previewAllocation` | groupId, amount, splitMethod, payerId, participants | AllocationPreview{shares[{accountId, share}]} (nothing persisted, no notifications) | AM | validation codes (§4.2) |
| `editPendingExpense` | expenseId, version, same fields as submit | ExpenseView | AM **and** actor = submitter | `NOT_FOUND`, `FORBIDDEN_ROLE`, `EXPENSE_NOT_PENDING`, `CONCURRENT_MODIFICATION`, validation codes |
| `getExpense` | expenseId | ExpenseView (+ decision, receipt metadata) | AM, or PARTY for former members | `NOT_FOUND` |
| `listGroupExpenses` | groupId, status?, from?, to?, page, size (≤ 100) | Page<ExpenseSummary> | AM | `NOT_FOUND`, `INVALID_DATE_RANGE` |
| `attachReceipt` **[COND-A; Release 2]** | expenseId, bytes, declaredContentType, filename | ReceiptView | actor = submitter; expense PENDING | `EXPENSE_NOT_PENDING`, `RECEIPT_TYPE_UNSUPPORTED`, `RECEIPT_TOO_LARGE` |
| `getReceipt` **[A]** | expenseId | stream + content type | as `getExpense` | `NOT_FOUND` |
| `approveExpense` | expenseId, version | ExpenseView (APPROVED) | ReviewPolicy | `NOT_FOUND`, `FORBIDDEN_ROLE`, `SELF_APPROVAL_NOT_ALLOWED`, `EXPENSE_NOT_PENDING`, `CONCURRENT_MODIFICATION` |
| `rejectExpense` | expenseId, version, reason? (≤ 500) | ExpenseView (REJECTED) | ReviewPolicy | same as approve |
| `listPendingReviewsForMe` | none | List<ExpenseSummary> where I may decide | self | none |

**SubmitExpenseCommand**

| Field | Type | Rules |
|---|---|---|
| groupId | UUID | actor is AM |
| amount | BigDecimal | > 0; scale ≤ group currency scale; ≤ 1,000,000 |
| expenseDate | LocalDate | ≤ today + 1 day |
| description | String | 1–200 after trim |
| splitMethod | EQUAL \| PERCENTAGE | required |
| payerId | UUID | ACTIVE member of the group; chosen independently of participants |
| participants | list of {accountId, percentage?} | ≥ 1 entry; distinct; all ACTIVE members; `percentage` required iff PERCENTAGE, each 0–100, scale ≤ 2, Σ = 100.00; must be absent for EQUAL |
| idempotencyKey | UUID? | optional |

**ExpenseView:**

- Scalars: id, groupId, submittedBy, payer, amount, currency, expenseDate, description, splitMethod, status, version, submittedAt, lastEditedAt.
- `reviewBlocked`: true when the expense is PENDING, its submitter is the only manager, and self-approval is off ([design.md §5.2](../design.md#52-reviewpolicycheckactorid-expense-e-groupcontext-ctx--decision)).
- `participants[{accountId, displayName, percentage?, share}]`.
- `review{reviewerId, decision, reason, decidedAt}?`.
- `receipt{contentType, sizeBytes, filename}?`.

### 4.2 Validation error codes (field-level; all returned together)

| Code | Condition |
|---|---|
| `AMOUNT_FORMAT_INVALID` | not a plain decimal string (e.g. empty, exponent form `1E+2`, grouping separators, currency symbol, JSON number) |
| `AMOUNT_NOT_POSITIVE` | amount ≤ 0 |
| `AMOUNT_PRECISION_EXCEEDED` | more decimal places than the currency allows |
| `AMOUNT_TOO_LARGE` | > 1,000,000 |
| `DATE_IN_FUTURE` | date > today + 1 |
| `DESCRIPTION_INVALID` | blank or > 200 |
| `PAYER_NOT_MEMBER` | payer not an ACTIVE member |
| `PARTICIPANTS_REQUIRED` | empty participant list |
| `PARTICIPANT_DUPLICATE` | same account twice |
| `PARTICIPANT_NOT_MEMBER` | participant not an ACTIVE member of this group |
| `PERCENTAGE_REQUIRED` / `PERCENTAGE_NOT_ALLOWED` | missing for PERCENTAGE / present for EQUAL |
| `PERCENTAGE_FORMAT_INVALID` | not a plain decimal string |
| `PERCENTAGE_NEGATIVE` / `PERCENTAGE_OUT_OF_RANGE` | < 0 / > 100 |
| `PERCENTAGE_PRECISION_EXCEEDED` | scale > 2 |
| `PERCENTAGES_NOT_100` | Σ ≠ 100.00 |

A request that tries to assign a share to a non-participant is impossible by construction: shares are computed only for listed participants. A client-supplied `share` field is ignored and never trusted (AC-009).

### 4.3 State behavior

- **Submit:** creates PENDING, computes and stores the shares, and inserts `EXPENSE_SUBMITTED` notifications for all other ACTIVE members, all in one transaction. **No balance change** (AC-010, AC-013).
- **Approve/reject:** takes the group lock, then a conditional update on `status = PENDING AND version = ?`, inserts the ReviewDecision, and inserts `EXPENSE_REVIEWED` notifications for the submitter, payer, and participants (excluding the reviewer). Approve: balances now include the expense (AC-011). Reject: balances never include it (AC-012).
- **ReviewPolicy (default, OQ-2):** actor is MGR, and (actor ≠ submitter or group.selfApprovalAllowed).

## 5. BalanceService and PaymentService (FR-009, FR-010)

| Operation | Input | Output | Authorization | Errors |
|---|---|---|---|---|
| `getGroupBalances` | groupId | GroupBalances{currency, pairs[{debtor, creditor, amount}] (non-zero only, amount > 0), memberNet[{member, net}]} | AM | `NOT_FOUND` |
| `getMyBalances` | none | MyBalances{byGroup[{group, currency, pairs involving me}], byCounterparty[{counterparty, currency, net}]} (totals per currency; never cross-currency) | self (includes PARTY pairs in former groups) | none |
| `recordPayment` *(idempotent)* | RecordPaymentCommand | PaymentView + updated pair balance | AM **and** actor = senderId | see below |
| `listPayments` | groupId, from?, to?, page, size | Page<PaymentView> | AM (former members: PARTY only) | `NOT_FOUND`, `INVALID_DATE_RANGE` |

**RecordPaymentCommand:** groupId, senderId, recipientId, amount, paymentDate, note? (≤ 200), confirmPossibleDuplicate (default false), idempotencyKey?

| Code | Condition |
|---|---|
| `PAYMENT_NOT_SENDER` | actor ≠ senderId (FR-010: the sender records) |
| `PAYMENT_SELF` | sender = recipient |
| `RECIPIENT_NOT_MEMBER` | recipient not an ACTIVE member |
| `AMOUNT_NOT_POSITIVE`, `AMOUNT_PRECISION_EXCEEDED`, `DATE_IN_FUTURE` | as for expenses |
| `NOTHING_OWED` | N(sender, recipient) ≤ 0 |
| `PAYMENT_EXCEEDS_OUTSTANDING` | amount > N(sender, recipient); the response includes the current outstanding amount (the actor is party to it) |
| `POSSIBLE_DUPLICATE_PAYMENT` | same sender/recipient/amount/date recorded in the last 24 h and `confirmPossibleDuplicate = false` |

**State behavior:**

1. Group lock.
2. Compute N(sender, recipient) and validate.
3. Insert with `recordedBy = actor`, `recordedAt = clock.now()`.
4. Commit. The balance reflects the payment immediately (AC-015, AC-016).

**PaymentView:** id, groupId, sender, recipient, amount, currency, paymentDate, note, recordedBy, recordedAt.

## 6. NotificationService and ReportService (FR-008, FR-011, FR-012)

| Operation | Input | Output | Authorization | Errors |
|---|---|---|---|---|
| `listMyNotifications` | unreadOnly?, page, size | Page<NotificationView{id, type, groupId, subjectId, summary, createdAt, read}> | self | none |
| `GroupService.listHistory` (NFR-005) | groupId, subjectType?, subjectId?, page, size | Page<AuditEventView{eventType, actor, subjectType, subjectId, occurredAt, details (redacted per viewer)}> | AM; former members: PARTY subjects only | `NOT_FOUND` |
| `markRead` | notificationIds[] | none | recipient only (IDs that are not the actor's are ignored, not reported) | none |
| `getSummary` (Release 1) | groupId? (null = all my groups) | Summary{perCurrency[{currency, totalPaidByMe, myTotalShare, pendingCount, owedToMe, iOwe}], paymentsSent, paymentsReceived, balances} | AM for groupId; else my ACTIVE groups + PARTY records | `NOT_FOUND` |
| `runReport` (FR-012; Release 2) | ReportQuery{from, to, groupId?, recordTypes ⊆ {EXPENSE, PAYMENT}, expenseStatuses ⊆ {PENDING, APPROVED, REJECTED} (default APPROVED), onlyMine: boolean} | ReportResult{query echo, rows[{type, date, groupId, description, amount, currency, status, payer/sender, participants/recipient, myShare?}], totals per currency, balanceMovement[{counterparty, currency, net}]} | groups restricted to the actor's ACTIVE memberships, plus PARTY records | `INVALID_DATE_RANGE` (from or to missing, or to < from), `NOT_FOUND` |

**Report semantics:**

- Dates are inclusive and use the business date (`expenseDate`/`paymentDate`). A record dated exactly on `from` or `to` is included.
- An empty result is a valid report with zero totals.
- Range limit: ≤ 5 years **[ENGINEERING DECISION]** → else `INVALID_DATE_RANGE`.

## 7. Error codes

Single enumeration used by A (problem `code`) and B (presenter messages).

| Code | Category | A: HTTP |
|---|---|---|
| `NOT_AUTHENTICATED` | auth | 401 |
| `INVALID_CREDENTIALS`, `RESET_TOKEN_INVALID` | auth | 401 / 400 |
| `RATE_LIMITED` | auth | 429 |
| `NOT_FOUND` | visibility (also used for "exists but not visible") | 404 |
| `FORBIDDEN_ROLE`, `PAYMENT_NOT_SENDER`, `SELF_APPROVAL_NOT_ALLOWED` | permission | 403 |
| `VALIDATION_FAILED` (with per-field codes from §4.2/§5) | validation | 400 |
| `EMAIL_UNAVAILABLE`, `ALREADY_MEMBER`, `INVITATION_ALREADY_PENDING`, `IDEMPOTENCY_KEY_REUSED`, `POSSIBLE_DUPLICATE_PAYMENT` | conflict | 409 |
| `EXPENSE_NOT_PENDING`, `INVITATION_NOT_PENDING`, `CONCURRENT_MODIFICATION` | state / concurrency | 409 |
| `CURRENCY_UNSUPPORTED`, `CURRENCY_LOCKED`, `PAYMENT_SELF`, `RECIPIENT_NOT_MEMBER`, `NOTHING_OWED`, `PAYMENT_EXCEEDS_OUTSTANDING`, `INVALID_DATE_RANGE`, `LAST_MANAGER`, `CANNOT_REMOVE_CREATOR`, `CANNOT_DEMOTE_CREATOR`, `MEMBER_HAS_OUTSTANDING_BALANCE`, `MEMBER_HAS_PENDING_EXPENSES` | business rule | 422 |
| `RECEIPT_TOO_LARGE` / `RECEIPT_TYPE_UNSUPPORTED` | upload | 413 / 415 |
| `INTERNAL_ERROR` | unexpected (with correlationId) | 500 |

## 8. Transaction boundaries and postconditions

Each command is **one transaction** (`TransactionRunner.inTransaction`). Lock and audit behavior follow [design.md §8](../design.md#8-transactions-and-concurrency).

| Operation | Lock | Postconditions on success (all in the same commit) |
|---|---|---|
| `register` | none | Account exists with a hashed password. B: 8 recovery codes stored hashed. |
| `completePasswordReset` / `recoverWithCode` / `changePassword` | account row (optimistic) | New hash. `password_changed_at = now`. Token or code consumed. Audit `PASSWORD_*`. Older sessions invalid (A). |
| `createGroup` | none | Group exists. The creator has an ACTIVE MANAGER membership. Audit `GROUP_UPDATED` (created). |
| `inviteMember` | unique pending index | One PENDING invitation. `INVITATION_RECEIVED` notification if the account exists. Audit. |
| `respondToInvitation` | invitation version | Invitation ACCEPTED/DECLINED. If accepted, membership ACTIVE MEMBER. Audit. |
| `setManager` / `removeMember` / `leaveGroup` | group row | Role or status changed. ≥ 1 manager remains (I-16). Removal only with zero balance and no pending involvement (I-17). Audit. |
| `submitExpense` | none | Expense PENDING with stored shares (Σ = amount). Notifications to other active members. Audit. **No balance change.** |
| `editPendingExpense` | expense version | Still PENDING. Shares recomputed. `version + 1`. Audit with before/after. |
| `approveExpense` / `rejectExpense` | group row, then expense version | Status APPROVED/REJECTED. One `ReviewDecision`. Notifications. Audit. APPROVED: included in balances from the next read. |
| `recordPayment` | group row | Immutable payment with `recordedBy = actor = sender` and `recordedAt` = server time. The pair balance has dropped by exactly the amount and is never reversed (I-13). Audit. |
| `attachReceipt` | expense version | One receipt row. The file is stored. The previous file is deleted after commit. Audit (`EXPENSE_EDITED`, receipt changed). |
| Queries (`get*`, `list*`, `run*`, `preview*`) | none (read-only) | No state change. Results limited by `AuthorizationService` scope. |
