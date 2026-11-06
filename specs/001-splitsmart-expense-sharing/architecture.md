# SplitSmart Architecture Proposal

## Purpose and Scope

This document is the required standalone architecture proposal for the SplitSmart project. It is derived from the authoritative customer baseline in `UserRequirement.md`, the clarified specification in `specs/001-splitsmart-expense-sharing/spec.md`, and the project constitution in `.specify/memory/constitution.md`.

This architecture package is a planning artifact only. It does not implement application code, does not change customer requirements, and does not amend the constitution.

## Architectural Drivers and Constraints

### Customer-driven requirements
- Individual accounts with sign-in and recovery
- Group creation and membership management
- Expense creation with payer, participants, allocation type, optional receipts, and review workflow
- Notifications for new group expenses
- Pending approval gating before balances change
- Pairwise balance tracking
- Payment recording and immediate balance adjustment
- Expense/balance/payment summaries and date-range/custom reports
- Primary web-based product experience

### Constitutional constraints
- Java 21 target for Java application code
- Maven-based build and verification
- JUnit 5 for automated tests
- Clear separation between UI, business logic, and persistence
- `BigDecimal` for all monetary values and calculation logic
- Large-remainder rounding and exact reconciliation rules for percentage and equal splits
- Human review of AI-generated planning/artifacts
- Swing is a technical implementation choice, not a requirement substitute

## Unresolved Product Decision: NEEDS CUSTOMER DECISION

The most important unresolved design gate is the platform decision:

### Alternative A — Recommended: Web-based product architecture
This is the customer-compliant baseline and the primary architecture direction.

- Fully satisfies `NFR-001` and the customer baseline requirement that the product is primarily web-based.
- Aligns with group workflows, account management, notifications, reports, and multi-user collaboration.
- Fits the Java 21 + Maven constitution and the clarified product requirements.
- Allows the business logic to be separated cleanly from browser/UI concerns.

### Alternative B — Swing desktop MVP only
This is a conditional alternative only if the customer explicitly approves changing the delivery scope and platform.

- Could be implemented as a Java Swing desktop app using the same domain model/services.
- May be acceptable as a narrower MVP only after a formal requirement/platform change is documented.
- Does not satisfy the current baseline because it contradicts `NFR-001` and the web-first requirement.
- Requires explicit written approval that changes the product scope and platform before implementation proceeds.

### Decision status
`NEEDS CUSTOMER DECISION` — the project must not silently assume Swing is the primary product or that the web requirement is optional.

## System Context

SplitSmart is a user-focused, group-based expense-sharing system. It supports users who create personal accounts, join groups, submit expenses, review pending items, view pairwise balances, record payments, and generate reports limited to authorized information.

The system boundaries include:
- Account and authentication services
- Group and membership management
- Expense submission and review
- Allocation and rounding logic
- Balance and payment calculation
- Notification generation
- Reporting and audit views
- Persistence and retrieval of records

## Proposed Architecture

### Recommended architecture style
A layered web application architecture with a domain-driven core and a browser-based UI.

Major layers:
1. Presentation layer
   - Browser-based web UI
   - Form validation and user interaction flow
   - Authentication/session handling
   - Rendered pages and API clients

2. Application service layer
   - Account services
   - Group and membership services
   - Expense creation and validation
   - Approval workflow orchestration
   - Balance recalculation and payment services
   - Reporting service
   - Notification publication

3. Domain model layer
   - Account
   - Group
   - GroupMembership
   - Expense
   - ExpenseParticipant
   - ReviewDecision
   - Payment
   - Notification
   - BalanceView
   - ReportRequest

4. Persistence / data access layer
   - Repositories or DAOs for account, group, expense, payment, review, and notification data
   - Query objects for reporting and pairwise balance retrieval

5. Security and authorization layer
   - Authentication and session/token validation
   - Group membership checks
   - Manager permissions enforcement
   - Personal-data and group-data visibility control

## Responsibilities and Boundaries

