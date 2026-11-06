# SplitSmart — Reconciliation Instructions for the Final Requirements and Design Baseline

**Purpose:** Give Claude Code an actionable, controlled workflow for reconciling the two independent proposals into one coherent, reviewable final documentation package. **Use Claude Code's detailed design as the starting point, not as an automatically approved specification.** The final package must cover requirements (including nonfunctional requirements), architecture, detailed design, data, interfaces, PlantUML diagrams, verification, traceability, and decisions.

**Execution context:** Run Claude Code from the repository root. This file is an instruction prompt; following it should create or update documentation, **not** implement the application. Do not commit or push the generated documents automatically.

## 1. Read the evidence before editing

Read the following existing files in full:

1. `UserRequirement.md` — authoritative customer baseline.
2. `specs/001-splitsmart-expense-sharing/spec.md` — clarified FR-001–FR-012, NFR-001–NFR-005, AC-001–AC-023, assumptions, and open questions.
3. `.specify/memory/constitution.md` — binding engineering and governance constraints.
4. `architecture/CROSS_REVIEW_REPORT.md` — comparison, findings F-01–F-11, gaps, and readiness assessment.
5. All existing documentation under `architecture/claude-code/`, particularly `design.md`, `architecture.md`, `data-model.md`, `research.md`, `traceability.md`, `contracts/`, `quickstart.md`, and `diagrams/`.
6. All Spec Kit planning/design documents under `specs/001-splitsmart-expense-sharing/`, particularly `architecture.md`, `plan.md`, `research.md`, `data-model.md`, `contracts/`, `quickstart.md`, `diagrams/`, and the requirements checklist.

Inspect the actual repository contents; do not assume a referenced file exists. If something is absent, record it rather than inventing its contents. You may inspect existing application/build configuration to distinguish proposed from implemented behavior.

**Authority order:** customer baseline first; clarified specification (unless it conflicts with the baseline); constitution for engineering constraints; then cross-review findings; then the independent designs as candidate solutions. Where sources conflict, identify the conflict, cite the exact file/section, and request the necessary decision. Agreement between two AI proposals is not customer approval.

## 2. Reconciliation rules

- Start from Claude Code's `design.md` and its associated architecture/model/contracts. Retain its useful implementation-level specificity **only where compatible with the requirements**. Incorporate useful or missing coverage from Spec Kit, rather than concatenating documents or discarding either proposal without analysis.
- For every significant choice, distinguish **CUSTOMER REQUIREMENT**, **CONSTITUTION CONSTRAINT**, **CLARIFIED SPECIFICATION**, **ENGINEERING DECISION (PROPOSED)**, **ASSUMPTION**, **DEFERRED WITH APPROVAL REQUIRED**, and **NEEDS CUSTOMER DECISION**. Include rationale and source references.
- Preserve the full customer scope. A candidate MVP or course-timebox does not silently remove receipt images, custom/date-range reports, account recovery, notifications, or other required features. Track release scope and deferred work separately, with explicit approval status.
- **Critical platform gate:** NFR-001 and AC-020 require a primarily web-based experience. Recommend the web-compliant architecture unless the customer explicitly approves a changed delivery scope. Swing-only does **not** meet NFR-001; keep Swing as a clearly labeled conditional alternative. Do not silently convert the baseline to Swing or describe an unapproved MVP as requirements-compliant.
- Preserve Java 21, Maven, JUnit 5, object-oriented separation, testable domain logic, `BigDecimal` for money, explicit currency precision, largest-remainder distribution, and PlantUML. Do not present unrun tests or unmeasured performance as verified.
- Resolve document-to-document contradictions where evidence supports resolution. Where customer authority is required, **do not guess**: keep a clearly marked decision gate, present options and consequences, and provide a safe conditional design. A "final" documentation package may still contain explicitly unresolved approvals; do not claim approval or implementation readiness.
- Avoid unnecessary framework/module complexity for a CIS 285 term project. Keep the design sufficiently detailed to implement and test, but justify optional modules and infrastructure.
- Do not edit or overwrite the original independent proposals, `UserRequirement.md`, `spec.md`, or the constitution. If changes to the authoritative baseline seem necessary, **propose** them in a change-request document for human/customer approval instead.

