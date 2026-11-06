# Research and Design Decisions: SplitSmart

## Decision Record Overview

This document records the design decisions that are explicitly supported by the customer requirements and the project constitution, and it separates them from engineering recommendations and unresolved customer approvals. The authoritative inputs are:

- `UserRequirement.md`
- `specs/001-splitsmart-expense-sharing/spec.md`
- `.specify/memory/constitution.md`

## Confirmed Customer Requirements

### 1) Web-first product baseline
**Source**: `UserRequirement.md`, NFR-001, AC-020

**Decision**: The primary product must be web-based.

**Status**: Confirmed by source documents.

**Reasoning**: The customer explicitly states “OS supported: primary web-based,” and the constitution states that Swing is an implementation decision and not a replacement for the customer requirement.

**Implication**: Any implementation that is primarily Swing-only must be labeled as a revised scope and requires recorded customer approval.

### 2) Expense sharing domain and user structure
**Source**: `UserRequirement.md`, FR-001–FR-012

**Decision**: The product manages individual accounts, group memberships, expense records, payments, and review states.

**Status**: Confirmed.

**Reasoning**: The product exists to support shared household, trip, and small-team expenses; the clarified spec adds role definitions and precise rules for payer selection, approval workflow, and privacy.

### 3) Financial correctness rules
**Source**: constitution + clarified specification

**Decision**: All monetary values use `BigDecimal` and rounding rules are explicit.

**Status**: Confirmed by constitution and acceptance scenarios.

**Reasoning**: The constitution explicitly requires BigDecimal and tests for financial behavior. The spec states amounts must be positive and currency-precise; percentage allocations total 100%; leftover cents use the largest-remainder method.

## Engineering Recommendations

### 4) Recommended technology stack
**Status**: Recommendation, not a customer requirement

**Recommendation**: Java 21 + Maven web application with browser UI and a relational database. Use Spring Boot 3.x if the implementation team wants a standard enterprise stack.

**Rationale**:
- Aligns with the constitution’s Java 21 and Maven requirements.
- Simplifies web delivery, security, validation, and testing.
- Keeps the UI separated from business logic and persistence.

**Alternative**: Swing-only desktop application as an MVP if customer approval changes the project scope.

**Why the alternative is rejected for the current scope**: It fails NFR-001 and contradicts the baseline product requirement.

### 5) Domain-driven architecture
**Status**: Recommendation

**Recommendation**: Use a layered architecture with clear ownership boundaries:
- API/UI layer: user interactions and browser presentation
- Application service layer: validation, calculation, and orchestration
- Domain model: invariant enforcement and ownership semantics
- Persistence layer: repositories and data access
- Security layer: membership and authorization checks

**Rationale**: This separates GUI concerns from business rules, which is required by the constitution and prevents logic from being embedded in Swing event handlers or view classes.

## Key Domain Decisions

### 6) Approval workflow and balance effects
**Source**: FR-006, FR-007, AC-010–AC-012, NFR-005

**Decision**: Submissions remain pending until approved or rejected; pending expenses do not affect balances; approved expenses affect balances; rejected expenses do not.

**Status**: Confirmed.

**Rationale**: This is a core trust and audit requirement. It prevents unreviewed expenses from altering balances and makes review outcomes visible in history.

### 7) Payment semantics and immediate settlement effect
**Source**: FR-010, AC-015–AC-016, clarification session

**Decision**: The payment sender records the payment; the balance changes immediately. Payment history retains who recorded it and when.

**Status**: Confirmed.

**Rationale**: This was clarified explicitly by the specification and matches real-time settlement behavior.

### 8) Allocation logic
**Source**: FR-004, FR-005, AC-007–AC-009, NFR-003

**Decision**: Equal splits and percentage splits are computed in the domain layer and validated before persistence.

**Status**: Confirmed.