### Presentation layer responsibilities
- Present sign-in, account recovery, group, expense, review, payment, and report screens
- Validate basic user input for usability
- Submit requests to backend services
- Render notifications and review states clearly
- Display pairwise balances and report outputs

This layer must not own critical financial rules or domain invariants.

### Application service responsibilities
- Enforce domain rules before writes occur
- Validate payer identity, participant membership, group currency precision, and percentage totals
- Execute the approval workflow
- Trigger notification creation when expenses are submitted
- Recompute or refresh pairwise balances after approval/payment changes
- Protect data access by authorized user/group scope

### Domain responsibilities
- Enforce business invariants and state transitions
- Represent owner and membership semantics
- Preserve historical attribution even after group membership changes
- Compute equal and percentage allocations with the largest-remainder rule
- Define review statuses: pending, approved, rejected
- Represent payments and their immediate balance impact

### Persistence responsibilities
- Persist immutable records and audit metadata
- Support report queries across date ranges and group/user scope
- Maintain historical data for expenses, payments, and review decisions
- Store notification state and access history where needed

## Runtime Interactions and Key Flows

### Account and group lifecycle
1. User creates an account with email and password.
2. User signs in and receives session context.
3. User creates a group or is added to a group by invitation.
4. Group creator or manager invites a user.
5. Invitee accepts before membership becomes active.
6. Authorization checks enforce member-only visibility and manager-only management rights.

### Expense submission flow
1. User chooses group and selects payer and participants.
2. Validation checks payer membership, positive amount, valid currency precision, valid allocation type, and valid percentages if percentage split is used.
3. Expense is saved as pending/review-required.
4. Notification is published to relevant group members.
5. Review queue shows the expense before balances change.

### Review and approval flow
1. Authorized approver opens pending expense for review.
2. Review decision is recorded as approved or rejected.
3. Pending expenses do not affect balances.
4. Approved expenses contribute to balance calculations.
5. Rejected expenses remain visible in history but do not affect balances.

### Payment flow
1. Sender records a payment with recipient, amount, and date.
2. Payment is persisted with metadata identifying who recorded it and when.
3. The corresponding pairwise balance is adjusted immediately.
4. Payment history remains available for reports and audit.

## Financial Model and Domain Rules

### Money and precision
All monetary values use `BigDecimal`.

- `amount` values are positive and aligned to the group currency precision
- Currency scale is explicit at both calculation and persistence boundaries
- No `float` or `double` is used for money

### Allocation rules
- Equal split: selected participants share the total exactly, reconciling to the smallest currency unit
- Percentage split: each participant percentage is nonnegative and totals exactly 100%
- Fractional cents are distributed using the largest-remainder method
- The final allocated shares must reconcile exactly to the expense total

### Balance rules
- Pending expenses do not affect balances
- Approved expenses affect balances
- Rejected expenses do not affect balances
- A payment recorded by the sender affects the balance immediately
- Historical records retain metadata about who recorded the payment and when

## Security, Privacy, and Authorization

### Identity and access
- Accounts are unique by email.
- Authentication must be enforced before group-scoped access
- A user may only view their own private account information and the information shared with groups they belong to

### Authorization boundaries
- Only group creators or designated managers can invite or remove members
- Expense review must be limited to authorized approvers
- Reports must filter by the user’s authorized groups and scope

### Privacy requirements
- Hidden or protected account, expense, balance, and payment data must not be exposed to unauthorized users
- UI and API layers must not bypass business-level access rules

## Persistence and Consistency Needs

The design does not assume a specific production database as a proven fact; it recommends a relational database for auditability, reporting, and transaction safety.

Recommended persistence model:
- `accounts`
- `groups`
- `group_memberships`
- `expenses`
- `expense_participants`
- `review_decisions`
- `notifications`
- `payments`
- `report_requests`

Recommended consistency principles:
- Store all financial records in the database, not in UI memory only
- Keep expense and payment history immutable from a reporting perspective
- Treat balance views as derived values computed from source records, with recomputation after approvals or payments
- Use transactions around approval/payment changes so that the audit trail and balance updates stay consistent