## 3. Required final output location and document set

Create a separate reconciled package under `architecture/final/` (not under `architecture/claude-code/` or the Spec Kit feature folder). Produce these files, using relative links among them:

| File | Required content |
| --- | --- |
| `README.md` | Package index; purpose; source precedence; scope; status; reading order; how to render diagrams; unresolved gates; explicit statement that this is a reconciled *candidate* pending human/customer review. |
| `requirements.md` | Consolidated, individually numbered functional requirements **FR-001–FR-012**, nonfunctional requirements **NFR-001–NFR-005**, acceptance criteria **AC-001–AC-023**, roles/actors, business rules, assumptions, dependencies, and scope/phasing. Preserve IDs and customer meaning. Do not invent approval. Define measurable NFR verification and separate suggested additional quality attributes from approved requirements. |
| `architecture.md` | Context, system boundary, web-compliant target and conditional Swing alternative, deployment, layers/modules, dependencies, security boundaries, major workflows, quality attributes, architectural decisions/tradeoffs, and risks. |
| `design.md` | Detailed, implementation-oriented design derived primarily from Claude's existing `design.md`: packages/classes/interfaces/services, algorithms, validation, use-case realizations, workflows, authorization, state transitions, error handling, concurrency/transactions, recovery, invitations, notifications, receipts, reporting, and test seams. |
| `data-model.md` | Conceptual/logical entities, attributes/types, identifiers, relationships/cardinalities, constraints/indexes, ownership, state and audit history, money precision, and derived-vs-stored balances. Explain migrations and integrity checks as proposals, not completed work. |
| `contracts/application-services.md` | Shared service operation signatures, DTOs, pre/postconditions, validation and error codes, authorization, state effects, and transaction boundaries. |
| `contracts/web-api.md` | Web REST/API endpoints, auth/recovery, invitations, membership, expenses/receipt upload and download, review, payments, balances, notifications, reports, pagination, idempotency, status/error formats, and security. |
| `contracts/ui-workflows.md` | Browser screens, actor permissions, forms, feedback, navigation, accessible interaction expectations, and use-case-to-screen mapping; Swing-specific UI only in a clearly conditional subsection or separate document. |
| `decisions.md` | Decision log with ID, topic, source, options, recommendation, rationale, owner/approval authority, status, date (only if known), impact, and affected artifacts. Highlight unresolved customer decisions. |
| `reconciliation-matrix.md` | Item-by-item record of Claude vs Spec Kit vs source requirement, final disposition (adopt/modify/reject/defer/open), evidence and reason. Explicitly address cross-review F-01–F-11. |
| `traceability.md` | Bidirectional mapping of **every** FR, NFR, and AC to design sections, data/contracts, UML diagrams, planned tests, and release status. Mark gaps and unverified criteria clearly. |
| `verification-plan.md` | Unit, integration, concurrency, security, contract, UI/end-to-end, UML validation, and performance tests with expected results; Java 21/Maven/JUnit 5 commands where applicable, labeled *proposed* until run. |
| `quickstart.md` | Planned prerequisites, setup/build/test/run instructions, diagram rendering, and concrete end-to-end walkthroughs; distinguish existing code from planned implementation. |
| `open-questions-and-change-requests.md` | Customer questions, decision blockers, unapproved deviations, proposed source requirement changes, responsible approver, impact, and required follow-up. |

Add other focused documents only when they materially improve clarity (e.g., `security.md` or `deployment.md`), and link them from the index. Do not duplicate inconsistent definitions across files.

## 4. Mandatory PlantUML package

