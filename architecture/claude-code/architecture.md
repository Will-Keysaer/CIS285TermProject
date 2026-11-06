# SplitSmart — Independent Architecture and Design Proposal (Claude Code)

| Field | Value |
|---|---|
| Status | **Draft proposal. Needs human review (Constitution VII).** Not approved, not implemented. |
| Date | 2026-11-05 |
| Authoritative inputs | `UserRequirement.md`, `specs/001-splitsmart-expense-sharing/spec.md`, `.specify/memory/constitution.md` |
| Independence | Written without reading or using any Spec Kit architecture/design outputs. When this was written, the Spec Kit plan artifacts (`plan.md`, `research.md`, `data-model.md`, `contracts/`, `quickstart.md`) did not exist in the repository. |
| Companion documents | [design.md](design.md) (detailed, implementation-level design) · [research.md](research.md) · [data-model.md](data-model.md) · [contracts/](contracts/README.md) · [quickstart.md](quickstart.md) · [traceability.md](traceability.md) · [diagrams/](diagrams/) |

Labels used throughout:

- **[FACT]**: stated in an authoritative input, or observed in the repository.
- **[DECISION]**: a design decision this proposal makes, with rationale in [research.md](research.md).
- **[PROPOSAL]**: a technology or policy choice that the requirements do not specify. Alternatives are listed.
- **[ASSUMPTION]**: something taken as true until someone confirms or corrects it.
- **[OPEN]**: an unresolved decision that needs customer or project-owner input.
- **[CONDITIONAL-B]**: applies only if the customer gives documented approval for the Swing alternative.

---

## 1. Executive summary

SplitSmart lets account holders share expenses inside groups. Members submit expenses with equal or percentage splits. Expenses are reviewed before they affect balances. Pairwise balances are tracked, payments are recorded, and members get reports. **[FACT]** The customer requires a **primarily web-based** product (NFR-001). **[FACT]** The constitution says Swing is only an implementation decision, and that a Swing-only desktop application does not satisfy NFR-001.

**The delivery platform is unresolved and potentially blocking (spec Open Question 1).** This proposal does not settle it. Instead it:

1. Defines a **platform-independent core** (domain and application layers, Maven modules `splitsmart-domain` and `splitsmart-application`). This core holds every business rule: money, allocation, approval, balances, payments, authorization policy, and reports. The core is identical under both alternatives and can be built and tested **before** the platform decision is made.
2. Describes two separate, unblended delivery architectures:
   - **Alternative A, Web (baseline-compliant):** a Java 21 Spring Boot modular monolith serving a JSON REST API and a browser client, backed by PostgreSQL.
   - **Alternative B, Swing desktop MVP [CONDITIONAL-B]:** a standalone Java 21 Swing application with an embedded file database. It is viable **only** with documented customer approval of the platform change. **It does not satisfy NFR-001.**
3. **Recommends Alternative A** as the default. It is the only alternative that meets the customer baseline. Alternative B is documented so the customer can make an informed choice. It is not presented as equivalent.

### 1.1 Scope labels

| Scope label | Contents |
|---|---|
| **Full customer scope** | FR-001 – FR-012, NFR-001 – NFR-005: accounts and recovery, groups, invitations, roles, expenses with equal/percentage splits and optional receipts, notifications, review/approval, pairwise balances, payments, summaries, and date-range reports, delivered primarily on the web. |
| **Candidate MVP scope** (from spec; platform-neutral functional slice) | FR-001 – FR-011 with one review step, in-app notifications, and basic summaries. |
| **Deferred from candidate MVP** (still in the customer scope) | Optional receipt images (part of FR-003), custom/date-range reports (FR-012), multiple approval levels, notification channels beyond in-app. |
| **NFR-001 status by alternative** | **A:** met by the MVP. **B:** **not met.** It stays an outstanding customer requirement that a later web delivery must fulfil. |
| **Decisions requiring customer approval** | Platform (OQ-1); approver rules and self-approval (OQ-2); notification channels (OQ-3); currencies (OQ-4); report export and filters (OQ-5); editing/deleting expenses (OQ-6); plus the new items in §11. |

---

## 2. Platform alternatives and decision analysis

### 2.1 The conflict, preserved

- **[FACT]** `UserRequirement.md`: "OS supported: primary web-based."
- **[FACT]** spec NFR-001: "A desktop Swing application alone does not satisfy this requirement."
- **[FACT]** constitution VIII: Swing as the primary interface "requires documented customer approval of the resulting change in delivery platform."
- **[FACT]** spec OQ-1 asks whether the customer approves a Swing desktop MVP. **There is no recorded answer.**

This proposal therefore treats the platform as **[OPEN] and blocking for presentation and infrastructure work**. It is **not blocking for the core** (see §2.5).

### 2.2 Alternative A: primarily web-based (recommended)

A single deployable Spring Boot application (modular monolith):

