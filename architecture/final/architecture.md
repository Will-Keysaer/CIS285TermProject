# SplitSmart: Architecture (Reconciled Candidate)

Status: **Reconciled candidate. Pending human and customer review. Not approved. Not implemented.** Based on Claude Code `architecture/claude-code/architecture.md`, with Spec Kit `specs/001-splitsmart-expense-sharing/architecture.md` and `plan.md` reconciled, and cross-review findings F-01–F-11 addressed ([reconciliation-matrix.md](reconciliation-matrix.md)). Requirements: [requirements.md](requirements.md). Detailed behavior: [design.md](design.md). Decisions: [decisions.md](decisions.md).

## 1. Purpose and scope

SplitSmart lets account holders share expenses inside groups:

- Expenses are split equally or by percentage, reviewed before they affect balances, and settled by recorded payments.
- Members get summaries and date-range reports.

The architecture covers the **full customer scope** (FR-001–FR-012, NFR-001–NFR-005). Release phasing is described in [requirements.md §7](requirements.md#7-scope-and-phasing). Receipts and custom reports (Release 2) stay in the planned delivery unless the customer approves otherwise (OQ-13).

## 2. Context and system boundary

```
             ┌─────────────────────── SplitSmart system boundary ───────────────────────┐
Guest ──────►│  Browser client ──HTTPS──► Web API ──► Application services ──► Domain    │
Account      │  (static HTML/JS)          (/api/v1)        │                             │
holder ─────►│                                             ├──► Persistence ──► PostgreSQL
             │                                             ├──► Receipt storage (private) │
             │                                             └──► Recovery email adapter ───┼──► SMTP relay (external)
             └───────────────────────────────────────────────────────────────────────────┘
```

**Inside the boundary:**

- Accounts and recovery, groups, memberships, invitations.
- Expenses, allocations, reviews.
- Notifications (in-app), derived balances, payments, reports, audit log.

**Outside the boundary:**

- The SMTP relay (recovery email only).
- The hosting platform (OD-1).
- The user's browser.

Not in scope: payment processing (SplitSmart only **records** payments), exchange rates, external identity providers.

Diagrams: [use-cases.puml](diagrams/use-cases.puml) (actors and use cases), [components.puml](diagrams/components.puml) (components and trust boundaries).

## 3. Platform decision gate (F-01) [NEEDS CUSTOMER DECISION, BLOCKING]

**Facts:**

- UR: "OS supported: primary web-based."
- SPEC NFR-001: "A desktop Swing application alone does not satisfy this requirement."
- CONST VIII: Swing as the primary interface "requires documented customer approval."
- SPEC OQ-1 is unanswered.

Both independent proposals reached the same recommendation. **That agreement is not approval.**

### 3.1 Alternative A: web (recommended, baseline-compliant)

- **Shape:** a single Spring Boot application (D-05). It serves a versioned JSON REST API ([contracts/web-api.md](contracts/web-api.md)) and a same-origin static browser client (D-06; [contracts/ui-workflows.md](contracts/ui-workflows.md)).
- **Data:** PostgreSQL (D-07).
- **Adapters:** receipt files behind a storage port (D-24), and SMTP for recovery (D-10).
- **Compliance:** satisfies NFR-001 and makes AC-020 testable.

### 3.2 Alternative B: Swing desktop MVP [COND-B: only with documented customer approval]

- **Shape:** a standalone desktop application (D-02). Swing views and presenters call the **same** application services. Data is an embedded H2 file.
- **Limitations:**
  - **NFR-001 and AC-020 are not met.**
  - All group members must use one installation on one computer.
  - Notifications are visible only on that machine.
  - The database file is readable by OS users unless it is encrypted.
- **Diagram:** [components-swing-conditional.puml](diagrams/components-swing-conditional.puml).

A Swing client talking to a server, and Swing connected directly to a shared database, were both rejected (D-02). The first costs nearly as much as A and still fails NFR-001. The second exposes database credentials on every client.

### 3.3 Comparison

| Dimension | A: Web | B: Swing [COND-B] |
|---|---|---|
| NFR-001 / AC-020 | **Met** | **Not met** |
| FR-001 recovery | Emailed single-use token | One-time recovery codes |
| FR-002 invitations | Across devices | Same installation only |
| FR-003 receipts (R2) | Server upload/download | Local copy, if built |
| FR-008 notifications | Any device | Same machine only |
| NFR-002 privacy | Server-enforced; TLS | App-enforced; file readable unless encrypted |
| Deployment | Host + PostgreSQL + SMTP + TLS | Installer per machine |
| Concurrency | Real concurrent users (design §8) | Single process; same core code |
| Testing | Core JUnit + MockMvc + PostgreSQL IT + browser E2E | Core JUnit + H2 IT + presenter tests |
| Relative complexity (qualitative, unmeasured) | Higher | Lower for a demo; does not progress toward NFR-001 except through the core |

### 3.4 Implications

- **Choosing A:** meets the baseline. Costs security, deployment, and front-end work. All release work counts toward the final product.
- **Choosing B (only with written approval):** cheaper to demonstrate. However:
  - NFR-001 stays outstanding.
  - Use across multiple devices is not supported.
  - The presentation and deployment work is largely discarded later.
  - It requires CR-06.
- **Either way:** `splitsmart.domain`, `splitsmart.application`, and `splitsmart.persistence` are reused unchanged. The core can be built first (D-03).

## 4. Layers, packages, and dependencies

| Layer (package) | Responsibility | May depend on | Platform |
|---|---|---|---|
| Domain (`splitsmart.domain`) | Money, allocation, the expense state machine, review policy, balance netting, payment rules, membership rules, date ranges, error codes | JDK only | Shared |
| Application (`splitsmart.application`) | Use-case services, authorization, transactions, idempotency, notifications, audit; **ports** for repositories, clock, hashing, recovery, receipts | Domain | Shared |
| Persistence (`splitsmart.persistence`) | JPA entities and repositories, aggregate SQL, Flyway migrations (adapters for application ports) | Application, Domain | Shared |
| Web (`splitsmart.web`) | REST controllers, DTO mapping, problem details, security configuration, SMTP and receipt adapters, static client | Application, Domain | [COND-A] |
| Desktop (`splitsmart.desktop`) | Swing views, presenters, composition root | Application, Domain | [COND-B] |

**One Maven module** (D-04; a change from Claude's five modules, per XR §4 and the Spec Kit layout). The dependency rules are enforced by ArchUnit (design §16). **No business rule lives in controllers, views, or event handlers** (CONST III).

**Technology (all [ENGINEERING DECISION], with exact versions pinned at implementation):**

- Java 21 and Maven [CONSTITUTION].
- JUnit 5 [CONSTITUTION].
- Spring Boot 3.x, Spring Security, Spring Data JPA, Flyway.
- PostgreSQL (A) and H2 (tests, B).
- ArchUnit.
- Optional: Testcontainers, jqwik, Playwright for Java.

Diagrams: [components.puml](diagrams/components.puml), [classes-detailed.puml](diagrams/classes-detailed.puml), [classes-application.puml](diagrams/classes-application.puml).

## 5. Major workflows

Each workflow is detailed in [design.md §15](design.md#15-workflow-walk-throughs-end-to-end).

### 5.1 Accounts and recovery (FR-001)

1. A guest registers with email and password, and signs in.
2. Recovery (A) uses an emailed single-use token (30 min, hash stored). The response is the same whether or not the email exists. All sessions are invalidated on reset. Diagram: [sequence-password-reset.puml](diagrams/sequence-password-reset.puml). Decision D-10.

### 5.2 Groups and invitations (FR-002)

- A manager invites by email. Each **invitation has its own ID** (D-32, XR F-03).
- Only the intended invitee can accept or decline it. Invitations expire after 14 days, checked lazily (D-39). Accepting twice is prevented by a version check.
- Role changes and removal are manager-only. The creator and the last manager are protected.
- Removing a member is blocked while they have a non-zero balance or are involved in a pending expense (OQ-8).
- History stays attributed to the original accounts (AC-004).
- Diagram: [states-invitation.puml](diagrams/states-invitation.puml).

### 5.3 Expense submission (FR-003 – FR-005, FR-008)

1. The submitter enters the expense. Validation collects all field errors. The payer is a separate ACTIVE member (and may also participate). Participants are distinct ACTIVE members.
2. The split is computed by largest remainder (§7).
3. The expense is saved as **PENDING**.
4. Notifications go to the **other** active members, and an audit event is written, in the same transaction.
5. Balances are unchanged.

### 5.4 Expense review (FR-006, FR-007)

- **Default policy (OQ-2):** managers decide. Submitters may not decide their own expense unless the group enables self-approval. There is one level.
- **Deadlock flag:** `reviewBlocked` shows when the submitter is the only manager and self-approval is off. A manager can designate another manager or enable self-approval.
- **Locking:** approval and rejection take the group lock and apply a version-checked conditional update. A reviewer can never approve content they have not seen.
- **Effect:** approved expenses enter balances from the next read. Rejected expenses never do.
- Diagrams: [sequence-expense-review.puml](diagrams/sequence-expense-review.puml), [states.puml](diagrams/states.puml), [activities.puml](diagrams/activities.puml).

### 5.5 Balances and payments (FR-009, FR-010)

- **Balances** are derived on read: approved shares owed to the payer, net of payments in both directions (D-18).
- **Recording a payment:**
  - Only the sender can record it.
  - The group lock is taken, then the amount the sender owes is computed.
  - **Overpayment is rejected** (D-25 / OQ-7), which satisfies AC-015.
  - Idempotency key and possible-duplicate confirmation apply (D-20).
  - The payment is immutable and takes effect immediately, with recorder and timestamp stored.
- Diagrams: [sequence-payment.puml](diagrams/sequence-payment.puml), [states-payment.puml](diagrams/states-payment.puml), [activities-payment.puml](diagrams/activities-payment.puml).

### 5.6 Reports (FR-011, FR-012)

- **Summary (Release 1):** totals per currency.
- **Custom report (Release 2):** inclusive business-date range, filters, and balance movement. Scope is always computed on the server from the caller's memberships (D-38).

## 6. Security boundaries, privacy, and data ownership (NFR-002)

| Trust boundary | Control |
|---|---|
| Internet ↔ application (A) | TLS + HSTS; session cookie `HttpOnly; Secure; SameSite=Lax`; CSRF token on state changes; CSP `default-src 'self'`; login and recovery rate limiting |
| Presentation ↔ application | `ActorContext` comes only from the authenticated session. Request bodies never carry the actor (the Spec Kit `approverAccountId` body field was rejected). |
| Application ↔ data | Every group-scoped query takes a scope from `AuthorizationService`. A non-visible resource → **404**. A visible resource with a missing role → **403**. |
| Application ↔ receipt storage | Magic-byte type check, size limit, re-encoding, random keys, authorized download only |
| Desktop file (B) | OS file permissions; optional H2 encryption (disclosed weakness) |

**Data ownership:**

| Data | Owner and visibility |
|---|---|
| Account profile | The account holder. Display name and email are shown inside shared groups. |
| Group data (expenses, allocations, decisions, payments, receipts, audit) | The group. Visible to active members, and to former members only for records they were party to (D-27). |
| Notifications | The recipient only |

Full matrix: [design.md §9.2](design.md#92-operation--relationship-matrix).

## 7. Financial integrity policy

| Rule | Source |
|---|---|
| `BigDecimal` only; explicit scale; `RoundingMode.UNNECESSARY` only | [CONSTITUTION] IV; D-12 |
| Amounts > 0 at currency precision; input rejected, never rounded | [CLARIFIED SPEC] FR-003; D-12 |
| Percentages ≥ 0, Σ = 100 exactly; precision 2 dp proposed | [CLARIFIED SPEC] FR-005; OQ-10 |
| Largest remainder on integer minor units; ties to lowest account-ID string; Σ shares = amount asserted | [CLARIFIED SPEC] FR-005, NFR-003; D-16, D-17 |
| Currency per group; scales {0,2,3}; ceiling 1,000,000 | [NEEDS CUSTOMER DECISION] OQ-4, OQ-11 |
| Balances derived from APPROVED shares and payments; Σ member nets = 0 | D-18; NFR-003 |
| Persistence `NUMERIC(19,4)` at currency scale; read with exact rescale | D-12 |

Worked examples and the algorithm: [design.md §3](design.md#3-split-calculation-allocationcalculator).

## 8. Deployment and runtime

### 8.1 Alternative A [COND-A]

- **Artifact:** one executable JAR (API + static client) on a JRE 21.
- **Services:** PostgreSQL, a private receipt volume, and an SMTP relay.
- **Configuration:** through environment variables. No secrets in the repository.
- **Start-up and health:** Flyway runs on start-up. `/actuator/health` only.
- **Environments:** `local` (Compose PostgreSQL; recovery link printed to the console), `test` (H2 plus an optional PostgreSQL profile), `demo/prod` (host per OD-1).
- **Scaling:** single instance assumed. The in-memory rate limiter would need a shared store for multiple instances.

### 8.2 Alternative B [COND-B]

- **Artifact:** a desktop JAR or `jpackage` installer bundling a Java 21 runtime.
- **Data:** an H2 file in the user profile directory. Flyway runs on start-up.
- **Single instance:** a second launch is blocked by the H2 file lock.

## 9. Quality attributes

| Attribute | Requirement | Architectural tactic | Verification |
|---|---|---|---|
| Web delivery | NFR-001 | Alternative A | VP-E2E (browser) |
| Privacy | NFR-002 | Central `AuthorizationService`; 404-not-403; server-side scope | Authorization matrix tests |
| Financial integrity | NFR-003 | Integer allocation; derived balances; `UNNECESSARY` rounding | Unit and property tests; reconciliation IT |
| Performance | NFR-004 | Indexed aggregates; paging; derived balances with measured fallback | Benchmark protocol (**unmeasured**) |
| Reviewability | NFR-005 | Status state machine; immutable decisions and payments; audit log | History API tests |
| Maintainability | CONST III | Layered packages + ArchUnit | `ArchitectureRulesTest` |
| Accessibility (suggested only) | QA-S1 | Semantic HTML forms; announced errors | Manual and automated a11y checks |

## 10. Architectural decisions and trade-offs

| Decision | Trade-off accepted |
|---|---|
| D-01: web recommended | More up-front work in exchange for baseline compliance |
| D-04: one module | Weaker compile-time separation; mitigated by ArchUnit |
| D-06: static JS client | Hand-written client code; no build toolchain |
| D-18: derived balances | Query cost on every read, in exchange for correctness by construction. Must be measured for NFR-004. |
| D-19: group row lock | Coarse lock; negligible contention for groups of ≤ 50 |
| D-25: reject overpayment | Users cannot pre-pay; pending OQ-7 |
| D-31: audit log | One extra insert per write, in exchange for reviewability |

The full log, including rejected options, is in [decisions.md](decisions.md).

## 11. Risks

| # | Risk | Mitigation |
|---|---|---|
| RK-1 | OQ-1 stays unanswered and presentation work stalls | Build the core first; escalate OQ-1 |
| RK-2 | Swing is chosen and later mistaken for NFR-001 compliance | Traceability marks NFR-001 ❌ under B; CR-06 |
| RK-3 | A's scope exceeds team capacity | One module; static client; Spring defaults; Release 1 / Release 2 phasing |
| RK-4 | Rounding defects | Integer algorithm; property tests; invariant assertion |
| RK-5 | Concurrency defects on the real DB | Concurrency IT on the selected DB (F-07) |
| RK-6 | NFR-004 missed | Measure early; materialized fallback |
| RK-7 | Malicious receipt uploads | Sniffing, re-encoding, limits, private storage |
| RK-8 | Review deadlock | `reviewBlocked` flag; manager designation |
| RK-9 | Proposed limits rejected by the customer | They are constants (OQ-4, OQ-10, OQ-11) |
| RK-10 | Registration reveals existing emails | Documented; OD-4 |

## 12. Constitution compliance (on paper; nothing executed)

| Principle | Assessment |
|---|---|
| I. Traceability | [traceability.md](traceability.md) maps every FR, NFR, and AC. Decisions are labeled separately from requirements. |
| II. Java 21 / Maven | Designed. JDK 21 is not present on the authoring machine (OD-2). No build has been run. |
| III. Separation | Layered packages + ArchUnit; the core is testable without a GUI or server |
| IV. `BigDecimal` | §7 |
| V. JUnit 5 | [verification-plan.md](verification-plan.md). No test exists yet. |
| VI. PlantUML | `diagrams/*.puml`. Render status is in [README.md §6](README.md#6-diagram-index). |
| VII. Human review | **Pending** for this whole package |
| VIII. Swing conditional | §3.2; never claimed to satisfy NFR-001 |
