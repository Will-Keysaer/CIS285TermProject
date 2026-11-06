# Implementation Plan: SplitSmart Expense Sharing

**Branch**: Not created in this working copy
**Date**: 2026-11-05
**Spec**: [spec.md](./spec.md)

## Summary

SplitSmart is a customer-driven expense-sharing application whose authoritative requirements are defined in `UserRequirement.md` and clarified in `specs/001-splitsmart-expense-sharing/spec.md`. The product must be primarily web-based, support accounts, group membership management, expense submission, approval gating, balance tracking, payment recording, and reporting. The design below preserves the full customer product scope while explicitly flagging the unresolved platform decision: a web-based implementation is required by NFR-001, while a Java Swing desktop MVP is only a conditional alternative that requires explicit customer approval and a change in delivery scope.

This plan recommends a Java 21 + Maven web application architecture with a domain model separated from UI and infrastructure concerns. The design uses BigDecimal for all money values, validation rules around equal and percentage splits, explicit approval state transitions, and pairwise balance calculations derived from approved expenses and payments. Implementation should proceed only after the customer confirms that the product remains web-first or approves a Swing-only MVP as a different project scope.

## Technical Context

**Language/Version**: Java 21

**Primary Dependencies**: Maven; recommended web stack is Spring Boot 3.x with Spring MVC / REST and Spring Data JPA; JUnit 5 for tests; PlantUML for architecture diagrams. These are engineering recommendations, not facts about an existing repository implementation.

**Storage**: Relational database (recommended: PostgreSQL for production, H2 for local development/test) accessed through repositories/DAO layer; persistence boundaries must be explicit and not mixed with Swing code. Exact DB choice is not yet approved as a repository fact and should be finalized during implementation.

**Testing**: Maven verification with JUnit 5, plus domain-level tests for split logic, approval flow, balance recalculation, and payment invariants.

**Target Platform**: Primary web application served through a browser; desktop Swing is a conditional alternative only if customer approval changes the delivery scope.

**Project Type**: Web application with a Java backend and browser-based UI; alternative Swing-only MVP remains outside the customer baseline unless approved.

**Performance Goals**: For a group of up to 50 members and 10,000 expense/payment records, at least 95% of core account, expense, balance, and report views should complete within 3 seconds under ordinary operating conditions (NFR-004).

**Constraints**:
- Web-first product requirement must remain visible until customer approval settles the platform question.
- All monetary values use BigDecimal; no float/double for money.
- Expense splits must reconcile exactly to the smallest supported currency unit.
- Pending expenses cannot affect balances; approved expenses can; rejected expenses do not.
- Payments recorded by the sender affect balances immediately and retain audit metadata.
- Only group creators/designated managers can manage membership.
- Privacy/security boundaries require scoped authorization to account data and group data.

**Scale/Scope**: Initial implementation targets a multi-user group expense app for household, trip, and small team use, with pairwise balances and historical reports. The design remains intentionally bounded to the customer scope and a future MVP slice, not a broader general ledger product.

## Architecture Direction

### Alternative A — Recommended: Web-based product architecture

This is the customer-compliant baseline and the primary design direction.

Components:
- Web UI layer: browser-based pages and API-driven interactions for sign-in, group management, expense entry, review, balances, payment recording, and reports.
- Application service layer: business logic for account authentication, expense validation, allocation calculation, approval workflow, and balance updates.
- Domain model: Account, Group, Membership, Expense, Allocation, ReviewDecision, Notification, Payment, BalanceView, ReportRequest.
- Persistence layer: repositories for groups, expenses, payments, notifications, and audit records.
- Security/authz layer: session-based auth, ownership checks, group authorization rules, and private-data access policies.

This design satisfies FR-001 through FR-012 and NFR-001 through NFR-005; it keeps web delivery as the primary product and respects the constitution.

### Alternative B — Swing desktop MVP only (conditional)

This alternative is valid only if the customer formally approves a change to the delivery platform. It is not a substitute for the web requirement under current requirements.

Consequences:
- The product would no longer satisfy NFR-001 as currently written.
- Account, group, expense, and payment flows would still be designable in Java Swing, but the design must be explicitly treated as a revised scope.
- The same domain and business logic could be shared with Alternative A, but UI and server assumptions would be different.
- Requirement coverage would be narrower unless the customer approves a reduced desktop-only MVP and the scope is revisited.

This alternative is marked `NEEDS CUSTOMER DECISION` and requires a documented approval of the platform change before implementation proceeds.

## Constitution Check