**Rules**:
- Amount must be positive and use the configured currency precision.
- Percentage totals must be exactly 100%, nonnegative, and valid per participant.
- Shares reconcile to total using largest-remainder distribution for leftover cents.
- Equal splits must sum exactly to the total in the smallest currency unit.
- A participant cannot be assigned a share if they are not part of the expense selection.

### 9) Privacy and authorization model
**Source**: NFR-002, AC-002, FR-002

**Decision**: Data access is scoped by account identity and group membership, and access decisions happen in service/security layers rather than in the UI.

**Status**: Confirmed.

**Rationale**: The product must not reveal private account, expense, balance, or payment details beyond a user’s own data and group-scoped data.

## Alternative Analysis

### Alternative A: Web-based product (recommended)
**Pros**:
- Satisfies NFR-001 and the customer’s primary platform requirement.
- Enables clear browser-based user flows, notifications, and reporting.
- Fits the constitution and the design syntax requirements.

**Cons**:
- Requires web UI and backend architecture beyond a desktop implementation.
- Needs careful auth/session and privacy design.

**Decision**: Recommended and compliant with the baseline.

### Alternative B: Swing-only desktop MVP (conditional)
**Pros**:
- May satisfy a narrower local desktop requirement if customer approval changes scope.
- Reuses Java domain logic and could be built rapidly for a limited audience.

**Cons**:
- Violates the current customer baseline because the product must be primarily web-based.
- Requires explicit approval and a formally revised project scope.
- Weakens requirement coverage and likely conflicts with the constitution’s “Swing is an implementation decision” principle.

**Decision**: Not selected unless customer approval changes scope. Mark as `NEEDS CUSTOMER DECISION`.

## Persistence and Consistency Recommendations

**Status**: Engineering recommendation

**Recommendation**:
- Use a relational schema with explicit tables for Account, Group, Membership, Expense, Payment, ReviewDecision, Notification, and Balance calculation artifacts.
- Keep the balance calculation as a derived view or query rather than an untrusted mutable source of truth.
- Store immutable approval and payment audit metadata.
- Treat the database as the authoritative persistence layer; avoid keeping critical business state only in UI memory.

**Reasoning**:
- Expense and payment records must be reviewable and auditable.
- Pairwise balance calculations can be derived from approved expenses and payment history.
- Reports need date-based, group-scoped, and user-scoped query support.

## Open Questions and Need-to-Resolve Items

### Needs customer decision
1. **Platform decision**: Keep the primary product web-based, or approve a Swing-only MVP as a revised scope.
2. **Notification channel**: In-app notifications are required by the spec; richer push or email notifications are explicitly deferred.
3. **Receipt image storage**: The design can support file storage or object storage, but the exact implementation and retention policy need approval during implementation.

### Engineering assumptions
- A relational database is the intended persistence mechanism.
- The product scope remains focused on small group expense splitting rather than enterprise-scale multi-tenant finance workflows.
- Security design will use standard session or token-based auth and role validation at the service layer.

## Traceability to Requirements

| Requirement(s) | Decision or rationale |
|---|---|
| FR-001, FR-002, NFR-002 | Account, group, and authorization rules are enforced in the domain/service layers |
| FR-003, FR-004, FR-005 | Expense creation and allocation policy are handled by domain services |
| FR-006, FR-007, NFR-005 | Approval workflow is explicit with pending/approved/rejected outcomes |
| FR-008 | Notification model is domain-backed and group-scoped |
| FR-009, FR-010 | Balance views and payment history follow immediate settlement semantics |
| FR-011, FR-012 | Report queries are scoped to authorized users and date-range filters |
| NFR-001 | Product platform choice remains web-first unless customer approval changes scope |
| AC-020–AC-023 | Performance and reviewability rules are designed into query/report strategy |

## Validation Notes

This research record is a planning artifact only. It does not claim that code or infrastructure is already implemented. It is reviewed as part of the design package and must be checked by a human before implementation proceeds.
