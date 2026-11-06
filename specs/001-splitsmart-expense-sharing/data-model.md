# Data Model: SplitSmart Expense Sharing

## Purpose

This data model captures the core entities, relationships, invariants, and validation rules required to implement the SplitSmart domain. It is derived from the authoritative requirements and the clarified specification. The model intentionally separates the domain from UI and infrastructure concerns.

## Core Entities

### 1. Account
Represents a person’s identity and private access profile.

Fields:
- `account_id` (UUID or surrogate key)
- `email` (unique, validated)
- `password_hash` (stored only as hash; not plaintext)
- `display_name`
- `created_at`
- `last_login_at`
- `status` (active, locked, recovered, deleted) — implementation choice

Rules and invariants:
- Each account must have a unique email.
- The account may only access its own private data and group-shared data to which it belongs.
- Passwords are not stored in plaintext.

Requirement coverage: FR-001, NFR-002

### 2. Group
Represents a shared spending context such as roommates, travelers, or coworkers.

Fields:
- `group_id`
- `name`
- `currency_code` (e.g., USD, EUR, CAD)
- `currency_scale` (smallest supported unit such as 0.01 for cent precision)
- `created_by_account_id`
- `created_at`
- `status`

Rules and invariants:
- A group has one creator and zero or more managers.
- Membership is controlled by the creator or designated managers.
- Historical expenses remain attributed to original participants even after a user is removed.

Requirement coverage: FR-002, AC-003, AC-004

### 3. GroupMembership
Links an account to a group and tracks role and invite state.

Fields:
- `membership_id`
- `group_id`
- `account_id`
- `role` (member, manager, creator)
- `invite_status` (pending, accepted, rejected, removed)
- `invited_by_account_id`
- `accepted_at`
- `removed_at`

Rules and invariants:
- Invitees must accept before membership is active.
- Only creator/designated managers may invite or remove members.
- Membership changes do not rewrite historical attribution of prior expenses.

Requirement coverage: FR-002, AC-003, AC-004

### 4. Expense
Represents a submitted group expense before and after review.

Fields:
- `expense_id`
- `group_id`
- `submitter_account_id`
- `payer_account_id` (selected separately from participants)
- `expense_date`
- `description`
- `amount` (BigDecimal)
- `currency_code`
- `allocation_type` (equal, percentage)
- `receipt_reference` (optional, file/object reference)
- `status` (pending, approved, rejected)
- `submitted_at`
- `reviewed_by_account_id` (nullable)
- `reviewed_at` (nullable)
- `review_note` (nullable)

Rules and invariants:
- `amount` must be positive and aligned to the group currency precision.
- `payer_account_id` must belong to the group.
- The payer may be excluded from the allocation set.
- Only approved expenses affect balances.
- Pending expenses remain visible but do not alter balances.
- Rejected expenses remain recorded with review outcome but do not change balances.

Requirement coverage: FR-003, FR-006, FR-007, AC-005, AC-010–AC-012

### 5. ExpenseParticipant
Represents a participant allocation for a specific expense.

Fields:
- `expense_participant_id`
- `expense_id`
- `account_id`
- `share_amount` (BigDecimal)
- `percentage` (BigDecimal or nullable depending on allocation type)
- `allocation_order` (for deterministic rounding order)

Rules and invariants:
- Each participant is a group member and must belong to the expense's group.
- For equal splits, the total share_amount equals the total expense amount.
- For percentage splits, percentages are nonnegative and total exactly 100%.
- `share_amount` is derived from the allocation logic and is stored for traceability and reporting.

Requirement coverage: FR-003, FR-004, FR-005, NFR-003, AC-007–AC-009

### 6. ReviewDecision
Tracks the final review outcome for an expense.

Fields:
- `review_decision_id`
- `expense_id`
- `approver_account_id`
- `decision` (approved, rejected)
- `decision_at`
- `reason` (optional)

Rules and invariants:
- Exactly one final review decision is allowed for a pending expense.
- Review decisions are retained for audit and reporting.

Requirement coverage: FR-007, NFR-005

### 7. Notification
Stores a notification to relevant group members when an expense is submitted.

Fields:
- `notification_id`
- `group_id`
- `expense_id`
- `recipient_account_id`
- `message`
- `status` (pending, delivered, read, dismissed)
- `created_at`
- `read_at` (nullable)

Rules and invariants:
- Notifications are created for group members other than the submitter when an expense is submitted.
- Notification content must include review status.

Requirement coverage: FR-008, AC-013

### 8. Payment
Represents a payment that reduces the balance between two users.

Fields:
- `payment_id`
- `group_id`
- `sender_account_id`
- `recipient_account_id`
- `amount` (BigDecimal)
- `payment_date`
- `recorded_by_account_id`
- `recorded_at`
- `note` (optional)

Rules and invariants:
- Sender and recipient must both belong to the group.
- Payment amount must be positive and within currency precision.
- Balance is adjusted immediately when the payment is recorded.
- A payment is preserved in historical records and identifies who recorded it and when.

Requirement coverage: FR-010, AC-015, AC-016

### 9. BalanceView (derived)
Represents the pairwise amount one user owes another user after expenses and payments.

Fields:
- `balance_id`
- `group_id`
- `from_account_id`
- `to_account_id`
- `amount_owed` (BigDecimal)
- `last_updated_at`

