# UI Workflows Contract (Browser primary; Swing conditional)

Status: **Reconciled candidate. Pending human review. Not implemented.**

- **Sources:** Spec Kit `contracts/ui-contracts.md` (screen concepts, dashboard, accessibility, validation responsibilities) merged with Claude Code `contracts/swing-ui.md` (presenter pattern, error mapping).
- **§1–§5** describe the **browser UI** [COND-A], the recommended baseline-compliant path.
- **§6** describes the **Swing UI** [COND-B]. It applies only with documented customer approval and does **not** satisfy NFR-001.

Every screen calls the operations in [web-api.md](web-api.md), which map onto [application-services.md](application-services.md). The UI **never** computes shares, balances, or authorization. It may hide controls for usability, but the server always enforces the rules (SK ui-contracts "UI validation responsibilities"; CONST III).

## 1. Navigation map

```
Sign in ─┬─ Register
         ├─ Forgot password → (email link) → Reset password
         └─► Dashboard
               ├─ My invitations (accept / decline)
               ├─ Notifications
               ├─ My balances (all groups)
               ├─ My summary / Reports (Release 2: custom date-range)
               ├─ Create group
               └─ Group ─┬─ Expenses (list, filter by status) ─┬─ New / edit expense
                         │                                     └─ Expense detail (review, receipt, history)
                         ├─ Review queue (managers)
                         ├─ Balances ── Record payment
                         ├─ Payments (history)
                         ├─ Members (managers: invite, role, remove; anyone: leave)
                         ├─ History (audit)
                         └─ Settings (managers)
```

## 2. Screens

| # | Screen | Who can open it | Content and form fields | Calls | Feedback and edge cases |
|---|---|---|---|---|---|
| U1 | Sign in | Guest | Email, password; links to Register and Forgot password | `POST /auth/login` | A generic "Email or password is incorrect" that does not reveal whether the account exists (SK). Lock-out message after the rate limit. |
| U2 | Register | Guest | Email, display name, password (12–128 characters) | `POST /auth/register` | Inline field errors; `EMAIL_UNAVAILABLE` |
| U3 | Forgot password | Guest | Email | `POST /auth/password-reset-requests` | Always shows "If an account exists, we sent a link" (uniform response) |
| U4 | Reset password | Holder of the reset link | New password ×2 (token read from the URL fragment) | `POST /auth/password-resets` | `RESET_TOKEN_INVALID` → "This link is invalid or expired"; on success, go to Sign in |
| U5 | Dashboard | Account holder | Profile summary; group cards with role badges; unread-notification count; pending invitations; items awaiting my review (SK dashboard) | `GET /me`, `/groups`, `/me/invitations`, `/me/notifications/unread-count`, `/me/pending-reviews` | Only my data and my groups (NFR-002) |
| U6 | Create group / Settings | Any holder (create); managers (settings) | Name, description, currency (default USD), self-approval toggle | `POST /groups`, `PATCH /groups/{id}` | `CURRENCY_LOCKED` after the first expense; `CONCURRENT_MODIFICATION` → "Reload" |
| U7 | Members | Active members (view); managers (actions) | Member list with roles; invite by email; make or revoke manager; remove; leave | `GET /groups/{id}`, invitations, role, `DELETE` member | Invite always says "Invitation recorded"; `MEMBER_HAS_OUTSTANDING_BALANCE`, `LAST_MANAGER`, `CANNOT_REMOVE_CREATOR` shown in plain language |
| U8 | My invitations | Invitee | Group name, inviter, date; Accept / Decline | `/me/invitations/{id}/accept`, `/decline` | `INVITATION_NOT_PENDING` → "This invitation is no longer valid" |
| U9 | Expense list | Active members | Date, description, amount, payer, status badge (PENDING / APPROVED / REJECTED, NFR-005); filters (status, dates) | `GET /groups/{id}/expenses` | Paged (50) |
| U10 | New / edit expense | Active members (new); submitter while PENDING (edit) | Amount, date, description, **payer dropdown** (active members) **separate from** the **participant checklist**, split method; percentage inputs when PERCENTAGE; live share preview; optional receipt (Release 2) | `POST /groups/{id}/expenses/preview` (debounced), `POST …/expenses` (with `Idempotency-Key`), `PUT /expenses/{id}`, `PUT …/receipt` | All field errors at once next to each field; running percentage total ("Total 90.00 %, must be 100.00 %"); the payer may also be ticked as a participant; banner "Pending review: balances not changed yet" after submit; `reviewBlocked` banner explaining who can review |
| U11 | Expense detail | Active members; former members if party | Fields, participant shares, payer, submitter, status, review decision and reason, receipt link (R2), history | `GET /expenses/{id}`, `/receipt`, `/groups/{id}/history?subjectId=` | Approve and Reject buttons shown only when `/me/pending-reviews` lists the expense; `EXPENSE_NOT_PENDING` / `CONCURRENT_MODIFICATION` → "This expense changed, reload and review again" |
| U12 | Review queue | Managers (policy per OQ-2) | Pending expenses I may decide, with receipt link; reason field for rejection | `/me/pending-reviews`, `/approval`, `/rejection` | `SELF_APPROVAL_NOT_ALLOWED` explained |
| U13 | Group balances | Active members | "X owes Y $Z" rows (non-zero only); my net; a **Record payment** button only on rows where I am the debtor | `GET /groups/{id}/balances` | Settled state ("All settled") |
| U14 | Record payment | Active member, as sender | Recipient (pre-filled), amount (pre-filled with the outstanding amount), date, note | `POST /groups/{id}/payments` (with `Idempotency-Key`) | `PAYMENT_EXCEEDS_OUTSTANDING` shows the outstanding amount; `POSSIBLE_DUPLICATE_PAYMENT` → confirm dialog, then resend with `confirmPossibleDuplicate=true`; success shows the updated balance immediately |
| U15 | Payments history | Active members; former members (party rows) | Sender, recipient, amount, date, recorded by, recorded at (NFR-005) | `GET /groups/{id}/payments` | Paged |
| U16 | Notifications | Recipient | List with unread markers; mark read | `/me/notifications`, `/read` | Badge polls the unread count every 60 s |
| U17 | My balances | Account holder | Per group, plus per counterparty and per currency | `GET /me/balances` | Currencies are never summed together |
| U18 | Summary | Account holder | Per-currency totals (paid, my share, pending count, owed to me, I owe, payments sent/received) | `GET /me/summary` | |
| U19 | Custom report (Release 2) | Account holder | From/to dates, group, record types, expense statuses, only-mine | `GET /me/reports` | `INVALID_DATE_RANGE` → "End date is before start date"; no table shown; "No matching records" for empty results |
| U20 | Group history | Active members | Audit events (submissions, edits with before/after, decisions, payments, membership changes) | `GET /groups/{id}/history` | Invitee emails visible to managers only |