### Constitution alignment
- Customer requirements and traceability: Pass — current design is derived from `UserRequirement.md` and `spec.md` and traceability is preserved in this document.
- Java 21 and Maven: Pass — this is the planned implementation baseline.
- Object-oriented separation: Pass — UI, domain, and persistence responsibilities are explicitly separated.
- Financial accuracy: Pass — BigDecimal, explicit scale and rounding rules, and balance invariants are specified.
- Automated testing: Pass — JUnit 5 test strategy is planned and requirement-driven.
- PlantUML design records: Pass — diagrams are generated in the design package.
- Human review of AI output: Pass — all generated documentation must be reviewed by a human before acceptance.
- Swing is an implementation decision: Pass — Swing remains clearly marked as conditional and not as satisfying the customer requirement.

### Planned gate status
- `NEEDS CUSTOMER DECISION`: Product platform decision remains unresolved until a customer confirms whether the primary product stays web-based or approves Swing as a revised delivery scope.

## Project Structure

```text
specs/001-splitsmart-expense-sharing/
├── spec.md
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   ├── web-api.md
│   └── ui-contracts.md
├── diagrams/
│   ├── splitsmart-architecture.puml
│   └── expense-workflow.puml
├── checklists/
│   └── requirements.md
└── tasks.md   # Not created in this planning step
```

Recommended implementation layout for the actual repository later (not pre-existing):

```text
src/
├── main/java/com/splitsmart/
│   ├── api/
│   ├── domain/
│   ├── service/
│   ├── persistence/
│   ├── security/
│   └── ui/
├── test/java/com/splitsmart/
│   ├── domain/
│   ├── service/
│   └── integration/
└── main/resources/
```

## Traceability Matrix

| Requirement | Design evidence | Status |
|---|---|---|
| FR-001 | Account domain model, auth/session service, privacy boundaries | Confirmed |
| FR-002 | Group, membership, permissions, invite acceptance flow | Confirmed |
| FR-003 | Expense entity, payer model, participant selection, receipt metadata | Confirmed |
| FR-004 | Equal split allocation service | Confirmed |
| FR-005 | Percentage split service with largest-remainder rounding | Confirmed |
| FR-006 | Expense review state machine; pending changes do not affect balances | Confirmed |
| FR-007 | ReviewDecision, approver authorization, approval/rejection outcomes | Confirmed |
| FR-008 | Notification domain model and event publication | Confirmed |
| FR-009 | Pairwise balance calculation and history | Confirmed |
| FR-010 | Payment entity with sender, recipient, amount, recording metadata | Confirmed |
| FR-011 | Report summaries and authorization filtering | Confirmed |
| FR-012 | Date-range report requests and custom filters | Confirmed |
| NFR-001 | Web-first architecture recommendation; Swing alternative explicitly conditional | Needs customer decision |
| NFR-002 | Authorization model and private data boundaries | Confirmed |
| NFR-003 | BigDecimal + largest remainder validation | Confirmed |
| NFR-004 | Performance targets and repository/query strategy | Assumption / engineering target |
| NFR-005 | Reviewability and settlement history design | Confirmed |
| AC-001–AC-023 | Cross-reference to service/domain invariants and acceptance scenarios | Confirmed |

## Key Design Decisions and Implementation Plan

1. Phase 0 — Requirements and decision gating
   - Confirm the web-first requirement is the approved product baseline.
   - If the customer approves a Swing-only MVP, create a revised scope document and treat it as a separate project decision.

2. Phase 1 — Domain and persistence design
   - Model domain entities and invariants.
   - Build money and split rules into domain services, not UI code.
   - Define repository interfaces and access-control policies.

3. Phase 2 — Application services and validation
   - Implement account management, group & membership management, expense creation, approval workflow, notifications, payment recording, and reports.
   - Add validation at service boundaries and expose consistent error payloads.

4. Phase 3 — Web/UI implementation and verification
   - Implement browser screens and API endpoints for the chosen web stack.
   - Run JUnit 5 tests covering all acceptance scenarios and financial edge cases.

## Risks, Open Questions, and Assumptions

- Open question: Is the customer willing to keep the full product web-based, or does the customer want to accept a desktop-only MVP? This is the single most important unresolved design gate.
- Assumption: The product uses a relational model to support auditability, reporting, and permissions.
- Assumption: Notification delivery is at least in-app; push/email channels are deferred unless customer approves them.
- Assumption: Receipt images are stored as metadata plus an object reference or file storage path, with validation rules and retention policy to be defined during implementation.
- Risk: If the platform decision is deferred too long, engineering effort may begin on a Swing implementation that violates NFR-001 and would need to be reworked.

## Notes

This plan is a design package only. It does not implement application code, does not add customer requirements, and does not change the constitution. It is intended to guide an implementation team after the platform decision is resolved.
