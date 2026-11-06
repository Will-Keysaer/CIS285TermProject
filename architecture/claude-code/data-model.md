# SplitSmart: Data Model (Claude Code proposal)

Status: **draft, needs human review.** Shared by Alternatives A and B unless marked. Behavior of these entities (algorithms, transactions, validation): [design.md](design.md). Diagram: [diagrams/domain-model.puml](diagrams/domain-model.puml). State machines: [diagrams/expense-states.puml](diagrams/expense-states.puml).

Conventions:

- **IDs:** all IDs are UUIDs.
- **Timestamps:** `Instant` in UTC (`TIMESTAMP WITH TIME ZONE`). They are set from the injected server `Clock`, never from the client.
- **Business dates:** `LocalDate` (`DATE`), entered by the user.
- **Money:** `BigDecimal` stored as `NUMERIC(19,4)`, always written at the group currency scale (see §3).
- **Table names:** shown in `snake_case`. `app_group` avoids the SQL keyword `GROUP`.
- **Optimistic locking:** `version` columns support it.

## 1. Entities

### 1.1 Account (`account`)
| Attribute | Type | Constraints |
|---|---|---|
| id | UUID | PK |
| email | VARCHAR(254) | NOT NULL, stored lower-cased, UNIQUE |
| display_name | VARCHAR(80) | NOT NULL, 1–80 chars after trim |
| password_hash | VARCHAR(100) | NOT NULL (BCrypt), never exposed |
| created_at | TIMESTAMPTZ | NOT NULL |
| password_changed_at | TIMESTAMPTZ | NOT NULL; sessions issued before this are invalid (A) |
| failed_login_count / locked_until | INT / TIMESTAMPTZ | Rate-limiting support **[PROPOSAL]** |
| version | BIGINT | Optimistic lock |

### 1.2 PasswordResetToken (`password_reset_token`): Alternative A
| Attribute | Type | Constraints |
|---|---|---|
| id | UUID | PK |
| account_id | UUID | FK → account |
| token_hash | CHAR(64) | SHA-256 of the 256-bit random token; UNIQUE |
| expires_at | TIMESTAMPTZ | created + 30 min |
| used_at | TIMESTAMPTZ | NULL until used; single use |

### 1.3 RecoveryCode (`recovery_code`): Alternative B [CONDITIONAL-B]
| Attribute | Type | Constraints |
|---|---|---|
| id | UUID | PK |
| account_id | UUID | FK → account |
| code_hash | VARCHAR(100) | BCrypt of a one-time code; 8 codes issued at registration |
| used_at | TIMESTAMPTZ | NULL until used |

### 1.4 Group (`app_group`)
| Attribute | Type | Constraints |
|---|---|---|
| id | UUID | PK |
| name | VARCHAR(80) | NOT NULL |
| description | VARCHAR(500) | NULL |
| currency_code | CHAR(3) | NOT NULL, ISO-4217 supported by `java.util.Currency`; **immutable once any expense exists** |
| created_by | UUID | FK → account; NOT NULL; immutable |
| self_approval_allowed | BOOLEAN | NOT NULL DEFAULT FALSE (OQ-2) |
| created_at | TIMESTAMPTZ | NOT NULL |
| version | BIGINT | Optimistic lock for detail updates. The row is also the **pessimistic lock target** for balance-affecting writes. |

### 1.5 Membership (`membership`)
| Attribute | Type | Constraints |
|---|---|---|
| id | UUID | PK |
| group_id | UUID | FK → app_group |
| account_id | UUID | FK → account |
| role | VARCHAR(10) | `MANAGER` \| `MEMBER` |
| status | VARCHAR(10) | `ACTIVE` \| `REMOVED` \| `LEFT` |
| joined_at | TIMESTAMPTZ | NOT NULL |
| ended_at | TIMESTAMPTZ | NULL while ACTIVE |
| ended_by | UUID | FK → account; who removed them (NULL if LEFT) |

Constraints:

- UNIQUE (group_id, account_id). A returning member reactivates the same row through a new invitation.
- The group creator's membership is always `MANAGER`/`ACTIVE`.
- Each group has ≥ 1 active manager.

### 1.6 Invitation (`invitation`)
| Attribute | Type | Constraints |
|---|---|---|
| id | UUID | PK |
| group_id | UUID | FK → app_group |
| invitee_email | VARCHAR(254) | As entered, lower-cased |
| invitee_account_id | UUID | FK → account; NULL if no account matches (invitation then never becomes actionable) |
| invited_by | UUID | FK → account (a manager at invite time) |
| status | VARCHAR(10) | `PENDING` \| `ACCEPTED` \| `DECLINED` \| `REVOKED` \| `EXPIRED` |
| created_at / responded_at / expires_at | TIMESTAMPTZ | expires_at = created + 14 days **[PROPOSAL]**; expiry is evaluated lazily on read |
| version | BIGINT | Optimistic lock; guards against a double accept/decline |