Rules and invariants:
- `amount_owed` is derived from approved expenses and recorded payments.
- It is not allowed to be negative solely due to overpayment; the system should not produce a negative balance from a payment alone beyond agreed semantics.
- Pairwise balances are recomputed from underlying source records after changes.

Requirement coverage: FR-009, AC-014, AC-015

### 10. ReportRequest and ReportResult
Support summary and custom report generation.

Fields:
- `report_request_id`
- `account_id`
- `group_id`
- `start_date` (nullable)
- `end_date` (nullable)
- `report_type` (expense, balance, payment, custom)
- `parameters` (JSON or structured values)
- `created_at`

Rules and invariants:
- Reports are limited to authorized data.
- Date-range filters must validate start/end, and no report may include a date range with mismatched boundaries.

Requirement coverage: FR-011, FR-012, AC-017–AC-019

## Relationships

```text
Account 1---* GroupMembership *---1 Group
Account 1---* Expense (submitter)
Account 1---* Expense (payer)
Expense 1---* ExpenseParticipant
Expense 1---1 ReviewDecision
Group 1---* Expense
Group 1---* Notification
Group 1---* Payment
Group 1---* BalanceView
Account 1---* Payment (sender/recipient)
Account 1---* ReportRequest
```

## Domain Invariants

1. Money is always represented as `BigDecimal`, never float/double.
2. Shared expense amounts and allocations reconcile to the total at the currency precision.
3. Percentage allocations sum to exactly 100% with no negative percentages.
4. Pending expenses never affect balances.
5. Approved expenses contribute to balances; rejected expenses do not.
6. Similar payment amounts reduce the corresponding pairwise balance immediately after recording.
7. Every account can access only its own private account information and group data they belong to.
8. The payer is separate from the participant selection and must belong to the group.
9. Historical attribution remains tied to the original participants even if group membership changes later.
10. Every expense and payment is reviewable/auditable by historical metadata.

## Validation Rules at Service Boundaries

### Expense validation
- `amount > 0`
- Currency precision matches the group configuration
- `description` is not blank
- Payment and participants belong to the same group
- `payer_account_id` belongs to the group
- `allocation_type` must be supported: equal or percentage
- For percentage split: all percentages are >= 0 and sum to 100.00 exactly
- For equal split: selected participants are the allocation set
- Receipt image is optional but valid when attached

### Payment validation
- `sender_account_id` and `recipient_account_id` belong to same group
- `amount > 0`
- Payment date is valid
- Payment is recorded by a valid group member
- Historical metadata is retained

### Authorization validation
- Account must be authenticated
- User must be a member of the group for any group-scoped action
- Managers/creator may add/remove members; others cannot
- Reports may return only authorized group and personal data

## State Transitions

### Expense lifecycle
```text
Draft -> Submitted -> PendingReview -> Approved
                     \-> Rejected
```

- `Draft` is an implementation detail that may exist before submission.
- `Submitted` or `PendingReview` states are equivalent for the business rule that expenses await review.
- On approval, the expense participates in balance calculations.
- On rejection, the expense is retained with a rejection outcome and does not affect balances.

### Payment lifecycle
```text
Recorded -> Immediately reflected in balance history -> Available in reports
```

### Notification lifecycle
```text
Created -> Delivered -> Read/Dismissed
```

## Data Ownership and Privacy Boundaries

- Account data is strictly personal and owned by the account holder.
- Group membership data is shared within the group but not outside it.
- Expense details are group-scoped and available only to members of the relevant group.
- Payment details and histories are also group-scoped and must be restricted to authorized group members.
- Authorization checks happen in service/security layers; UI must not bypass them.

## Suggested persistence design

Recommended storage design (implementation choice, not pre-existing repository fact):

- `accounts`
- `groups`
- `group_memberships`
- `expenses`
- `expense_participants`
- `review_decisions`
- `notifications`
- `payments`
- `report_requests`

On implemention, create indexes for:
- `group_memberships(group_id, account_id)`
- `expenses(group_id, status, expense_date)`
- `payments(group_id, payment_date)`
- `notifications(recipient_account_id, status)`

## Traceability

| Requirement(s) | Entity/Model area |
|---|---|
| FR-001 | Account |
| FR-002 | Group, GroupMembership |
| FR-003 | Expense, ExpenseParticipant |
| FR-004 | ExpenseParticipant / split logic |
| FR-005 | ExpenseParticipant / percentage validation |
| FR-006, FR-007 | Expense, ReviewDecision |
| FR-008 | Notification |
| FR-009 | BalanceView |
| FR-010 | Payment |
| FR-011, FR-012 | ReportRequest, ReportResult |
| NFR-002 | Account + authorization boundaries |
| NFR-003 | ExpenseParticipant + BigDecimal rules |
| NFR-005 | ReviewDecision + Payment + historical audit |

## Validation Strategy

Implementation tests should verify:
- Positive amount rejection and invalid percentage totals fail.
- Largest-remainder rounding yields exact totals.
- Pending expense does not affect balances.
- Approved expense updates pairwise balances correctly.
- Rejected expense leaves balances unchanged.
- Payment immediately updates balances and preserves metadata.
- Reports respect date ranges and account/group authorization.

## Notes

This data model is for planning purposes only. It is not a claim that the repository already contains these tables or classes. It is intended to guide a Java 21 implementation after the architecture decision is finalized.
