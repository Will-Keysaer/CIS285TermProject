# REST API Contract: Alternative A (Web)

> **CONDITIONAL:** applies only if OQ-1 is resolved in favor of the web platform (the recommended alternative). Every rule here **maps onto** [application-services.md](application-services.md). Where they differ, the application-service contract is authoritative and this file must be corrected.

Status: **draft, needs human review.** Base path: `/api/v1`. Same-origin with the browser client.

## 1. Authentication and session

- **Session:** Spring Security server session. Cookie `SESSION` with `HttpOnly; Secure; SameSite=Lax`.
- **CSRF:** cookie `XSRF-TOKEN` (readable by JS). Every state-changing request (POST/PUT/PATCH/DELETE) must echo it in the `X-XSRF-TOKEN` header. Otherwise the response is 403 `CSRF_INVALID`.
- **Unauthenticated access:** to protected endpoints → 401 `NOT_AUTHENTICATED`.
- **Session expiry:** idle timeout 30 min, absolute 12 h **[PROPOSAL]**.

## 2. Conventions

**Encoding**

- Content type is `application/json; charset=utf-8`. Receipt upload uses `multipart/form-data`.
- **Money is a JSON string** with a plain decimal, for example `"42.10"`. Not a number, not exponent notation.
- Percentages are strings too (`"33.33"`).
- Dates are `YYYY-MM-DD`. Timestamps are ISO-8601 UTC (`2026-11-05T19:30:00Z`).
- IDs are UUID strings.

**Paging and concurrency**

- Paging: `?page=0&size=50` (size ≤ 100). Response shape is `{ "items": [...], "page": 0, "size": 50, "totalItems": 123 }`.
- Optimistic concurrency: mutable resources return `version`. Updates and review actions must send it in the body. A mismatch → 409 `CONCURRENT_MODIFICATION`.
- Idempotency: `POST` create endpoints marked ⓘ accept an `Idempotency-Key: <uuid>` header.

**Errors** (RFC 9457):

```json
{
  "type": "https://splitsmart.example/problems/validation-failed",
  "title": "Validation failed",
  "status": 400,
  "code": "VALIDATION_FAILED",
  "detail": "The expense could not be accepted.",
  "correlationId": "1f6c…",
  "errors": [
    { "field": "participants[1].percentage", "code": "PERCENTAGE_NEGATIVE", "message": "Percentage must be 0 or more." },
    { "field": "participants", "code": "PERCENTAGES_NOT_100", "message": "Percentages total 90.00; they must total exactly 100.00." }
  ]
}
```

The `type` URI host is a placeholder.

**Visibility**

