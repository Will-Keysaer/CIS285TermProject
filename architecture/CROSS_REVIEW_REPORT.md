# SplitSmart — Independent Architecture Cross-Review

**Review status:** Documentation review only; **NOT READY** for implementation without decisions and corrections.  
**Repository:** `Will-Keysaer/CIS285TermProject`, `master` (documents available at review time, 2026-11-05).  
**Scope:** Compare Spec Kit's proposal in `specs/001-splitsmart-expense-sharing/` with Claude Code's proposal in `architecture/claude-code/`. This report is an independent review, **not** a modification to either proposal. No application code, tests, build, or performance benchmark was run.

## 1. Executive summary

Both proposals preserve the central customer/platform conflict: `UserRequirement.md` requires a **primarily web-based** product, while a Swing-only MVP would require documented customer approval and would not satisfy the current web requirement (`NFR-001`, `AC-020`, constitution VIII). Both propose a layered Java 21/Maven architecture, JUnit 5 tests, `BigDecimal` money, approval-gated balances, and exact split reconciliation.

**Claude Code provides the stronger implementation-level design**: a standalone `design.md`, explicit module/class responsibilities, state machines, algorithms, concurrency policy, detailed contracts, and a traceability document. **Spec Kit provides a coherent higher-level plan**, but its contracts and operational details are less complete. Neither design is automatically correct or approved. Most importantly, neither can be accepted as the chosen implementation path until the platform decision is recorded. The two proposals should not be merged mechanically: several Claude decisions are proposals rather than customer requirements.

**Verdict: NOT READY** for platform-specific implementation. The platform-independent domain model, allocation rules, and unit-test planning can proceed after human review. See blockers below.

## 2. Source baseline and artifacts inspected

**Authoritative:** `UserRequirement.md`; `specs/001-splitsmart-expense-sharing/spec.md` (FR-001–FR-012, NFR-001–NFR-005, AC-001–AC-023); `.specify/memory/constitution.md` (principles I–VIII).

**Spec Kit:** `architecture.md`, `plan.md`, `research.md`, `data-model.md`, `quickstart.md`, `contracts/web-api.md`, `contracts/ui-contracts.md`, `diagrams/splitsmart-architecture.puml`, `diagrams/expense-workflow.puml`, and `checklists/requirements.md`, all under `specs/001-splitsmart-expense-sharing/`.

**Claude Code:** `architecture.md`, `design.md`, `research.md`, `data-model.md`, `quickstart.md`, `traceability.md`, `contracts/README.md`, `contracts/application-services.md`, `contracts/rest-api.md`, `contracts/swing-ui.md`, and the `.puml` files under `diagrams/`, all under `architecture/claude-code/`.

**Evidence limitations:** The primary architecture, model, research, quickstart, and contract documents were inspected directly; diagram source and checklist files were identified in the repository but not independently rendered or validated. No code exists in this review to prove API behavior, database migrations, timing, security, or test success. External research citations were not independently verified. A missing standalone `design.md` on the Spec Kit side is **not itself a defect**: its design information is distributed across its plan, architecture, model, and contracts.

## 3. Requirements coverage matrix

Legend: **Designed** = explicit proposed design, not implemented; **Partial** = covered at high level or with missing contract detail; **Conditional** = platform-dependent; **Unverified** = requires execution or customer decision.