## 3. Use case → screen mapping

| Use case ([use-cases.puml](../diagrams/use-cases.puml)) | Screens | Requirement |
|---|---|---|
| Register / sign in | U1, U2 | FR-001 |
| Recover account | U3, U4 | FR-001 |
| Create and manage group | U6, U7 | FR-002 |
| Invite / accept invitation | U7, U8 | FR-002 |
| Submit expense (equal or percentage) | U10 | FR-003–FR-005 |
| Attach receipt (R2) | U10, U11 | FR-003 |
| Receive notifications | U5, U16 | FR-008 |
| Review expense | U11, U12 | FR-006, FR-007 |
| View balances | U13, U17 | FR-009 |
| Record payment | U14, U15 | FR-010 |
| View summary | U18 | FR-011 |
| Run date-range report (R2) | U19 | FR-012 |
| View history | U9, U11, U15, U20 | NFR-005 |

## 4. Interaction rules

1. **Money input:** a text field with a plain decimal format. The browser sends the string unchanged and never parses it into a float for arithmetic.
2. **Previews come from the server** (`/expenses/preview`). The client never calculates shares.
3. **Double submits:** buttons are disabled while a request is in flight. An `Idempotency-Key` is generated when the form opens.
4. **Problem codes:** every code in [application-services.md §7](application-services.md#7-error-codes) has a plain-language message. Field errors are attached to fields through `errors[].field`.
5. **Unauthorized navigation:** a 404 shows a generic "Not found", and a 401 redirects to Sign in.
6. **Status language:** the words "Pending review", "Approved", "Rejected", and "Payment (settlement)" are used consistently (AC-023).

## 5. Accessibility expectations (suggested, QA-S1; not an approved requirement)

- Every input has a visible `<label>`. Errors are linked with `aria-describedby` and announced through an `aria-live` region.
- All actions can be done by keyboard. Focus is visible. Dialogs trap focus and return it when closed.
- Status is shown with text, not color alone.
- Contrast is at least 4.5:1. The layout stays usable at 400 px width and at 200% zoom.

## 6. Conditional Swing desktop UI [COND-B]

> Build this only after **documented customer approval** of OQ-1. A Swing UI **does not satisfy NFR-001 or AC-020**.

- **Structure: Model–View–Presenter.**
  - **Views** (Swing panels): layout and event forwarding only. They do no `BigDecimal` arithmetic (ArchUnit).
  - **Presenters:** plain Java with no `javax.swing` imports. They run service calls in `SwingWorker`s and map `ErrorCode`s to messages.
  - **`SessionHolder`:** holds the `ActorContext`.
- **Screens:** U1–U20 have Swing equivalents calling the **same application services** directly. The differences are:
  - Recovery uses one-time recovery codes shown once at registration (U3/U4 become "Recover with code").
  - Notifications refresh at sign-in and on screen change.
  - Invitations can only be accepted by a user signing in on the **same installation**.
  - Receipts and custom reports (R2) follow the same service contracts if built.
- **Limitations to disclose:** a single machine; notifications only on that machine; the DB file is readable unless encrypted; **NFR-001 is unmet**.