Create **maintainable PlantUML source** in `architecture/final/diagrams/`. At minimum include all six requested diagram types below, plus a useful domain/data diagram. Each source must have a descriptive title, actor/component names consistent with the final documents, and a short reference in the relevant prose. Use `.puml` files; optionally render SVG/PNG if the required renderer is available. **Never claim rendering succeeded unless it was run.**

| Source file | Diagram and minimum coverage |
| --- | --- |
| `use-cases.puml` | **Use-case diagram:** actors (guest, account holder, group member, manager/reviewer, invited user as appropriate); registration/recovery, groups/invitations, expense submission/splitting, approvals, notifications, balances, payments, summaries and date-range reports. Clarify roles and authorization rather than depicting every actor as privileged. |
| `components.puml` | **Component diagram:** browser/UI, API/controllers, application services, domain services, repositories/persistence, database, recovery/notification and receipt adapters; provided/required interfaces, dependencies, and trust boundaries. Reflect the recommended web path; identify conditional alternatives separately. |
| `classes-detailed.puml` | **Detailed class diagram:** Account, Group, Membership, Invitation, Expense, ExpenseParticipant/Allocation, ReviewDecision, Payment, Notification, Money, allocation/balance services, policies, repositories, and key service interfaces; meaningful attributes, operations, types, multiplicities, ownership and dependency arrows. Avoid one unreadable giant class diagram: split into additional linked class diagrams by subsystem if necessary, while retaining this overview. |
| `sequence-expense-review.puml` | **Sequence diagram:** submission, validation, equal/percentage allocation, persistence, pending state, notification, reviewer authorization, approve/reject, transaction and balance visibility. Show failure/alternate paths. |
| `sequence-payment.puml` | **Sequence diagram:** sender-recorded payment, authorization, outstanding calculation, overpayment policy, idempotency/duplicate behavior, transaction/concurrency, persistence, immediate derived balance and audit. |
| `states.puml` | **State diagrams:** at least expense PENDING → APPROVED/REJECTED and relevant guards/actions; include invitation lifecycle and payment lifecycle as separate state files if clearer. Make invalid transitions explicit. |
| `activities.puml` | **Activity diagrams:** expense submission/review and payment settlement, including branches for invalid input, permissions, rejection, approval, and updates. Split into multiple files if clarity improves. |
| `domain-model.puml` | **Domain/ER-style diagram:** entities and cardinalities, key invariants, invitation vs membership, payer vs participants, audit history, and derived balances. |

Use `@startuml`/`@enduml`, appropriate PlantUML syntax, and consistent IDs/terminology. Provide a diagram index in `README.md` mapping each diagram to requirements and design sections. Check syntax by rendering if tooling is installed; otherwise explicitly state **NOT RENDERED** and describe how to validate. Cross-check diagrams against the data model, contracts, authorization matrix, and state machines. Do not include unsupported behaviors just to make a diagram look complete.

## 5. Mandatory issues to reconcile

Address these explicitly, with a disposition in `reconciliation-matrix.md` and linked decision/test evidence:

