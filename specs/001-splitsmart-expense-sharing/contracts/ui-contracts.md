# SplitSmart UI Contract and Interaction Model

## Purpose

This document defines the expected browser experience for the recommended web-first architecture. It is intentionally independent of a particular front-end framework and captures the major user flows, responsibilities, and validation boundaries that the UI must enforce.

## Primary UI concept

The product is a browser-based application organized by a user dashboard and group context. The UI should present:
- My account and private profile
- Groups I belong to
- Shared expenses for each group
- Review queue for pending expenses
- Pairwise balances
- Payment entry and history
- Notifications
- Custom reports and date filters

## Screen and workflow contracts

### 1. Sign-in and recovery
**Screen**: Landing / sign-in page

Required fields:
- Email
- Password

Secondary actions:
- Recover account
- Create new account

Validation:
- Email format validation
- Password length and complexity policy
- Authentication failure should show a generic error message without revealing account existence

Requirement coverage: FR-001, NFR-002

### 2. Dashboard
**Screen**: Account dashboard

Displays:
- User profile summary
- Group cards and role badges
- Pending notifications count
- Recent activity or open review items

Rules:
- User sees only their private data and the data of groups to which they belong.

### 3. Group management screen
**Screen**: Group detail and membership panel

Allows:
- Create group
- Invite member
- Accept invitation
- Remove member
- Update group metadata if applicable

Authorization:
- Only creator/designated managers may invite or remove members

### 4. Expense entry screen
**Screen**: New expense form

Required fields:
- Date
- Description
- Amount
- Allocation method: equal or percentage
- Payer
- Participants
- Optional receipt

Rules:
- Payer selected separately from participants
- Payer must belong to the group
- Participant set is validated before submission
- Net allocation must reconcile to the total amount

### 5. Expense review screen
**Screen**: Pending expense review queue

Shows:
- Expense date, description, amount, payer, participants, receipt reference
- Review decision buttons: approve / reject
- Review notes and audit trail

Rules:
- Approved expenses affect balances
- Rejected expenses remain in history but do not affect balances

### 6. Balance and payment screen
**Screen**: Pairwise balances and payment form

Displays:
- Amount owed by user A to user B
- Payment transaction list
- Payment recording form

Rules:
- Payment is recorded by sender and immediately affects balances
- Payment history retains who recorded it and when

### 7. Reports screen
**Screen**: Reporting and filters

Supports:
- Expense summary
- Balance summary
- Payment summary
- Custom date-range reports

Validation:
- Date ranges must be valid and cannot end before they start
- Results are filtered by account/group authorization

## UI validation responsibilities

The UI should validate user input for usability, but it must not be the sole enforcement point for business rules. The real business logic and financial invariants must live in domain/application services.

This means the UI is responsible for:
- field validation and formatting
- accessible errors and progress states
- authorization-aware visibility
- handling API errors consistently

The backend/service layer is responsible for:
- final validation
- financial calculations
- balances and payment updates
- historical and audit-safe persistence

## Interaction patterns

### Expense allocation patterns
- Equal split: the UI should show selected participants and the derived per-person share.
- Percentage split: the UI should show percentages and indicate when 100% is reached or if invalid totals remain.

### Balance display patterns
- Display pairwise balances in the user’s chosen group context.
- Highlight pending review items separately from settled or approved records.

### Notification patterns
- Show a badge or list of notifications when a new expense is submitted.
- Include review status in the notification text.

## Accessibility and privacy expectations

- Use role-based visibility to hide or disable actions outside authorization.
- Avoid exposing private account information or other groups’ data.
- Keep the review and payment history understandable to the end user.
- Use clear labels for pending/approved/rejected states.

## Traceability

| Requirement(s) | UI contract area |
|---|---|
| FR-001 | sign-in, account recovery |
| FR-002 | group management and members |
| FR-003, FR-004, FR-005 | expense creation and allocation |
| FR-006, FR-007 | expense review |
| FR-008 | notifications |
| FR-009, FR-010 | balance and payment views |
| FR-011, FR-012 | reports and date-range filters |
| NFR-002 | privacy and scope gating |
| NFR-001 | web-first browsing experience |

## Notes

This UI contract is intended for the recommended web-first product. If the customer approves a Swing-only scope later, a different UI contract will be required, and it must not be treated as satisfying the current web requirement.
