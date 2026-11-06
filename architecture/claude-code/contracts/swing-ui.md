# Swing UI / Presenter Contract: Alternative B

> **[CONDITIONAL-B]:** applies **only** if the customer gives **documented approval** of a Swing desktop MVP (OQ-1). A Swing application **does not satisfy NFR-001** (primarily web-based). NFR-001 stays an outstanding customer requirement under this alternative. Do not build this without that approval.

Status: **draft, needs human review.** Module: `splitsmart-desktop`. Calls the same [application services](application-services.md); no business rules live here.

## 1. Structure (Model–View–Presenter)

| Element | Responsibility | Must NOT |
|---|---|---|
| `*View` (Swing `JPanel`/`JDialog`) | Lay out widgets; forward user events to the presenter; render view-models; show field errors | Compute splits or balances, validate business rules, access persistence, hold `BigDecimal` arithmetic |
| `*Presenter` (plain Java) | Convert widget input to commands; call application services in a `SwingWorker`; map results and `ErrorCode`s to view-models | Import `javax.swing` (keeps presenters unit-testable without a GUI) |
| `*ViewModel` | Immutable display data (strings already formatted) | n/a |
| `SessionHolder` | Holds `ActorContext` after sign-in; cleared on sign-out | n/a |
| `DesktopApplication` | Composition root: builds the DB, persistence adapters, services, and presenters; starts the UI on the EDT | n/a |

Threading contract:

- Views are touched only on the Event Dispatch Thread.
- Every service call runs off-EDT (`SwingWorker`).
- Buttons are disabled while a call is in flight, which prevents double submission. Idempotency keys are still sent.

## 2. Screens → service calls

| Screen | User actions | Presenter → service | Error mapping (examples) |
|---|---|---|---|
| Sign-in | sign in; go to register; recover | `AccountService.authenticate` | `INVALID_CREDENTIALS` → "Email or password is incorrect" |
| Register | submit | `register` → **show 8 recovery codes once** with "I have saved these" confirmation | `EMAIL_UNAVAILABLE`, field codes inline |
| Recover access | email + recovery code + new password | `recoverWithCode` | `INVALID_CREDENTIALS` (uniform) |
| My groups (home) | open group; create group; view invitations; notification badge | `listMyGroups`, `listMyInvitations`, `listMyNotifications(unreadOnly)` | n/a |
| Create/edit group | name, description, currency, self-approval toggle | `createGroup` / `updateGroup` | `CURRENCY_LOCKED`, `CONCURRENT_MODIFICATION` → "Reload and try again" |
| Group detail: Members tab | invite by email; remove; make/revoke manager (MGR only; controls hidden for others, and the server-side check still applies) | `inviteMember`, `removeMember`, `setManager` | `MEMBER_HAS_OUTSTANDING_BALANCE`, `LAST_MANAGER`, `FORBIDDEN_ROLE` |
| Invitations | accept / decline | `respondToInvitation` | `INVITATION_NOT_PENDING` |
| Group detail: Expenses tab | list with status badges PENDING/APPROVED/REJECTED (NFR-005); filter by status | `listGroupExpenses` | n/a |
| Expense form | amount, date, description, payer (dropdown of active members), participants (checkbox list, **separate** from payer), split method; percentage fields when PERCENTAGE; live preview of shares | `submitExpense` / `editPendingExpense`. The **share preview** is the server-computed result of a dry-run call `ExpenseService.previewAllocation` **[PROPOSAL]**, so the view never computes shares | all §4.2 codes shown beside the relevant field |
| Expense detail | view shares, review decision; approve/reject (only if `listPendingReviewsForMe` contains it) | `getExpense`, `approveExpense`, `rejectExpense` | `SELF_APPROVAL_NOT_ALLOWED`, `EXPENSE_NOT_PENDING` |
| Group detail: Balances tab | pairwise list "X owes Y $Z"; "Record payment" from a row where I am the debtor | `getGroupBalances` | n/a |
| Record payment dialog | recipient (pre-filled), amount (pre-filled with outstanding), date, note | `recordPayment` (sender = me) | `PAYMENT_EXCEEDS_OUTSTANDING` → shows outstanding; `POSSIBLE_DUPLICATE_PAYMENT` → confirm dialog, retry with `confirmPossibleDuplicate = true` |
| Group detail: Payments tab | history with recordedBy/recordedAt (NFR-005) | `listPayments` | n/a |
| Summary | per-group or all-groups summary | `ReportService.getSummary` | n/a |
| Notifications | list; mark read | `listMyNotifications`, `markRead` | n/a |

Deferred in the candidate MVP, retained in full scope:

- Receipt attachment.
- Custom date-range reports (`runReport`).

If they are added later, they follow application-services.md unchanged.

## 3. Shared-contract operation used by this UI

`ExpenseService.previewAllocation` (defined in [application-services.md §4.1](application-services.md#41-commands-and-queries)) returns the shares or validation errors without persisting anything. Authorization: AM. It is **recommended for B** so that the view never computes shares, and **optional for A** (`POST /groups/{groupId}/expenses/preview`).

## 4. Platform limitations to disclose to the customer

1. All group members must use the **same installation on the same computer**.
2. Notifications appear only when the recipient next signs in on that machine.
3. Anyone with OS access to the DB file can read it unless file encryption is enabled.
4. NFR-001 is **not met**.