- Anything the caller cannot see returns **404 `NOT_FOUND`**, including resources in groups they do not belong to. 403 is used only when the caller can see the resource but lacks the role or permission.
- HTTP status mapping: [application-services.md §7](application-services.md#7-error-codes).

## 3. Endpoints

Legend: **Auth** = who may call (AM = active member, MGR = manager, PARTY = party to the record).

### 3.1 Accounts (FR-001)

| Method & path | Auth | Request | Success | Errors |
|---|---|---|---|---|
| `POST /auth/register` | public | `{email, displayName, password}` | 201 `AccountView`; signed in | 400, 409 `EMAIL_UNAVAILABLE` |
| `POST /auth/login` | public | `{email, password}` | 200 `AccountView`; new session ID (session-fixation protection) | 401 `INVALID_CREDENTIALS`, 429 |
| `POST /auth/logout` | signed in | none | 204 | none |
| `POST /auth/password-reset-requests` | public | `{email}` | **202 always** | 429 |
| `POST /auth/password-resets` | token holder | `{token, newPassword}` | 204; all sessions for the account invalidated | 400 `RESET_TOKEN_INVALID`, 400 |
| `GET /me` | signed in | none | 200 `AccountView {id, email, displayName}` | 401 |
| `PUT /me/password` | signed in | `{currentPassword, newPassword}` | 204 | 401, 400 |

### 3.2 Groups, memberships, invitations (FR-002)

| Method & path | Auth | Request | Success | Errors |
|---|---|---|---|---|
| `POST /groups` | signed in | `{name, description?, currencyCode?, selfApprovalAllowed?}` | 201 `GroupView` | 400, 422 `CURRENCY_UNSUPPORTED` |
| `GET /groups` | signed in | none | 200 `[GroupSummary]` | none |
| `GET /groups/{groupId}` | AM | none | 200 `GroupView` (with `members[]`) | 404 |
| `PATCH /groups/{groupId}` | MGR | `{version, name?, description?, currencyCode?, selfApprovalAllowed?}` | 200 `GroupView` | 403, 404, 409, 422 `CURRENCY_LOCKED` |
| `POST /groups/{groupId}/invitations` | MGR | `{email}` | **202** `{status: "INVITATION_RECORDED"}` (uniform) | 403, 404, 409 `ALREADY_MEMBER` / `INVITATION_ALREADY_PENDING` |
| `GET /groups/{groupId}/invitations` | MGR | none | 200 `[{id, email, status, createdAt, expiresAt}]` | 403, 404 |
| `DELETE /groups/{groupId}/invitations/{invitationId}` | MGR | none | 204 (revoked) | 403, 404, 409 `INVITATION_NOT_PENDING` |
| `GET /me/invitations` | signed in | none | 200 `[InvitationView]` (PENDING, addressed to me) | none |
| `POST /me/invitations/{invitationId}/accept` | invitee | none | 200 `MembershipView` | 404, 409 `INVITATION_NOT_PENDING` |
| `POST /me/invitations/{invitationId}/decline` | invitee | none | 204 | 404, 409 |
| `PUT /groups/{groupId}/members/{accountId}/role` | MGR | `{role: "MANAGER" \| "MEMBER"}` | 200 `MembershipView` | 403, 404, 422 `LAST_MANAGER` / `CANNOT_DEMOTE_CREATOR` |
| `DELETE /groups/{groupId}/members/{accountId}` | MGR (or self = leave) | none | 204 | 403, 404, 422 `CANNOT_REMOVE_CREATOR` / `LAST_MANAGER` / `MEMBER_HAS_OUTSTANDING_BALANCE` / `MEMBER_HAS_PENDING_EXPENSES` |

`GroupView`:

```json
{ "id": "…", "name": "Apartment 4B", "description": null, "currencyCode": "USD", "currencyScale": 2,
  "createdBy": "…", "selfApprovalAllowed": false, "myRole": "MANAGER", "version": 3,
  "members": [ { "accountId": "…", "displayName": "Alex", "email": "alex@example.com", "role": "MANAGER" } ] }
```

### 3.3 Expenses and review (FR-003 – FR-008)

| Method & path | Auth | Request | Success | Errors |
|---|---|---|---|---|
| `POST /groups/{groupId}/expenses` ⓘ | AM | `SubmitExpenseRequest` | 201 `ExpenseView` (status `PENDING`); `Location` header | 400 (field codes), 404, 409 `IDEMPOTENCY_KEY_REUSED` |
| `POST /groups/{groupId}/expenses/preview` *(optional for A)* | AM | `SubmitExpenseRequest` | 200 `{shares: [{accountId, share}]}`; nothing persisted | 400 (field codes), 404 |
| `GET /groups/{groupId}/expenses?status=&from=&to=&page=&size=` | AM | none | 200 `Page<ExpenseSummary>` | 404, 422 `INVALID_DATE_RANGE` |
| `GET /expenses/{expenseId}` | AM / PARTY | none | 200 `ExpenseView` | 404 |
| `PUT /expenses/{expenseId}` | submitter, PENDING | `SubmitExpenseRequest` + `version` | 200 `ExpenseView` | 400, 403, 404, 409 `EXPENSE_NOT_PENDING` / `CONCURRENT_MODIFICATION` |
| `PUT /expenses/{expenseId}/receipt` *(full scope)* | submitter, PENDING | multipart `file` (JPEG/PNG ≤ 5 MB) | 200 `ReceiptView` | 403, 404, 409, 413, 415 |
| `GET /expenses/{expenseId}/receipt` *(full scope)* | AM / PARTY | none | 200 image bytes; `Content-Disposition: attachment`; `X-Content-Type-Options: nosniff`; `Cache-Control: private, no-store` | 404 |
| `POST /expenses/{expenseId}/approval` | ReviewPolicy | `{version}` | 200 `ExpenseView` (`APPROVED`) | 403 `FORBIDDEN_ROLE` / `SELF_APPROVAL_NOT_ALLOWED`, 404, 409 |
| `POST /expenses/{expenseId}/rejection` | ReviewPolicy | `{version, reason?}` | 200 `ExpenseView` (`REJECTED`) | as above |
| `GET /me/pending-reviews` | signed in | none | 200 `[ExpenseSummary]` | none |

`SubmitExpenseRequest`:

```json
{
  "amount": "100.00",
  "expenseDate": "2026-11-04",
  "description": "Groceries",
  "splitMethod": "PERCENTAGE",
  "payerId": "a1…",
  "participants": [
    { "accountId": "b2…", "percentage": "50.00" },
    { "accountId": "c3…", "percentage": "50.00" }
  ]
}
```

Notes:

- The payer `a1…` is not a participant here. That is valid (FR-003); `a1…` is owed 100.00.
- For `"splitMethod": "EQUAL"`, `percentage` must be omitted.
- Any `share` field the client sends is ignored.

`ExpenseView`:

```json
{
  "id": "e9…", "groupId": "g1…", "status": "PENDING", "version": 0,
  "amount": "100.00", "currencyCode": "USD", "expenseDate": "2026-11-04", "description": "Groceries",
  "splitMethod": "PERCENTAGE",
  "submittedBy": { "accountId": "b2…", "displayName": "Blake" },
  "payer": { "accountId": "a1…", "displayName": "Alex" },
  "participants": [
    { "accountId": "b2…", "displayName": "Blake", "percentage": "50.00", "share": "50.00" },
    { "accountId": "c3…", "displayName": "Casey", "percentage": "50.00", "share": "50.00" }
  ],
  "submittedAt": "2026-11-05T19:30:00Z", "lastEditedAt": null,
  "reviewBlocked": false,
  "review": null,
  "receipt": null
}
```

After a decision, `review` becomes `{ "reviewer": {…}, "decision": "APPROVED", "reason": null, "decidedAt": "…" }`.

### 3.4 Balances and payments (FR-009, FR-010)

| Method & path | Auth | Request | Success | Errors |
|---|---|---|---|---|
| `GET /groups/{groupId}/balances` | AM | none | 200 `GroupBalances` | 404 |
| `GET /me/balances` | signed in | none | 200 `MyBalances` | none |
| `POST /groups/{groupId}/payments` ⓘ | AM, actor = sender | `RecordPaymentRequest` | 201 `{payment: PaymentView, balance: {debtor, creditor, amount}}` | 400, 403 `PAYMENT_NOT_SENDER`, 404, 409 `POSSIBLE_DUPLICATE_PAYMENT`, 422 `PAYMENT_SELF` / `RECIPIENT_NOT_MEMBER` / `NOTHING_OWED` / `PAYMENT_EXCEEDS_OUTSTANDING` |
| `GET /groups/{groupId}/payments?from=&to=&page=&size=` | AM / PARTY | none | 200 `Page<PaymentView>` | 404, 422 |

`GroupBalances`:

```json
{ "groupId": "g1…", "currencyCode": "USD",
  "pairs": [ { "debtor": {"accountId":"b2…","displayName":"Blake"}, "creditor": {"accountId":"a1…","displayName":"Alex"}, "amount": "50.00" } ],
  "memberNet": [ { "accountId": "a1…", "net": "100.00" }, { "accountId": "b2…", "net": "-50.00" }, { "accountId": "c3…", "net": "-50.00" } ] }
```

`pairs` lists only non-zero balances, with a positive `amount` and direction given by debtor/creditor. `memberNet` sums to `0` (I-15).

`RecordPaymentRequest`:

```json
{ "senderId": "b2…", "recipientId": "a1…", "amount": "50.00", "paymentDate": "2026-11-05",
  "note": "Venmo", "confirmPossibleDuplicate": false }
```

`PAYMENT_EXCEEDS_OUTSTANDING` problem includes `"outstanding": "50.00"`.

`PaymentView`: `{id, groupId, sender, recipient, amount, currencyCode, paymentDate, note, recordedBy, recordedAt}`.

### 3.5 Notifications (FR-008)

| Method & path | Auth | Request | Success |
|---|---|---|---|
| `GET /me/notifications?unreadOnly=&page=&size=` | signed in | none | 200 `Page<NotificationView>` |
| `GET /me/notifications/unread-count` | signed in | none | 200 `{count}` (the client may poll every 60 s **[PROPOSAL]**) |
| `POST /me/notifications/read` | signed in | `{ids: [...]}` | 204 |

### 3.6 Reports (FR-011, FR-012)

| Method & path | Auth | Request | Success | Errors |
|---|---|---|---|---|
| `GET /me/summary?groupId=` | signed in (AM if groupId) | none | 200 `Summary` | 404 |
| `GET /me/reports?from=&to=&groupId=&recordTypes=EXPENSE,PAYMENT&expenseStatuses=APPROVED&onlyMine=false` *(full scope; deferred in candidate MVP)* | signed in | none | 200 `ReportResult` | 404, 422 `INVALID_DATE_RANGE` |

Report rules are in application-services.md §6: inclusive business-date boundaries, scoping to the caller's groups, and empty result = valid.

## 4. Non-functional contract notes

- **Response headers:** `Strict-Transport-Security`, `Content-Security-Policy: default-src 'self'`, `X-Content-Type-Options: nosniff`, `Referrer-Policy: same-origin`.
- **Performance target (NFR-004):** p95 ≤ 3 s for `GET /me`, `GET /groups/{id}/expenses`, `GET /groups/{id}/balances`, `GET /me/summary`, and `GET /me/reports` on the defined dataset. This is a target to be measured, not a guarantee.
- **Versioning:** a breaking change → `/api/v2`. Additive fields are allowed in v1.