| Requirement | Customer meaning | Spec Kit | Claude Code | Gap / verdict |
|---|---|---|---|---|
| FR-001 | Accounts, login, recovery | Partial: account model and UI recovery, but web API lacks a password-reset endpoint | Designed: account recovery, reset-token API and flow | Spec Kit contract gap; AC-001 not yet verifiable |
| FR-002 | Groups, manager controls, accepted invitations | Partial: roles and invitation acceptance, but acceptance API identifies a member rather than a specific invitation | Designed: invitation IDs, accept/decline, roles, removal | Spec Kit contract needs refinement; AC-003/004 |
| FR-003 | Expense fields, payer, participants, optional receipt | Designed in model; receipt reference, no explicit upload endpoint | Designed; receipt upload is full scope/deferred MVP | Receipt feature must not disappear from full scope; AC-005/006 |
| FR-004 | Equal splits | Designed | Designed with algorithm/test cases | Verify cent reconciliation; AC-007/021 |
| FR-005 | Percentage splits, exactly 100% | Designed | Designed with integer basis points/largest remainder | Claude's 2-decimal percentage limit is a proposal; AC-008/009/021 |
| FR-006 | Pending expenses do not affect balances | Designed | Designed with state machine and query rules | Needs tests; AC-010 |
| FR-007 | Review, approve/reject | Designed | Designed with manager policy, versions and race handling | Approver/self-approval policy needs customer decision; AC-011/012 |
| FR-008 | Notify other group members of submissions | Designed | Designed with notification records | Delivery channel remains open; AC-013 |
| FR-009 | Pairwise balances | Designed, derived from approved expenses/payments | Designed, derived query and netting | Reconciliation/concurrency tests needed; AC-014/015 |
| FR-010 | Sender-recorded payment, immediate balance effect | Designed | Designed with locks, audit, idempotency | Overpayment handling needs verification; AC-015/016 |
| FR-011 | Authorized summaries | Designed | Designed with summary calculations | Need implementation/security tests; AC-017 |
| FR-012 | Custom/date-range reports | Designed | Designed, deferred from candidate MVP | Deferral does not remove full requirement; AC-018/019 |
| NFR-001 | Primary web experience | Web alternative satisfies on paper; Swing conditional | Web alternative satisfies on paper; Swing conditional | **BLOCKED** pending platform decision; AC-020 |
| NFR-002 | Privacy and authorization | Designed at high level | Designed with operation/relationship matrix | Requires adversarial tests; AC-002 |
| NFR-003 | Financial integrity | Designed | Detailed money/allocation invariants | Tests, currency rules, edge cases unverified; AC-021 |
| NFR-004 | 95% of views within 3 s, 50 users/10k records | Performance target noted | Proposed indexes, paging and performance harness | **Unmeasured** in both; AC-022 |
| NFR-005 | Expense states and payment history | Designed | Designed with lifecycle and audit records | Verify visibility and immutable history; AC-023 |

**Acceptance-criteria audit:** AC-001–AC-004 are covered in account/group designs, subject to the Spec Kit API gaps; AC-005–AC-009 in expense and allocation designs; AC-010–AC-013 in review and notification designs; AC-014–AC-016 in balances/payments; AC-017–AC-019 in reporting; AC-020 remains platform-blocked; AC-021 is a testable algorithm requirement; AC-022 is unmeasured; AC-023 needs audit/history tests. This is **design coverage**, not passed acceptance testing.

## 4. Architecture and detailed-design comparison

| Dimension | Spec Kit | Claude Code | Assessment |
|---|---|---|---|
| Platform | Web-first recommendation, conditional Swing | Same, explicit A/B analysis | Sound agreement; customer decision unresolved |
| Structure | Presentation / application / domain / persistence separation | Named Maven modules, classes, ports, adapters | Claude more actionable; avoid adopting its framework choices as facts |
| Money | `BigDecimal`, currency precision, largest remainder | `BigDecimal` value object, basis points, integer minor-unit allocation, deterministic tie-break | Claude more testable; validate supported currencies/limits |
| Persistence | Derived balances and transactional updates proposed | Detailed SQL/query, optimistic versions, group locks, indexes | Claude more concrete; must test concurrency and DB compatibility |
| Security | Group-scoped authorization and privacy | Detailed operation matrix, reset flow, CSRF, visibility rules | Claude stronger; security is unverified |
| Contracts | Browser UI and web API; some operations underspecified | Web REST, application services, conditional Swing presenter contracts | Claude more complete; platform conditionality handled clearly |
| Notifications | Notification entity and submission flow | Recipient rules, records, unread count, future channels | Claude more specific; customer channel unresolved |
| Reporting | Summaries and custom date ranges | Explicit aggregate definitions and filtering semantics | Claude more actionable; date/filter policy needs sign-off |
| Diagrams | PlantUML sources present | PlantUML sources present | Source files exist; diagram consistency/rendering not independently verified |
| Tests | Proposed JUnit 5 and end-to-end scenarios | Named unit/integration/concurrency/security/performance tests | Claude more detailed; neither has executed application tests |
| Complexity | Smaller conceptual plan | More prescriptive modules and policies | Claude may be over-specified for a term project; choose deliberately |

## 5. Prioritized findings