- **Presentation:** a JSON REST API (`/api/v1`, see [contracts/rest-api.md](contracts/rest-api.md)) plus a browser single-page client served from the same origin. **[PROPOSAL]**
- **Application and domain:** the shared core.
- **Infrastructure:** PostgreSQL through Spring Data JPA with Flyway migrations, receipt file storage, an SMTP email adapter for account recovery, and in-app notifications stored in the database. **[PROPOSAL]**

### 2.3 Alternative B: Java Swing desktop MVP [CONDITIONAL-B]

A standalone desktop application:

- **Presentation:** Swing views with presenters/view-models (see [contracts/swing-ui.md](contracts/swing-ui.md)).
- **Application and domain:** the same shared core.
- **Infrastructure:** an embedded H2 file database on the local machine, with the same JPA persistence module and Flyway migrations. **[PROPOSAL]**

There is no server. All users of a group must use the **same installation on the same computer** (for example, a shared household PC). This is a structural limitation, not an implementation detail.

> A third variant, a Swing client talking to a networked server, was considered and **rejected for this proposal**. It requires building most of Alternative A's server (API, authentication, hosting) and still does not satisfy NFR-001. In effect it is a blend of A and B that costs more than either. See [research.md R-02](research.md#r-02-swing-variant).

### 2.4 Comparison

| Dimension | A: Web | B: Swing desktop [CONDITIONAL-B] |
|---|---|---|
| **NFR-001 (primary web)** | **Satisfied** | **Not satisfied.** It stays an open customer requirement. |
| FR-001 accounts / recovery | Email + password; recovery by emailed one-time reset token | Email + password stored locally; recovery by **one-time recovery codes** shown at registration. Email recovery would need SMTP from the desktop app, which is possible but adds credential distribution problems. |
| FR-002 groups / invitations | Invitations reach users on any device; acceptance happens in their own session | Invitee must sign in **on the same installation** to accept. People on different computers cannot be invited in any meaningful way. |
| FR-003 receipt images | Upload to server storage, authorized download | Deferred in the candidate MVP. If added, a local file copy. |
| FR-008 notifications | In-app notification inbox, visible on next request from any device | In-app inbox visible only when the recipient next signs in **on that machine** |
| FR-009/010 balances, payments | Shared, consistent, multi-user, concurrent | Correct, but only for users of that installation. Effectively one user at a time. |
| FR-011/012 reports | Browser views | Swing views. FR-012 is deferred in the candidate MVP. |
| Security / privacy (NFR-002) | Server-enforced authorization; HTTPS; data not on user devices | Authorization is enforced by the app, but **anyone with file-system access to the DB file can read all groups' data** unless the file is encrypted. Weaker privacy boundary. |
| Deployment | A host (container or VM), PostgreSQL, HTTPS certificate, SMTP account | Install a JDK 21 runtime (or a `jpackage` bundle) on each machine. No server. |
| Concurrency | Real concurrent users; needs transactions and locking (§6.3) | Single process. Locking is still implemented in the shared core but rarely contended. |
| UX | Any modern browser; responsive; reachable from phones | Desktop only; one machine; familiar to course staff |
| Testing | Core JUnit tests + MockMvc API tests + PostgreSQL integration tests + browser smoke tests | Core JUnit tests + H2 integration tests + presenter tests + optional AssertJ-Swing GUI smoke tests |
| Relative complexity (qualitative, not measured) | **Higher**: auth/session/CSRF, hosting, email, a front-end build | **Lower** for a demo, but the work does not move toward NFR-001 except through the shared core |
| Path to full customer scope | Direct | Requires a second presentation and infrastructure build (web) later. Only the core carries over. |

### 2.5 Implications of each choice

- **Choosing A** meets the baseline. It costs more up front: security plumbing, deployment, and a front end. All MVP work moves toward the full product.
- **Choosing B** (only with documented approval) gives a cheaper demonstrable MVP. However:
  - (a) NFR-001 remains unmet and must be tracked as outstanding.
  - (b) Multi-person, multi-device use is not supported in practice.
  - (c) The Swing presentation and embedded-DB setup are largely throw-away when the web product is built.
  - (d) Privacy is weaker.
- **Either way**, the shared core (`splitsmart-domain`, `splitsmart-application`) and the persistence schema are reused unchanged. This is the main hedge. **[DECISION]** Start with the core and its JUnit tests now, and do not start presentation work until OQ-1 is answered.

### 2.6 Recommendation

**[DECISION] Recommend Alternative A.** Alternative B may be adopted only if the customer provides written approval stating that a Swing desktop MVP is acceptable and that NFR-001 remains an outstanding requirement. The approval must be recorded in the project (constitution I and VIII). Without that approval, Alternative B must not be built as the product's primary interface.

---

## 3. Logical architecture

> Class-level responsibilities, method contracts, algorithms, and transaction details for everything in this section are in [design.md](design.md).

### 3.1 Layers and Maven modules [PROPOSAL]

