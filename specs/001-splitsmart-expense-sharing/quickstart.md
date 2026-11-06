# Quickstart and Validation Guide

## Purpose

This quickstart is a planning artifact for the SplitSmart design package. It describes the prerequisites, implementation path, and validation scenarios for a Java 21 web application that follows the customer and constitutional requirements. It does not make claims that application code already exists or that the project is built.

## Prerequisites

- Java 21 JDK
- Maven 3.9+
- Git
- Web browser for product validation
- Optional: Postgres for a production-like local validation environment
- Optional: IDE such as IntelliJ IDEA or VS Code with Java support

## Recommended Implementation Baseline

- Java 21
- Maven
- Web backend: Spring Boot 3.x (recommended, not required by the customer baseline)
- Relational database: PostgreSQL for production or H2 for local development/test
- Test framework: JUnit 5
- Diagramming: PlantUML

Note: the stack above is an engineering recommendation and must be reviewed and approved by the implementation team. It is not a customer requirement and not a claim that the repository already contains those libraries.

## Bootstrap Commands

When implementation begins in the target repository, the project would be expected to run with commands along the following lines:

```bash
mvn clean verify
mvn test
mvn spring-boot:run
```

If the implementation team chooses a different Java web stack, the equivalent Maven commands still apply, but the profile names and packaging structure may differ.

## End-to-End Validation Scenarios

### 1. Account creation and sign-in
- Create a user account with email + password
- Sign in and confirm the user sees only their own data and group-shared data
- Attempt a login with an invalid password and confirm failure
- Validate account-recovery flow if implemented

Expected result: FR-001, NFR-002, AC-001, AC-002

### 2. Group and membership management
- Create a group
- Add a member via invitation
- Verify invitee must accept before joining
- Remove a manager or member from the group
- Confirm historical expenses retain original attribution

Expected result: FR-002, AC-003, AC-004

### 3. Expense creation and split logic
- Submit an equal-split expense with a positive amount
- Submit a percentage-split expense totaling 100%
- Attempt invalid values (zero amount, negative percentage, non-matching total, non-participant) and ensure rejection
- Attach or omit a receipt image and validate both cases

Expected result: FR-003, FR-004, FR-005, AC-005–AC-009, NFR-003

### 4. Approval workflow and notifications
- Submit a pending expense
- Confirm it does not change balances until approval
- Approve the expense and verify the balance updates
- Reject an expense and verify no balance change
- Verify that group members receive a notification of the new expense and review status

Expected result: FR-006, FR-007, FR-008, AC-010–AC-013, NFR-005

### 5. Payment tracking and pairwise balances
- Record a payment from one member to another
- Confirm the balance changes immediately and history records who recorded the payment and when
- View pairwise balances and verify matching amounts owed

Expected result: FR-009, FR-010, AC-014–AC-016

### 6. Reports and date filtering
- Generate expense summary report for one group/user
- Generate balance report and payment history report
- Apply valid date ranges and verify results match source data
- Validate invalid date window handling

Expected result: FR-011, FR-012, AC-017–AC-019

### 7. Web-first product validation
- Launch the product via the browser and complete the supported journeys
- Confirm the browser-based experience is the primary path
- If a Swing desktop alternative is considered, ensure it is labeled as a separate approved scope rather than the primary product

Expected result: NFR-001, AC-020

## Verification checklist for implementation

- Maven verification passes for all domain and service-level tests.
- JUnit 5 tests cover each acceptance condition.
- Financial tests cover rounding, splits, approvals, and payment effects.
- Authorization tests confirm private data boundaries.
- Validation tests reject invalid expense values and date-window requests.

## Notes

- This is not a build or run confirmation for an existing app. It is a plan for the implementation team.
- No implementation code should be written until the web-first product decision is confirmed or a customer-approved Swing MVP scope is formally adopted.
- The design package must be reviewed by a human before moving to code implementation.