| ID | Severity | Affected | Evidence | Finding and recommended action |
|---|---|---|---|---|
| F-01 | **CRITICAL / gate** | Both | `UserRequirement.md` “OS supported: primary web-based”; spec NFR-001/AC-020/Open Question 1; constitution VIII; both `architecture.md` platform sections | The platform is unresolved. Obtain and record customer approval before treating Swing as the primary MVP. Otherwise build for web. Do not claim Swing-only meets NFR-001. Both proposals correctly flag this; the **decision**, not their documentation, is the blocker. |
| F-02 | **HIGH** | Spec Kit | `contracts/web-api.md` authentication endpoints; spec FR-001/AC-001 | The web API lists account creation, login and current-account retrieval, but no explicit password-reset request/completion contract. Add recovery endpoints, token expiry/use rules, authorization and tests. |
| F-03 | **HIGH** | Spec Kit | `contracts/web-api.md` `POST /api/groups/{groupId}/members/{accountId}/accept`; `data-model.md` `GroupMembership`; spec FR-002/AC-003 | Invitation acceptance is underspecified: an account/member path does not identify which pending invitation is accepted. Specify invitation identity, recipient-only authorization, expiry, duplicate acceptance and membership transition. A separate `Invitation` entity is preferable to conflating invitation and membership states. |
| F-04 | **HIGH** | Both / customer | spec Open Question 2; Spec Kit review design; Claude `architecture.md` §4.4 and `design.md` §5.2 | The exact reviewer policy and self-approval rule remain undecided. Claude proposes manager-only, single-step, self-approval disabled by default and detects a single-manager deadlock. Record customer policy and ensure every group can actually complete review. |
| F-05 | **MEDIUM** | Spec Kit | `contracts/web-api.md` expense submission `receiptReference`; spec FR-003/AC-006 | Receipt attachment is modeled but upload/download, size/type, authorization and retention behavior are not sufficiently contracted. Define a secure flow for full scope; clearly mark any MVP deferral. |
| F-06 | **MEDIUM** | Claude Code | `design.md` §§2–3, §17 DD-1–DD-4; `research.md` R-08/R-09; spec FR-005 | Claude introduces a maximum of two percentage decimal places, a fixed set of currency scales, UUID-string tie-breaking and an amount ceiling. These are engineering proposals, **not** customer-approved requirements. Validate that these limits fit the intended product before implementation. |
| F-07 | **MEDIUM** | Claude Code | `design.md` §§7–8; `research.md` R-11/R-15; spec AC-015/016 | Group-row locking, overpayment rejection and idempotency are reasonable, but correctness under concurrent approval/payment/edit operations needs integration tests against the selected DB. Verify transaction isolation and duplicate behavior rather than assuming the design guarantees it. |
| F-08 | **MEDIUM** | Both | spec NFR-004/AC-022; Spec Kit `plan.md` performance context; Claude `architecture.md` §6.2 and `quickstart.md` S10 | Neither has measured the 3-second/95th-percentile requirement. Define the environment, dataset, view timing boundaries and benchmark before claiming compliance. |
| F-09 | **MEDIUM** | Claude Code | `architecture.md` §1.1; `traceability.md` §4; spec FR-003/FR-012 | Candidate MVP defers receipt images and custom reports. They remain customer requirements; record an approved scope/phasing decision, rather than treating deferral as deletion. |
| F-10 | **LOW** | Both | Actual paths under `specs/…` and `architecture/claude-code/` | Different output directories and lack of a Spec Kit `design.md` are organizational, not functional defects. Cross-review should use actual paths; no forced relocation needed. |
| F-11 | **LOW** | Claude Code | `quickstart.md` prerequisites and proposed commands | The quickstart records a JDK 25 environment observation and explicitly labels proposed commands unverified. Do not treat that as proof of the reviewer's environment or that Java 21 builds succeed. |

## 6. Domain and financial-integrity audit

**Equal/percentage allocation:** Both designs require nonnegative shares and exact reconciliation; Claude specifies minor-unit arithmetic and deterministic largest-remainder ordering (`design.md` §3). This is a strong design choice, but test the zero/one-participant cases, ties, fractional cents, extreme valid amounts, and multi-currency scales. Neither documentation package constitutes executed proof.

**Payer versus participants:** Both preserve the clarified rule that the payer is separately selected, belongs to the group, and may be excluded from the allocation (spec FR-003). Test payer-as-participant and payer-not-participant cases.

**Approval:** Both exclude pending and rejected expenses from balances; approved expenses enter balance derivation. Claude additionally specifies version checks and race handling (`design.md` §§5, 8). The reviewer policy still needs a product decision.