```
splitsmart (parent POM, Java 21, JUnit 5)
├── splitsmart-domain          shared   pure Java; no framework dependencies
├── splitsmart-application     shared   use-case services + ports (interfaces)
├── splitsmart-persistence     shared   JPA entities/repositories, Flyway migrations
├── splitsmart-web             A only   Spring Boot app: REST controllers, security, SPA assets, SMTP, receipt storage
└── splitsmart-desktop         B only   [CONDITIONAL-B] Swing views, presenters, local wiring
```

Dependency rule (enforced by module boundaries and an ArchUnit test, **[PROPOSAL]**):

```
web | desktop  ──►  application  ──►  domain
      │                  ▲
      └──► persistence ──┘   (persistence implements application ports; depends on application + domain)
```

- `domain` depends on nothing but the JDK.
- `application` depends only on `domain`. It defines **ports**: repository interfaces, `Clock`, `PasswordHasher`, `NotificationPublisher`, `ReceiptStorage`, `RecoveryChannel`, `TransactionRunner`.
- Presentation modules never touch persistence entities. They call application services with command objects and receive result DTOs.
- **Constitution III:** no business rule, split calculation, or balance computation in controllers, Swing listeners, or views.

Diagrams: [component-web.puml](diagrams/component-web.puml), [component-swing.puml](diagrams/component-swing.puml), [context-web.puml](diagrams/context-web.puml).

### 3.2 Domain layer (`splitsmart-domain`)

