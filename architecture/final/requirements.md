# SplitSmart: Consolidated Requirements (Reconciled Candidate)

Status: **Reconciled candidate. Pending human and customer review.** This document **restates** the authoritative requirements with the same IDs and meaning, and adds measurement methods, business rules, and phasing. It does **not** replace or amend them.

**Sources, in precedence order:**

1. `UserRequirement.md` (UR): customer baseline.
2. `specs/001-splitsmart-expense-sharing/spec.md` (SPEC): clarified FR, NFR, and AC; clarification session 2026-10-29.
3. `.specify/memory/constitution.md` (CONST): engineering constraints.

If wording here appears to differ from SPEC, **SPEC governs**. Report the difference as a defect in this document.

**Labels** ([README.md §3](README.md#3-labels-used-in-this-package)): **[CUSTOMER REQUIREMENT]**, **[CLARIFIED SPEC]**, **[CONSTITUTION]**, **[ASSUMPTION]**, **[NEEDS CUSTOMER DECISION]**, **[DEFERRED – APPROVAL REQUIRED]**. No requirement in this document has been newly approved.

## 1. Actors and roles

| Actor | Definition (SPEC "Actors" unless noted) | Key permissions (design.md §9.2) |
|---|---|---|
| Guest | Person without a session (derived from FR-001) | Register, sign in, request recovery |
| Account holder | Signed-in person; sees only their own private information and their groups' shared information | Own profile, notifications, invitations, summaries |
| Invited user | Account holder with a PENDING invitation (FR-002, AC-003) | Accept or decline **their own** invitation only |
| Group member | ACTIVE member of a group | View group data; submit expenses; record payments **as sender**; view balances and reports |
| Group creator / manager | Member with MANAGER role; the creator is always a manager | Member management; group settings; review (per OQ-2 default) |
| Expense submitter | Member who records an expense | Edit their own PENDING expense; attach a receipt |
| Expense participant | Member included in an expense's allocation | Sees their share |
| Payer | Member who paid. Selected separately; **may or may not** be a participant (FR-003). | Owed by participants |
| Approver / reviewer | Member authorized to decide an expense (SPEC Actors). Default: managers (OQ-2). | Approve or reject |
| Former member | Removed or left member (derived; D-27) | Read-only access to records they were party to |

## 2. Functional requirements

All FRs are **[CUSTOMER REQUIREMENT]** as clarified in SPEC. Clarified details are marked **[CLARIFIED SPEC]**.

| ID | Requirement (SPEC wording, condensed without change of meaning) | UR source | Clarifications / notes | Acceptance |
|---|---|---|---|---|
| FR-001 | Each person can create an individual account with **email and password** [CLARIFIED SPEC], sign in to access their own expense and balance information, and **recover access** if they cannot sign in. | "User accounts" | Clarification Q2: email + password with recovery | AC-001, AC-002 |
| FR-002 | Users can create and manage groups and their membership. **Only the creator or designated managers may invite or remove members. An invitee must accept before joining.** [CLARIFIED SPEC] | "Group management" | Clarification Q3 | AC-003, AC-004 |
| FR-003 | A group member can create an expense with a **positive amount at the group's currency precision**, date, description, allocation method, **one payer who belongs to the group**, the involved participants, and an **optional receipt image**. The payer is selected **separately** from participants and **may be excluded** from the allocation. [CLARIFIED SPEC] | "Expense creation" | Clarifications Q1 and Q4 | AC-005, AC-006, AC-009 |
| FR-004 | Expenses can be divided **equally** among the selected participants. | "equally split" | | AC-007 |
| FR-005 | Expenses can be divided by **nonnegative percentages totaling exactly 100%**. Shares are rounded to the smallest currency unit using **largest remainder** and **reconcile to the total**. [CLARIFIED SPEC] | "split by %" | Clarification Q4. Percentage precision: [NEEDS CUSTOMER DECISION] OQ-10. | AC-008, AC-009, AC-021 |
| FR-006 | A submitted expense stays out of balances **until approved**. | "Approval workflow" | | AC-010 |
| FR-007 | A review and approval process records whether a submitted expense was **approved or rejected**. | "Approval workflow… single review or multiple levels" | Approver policy: [NEEDS CUSTOMER DECISION] OQ-2. SPEC assumes a single step initially. | AC-011, AC-012 |
| FR-008 | **Other members** of the group are notified when a new expense is submitted. | "Notification system" | Channel: in-app [ASSUMPTION]; others OQ-3 | AC-013 |
| FR-009 | The system tracks how much **each user owes every other user** based on **approved** expenses and recorded payments. | "Balance tracking" | | AC-014, AC-015 |
| FR-010 | The **payment sender** records the payment's sender, recipient, amount, and date. It **immediately** affects the balance. History identifies **who recorded it and when**. [CLARIFIED SPEC] | "Payment tracking" | Clarification Q5 | AC-015, AC-016 |
| FR-011 | Reports or summaries of users' expenses, balances, and payments, **limited to information they may access**. | "Reporting" | | AC-017 |
| FR-012 | Users can generate reports for **specified date ranges** and supported parameters. | "custom reports with specified date ranges and other parameters" | Additional filters and export: OQ-5 | AC-018, AC-019 |

## 3. Nonfunctional requirements and measurement

| ID | Requirement (SPEC) | Type | Measurable verification method (proposed; [verification-plan.md](verification-plan.md)) | Acceptance |
|---|---|---|---|---|
| NFR-001 | The product **MUST be primarily web-based**. A Swing desktop application alone does **not** satisfy this. | [CUSTOMER REQUIREMENT] | Browser E2E: scenarios S1–S9 complete in a current Chromium-based browser and in Firefox through the web UI only (VP-E2E). **Fails by definition under the Swing alternative.** | AC-020 |
| NFR-002 | A signed-in user is limited to their own private information and their groups' shared information. Unauthorized attempts do not reveal protected expense, balance, or payment details. | [CLARIFIED SPEC] | Authorization matrix tests: 100% of cells in design.md §9.2 behave as specified. Non-member requests return 404 with no details (SC-005). | AC-002 |
| NFR-003 | Approved shares reconcile to the total at the smallest currency unit. Fractional remainders use largest remainder. Balance and payment totals reconcile to the records. | [CLARIFIED SPEC] + [CONSTITUTION] IV | Allocation table and property tests; Σ memberNet = 0; SQL vs. in-memory balance equality (SC-002, SC-003) | AC-021 |
| NFR-004 | With up to 50 members and 10,000 expense/payment records, **≥ 95%** of account, expense, balance, and report views are available within **3 s** under ordinary operating conditions. | [CLARIFIED SPEC]; SPEC marks it a **planning assumption, not an SLA** | Benchmark protocol in [verification-plan.md §6](verification-plan.md#6-performance-protocol-nfr-004--ac-022). **Not measured.** CR-03 proposes adding the protocol to SPEC. | AC-022 |
| NFR-005 | Review outcomes and payments stay identifiable in group history (pending, approved, rejected, settled). | [CLARIFIED SPEC] | History and audit API tests; UI status badges (S5, S7) | AC-023 |

### 3.1 Suggested additional quality attributes (NOT approved requirements)

These are engineering suggestions only. They must not be reported as customer requirements.

| ID | Suggestion | Rationale | Status |
|---|---|---|---|
| QA-S1 | Accessibility: target WCAG 2.1 AA for the web UI (keyboard operation, labels, contrast, errors announced) | Spec Kit ui-contracts "accessible errors"; good practice | Suggested |
| QA-S2 | Supported browsers: current Chrome/Edge, Firefox, Safari | Makes NFR-001 testable | Suggested |
| QA-S3 | Security baseline: TLS, secure cookies, CSRF, CSP, login rate limiting | Supports NFR-002 | Suggested; designed in design §13 |
| QA-S4 | Daily database backup | Data durability | Suggested; deployment-dependent (OD-1) |
| QA-S5 | Responsive layout usable at about 400 px width | Mobile access to a web product | Suggested |

## 4. Acceptance criteria

SPEC text is condensed without changing meaning. Verification for each criterion is in [traceability.md](traceability.md).

| AC | Req | Given / When / Then |
|---|---|---|
| AC-001 | FR-001 | A person without an account creates one with email and password → they can sign in, access their own expense and balance information, and use account recovery if they cannot sign in. |
| AC-002 | FR-001, NFR-002 | Two signed-in account holders → each sees only their own private information and information shared with groups they belong to. |
| AC-003 | FR-002 | A creator or designated manager invites another account holder → the invitee joins only after accepting. |
| AC-004 | FR-002 | A non-manager member attempts to invite or remove members → denied. A manager removes a member or updates group details → the group reflects the change, and prior expense records remain attributable to their original participants. |
| AC-005 | FR-003 | A member submits an expense with a positive amount at the group's currency precision, date, description, allocation method, selected payer, and participants → the expense records those values and identifies its submitter. |
| AC-006 | FR-003 | A supported receipt image is attached → associated with the expense. No image → the expense can still be submitted. |
| AC-007 | FR-004 | Equal split across selected participants → shares sum to the entered amount. |
| AC-008 | FR-005 | Valid percentages totaling 100% → shares sum to the entered amount. |
| AC-009 | FR-003, FR-005 | Zero or negative amount, excess precision, negative percentages, percentages not totaling exactly 100%, or a share assigned to a non-participant → identified as invalid and not accepted. |
| AC-010 | FR-006 | A newly submitted expense awaiting review → visibly pending and does not change any balance. |
| AC-011 | FR-007 | An authorized approver approves → the allocation is accepted and reflected in balances. |
| AC-012 | FR-007 | An authorized approver rejects → not included in balances, and the outcome is visible to relevant members. |
| AC-013 | FR-008 | Successful submission → the **other** members are notified of the new expense and its review status. |
| AC-014 | FR-009 | Approved expenses → a member sees the amount owed by each user to every other user. |
| AC-015 | FR-009, FR-010 | A recorded payment → the owed amount is reduced by the payment and **does not become negative solely because of an overpayment**. |
| AC-016 | FR-010 | The sender records sender, recipient, amount, and date → it immediately affects the balance, and history identifies who recorded it and when. |
| AC-017 | FR-011 | A member opens a summary → expense, balance, and payment categories for information they are authorized to view. |
| AC-018 | FR-012 | A custom report with a valid date range → only records in that range and matching the selected parameters are included. |
| AC-019 | FR-012 | An invalid date range → the system explains it and does not present a valid report. |
| AC-020 | NFR-001 | A supported web browser → the account holder completes the supported journeys through the primary web experience. |
| AC-021 | NFR-003 | Fractional-cent equal or percentage allocations → leftover cents by largest remainder; shares and balances reconcile at the smallest unit. |
| AC-022 | NFR-004 | The 50-member / 10,000-record dataset under ordinary conditions → ≥ 95% of account, expense, balance, and report views within 3 s. |
| AC-023 | NFR-005 | Submitted expenses and payments → each expense identifiable as pending, approved, or rejected, and payments identifiable as settlements. |

Success criteria SC-001 to SC-006 in SPEC remain in force. Verification is mapped in [traceability.md §5](traceability.md#5-success-criteria-sc-001sc-006).

## 5. Business rules

| ID | Rule | Label / source |
|---|---|---|
| BR-01 | Money uses `BigDecimal` only; scale is explicit at calculation and persistence boundaries | [CONSTITUTION] IV |
| BR-02 | Amount > 0 with no more decimals than the group currency allows. Input is rejected, never rounded. | [CLARIFIED SPEC] FR-003; rejecting rather than rounding is D-12 |
| BR-03 | Percentages ≥ 0 and Σ = exactly 100 | [CLARIFIED SPEC] FR-005 |
| BR-04 | Shares are computed by largest remainder and Σ shares = amount exactly. Ties go to the lowest account ID (string order). | [CLARIFIED SPEC] FR-005/NFR-003; tie-break D-17 |
| BR-05 | The payer is one ACTIVE group member, chosen separately from participants, and may or may not participate | [CLARIFIED SPEC] FR-003 |
| BR-06 | Participants are a non-empty set of distinct ACTIVE members. Shares exist only for participants. | [CLARIFIED SPEC] AC-009; non-empty set is derived |
| BR-07 | Only APPROVED expenses affect balances. PENDING and REJECTED never do. | [CLARIFIED SPEC] FR-006/007, AC-010–012 |
| BR-08 | Only the payment sender records a payment. It takes effect immediately. Recorder and record time are stored. | [CLARIFIED SPEC] FR-010 |
| BR-09 | A payment may not exceed what the sender currently owes the recipient in that group | D-25; [NEEDS CUSTOMER DECISION] OQ-7 (confirm) |
| BR-10 | Only the creator or managers invite or remove members. An invitee must accept. Acceptance is by the intended invitee only. | [CLARIFIED SPEC] FR-002; invitee-only is D-32 |
| BR-11 | Historical expenses, decisions, and payments remain attributed after membership ends | [CLARIFIED SPEC] AC-004 |
| BR-12 | Users see only their own private data and their groups' data. Former members see only records they were party to. | [CLARIFIED SPEC] NFR-002; former members D-27 |
| BR-13 | Notifications go to the **other** active members on submission | [CLARIFIED SPEC] FR-008, AC-013 |
| BR-14 | Report date ranges are inclusive on business dates. `to < from` is invalid. Results are limited to authorized data. | [CLARIFIED SPEC] FR-011/012, AC-018/019; inclusive boundaries D-38 |
| BR-15 | Default reviewer policy: managers; no self-approval unless the group enables it; single level | [NEEDS CUSTOMER DECISION] OQ-2 (D-21) |

## 6. Assumptions and dependencies

| ID | Assumption | Source |
|---|---|---|
| AS-1 | One currency per group, fixed after the first expense; default USD | SPEC Assumptions; OQ-4 |
| AS-2 | Percentages have at most 2 decimals | Proposed; OQ-10 |
| AS-3 | Single authorized review step; approvers = managers; self-approval off by default | SPEC Assumptions; OQ-2 |
| AS-4 | Notifications are in-app | SPEC Assumptions; OQ-3 |
| AS-5 | A person must have an account before being added as a member | SPEC Assumptions |
| AS-6 | Payments settle balances within one group; no cross-group netting | Proposed |
| AS-7 | Overpayments are rejected | Interpretation of AC-015; OQ-7 |
| AS-8 | Pending expenses are editable by the submitter; approved and rejected are immutable; no deletion | OQ-6 |
| AS-9 | Expense and payment dates ≤ today + 1 day | Proposed |
| AS-10 | Maximum amount 1,000,000 major units | Proposed; OQ-11 |
| AS-11 | The NFR-004 dataset and target are planning assumptions, not an SLA | SPEC Assumptions |
| AS-12 | "Payer may not be a participant" (Claude prompt) is read as "need not be". SPEC FR-003 governs. | UD-5 |

**Dependencies:**

| Dependency | Needed for |
|---|---|
| Customer answers OQ-1 to OQ-13 | Decisions |
| JDK 21 and Maven 3.9+ on the build machine | CONST II; OD-2 |
| [COND-A] PostgreSQL | Production and concurrency tests |
| [COND-A] SMTP relay | Recovery email |
| [COND-A] Writable private storage | Receipts |
| [COND-A] Hosting | OD-1 |

## 7. Scope and phasing

| Scope element | Label | Content |
|---|---|---|
| **Full customer scope** | [CUSTOMER REQUIREMENT] | FR-001–FR-012, NFR-001–NFR-005, AC-001–AC-023. Primary web delivery. |
| **Release 1 (candidate MVP)** | Engineering proposal (D-33) | FR-001–FR-011 with a single review level, in-app notifications, basic summaries. Web, unless OQ-1 says otherwise. |
| **Release 2** | Engineering proposal (D-33) | FR-003 receipt images (AC-006); FR-012 custom/date-range reports (AC-018, AC-019). **Planned within the delivered product.** |
| Removal of Release 2 items from the delivery | **[DEFERRED – APPROVAL REQUIRED]** | Requires OQ-13 approval. Until then they remain committed scope. |
| Multiple approval levels; channels beyond in-app; report export | [NEEDS CUSTOMER DECISION] | OQ-2, OQ-3, OQ-5. Not required until the customer asks. |
| Swing desktop MVP | [NEEDS CUSTOMER DECISION] | OQ-1. Even if approved, it leaves NFR-001/AC-020 **unmet**. |
| Out of scope unless requested | [ASSUMPTION] | Account deletion (OQ-9), exchange rates (OQ-4), cross-group netting (AS-6) |
