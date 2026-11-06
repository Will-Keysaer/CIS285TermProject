# SplitSmart Web API Design Contract

## Scope

This contract describes the web API shape for the recommended web-based product architecture. It is a design artifact only and is explicitly conditioned on the product’s web-first requirement still applying. If a Swing-only product is later approved by the customer, this contract becomes a separate alternative rather than the primary implementation contract.

## Base assumptions
- API uses JSON over HTTPS
- Authentication is required for all group-scoped actions
- Endpoints are organized by domain areas: accounts, groups, expenses, payments, notifications, reports
- All responses use consistent error structure
- Money values are serialized as strings to preserve exact `BigDecimal` precision

## Common response conventions

### Success response
```json
{
  "success": true,
  "data": { },
  "timestamp": "2026-11-05T19:54:24Z"
}
```

### Error response
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Expense amount must be greater than zero.",
    "details": ["amount"]
  }
}
```

## Authentication and account endpoints

### POST /api/accounts
Create an account.

Request body:
```json
{
  "email": "user@example.com",
  "password": "StrongPass!123",
  "displayName": "Taylor"
}
```

Behavior:
- Validates email format and password rules
- Stores only a hash of the password
- Returns created account ID and basic profile details

### POST /api/auth/login
Sign in.

Request body:
```json
{
  "email": "user@example.com",
  "password": "StrongPass!123"
}
```

Expected result:
- Returns session token or authenticated user context
- Applies privacy and authorization boundaries to later requests

### GET /api/accounts/me
Returns signed-in account details and group memberships.

## Group endpoints

### POST /api/groups
Create a group.

Request body:
```json
{
  "name": "Apartment 4B",
  "currencyCode": "USD",
  "currencyScale": 2
}
```

### POST /api/groups/{groupId}/invitations
Invite a user to the group.

Request body:
```json
{
  "inviteeEmail": "roommate@example.com",
  "role": "member"
}
```

Behavior:
- Only creator or designated managers can invite members.
- Invitation must be accepted before membership becomes active.

### POST /api/groups/{groupId}/members/{accountId}/accept
Accept invitation.

### DELETE /api/groups/{groupId}/members/{accountId}
Remove a member as authorized by manager or creator.

## Expense endpoints

### POST /api/groups/{groupId}/expenses
Create an expense.

Request body:
```json
{
  "description": "Groceries",
  "expenseDate": "2026-11-01",
  "amount": "129.50",
  "allocationType": "equal",
  "payerAccountId": "acc-123",
  "participantIds": ["acc-234", "acc-345"],
  "receiptReference": null
}
```

Behavior:
- Validates amount > 0 and proper currency precision
- Validates payer belongs to group
- Validates participants belong to group
- Stores expense as `pending` review state
- Notification is emitted to group members after submission

### GET /api/groups/{groupId}/expenses/{expenseId}
Returns expense detail, allocation, and review status.

### POST /api/groups/{groupId}/expenses/{expenseId}/review
Review an expense.

Request body:
```json
{
  "decision": "approved",
  "approverAccountId": "acc-111",
  "note": "Looks correct"
}
```

Behavior:
- Only an authorized approver can approve or reject.
- Approved expenses affect balances; rejected ones do not.

## Payment endpoints

### POST /api/groups/{groupId}/payments
Record a payment between two group members.

Request body:
```json
{
  "senderAccountId": "acc-123",
  "recipientAccountId": "acc-456",
  "amount": "35.00",
  "paymentDate": "2026-11-03",
  "note": "Rent partial payment"
}
```

Behavior:
- Updates balances immediately
- Stores recording metadata: who recorded it and when

### GET /api/groups/{groupId}/payments
List payment history.

## Balance and report endpoints

### GET /api/groups/{groupId}/balances
Returns pairwise balances for the signed-in user or group visibility scope.

### GET /api/groups/{groupId}/reports/summary
Request summary data for user expenses, balances, and payments.

### GET /api/groups/{groupId}/reports/custom
Date-range and custom report request.

Query parameters:
- `startDate`
- `endDate`
- `reportType`
- `accountId` (optional for user-level scope)

## Notification endpoints

### GET /api/accounts/me/notifications
Returns pending and recent notifications.

### PATCH /api/accounts/me/notifications/{notificationId}/read
Marks notification as read.

## Contract assumptions and invariants

- All monetary values are strings in order to preserve exact decimal precision.
- Percentage totals must total exactly 100.00 before the API accepts the expense.
- Expense and payment history remain available in the system even after group membership changes.
- Group members cannot read data outside the groups they belong to.
- A pending expense returns to the caller with a status that confirms it has not affected balances yet.

## Validation and errors

Common validation error codes:
- `AUTH_REQUIRED`
- `FORBIDDEN`
- `NOT_FOUND`
- `VALIDATION_ERROR`
- `PERCENTAGE_TOTAL_INVALID`
- `INVALID_DATE_RANGE`
- `BALANCE_UPDATE_ERROR`

Each error should identify the responsible field and the rule that failed.

## Traceability

| Requirement(s) | API contract area |
|---|---|
| FR-001, NFR-002 | auth/login, accounts/me |
| FR-002 | groups, invitations, membership endpoints |
| FR-003, FR-004, FR-005 | expenses create + validation |
| FR-006, FR-007 | expense review endpoint |
| FR-008 | notifications |
| FR-009 | balances |
| FR-010 | payments |
| FR-011, FR-012 | reports |
| NFR-001 | web API contract remains the primary product contract |

## Notes

This API design is a planning artifact for the recommended web-first architecture. It does not represent a codebase that already exists, and it remains invalid as the primary contract if the customer approves a Swing-only project scope instead.