## Deployment and Runtime Model

The recommended runtime model is a browser-based Java web application with a backend service layer and a relational database.

Typical runtime components:
- Browser clients for web UI access
- Java web server/application runtime
- Service layer for domain workflows
- Persistence layer to relational DB
- Notification service or internal events within the app boundary

This is the preferred runtime model until the customer approves a different platform scope.

## Technology Recommendations

### Java and build stack
- Java 21
- Maven
- JUnit 5

### Recommended implementation style
- A Java web stack such as Spring Boot (recommended, not forced by the customer requirement)
- Relational database such as PostgreSQL for production-like validation, with H2 or equivalent for local development/test if desired

### Why these recommendations fit
- They align with the constitution and the customer requirements.
- They provide a clear separation between web/UI, service logic, and persistence.
- They are suitable for transactional financial workflows, reports, and tests.

### Alternative technology note
A Swing-only desktop design can be architected with the same domain layer, but it is not compatible with the present web-first customer requirement unless the customer approves the scope change explicitly.

## Quality Attributes and Risks

### Quality attributes
- Security and privacy of account and group data
- Financial integrity and exact reconciliation
- Reviewability and audit trail
- Usability for small group collaboration
- Performance under moderate group sizes and record volumes

### Risks and trade-offs
- Platform decision risk: if Swing is assumed early, the design may violate NFR-001.
- Rounding risk: incorrect percentage splitting can break exact reconciliation.
- Approval workflow risk: if approvals are not enforced centrally, pending expenses could affect balances incorrectly.
- Privacy risk: if authorization checks are only in UI code, protected data could be leaked.

## Test Strategy and Traceability

### Test strategy
The implementation should include JUnit 5 tests for:
- Account creation and sign-in rules
- Group creation and membership permissions
- Equal-split rounding accuracy
- Percentage split totals and largest-remainder behavior
- Pending versus approved/rejected expense effect on balances
- Payment immediate impact and recorded metadata
- Authorization and privacy boundaries
- Date-range reports and custom summary behavior

### Requirement traceability
| Requirement(s) | Architectural evidence |
|---|---|
| FR-001, NFR-002 | Account auth model, authorization boundaries, private data access rules |
| FR-002 | Group and membership services, manager permissions |
| FR-003, FR-004, FR-005 | Expense submission, equal split and percentage split domain logic |
| FR-006, FR-007, NFR-005 | Approval workflow, review decision, history and audit state |
| FR-008 | Notification service and group event publication |
| FR-009, FR-010 | Balance engine and payment recording semantics |
| FR-011, FR-012 | Report service and date-range filtering |
| NFR-001 | Web-first product architecture, with Swing alternative as conditional |
| NFR-003 | BigDecimal and largest-remainder rules in domain layer |
| AC-001–AC-023 | Acceptance criteria map to service and workflow validation |

## Open Questions and Assumptions

### Needs customer approval
- Whether the product remains web-first as the primary product, or a Swing-only desktop MVP is approved as a new scope.
- Whether additional notification channels beyond in-app notifications are required.
- Exact receipt-image storage and retention policy.

### Engineering assumptions
- A relational database is the appropriate persistence technology for the domain.
- The project remains scoped to expense sharing rather than a broader financial platform.
- The implementation team can enforce the service-layer validation and authorization rules centrally.

## Validation Performed

Validation performed for this planning artifact:
- Confirmed the updated prompt requirement for a standalone `architecture.md`.
- Created the standalone architecture proposal at [architecture.md](C:/Users/willkeysaer/GitHub/CIS_285/Lab6/specs/001-splitsmart-expense-sharing/architecture.md).
- Checked the design package for completeness against the project requirements and the prompt requirements.

This is a documentation-only validation. No application implementation was created, and no build or test run was claimed because the project does not yet contain an implementation.

## Final Notes

This architecture proposal deliberately keeps the web-first requirement visible and unresolved only where the customer decision is still pending. It makes the Swing alternative explicit and conditional rather than pretending it satisfies the baseline requirement.