1. **F-01 / platform:** web-first baseline vs conditional Swing; approval gate and AC-020.
2. **F-02 / recovery:** explicit password-reset request/completion flows, token lifetime/single-use, account privacy, and tests.
3. **F-03 / invitations:** invitation identity, acceptance by intended invitee, expiry, duplicate acceptance, role changes, removal and historical attribution.
4. **F-04 / review:** who can approve/reject, self-approval, single-manager deadlock, single/multi-level approval scope, race handling; seek customer decision.
5. **F-05 / receipts:** secure optional image upload/download, validation, ownership/authorization, storage/retention, full scope vs any approved deferral.
6. **F-06 / money policies:** `BigDecimal`, supported currency scales, percentage precision, amount ceilings, exact equal/percentage reconciliation, largest-remainder deterministic tie-break; label unapproved limits as proposals.
7. **F-07 / payments and concurrency:** sender-only recording, immediate balance effects, no negative owed balance solely due to overpayment, duplicate/idempotency policy, transaction isolation, approval/payment/edit races.
8. **F-08 / performance:** NFR-004/AC-022 benchmark at up to 50 group members and 10,000 expense/payment records; at least 95% of specified views within 3 seconds; define workload, timing boundaries and environment. Do not claim measurements.
9. **F-09 / scope:** receipts and custom/date-range reporting remain in full scope even if candidate MVP defers them; approval and delivery phases explicit.
10. **F-10 / organization:** keep original Spec Kit and Claude outputs in place; create final docs only under `architecture/final/`.
11. **F-11 / reproducibility:** Java 21 requirement, Maven verification, JUnit 5, and proposed vs actually executed commands.
12. **Cross-cutting:** group privacy and former-member visibility; payer may be a group member **without** being an expense participant; pending/rejected expenses never affect balances; approved expenses do; audit/history survives relevant changes; notifications target appropriate other group members; date-range reporting respects permissions.

## 6. Required reconciliation workflow

**Phase A — Inventory and gap analysis (no edits):** List all available source documents, compare requirement coverage, inventory existing UML sources, extract contradictions and open decisions. Identify what is explicitly approved versus merely proposed. Use the cross-review report as a checklist, not unquestionable truth.

**Phase B — Decision register:** Draft `decisions.md` and `open-questions-and-change-requests.md`. Recommend sensible engineering defaults, but keep customer-owned choices unresolved. Where the platform is not approved, make the web-compliant architecture the primary documented path and Swing an unapproved alternative.

**Phase C — Requirements baseline:** Draft `requirements.md` without weakening customer scope. Preserve FR/NFR/AC identifiers, add source citations, define acceptance/measurement methods, and label candidate release phases separately.

**Phase D — Technical reconciliation:** Use Claude's detailed design as the working basis. Compare each major subsystem against Spec Kit and the source requirements. Produce a consistent architecture, detailed design, data model, interface contracts, and security/transaction policies. Explain significant deviations from Claude.

**Phase E — UML:** Generate all diagram types in §4 from the reconciled design, not from assumptions. Check relationships, cardinalities, method names, state guards, API calls and actor privileges against the text.

**Phase F — Traceability and verification:** Complete the reconciliation matrix, FR/NFR/AC traceability, test scenarios, performance protocol and quickstart. Mark `PROPOSED`, `UNVERIFIED`, `BLOCKED`, and `APPROVED` accurately. Do not mark any item approved without a documented human/customer decision.

**Phase G — Quality gate:** Inspect every generated Markdown link and diagram reference; check all expected files exist; check for contradictory state names, data fields, service names and endpoint contracts; verify no FR/NFR/AC is missing; review F-01–F-11 dispositions; render PlantUML if available; record exactly what was and was not validated. Fix documentation inconsistencies found. Do not claim code builds, tests, security controls or NFR-004 benchmarks passed without executing them.

## 7. Completion report and stopping conditions

At the end, provide:

- Exact paths of all generated/modified files in `architecture/final/`.
- A concise summary of the chosen *proposed* architecture and differences from Claude's starting design.
- A table of **open decisions**, owner/approval needed, impact, and what work is blocked.
- Coverage totals for FR-001–FR-012, NFR-001–NFR-005, AC-001–AC-023, and cross-review F-01–F-11.
- Diagram inventory and render/validation status.
- Checks actually executed vs planned only, with any failures or unavailable tools.
- Explicit readiness verdict: **READY FOR HUMAN REVIEW**, **READY WITH APPROVED CONDITIONS**, or **BLOCKED**; never claim implementation-ready merely because the documentation exists.

**Safety/scope:** Documentation and PlantUML only. No application implementation, no fabricated customer decisions, no overwriting original proposals or authoritative source files, no silent baseline changes, and **no commit/push/PR** during the reconciliation run. Human review is required under constitution VII.