**Balances:** Both favor deriving pairwise balances from approved expense allocations and payments rather than storing an independently mutable balance. This reduces synchronization risks. Test bilateral netting, payer's own share, group isolation, former members and transaction boundaries.

**Payments:** Both follow the sender-records/immediate-effect clarification. Claude proposes overpayment rejection, idempotency and group locks. Test duplicate submissions, concurrent payments, settlement to exactly zero, and historical audit fields. An overpayment must not produce a negative amount owed merely because it was entered (AC-015).

## 7. Constitution compliance

| Principle | Spec Kit | Claude Code | Verdict |
|---|---|---|---|
| I. Customer baseline/traceability | Requirement matrix in `plan.md` | Extensive `traceability.md` | Designed; full verification pending |
| II. Java 21/Maven | Explicit | Explicit module/build proposal | Compliant on paper; no build run |
| III. Object-oriented separation | Explicit layered boundaries | Explicit domain/application/presentation modules | Compliant on paper |
| IV. `BigDecimal` money | Explicit | Explicit value objects and minor-unit calculations | Compliant on paper; test required |
| V. JUnit 5 | Proposed tests | Named unit/integration suites | Compliant on paper; tests not run |
| VI. PlantUML | Diagram sources present | Diagram sources present | Source present; rendering/consistency not verified |
| VII. Human review | Required | Draft/human-review notices | Both appropriately unapproved |
| VIII. Swing is conditional | Explicit | Explicit | Both preserve the conflict; approval still missing |

## 8. Agreements, disagreements, and risks of shared assumptions

**Sound agreements:** Web-first satisfies the baseline; Swing-only cannot satisfy NFR-001; separation of UI and business logic; exact monetary reconciliation; pending expenses excluded from balances; sender-recorded payments affect balances immediately.

**Meaningful differences:** Claude specifies a more concrete module decomposition, technology stack, security controls, database strategy, detailed algorithms, approval policy, and concurrency mechanism. Spec Kit leaves more of these as implementation choices. Claude's specificity is useful for coding, but also creates more assumptions requiring review.

**Potential shared blind spots:** Both need actual acceptance tests, privacy/security tests, performance measurements, and documented customer decisions. Agreement between AI-generated proposals does not establish correctness.

## 9. Recommended direction

1. **Retain** the web-first architecture as the recommendation consistent with current requirements; retain Swing only as an explicitly conditional alternative.
2. **Use Claude's `design.md` as a detailed candidate implementation reference**, not as an approved specification. Evaluate its module count and technology choices against course time and scope.
3. **Keep Spec Kit's plan and architecture** as an independent proposal; there is no need to create a matching folder or `design.md` solely for symmetry.
4. **Repair the Spec Kit API gaps** around account recovery, invitation identity and receipt handling before relying on its contracts for implementation.
5. **Resolve customer decisions**: platform, review/self-approval policy, notification channel, supported currencies, report scope, and deferred-feature approval. Record which decisions the team can make independently.
6. **Implement and test a platform-independent core first** only after human review of the shared domain invariants, without assuming a final UI or hosting choice.

## 10. Readiness checklist

- [ ] Record customer decision on web vs. Swing and update scope/traceability if changed.
- [ ] Confirm reviewer/self-approval policy, including the one-manager case.
- [ ] Confirm MVP deferrals for receipt images and custom reports, or include them in the planned release.
- [ ] Finalize recovery and invitation contracts.
- [ ] Review and approve monetary/currency/percentage limits and tie-breaking.
- [ ] Execute JUnit 5 split, approval, payment, balance, and authorization tests.
- [ ] Execute database concurrency/integration tests against the chosen persistence technology.
- [ ] Render/check PlantUML diagrams against the final contracts and model.
- [ ] Measure NFR-004 with the specified 50-member/10,000-record dataset.
- [ ] Complete human review of both AI-generated proposals.

## 11. Final verdict

**NOT READY** for implementation of a chosen platform. The proposals are useful and substantially cover the requirements at the design level, but customer/platform and approval-policy gates remain, Spec Kit's API contracts need refinement, and neither proposal has demonstrated executable correctness, security or performance. **READY FOR LIMITED DOMAIN-LEVEL PLANNING** once the team reviews the invariant and testing decisions.

**File produced:** `CROSS_REVIEW_REPORT.md` as a local downloadable review artifact. It has **not** been committed or pushed to the repository. Source documents and proposal files were not modified.