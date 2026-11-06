# SplitSmart: Open Questions and Change Requests

Status: **Reconciled candidate. Pending human and customer review.** Nothing in this document has been answered or approved. **No authoritative source has been modified.** `UserRequirement.md`, `spec.md`, and the constitution are unchanged. The change requests below are *proposals* for the customer or owner to accept or reject.

## 1. Customer questions

Owner: **Customer** (through the project owner or instructor acting as customer liaison).

| ID | Question | Source | Default used in the design | Impact if the answer differs | Blocks | Required follow-up |
|---|---|---|---|---|---|---|
| **OQ-1** | **Must the MVP itself be web-based, or does the customer approve a Swing desktop MVP (which would leave NFR-001/AC-020 unmet)?** | SPEC OQ-1; UR; CONST VIII; XR F-01 | Web (Alternative A) | Swing: build the `splitsmart.desktop` package per [ui-workflows.md §6](contracts/ui-workflows.md#6-conditional-swing-desktop-ui-cond-b), record NFR-001 as **unmet**, and raise CR-06 | **Yes: all presentation and deployment work** | Written answer recorded in this repository |
| OQ-2 | Who may approve or reject an expense? May submitters approve their own? When are multiple approval levels needed? | SPEC OQ-2; XR F-04 | Managers; self-approval off unless enabled per group; single level | `ReviewPolicy` change only. Multi-level would use the reserved `ReviewDecision.level`. | No | Answer, then CR-04 |
| OQ-3 | Which notification channels are required beyond in-app? | SPEC OQ-3 | In-app only | Add a `NotificationPublisher` adapter (e.g. SMTP) | No | Answer |
| OQ-4 | Which currencies, and is exchange-rate handling required? | SPEC OQ-4; XR F-06 | One currency per group; scales {0,2,3}; default USD; no conversion | Multi-currency inside one group is a significant model change | No | Answer, then CR-05 |
| OQ-5 | Should reports be exportable? Which filters beyond date range? | SPEC OQ-5 | No export; filters: group, record type, expense status, only-mine | Additive endpoints and filters | No | Answer |
| OQ-6 | Can members edit or delete pending or approved expenses? What history must be kept? | SPEC OQ-6 | Submitter edits pending only; no deletion; audit keeps before/after | Deletion or approved-edit would need reversal semantics | No | Answer |
| OQ-7 | Confirm that an overpayment is **rejected** rather than recorded as a credit | SPEC AC-015; D-25 | Reject | Credit would allow reversed balances, which must be reconciled with AC-015 wording | No | Confirmation, then CR-02 |
| OQ-8 | May a member with an unsettled balance leave or be removed? | SPEC edge case; D-26 | Blocked until settled | Allowing it needs read-only carry-over rules | No | Answer |
| OQ-9 | Is account deletion required? What is the retention policy for financial history and audit data? | Not in SPEC; D-37 | Out of scope; data retained | Anonymization design needed | No | Answer |
| OQ-10 | How many decimal places may percentages have? | SPEC FR-005 (unspecified); D-14; XR F-06 | 2 | Validation constant only | No | Answer, then CR-05 |
| OQ-11 | Is there a maximum expense or payment amount? | Not in SPEC; D-15; XR F-06 | 1,000,000 major units | Constant only | No | Answer |
| OQ-12 | Receipt limits and retention: file types, maximum size, how long images are kept | SK research "Receipt image storage"; D-24; XR F-05 | JPEG/PNG; ≤ 5 MB; kept as long as the expense exists | Storage adapter and configuration | No | Answer |
| OQ-13 | Release phasing: are receipts (FR-003) and custom date-range reports (FR-012) in the final delivery as planned (Release 2)? Any removal needs explicit approval. | SPEC "Deferred from the candidate MVP"; XR F-09; D-33 | **Both included** in the planned delivery | A cut must be recorded as an approved deviation; traceability would mark them deferred | Only if a cut is proposed | Written confirmation |

## 2. Project-owner decisions

| ID | Decision | Default | Impact |
|---|---|---|---|
| OD-1 | Hosting for A | Local demo | Deployment only |
| OD-2 | JDK 21 install vs. JDK 25 with `--release 21` | Install JDK 21 | Reproducibility (F-11). The observed machine has JDK 25.0.2 and no Maven. |
| OD-3 | Front-end technology (D-06) | Static HTML/JS | Front-end effort |
| OD-4 | Email verification at registration (D-11) | Not in Release 1 | Registration flow |

## 3. Decision blockers

| Blocker | Blocked work | Work that can proceed now |
|---|---|---|
| OQ-1 (platform) | `splitsmart.web` or `splitsmart.desktop`; deployment; AC-020 verification; browser or Swing E2E tests | After human review: `splitsmart.domain`, `splitsmart.application`, `splitsmart.persistence`, and their JUnit 5 tests (D-03) |
| Human review of this package (CONST VII) | Treating any document as a baseline | Review itself |
| OD-2 / tooling | Any claim that `mvn verify` passes | Documentation |

## 4. Unapproved deviations and interpretations

These are places where the design goes beyond or interprets the sources. None is approved.

| ID | Item | Nature | Status |
|---|---|---|---|
| UD-1 | Release phasing (R1/R2) | Delivery sequencing. Not a scope change while both releases are in the delivery. | Pending OQ-13 |
| UD-2 | Overpayment rejection | Interpretation of AC-015 | Pending OQ-7 |
| UD-3 | Proposed limits (percentage dp, currency scales, amount ceiling, receipt size) | Engineering defaults for unspecified values | Pending OQ-4, OQ-10, OQ-11, OQ-12 |
| UD-4 | Former members keep read-only access to their own party records | Interpretation of FR-001 and NFR-002 | PROPOSED (D-27) |
| UD-5 | The Claude Code prompt's phrase "payer… may not be a participant" | Read as "**need not** be a participant", per SPEC FR-003 and the 2026-10-29 clarification. The payer **may** also be a participant. SPEC takes precedence over the prompt. | Resolved by source precedence. Recorded for transparency. |
| UD-6 | Audit log (D-31) | Addition supporting NFR-005 | PROPOSED |

## 5. Proposed change requests to authoritative sources (NOT APPLIED)

| ID | Target | Proposed change | Reason | Approver | Precondition |
|---|---|---|---|---|---|
| CR-01 | SPEC §"Proposed Java Swing MVP Subset" | Rename to a platform-neutral "Candidate Release Scope". State that the functional slice applies to whichever platform is approved. | The current heading ties the candidate MVP to Swing although the baseline is web. This invites the misreading XR F-01 warns about. | Customer / spec owner | None |
| CR-02 | SPEC FR-010 / AC-015 | Add: "A payment greater than the amount the sender currently owes the recipient is rejected." | Makes overpayment behavior explicit and testable | Customer | OQ-7 confirmed |
| CR-03 | SPEC NFR-004 / AC-022 | Add the measurement protocol (environment, views, timing boundary, dataset), or reference verification-plan §6 | "Ordinary operating conditions" is not measurable as written (XR F-08) | Customer / spec owner | None |
| CR-04 | SPEC FR-007 | Record the approved reviewer policy, self-approval rule, and level count | Closes OQ-2 | Customer | OQ-2 answered |
| CR-05 | SPEC FR-003 / FR-005 / Assumptions | Record supported currencies, percentage precision, and amount ceiling | Closes OQ-4, OQ-10, OQ-11 | Customer | Answers received |
| CR-06 | UR / SPEC NFR-001 / CONST VIII record | **Only if** the customer approves Swing: record the platform change, its approval, and that NFR-001 is unmet for the MVP | Constitution I and VIII require documented approval before the baseline changes | Customer | OQ-1 = Swing |

## 6. Required follow-up

1. The project owner obtains written answers to OQ-1 (first) and OQ-2 to OQ-13.
2. Update [decisions.md](decisions.md) statuses and dates **only** from the documented answers.
3. If a change request is accepted, the spec owner edits the authoritative source. This package is then updated to cite the new version.
4. A human reviews this package (CONST VII) before any document is treated as a baseline.