Constraint: at most one `PENDING` invitation per (group_id, invitee_email). In PostgreSQL this is a partial unique index; in H2 it is enforced in the application.

### 1.7 Expense (`expense`)
| Attribute | Type | Constraints |
|---|---|---|
| id | UUID | PK |
| group_id | UUID | FK → app_group |
| submitted_by | UUID | FK → account; an active member at submission |
| payer_id | UUID | FK → account; an **active member** at submission/edit; selected separately from participants |
| amount | NUMERIC(19,4) | > 0; scale ≤ currency scale; ≤ 1,000,000 major units (AS-10) |
| currency_code | CHAR(3) | Copied from the group at submission (snapshot) |
| expense_date | DATE | ≤ today + 1 day (AS-9) |
| description | VARCHAR(200) | NOT NULL, 1–200 chars |
| split_method | VARCHAR(10) | `EQUAL` \| `PERCENTAGE` |
| status | VARCHAR(10) | `PENDING` \| `APPROVED` \| `REJECTED` |
| submitted_at | TIMESTAMPTZ | NOT NULL |
| last_edited_at | TIMESTAMPTZ | NULL unless edited while pending |
| idempotency_key | UUID | NULL allowed; UNIQUE (group_id, submitted_by, idempotency_key) |
| idempotency_request_hash | CHAR(64) | SHA-256 of the canonical command; detects key reuse with a different body ([design.md §8.4](design.md#84-idempotency-idempotencyguard)) |
| version | BIGINT | Optimistic lock |

### 1.8 ExpenseParticipant / allocation (`expense_participant`)
| Attribute | Type | Constraints |
|---|---|---|
| expense_id | UUID | FK → expense; part of PK |
| account_id | UUID | FK → account; part of PK; an active member at submission |
| percentage | NUMERIC(5,2) | NULL for EQUAL; 0.00–100.00 for PERCENTAGE |
| share_amount | NUMERIC(19,4) | ≥ 0; computed by `AllocationCalculator` and **stored**, never recomputed |

### 1.9 ReceiptAttachment (`receipt_attachment`): Alternative A, full scope (deferred in candidate MVP)
| Attribute | Type | Constraints |
|---|---|---|
| expense_id | UUID | PK, FK → expense (0..1 per expense) |
| storage_key | VARCHAR(100) | Random. Not derived from the filename. |
| content_type | VARCHAR(20) | `image/jpeg` \| `image/png` (verified by magic bytes) |
| size_bytes | INT | ≤ 5 MB |
| original_filename | VARCHAR(255) | Sanitized; display only |
| uploaded_by / uploaded_at | UUID / TIMESTAMPTZ | Uploader must be the submitter; allowed while PENDING only |

### 1.10 ReviewDecision (`review_decision`)
| Attribute | Type | Constraints |
|---|---|---|
| id | UUID | PK |
| expense_id | UUID | FK → expense |
| reviewer_id | UUID | FK → account; authorized by ReviewPolicy at decision time |
| decision | VARCHAR(10) | `APPROVED` \| `REJECTED` |
| reason | VARCHAR(500) | Optional; recommended for rejections |
| decided_at | TIMESTAMPTZ | NOT NULL |
| level | SMALLINT | Always 1 in this proposal. Reserved for multi-level approval (OQ-2). |

Constraint: UNIQUE (expense_id, level). With single-level review there is exactly one decision per decided expense.

### 1.11 Notification (`notification`)
| Attribute | Type | Constraints |
|---|---|---|
| id | UUID | PK |
| recipient_id | UUID | FK → account |
| type | VARCHAR(30) | `EXPENSE_SUBMITTED` \| `EXPENSE_REVIEWED` \| `INVITATION_RECEIVED` |
| group_id | UUID | FK → app_group |
| subject_id | UUID | Expense ID or invitation ID |
| summary | VARCHAR(200) | Pre-rendered text, e.g. "Alex submitted 'Groceries' ($42.10), pending review". Generated from data the recipient may already see. |
| created_at | TIMESTAMPTZ | NOT NULL |
| read_at | TIMESTAMPTZ | NULL = unread |

### 1.12 Payment (`payment`)
| Attribute | Type | Constraints |
|---|---|---|
| id | UUID | PK |
| group_id | UUID | FK → app_group |
| sender_id | UUID | FK → account; active member; **= recorded_by** (FR-010) |
| recipient_id | UUID | FK → account; active member; ≠ sender_id |
| amount | NUMERIC(19,4) | > 0; scale ≤ currency scale; ≤ outstanding N(sender, recipient) at record time |
| currency_code | CHAR(3) | Group currency snapshot |
| payment_date | DATE | ≤ today + 1 day |
| note | VARCHAR(200) | Optional |
| recorded_by | UUID | FK → account (the authenticated actor) |
| recorded_at | TIMESTAMPTZ | Server clock |
| idempotency_key | UUID | UNIQUE (group_id, recorded_by, idempotency_key) |
| idempotency_request_hash | CHAR(64) | As for expense |

Payments are **immutable** in this proposal. Voiding or reversing a payment is out of scope ([OPEN], related to OQ-6).

### 1.13 Balance: derived, not stored
`PairwiseBalance(groupId, debtorId, creditorId, amount, currencyCode)` is a **read model** computed by `BalanceService` (formula in §4). Balances are **derived** and not materialized. Rationale:

- A single source of truth: the stored allocations plus payments.
- Reconciliation holds by construction (NFR-003, SC-003).
- No lost updates under concurrent approvals.

A materialized `pair_balance` table is a documented fallback **only** if the NFR-004 measurement fails ([research R-10](research.md#r-10-balance-storage)).

### 1.14 Report: not stored
`ReportQuery` (scope, date range, record types, statuses, party filter) and `ReportResult` (rows, totals, balance movement) are value objects computed on demand from authorized records. No report entity is persisted. Saved reports and exports are out of scope until OQ-5 is answered.

## 2. Relationships

```
Account 1 ── * Membership * ── 1 Group
Group   1 ── * Invitation     (invited_by → Account; invitee_account_id → Account 0..1)
Group   1 ── * Expense        (submitted_by, payer_id → Account)
Expense 1 ── 1..* ExpenseParticipant ── 1 Account
Expense 1 ── 0..1 ReceiptAttachment   [A]
Expense 1 ── 0..1 ReviewDecision      (0..* when multi-level is introduced)
Group   1 ── * Payment        (sender_id, recipient_id, recorded_by → Account)
Account 1 ── * Notification
Account 1 ── * PasswordResetToken [A] | RecoveryCode [B]
```

Indexes **[PROPOSAL]** (for NFR-004):

- `expense(group_id, status, expense_date)`
- `expense_participant(account_id)`
- `payment(group_id, payment_date)`
- `payment(group_id, sender_id, recipient_id)`
- `membership(account_id, status)`
- `notification(recipient_id, read_at, created_at DESC)`
- `invitation(invitee_account_id, status)`

## 3. Money rules

| Rule | Detail |
|---|---|
| M1 Type | `BigDecimal` only. No `double`/`float` for money anywhere, including DTOs and JSON mapping. JSON amounts are **strings** (`"42.10"`) to avoid client floating-point parsing. |
| M2 Scale | `s = Currency.getInstance(code).getDefaultFractionDigits()`. Only currencies with s ∈ {0, 2, 3} are accepted. Currencies with no fraction-digit definition (s = −1) are rejected. |
| M3 Input | Valid iff `amount > 0` and `amount.stripTrailingZeros().scale() ≤ s`. Normalized with `setScale(s)` (exact). Never rounded. |
| M4 Persistence | Written at scale `s` into `NUMERIC(19,4)`. Read back with `setScale(s)` (exact). |
| M5 Arithmetic | Sums and differences are exact. Division occurs **only** inside `AllocationCalculator`, which operates on integer minor units. |
| M6 Equality | `compareTo`, never `equals`. |
| M7 Display | Formatted per currency at scale `s`. Display never alters stored values. |

Allocation algorithm and worked examples: [architecture.md §6.1](architecture.md#61-monetary-precision-and-allocation-decision).

## 4. Balance derivation

For a group *g* and members *a ≠ b*:

```
D(a→b) = Σ share_amount  FROM expense_participant ep JOIN expense e
         WHERE e.group_id = g AND e.status = 'APPROVED' AND e.payer_id = b AND ep.account_id = a
P(a→b) = Σ amount FROM payment WHERE group_id = g AND sender_id = a AND recipient_id = b
N(a,b) = D(a→b) − D(b→a) − P(a→b) + P(b→a)
```

Implemented as one aggregate SQL query per group that returns directed sums. `PairwiseBalanceCalculator` in the domain nets them. The same calculator is unit-tested independently of SQL, and an integration test asserts that SQL and in-memory results agree.

## 5. Invariants

**Money and allocation**

- **I-1:** For every expense, `Σ share_amount = amount` exactly (all statuses, enforced at write).
- **I-2:** Every `share_amount ≥ 0` and has scale ≤ currency scale.
- **I-3:** PERCENTAGE: every percentage is in [0, 100] with scale ≤ 2, and Σ percentage = 100.00 exactly. EQUAL: all percentages NULL, and shares differ by at most one minor unit.
- **I-4:** An expense's currency equals its group's currency.

**Payer and participant roles**

- **I-5:** `payer_id` refers to an active member of the expense's group at submission/edit time.
- **I-6:** The participant set is non-empty and has no duplicates. Every participant was an active member at submission/edit time. Allocations exist **only** for participants (no shares for non-participants).
- **I-7:** The payer is chosen independently. The payer **may or may not** be a participant (spec FR-003). A payer's own share produces no debt.

**Approval**

- **I-8:** Only `APPROVED` expenses contribute to D(·). `PENDING` and `REJECTED` contribute nothing (FR-006, AC-010, AC-012).
- **I-9:** Status transitions are only PENDING → APPROVED and PENDING → REJECTED. Both are terminal. Each decided expense has exactly one `ReviewDecision` (level 1) whose decision matches its status.
- **I-10:** The reviewer was authorized under ReviewPolicy at decision time. Reviewer ≠ submitter unless `self_approval_allowed`.
- **I-11:** Amount, payer, participants, split, and receipt may change only while PENDING (by the submitter).

**Payments and reconciliation**

- **I-12:** `sender_id ≠ recipient_id`, both active members at record time, and `recorded_by = sender_id`.
- **I-13:** At record time, `0 < amount ≤ N(sender, recipient)` (evaluated under the group lock). So **no payment alone ever reverses a balance's direction** (AC-015).
- **I-14:** `recorded_at` is the server time of the commit. Payments are immutable and immediately included in N (AC-016).
- **I-15 (reconciliation):** For every group and pair, the displayed balance = the formula in §4 over the stored rows. Summed over all members, the net positions in a group total zero.

**Membership**

- **I-16:** Each group has ≥ 1 ACTIVE MANAGER. The creator's membership cannot be removed or demoted.
- **I-17:** A member with N ≠ 0 against anyone in the group, or who is payer or participant of a PENDING expense, cannot be removed or leave (D-2).
- **I-18:** Historical expenses, decisions, and payments keep their original account IDs after membership ends (AC-004).

## 6. Lifecycles and state transitions

### 6.1 Expense
| From | Event | Guard | To |
|---|---|---|---|
| (new) | submit | active member; validation passes | PENDING (+ notifications to other active members) |
| PENDING | edit | actor = submitter; version matches; validation passes | PENDING (allocations recomputed; version+1) |
| PENDING | approve | ReviewPolicy allows; version matches | APPROVED (+ ReviewDecision; affects balances) |
| PENDING | reject | ReviewPolicy allows; version matches | REJECTED (+ ReviewDecision; never affects balances) |
| APPROVED / REJECTED | any | n/a | `EXPENSE_NOT_PENDING` |

### 6.2 Invitation
| From | Event | Actor | To |
|---|---|---|---|
| (new) | invite | manager | PENDING |
| PENDING | accept | invitee | ACCEPTED (membership ACTIVE/MEMBER) |
| PENDING | decline | invitee | DECLINED |
| PENDING | revoke | manager | REVOKED |
| PENDING | time passes expires_at | system (checked lazily on read) | EXPIRED |

### 6.3 Membership
| From | Event | Actor | Guard | To |
|---|---|---|---|---|
| (none) / REMOVED / LEFT | invitation accepted | invitee | n/a | ACTIVE (MEMBER) |
| ACTIVE MEMBER | designate manager | manager | n/a | ACTIVE MANAGER |
| ACTIVE MANAGER | revoke manager | manager | not creator; ≥ 1 manager remains | ACTIVE MEMBER |
| ACTIVE | remove | manager | not creator; I-16, I-17 | REMOVED |
| ACTIVE | leave | self | not creator; I-16, I-17 | LEFT |

### 6.4 Notification
Created unread. `read_at` is set when the recipient marks it read. No deletion in the MVP.

### 6.5 Payment
Recorded once and immutable.

## 7. Platform differences

| Element | A: Web | B: Desktop [CONDITIONAL-B] |
|---|---|---|
| Recovery entity | PasswordResetToken | RecoveryCode |
| ReceiptAttachment | Full scope (deferred in MVP) | Deferred |
| DB engine | PostgreSQL | H2 file |
| Partial unique index (invitations) | DB-enforced | Application-enforced |

Everything else is identical. The same Flyway scripts are used, with one Postgres-only migration for the partial index guarded by a vendor-specific location (`db/migration/{vendor}`).