| Component | Responsibility |
|---|---|
| `Money`, `CurrencyPolicy` | Immutable `BigDecimal` amount plus ISO-4217 currency. Scale is fixed to the currency's minor-unit digits. Rejects excess precision; never rounds silently (§6.1). |
| `Percentage` | `BigDecimal` in [0, 100] with at most 2 decimal places **[ASSUMPTION]**. |
| `AllocationCalculator` | Equal and percentage splits using largest-remainder distribution with a deterministic tie-break. Guarantees Σshares = amount (§6.1). |
| `Expense` (aggregate) | Holds payer, participants, allocations, and status. Enforces the state machine PENDING → APPROVED or REJECTED, and edits only while PENDING. |
| `ReviewPolicy` | Decides whether an actor may approve or reject a given expense (role and self-approval rules). |
| `PairwiseBalanceCalculator` | Computes the net balance per member pair from approved allocations and payments (§6.2). |
| `PaymentRules` | Payment validation: positive, sender ≠ recipient, not over the outstanding amount, date not in the future. |
| `GroupMembershipRules` | Role checks, invitation transitions, removal preconditions (last manager, outstanding balance). |
| `DateRange` | Inclusive [from, to]. Invalid if `to < from`. |
| Domain exceptions | `DomainRuleViolation` with a stable `ErrorCode` (see [contracts/application-services.md §7](contracts/application-services.md#7-error-codes)). |

### 3.3 Application layer (`splitsmart-application`)

One service per use-case area. Each public method:

1. Takes an `ActorContext`, the authenticated account ID supplied by presentation and never by the client body.
2. Authorizes the call.
3. Validates the command.
4. Runs inside a single transaction through `TransactionRunner`.
5. Returns an immutable result.

| Service | Use cases | FRs |
|---|---|---|
| `AccountService` | register, authenticate, start/complete recovery, change password, view own profile | FR-001 |
| `GroupService` | create group, update details, designate/revoke manager, invite, accept/decline/revoke invitation, remove member, leave, list my groups | FR-002 |
| `ExpenseService` | preview allocation, submit, edit pending, get, list group expenses, attach/get receipt | FR-003 – FR-006 |
| `ReviewService` | approve, reject, list pending reviews for me | FR-006, FR-007 |
| `NotificationService` | publish on submit/review/invite; list, mark read | FR-008 |
| `BalanceService` | group pairwise balances, my balances across groups | FR-009 |
| `PaymentService` | record payment, list payments | FR-010 |
| `ReportService` | summary, custom date-range report | FR-011, FR-012 |
| `AuthorizationService` | `requireActiveMember`, `requireManager`, `requireParty`; used by all services | NFR-002 |

Full contracts: [contracts/application-services.md](contracts/application-services.md).

### 3.4 Infrastructure

| Component | A: Web | B: Desktop [CONDITIONAL-B] |
|---|---|---|
| Persistence | `splitsmart-persistence` on PostgreSQL **[PROPOSAL]** | Same module on embedded H2 file DB **[PROPOSAL]** |
| Migrations | Flyway (same scripts; Postgres-compatible SQL subset that H2 accepts in PostgreSQL mode) | Same |
| Password hashing | BCrypt via Spring Security `PasswordEncoder` behind the `PasswordHasher` port **[PROPOSAL]** | Same library, same port |
| Recovery channel | `EmailRecoveryChannel` (SMTP) **[PROPOSAL]**; console adapter in dev | `RecoveryCodeChannel` (one-time codes, hashed at rest) |
| Notifications | DB-backed in-app inbox; `NotificationPublisher` port allows email later (OQ-3) | DB-backed in-app inbox |
| Receipts | `FileSystemReceiptStorage` on a private volume **[PROPOSAL]**; object storage alternative | Deferred |
| Clock | `java.time.Clock` (UTC) injected | Same |

---

## 4. Major workflows

Sequence diagrams: [seq-expense-submit-approve.puml](diagrams/seq-expense-submit-approve.puml), [seq-payment-record.puml](diagrams/seq-payment-record.puml). State machine: [expense-states.puml](diagrams/expense-states.puml).

### 4.1 Account and recovery (FR-001)

1. **Register.** Email (normalized to lower case) + password (min 12 chars **[PROPOSAL]**) → `Account` created with a BCrypt hash.
2. **Sign in.**
   - A: Spring Security form/JSON login creates a server session. The cookie is `HttpOnly; Secure; SameSite=Lax`.
   - B: the app keeps an in-memory `ActorContext` until sign-out.
3. **Recover.**
   - A: request reset → the response is identical whether or not the email exists → a 256-bit random token is emailed. Only its SHA-256 hash is stored. It expires in 30 min, is single-use, and all sessions are invalidated on reset.
   - B: enter email + one unused recovery code + new password.

### 4.2 Group creation and membership (FR-002)

1. **Create group.** Name, optional description, currency (default `USD` **[ASSUMPTION]**). The creator gets membership `MANAGER`, and `Group.createdBy` is recorded.
2. **Invite.** Allowed for managers only. The invitee is an existing account identified by email (spec assumption). The API reply is the same whether or not the email has an account (§5.3). An `Invitation(PENDING)` is created and an in-app notification goes to the invitee if the account exists.
3. **Accept or decline.** The invitee only. Accept creates an `ACTIVE MEMBER` membership. The invitee is **not** a member until they accept (AC-003).
4. **Remove a member.** Managers only.
   - Rejected if the target is the creator, the last remaining manager, or has a non-zero balance or pending expenses in the group **[DECISION]**.
   - Membership becomes `REMOVED`.
   - Historical expenses and payments keep their original account references (AC-004).
5. **Designate or revoke a manager.** The creator or a manager may designate. The creator's manager role cannot be revoked.

### 4.3 Expense submission (FR-003 – FR-005, FR-008)

1. An active member submits: amount, date, description, split method, payer, participants, percentages (if PERCENTAGE), and an optional receipt.
2. Validation:
   - Amount > 0, with no more decimal places than the group currency allows (§6.1).
   - Payer is an active member.
   - Participants are a non-empty set of distinct active members.
   - The payer may or may not be among the participants (FR-003).
   - Percentages exist for exactly the participants, each ≥ 0, scale ≤ 2, sum exactly 100.00.
   - Date is not later than today + 1 day (timezone tolerance) **[ASSUMPTION]**.
3. `AllocationCalculator` computes shares. The stored shares are authoritative, so the split is never recomputed later.
4. The expense is saved as **PENDING**. It does not affect balances (FR-006, AC-010).
5. In the **same transaction**, an `EXPENSE_SUBMITTED` notification is inserted for every other active member (FR-008, AC-013).
6. Receipt (A): uploaded in a separate request after creation (`PUT .../receipt`). This keeps expense JSON simple and lets submission succeed without an image (AC-006).

### 4.4 Review (FR-006, FR-007)

1. **Default review policy** **[PROPOSAL, pending OQ-2]**:
   - Approvers are the group's **managers**.
   - The submitter may **not** approve or reject their own expense unless the group setting `selfApprovalAllowed = true`. The default is false.
   - One approval is required (single-level).
2. **Approve:** the conditional transition PENDING → APPROVED, using the expense's `version`, creates a `ReviewDecision`. Balances then include the expense (AC-011).
3. **Reject:** PENDING → REJECTED with an optional reason. Balances never include it (AC-012).
4. **Edit while pending:** the submitter may edit. Approved and rejected expenses are immutable **[DECISION, pending OQ-6]**.
5. **Notifications:** an `EXPENSE_REVIEWED` notification goes to the submitter, payer, and participants (except the reviewer).
6. **Edge case: the submitter is the only manager and self-approval is off.** The expense stays pending. The UI tells the submitter that another manager must be designated or self-approval enabled.

### 4.5 Balances and payments (FR-009, FR-010)

1. **Balances** are derived on read from approved allocations and payments (§6.2). They are never stored as a running total **[DECISION]**.
2. **Record payment:** the actor must be the **sender** (FR-010). Sender and recipient must be distinct active members.
3. **Amount and date:** amount > 0 at the currency scale; payment date ≤ today + 1 day.
4. **Serialization:** inside the transaction the group row is locked, then the outstanding amount the sender owes the recipient is computed. The amount must be ≤ that outstanding amount. Otherwise the error is `PAYMENT_EXCEEDS_OUTSTANDING` **[DECISION]**, which satisfies AC-015's "does not become negative solely because of an overpayment."
5. **Persist:** `recordedBy = actor`, `recordedAt = clock.now()`. The balance reflects the payment immediately (AC-016).
6. **Retries:** prevented by an idempotency key.
7. **Possible duplicates:** a payment with the same sender, recipient, amount, and date in the last 24 h requires explicit confirmation (`confirmPossibleDuplicate = true`).

### 4.6 Reports (FR-011, FR-012)

- **Summary** (MVP), per group or across all of my groups:
  - Total approved expenses I paid.
  - My total approved share.
  - Pending count.
  - Current pairwise balances involving me.
  - Payments sent and received.
- **Custom report** (deferred in the candidate MVP; full scope):
  - Inclusive date range on the **business date** (`expenseDate`, `paymentDate`).
  - Filters: group (one of mine, or all), record type (expenses, payments, both), expense status, and "only records I am a party to."
  - Output: matching records, totals, and net balance movement in range.
  - `to < from` → `INVALID_DATE_RANGE` (AC-019).
  - Export is out of scope until OQ-5 is answered.

---

## 5. Security, privacy, authorization, data ownership (NFR-002)

### 5.1 Authorization model

| Rule | Enforced in |
|---|---|
| Every operation needs an authenticated `ActorContext`, except register, sign-in, and recovery. | Presentation (A: Spring Security filter chain; B: sign-in gate) |
| Group-scoped reads and writes require an `ACTIVE` membership in that group. | `AuthorizationService.requireActiveMember` (application layer, **not** UI) |
| Invite, remove, designate manager, and update group require `MANAGER`. | `requireManager` |
| Approve and reject follow `ReviewPolicy` (manager; not the submitter unless allowed). | Domain |
| Edit a pending expense: submitter only. Record a payment: sender only. Accept or decline an invitation: invitee only. | Application services |
| Former members (`REMOVED`/`LEFT`) keep **read-only** access to records they were a party to (payer, participant, sender, recipient), so FR-001 "access their own information" holds. They lose access to the rest of the group. | `requireParty` |
| A notification, profile, or invitation is visible only to its owner or recipient. | Application services |

All IDs in the API are random UUIDs **[DECISION]**. That does not replace authorization checks; it makes enumeration impractical.

### 5.2 No information leakage

- **A:** a resource in a group where the caller is not a member returns **404 `NOT_FOUND`**, not 403. Existence is never confirmed. A member who lacks a role gets **403 `FORBIDDEN_ROLE`**; this reveals nothing they could not already see.
- Error messages never include other users' amounts or balances.
- Report and summary queries are always scoped by group IDs from the caller's memberships, computed on the server.

### 5.3 Account-enumeration posture [DECISION]

- Recovery requests and invitations return uniform responses.
- **Registration does reveal** "email unavailable." This is a usability trade-off, recorded as a risk. The alternative is a verification-email-first sign-up flow (see §11).

### 5.4 Web-specific controls (A)

- TLS everywhere; HSTS.
- Session cookie flags as in §4.1.
- CSRF protection on with Spring Security's cookie token (`XSRF-TOKEN`, double-submit) for the same-origin SPA.
- Content Security Policy `default-src 'self'`.
- Login and recovery rate limiting per IP and per account **[PROPOSAL]**: 5 failures per 15 min, then a delay.
- Receipts:
  - Only JPEG and PNG are accepted, checked by magic bytes, not just by `Content-Type`.
  - Max 5 MB **[PROPOSAL]**.
  - Stored under a random key outside the web root.
  - Served only through the authorized endpoint with `Content-Disposition: attachment` and `X-Content-Type-Options: nosniff`.
  - EXIF metadata (for example GPS) is stripped on upload **[PROPOSAL]**.

### 5.5 Desktop-specific controls (B) [CONDITIONAL-B]

- The DB file lives in the OS user-profile directory.
- Optional H2 file encryption **[PROPOSAL]**. Without it, the privacy boundary depends on OS file permissions. This weakness must be disclosed to the customer.
- Passwords and recovery codes are hashed exactly as in A.

### 5.6 Data ownership

| Data | Owner and visibility |
|---|---|
| Account profile and credentials | The account holder only. Never exposed to other users except display name and email inside shared groups. |
| Groups, expenses, allocations, review decisions, payments | The group. Visible to its active members, and to former members only for records they were party to. |
| Receipts | The group, under the same visibility as the expense. |
| Notifications | The recipient only. |

Deleting an account is **out of scope [OPEN]**. Financial history must stay attributable (AC-004), so deletion would need anonymization rules the customer has not given.

---

## 6. Persistence, transactions, consistency, errors

### 6.1 Monetary precision and allocation [DECISION]

**Representation and scale**

- `java.math.BigDecimal` everywhere. `double` and `float` are banned for money (enforced by an ArchUnit rule plus code review).
- **Currency scale:** the group's ISO-4217 currency fixes the scale at `java.util.Currency.getDefaultFractionDigits()` (USD → 2, JPY → 0). The currency is chosen at group creation and **cannot change** after the first expense.
- **Input:** an amount is accepted only if it is > 0 and `amount.stripTrailingZeros().scale() <= currencyScale`. `12.50` and `12.5` are accepted for USD; `12.505` is rejected with `AMOUNT_PRECISION_EXCEEDED`. **Input is never rounded.** After validation, the value is normalized with `setScale(currencyScale)`, which is exact.
- **Maximum:** amount ≤ 1,000,000.00 in major units **[ASSUMPTION]**, to bound arithmetic and catch typos.
- **Persistence:** `NUMERIC(19,4)` columns. Values are always written at the currency scale and read back with `setScale(currencyScale)`, which is exact because scale 4 is at least any supported scale. The column type is the same for every currency.
- **Comparisons:** always `compareTo`, never `equals`, because `BigDecimal.equals` is scale-sensitive.

**Allocation algorithm** (work in integer minor units, so no division rounding is ambiguous):

```
A      = amount.movePointRight(scale)            // exact integer, e.g. $100.00 → 10000
order  = participants sorted by accountId (UUID string, ascending)   // deterministic tie-break
EQUAL:      raw_i = A / n                         // exact rational
PERCENTAGE: raw_i = A × p_i / 100                 // exact: p_i has scale ≤ 2, so A×p_i/100 has scale ≤ 4
floor_i = ⌊raw_i⌋ ;  rem_i = raw_i − floor_i
L = A − Σ floor_i                                 // 0 ≤ L < n
sort participants by rem_i descending, then by `order`
give +1 minor unit to the first L participants
share_i = (floor_i + bonus_i).movePointLeft(scale)
assert Σ share_i == amount                        // invariant; failure is a bug → exception
```

Worked examples (these become JUnit cases):

| Case | Input | Shares |
|---|---|---|
| Equal, 3 ways | $100.00, participants P1<P2<P3 | 33.34, 33.33, 33.33 |
| Percent | $10.00 at 33.33 / 33.33 / 33.34 | raw 333.3 / 333.3 / 333.4 → floors 999, L=1 → 3.33, 3.33, **3.34** |
| Tie on remainder | $0.05 at 50 / 50 | raw 2.5 / 2.5 → L=1, tie → first by order: 0.03, 0.02 |
| JPY, equal | ¥1000 among 3 | 334, 333, 333 |
| 0% participant | $20.00 at 100 / 0 | 20.00, 0.00 (allowed: percentages are non-negative) |

### 6.2 Balances: derived, not materialized [DECISION]

For a group, with `share_e(x)` the stored allocation of approved expense *e* to member *x*:

```
D(a→b) = Σ over APPROVED expenses e with payer(e)=b of share_e(a)      (a ≠ b)
P(a→b) = Σ payments with sender a, recipient b
N(a,b) = D(a→b) − D(b→a) − P(a→b) + P(b→a)
N > 0 ⇒ "a owes b N";  N < 0 ⇒ "b owes a |N|";  N = 0 ⇒ settled
```

A payer's own share creates no debt. PENDING and REJECTED expenses contribute nothing.

**Why derived:**

- There is one source of truth: allocations and payments. Balances reconcile to the underlying records **by construction** (NFR-003, SC-003).
- No running totals can drift.
- Concurrent approvals cannot lose updates, because there is no counter to race on.

**Performance:** at NFR-004 scale (≤ 50 members, ≤ 10,000 records per group) the derivation is one indexed `GROUP BY` aggregate over at most tens of thousands of allocation rows. This is expected to be well within budget. **This is an expectation, not a measurement.** It must be verified by the NFR-004 performance test. If the test fails, add a materialized `pair_balance` table maintained in the same transaction as approvals and payments, with a reconciliation check against the derived value. That fallback is documented but not built.

**Cross-group view:** "my balances" lists my non-zero pairs per group, plus a per-counterparty total **per currency**. Different currencies are never summed together. Payments settle balances **within one group** **[ASSUMPTION]**.

### 6.3 Transactions and concurrency

| Operation | Consistency mechanism |
|---|---|
| Submit expense + allocations + notifications | One transaction. All-or-nothing. |
| Approve or reject | One transaction. `UPDATE expense SET status=?, version=version+1 WHERE id=? AND status='PENDING' AND version=?`. Zero rows → `EXPENSE_NOT_PENDING` or `CONCURRENT_MODIFICATION` (HTTP 409). Two simultaneous approvers: exactly one wins. Approve vs. reject: exactly one wins. Edit vs. approve: whichever commits first. The edit carries `version`, so a stale approval fails. |
| Edit pending expense | Optimistic lock on `version`. Allocations are replaced in the same transaction. |
| Record payment | One transaction. `SELECT … FROM app_group WHERE id=? FOR UPDATE` serializes balance-affecting writes per group. Then compute N(sender, recipient), validate, insert. This prevents two concurrent payments that are each ≤ outstanding but sum to more. |
| Approve (lock) | Also takes the group row lock, so a payment validation always sees a consistent set of approved expenses. Contention is negligible at ≤ 50 members. |
| Membership removal | Takes the group lock, then checks balance = 0 and no pending involvement. |
| Idempotency | `payment(group_id, recorded_by, idempotency_key)` and `expense(group_id, submitted_by, idempotency_key)` are unique. A retry with the same key and same body returns the original result; same key with a different body → `IDEMPOTENCY_KEY_REUSED`. |

Isolation level: PostgreSQL default `READ COMMITTED`, plus the explicit row lock and conditional updates above. H2 (B) gives the same semantics through `SELECT … FOR UPDATE`.

### 6.4 Error handling

- **Domain and application:** a typed `DomainRuleViolation(ErrorCode, field?, message)`, `NotFoundException`, `ForbiddenException`, and `ConcurrencyConflictException`. Validation collects **all** field errors for a command, not just the first.
- **A:** RFC 9457 `application/problem+json` with a stable `code` and `errors[]` (see [contracts/rest-api.md §2](contracts/rest-api.md#2-conventions)). 400 validation, 401 unauthenticated, 403 role, 404 not found or not visible, 409 state/concurrency/duplicate, 413 receipt too large, 415 unsupported type, 422 business-rule violation, 429 rate limited.
- **B:** presenters map the same `ErrorCode`s to inline field messages or dialogs (see [contracts/swing-ui.md](contracts/swing-ui.md)).
- **Unexpected errors:** logged with a correlation ID. The user sees a generic message. No stack traces or financial data in responses.
- **Logging:** never log passwords, tokens, recovery codes, or receipt contents.

---

## 7. Deployment and runtime

### 7.1 Alternative A [PROPOSAL]

- **Artifact:** a single executable Spring Boot JAR (`splitsmart-web`) containing the API and the built SPA assets. Optional container image.
- **Runtime:** JRE 21. PostgreSQL 16 (managed or self-hosted). A persistent private volume (or object storage) for receipts. An SMTP relay for recovery email.
- **Configuration:** environment variables for DB URL and credentials, SMTP, receipt path, and allowed origin. No secrets in the repository.
- **Environments:**
  - `local`: Docker Compose PostgreSQL + console email adapter.
  - `test`: H2 for fast tests, and PostgreSQL for the integration profile.
  - `demo/prod`: hosted.
- **Operations:**
  - Flyway runs migrations at start-up.
  - A health endpoint (`/actuator/health`) is exposed; other actuator endpoints are disabled.
  - Daily DB backup **[PROPOSAL]**.
- **Hosting provider:** **[OPEN]**. Course infrastructure, a university VM, or a PaaS. It does not affect the architecture.

### 7.2 Alternative B [CONDITIONAL-B]

- **Artifact:** `splitsmart-desktop` JAR, or a native installer via `jpackage` that bundles a Java 21 runtime **[PROPOSAL]**.
- **Data:** an H2 file in `%APPDATA%/SplitSmart` (Windows) or `~/.splitsmart`. Flyway migrates on start.
- **Backup:** the user copies the DB file. No multi-machine sync.
- **Threading:** all UI work on the Event Dispatch Thread. Application-service calls run in `SwingWorker`s so the UI does not freeze.

---

## 8. Testing and verification strategy

| Level | Scope | Tooling [PROPOSAL] | Runs in |
|---|---|---|---|
| Domain unit | Money, AllocationCalculator (incl. §6.1 table + property-based "Σ = amount" over random inputs), state machine, ReviewPolicy, PairwiseBalanceCalculator, PaymentRules, DateRange | JUnit 5 (+ jqwik for properties, optional) | `mvn verify` (all modules) |
| Application | Each service with in-memory port fakes: authorization denials, validation, notifications created, idempotency | JUnit 5 + fakes (no Spring, no GUI) | `mvn verify` |
| Persistence integration | Repositories, migrations, balance aggregate SQL = domain calculator, row lock and conditional update under two threads | JUnit 5 + H2 (default) / PostgreSQL via Testcontainers (profile `-Ppostgres`) | `mvn verify` / `mvn verify -Ppostgres` |
| API contract (A) | Every endpoint in rest-api.md: status codes, problem codes, 404-not-403 rule, CSRF | Spring MockMvc + JUnit 5 | `mvn verify` |
| Presenter (B) | Presenters call services and map errors; no Swing needed | JUnit 5 | `mvn verify` |
| E2E smoke | Quickstart scenarios S1–S8 | A: Playwright for Java (optional); B: manual script / AssertJ-Swing (optional) | manual or `-Pe2e` |
| Performance (NFR-004) | Seed 50 members + 10,000 records; measure p95 of account/expense/balance/report views ≤ 3 s | JUnit-driven HTTP timing harness or Gatling **[PROPOSAL]** | `-Pperf`, on demand |

**Constitution V:** GUI and E2E tests supplement but never replace the business-logic tests.

---

## 9. Risks and trade-offs

| # | Risk / trade-off | Impact | Mitigation |
|---|---|---|---|
| R1 | Platform decision (OQ-1) stays unresolved | Presentation work cannot start; schedule slip | Build the core first (§2.5); escalate OQ-1 now |
| R2 | B is chosen and then mistaken for meeting NFR-001 | Customer baseline silently weakened (violates constitution) | Traceability marks NFR-001 **NOT MET** under B; written approval required |
| R3 | A's added complexity (security, deployment, front end) exceeds team capacity | Late MVP | Same-origin SPA (no CORS); single JAR; Spring Security defaults; Thymeleaf fallback ([research R-04](research.md#r-04-web-stack-a)) |
| R4 | Rounding or reconciliation defects | Financial integrity (NFR-003) | Integer-minor-unit algorithm, invariant assertion, property tests, derived balances |
| R5 | Concurrent payments over-settle | Wrong balances | Group row lock + validation in one transaction (§6.3) |
| R6 | Approval deadlock (submitter is the only manager) | Expenses stuck pending | Clear UI message; designate a manager or enable self-approval (OQ-2) |
| R7 | Derived balances too slow at scale | NFR-004 missed | Indexed aggregate; measure; materialized fallback (§6.2) |
| R8 | Receipt uploads used as an attack vector | Security | Type sniffing, size limit, private storage, attachment disposition |
| R9 | Registration reveals existing emails | Minor privacy leak | Documented; verification-first sign-up as an alternative |
| R10 | B's shared-file DB readable by any OS user | Privacy (NFR-002) | Disclose; optional H2 encryption |

---

## 10. Assumptions

| ID | Assumption | Source |
|---|---|---|
| AS-1 | One currency per group, fixed after the first expense; default USD | spec Assumptions; OQ-4 |
| AS-2 | Percentages have at most 2 decimal places | This proposal |
| AS-3 | Approvers = managers; self-approval off by default; one approval level | Spec assumption (single review step); OQ-2 |
| AS-4 | Notifications are in-app only | spec Assumptions; OQ-3 |
| AS-5 | Invitees must already have an account | spec Assumptions |
| AS-6 | Payments settle balances within one group; no cross-group netting | This proposal |
| AS-7 | Overpayments are rejected rather than creating reverse debt | Interpretation of AC-015 |
| AS-8 | Approved/rejected expenses are immutable; pending ones are editable by the submitter | OQ-6 |
| AS-9 | Expense and payment dates may not be later than today + 1 day | This proposal |
| AS-10 | Max amount 1,000,000.00 major units | This proposal |
| AS-11 | The NFR-004 dataset is a planning target, not an SLA | spec Assumptions |
| AS-12 | The prompt's phrase "payer … may not be a participant" is read as "need not be a participant", consistent with spec FR-003 and the 2026-10-29 clarification. A payer **may** also be a participant. | spec FR-003 takes precedence |

## 11. Open decisions

| ID | Decision needed | Owner | Blocking? |
|---|---|---|---|
| OQ-1 | **Web vs. Swing MVP** | Customer | **Yes, for presentation and infrastructure** |
| OQ-2 | Who approves; self-approval; when multi-level | Customer | No (default AS-3) |
| OQ-3 | Notification channels beyond in-app | Customer | No |
| OQ-4 | Currencies, multi-currency, exchange rates | Customer | No (AS-1) |
| OQ-5 | Report export; filters beyond date range | Customer | No |
| OQ-6 | Edit/delete of pending and approved expenses; history retention | Customer | No (AS-8) |
| D-1 | Reject overpayments vs. record as credit | Customer/owner | No (AS-7) |
| D-2 | Member removal with non-zero balance: block (proposed) vs. allow with carry-over | Customer/owner | No |
| D-3 | Account deletion / data-retention policy | Customer | No (out of scope) |
| D-4 | Email verification at registration | Owner | No |
| D-5 | Hosting environment (A) | Owner | Before deployment |
| D-6 | Front-end technology (A): SPA vs. Thymeleaf | Team | Before A's UI work |

---

## 12. Constitution compliance assessment

| Principle | Assessment |
|---|---|
| I. Customer requirements and traceability | **Compliant.** All FR/NFR are mapped in [traceability.md](traceability.md). Implementation choices are labeled [PROPOSAL]/[DECISION], separate from requirements. No requirement is removed; deferrals are labeled. |
| II. Java 21 and Maven | **Compliant by design.** Parent POM sets `maven.compiler.release=21`; all modules build with `mvn verify`. *Note:* the only JDK observed on the authoring machine is 25; builds must still target release 21 (see [quickstart.md](quickstart.md)). |
| III. Object-oriented separation | **Compliant by design.** Domain and application modules have no UI or framework dependencies. Business rules are testable without a GUI or server. An ArchUnit test enforces it. |
| IV. Financial accuracy | **Compliant by design.** `BigDecimal` only. Explicit scale at input, calculation, and persistence (§6.1). Rounding tests are specified. |
| V. Automated testing | **Compliant by design.** JUnit 5 at every layer through Maven. GUI tests do not replace logic tests. |
| VI. PlantUML | **Compliant.** Diagrams in `diagrams/*.puml`. Not rendered or verified by this author unless noted in the final report. |
| VII. Human review | **Pending.** This whole package is AI-generated and is **not** approved until a human reviews it. |
| VIII. Swing is an implementation decision | **Compliant.** Swing appears only as conditional Alternative B. It is never claimed to satisfy NFR-001, and its adoption requires documented approval. |
