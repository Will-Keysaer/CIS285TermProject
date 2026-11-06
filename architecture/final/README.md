# SplitSmart: Reconciled Requirements and Design Package

> **This is a reconciled CANDIDATE, pending human review (Constitution VII) and customer decisions.**
>
> - It is **not** an approved specification. Nothing in it is APPROVED.
> - No application code exists. No test, build, security control, or performance benchmark has been executed.
> - The delivery platform (web vs. Swing, OQ-1) is **unresolved and blocking** for presentation work.

## 1. Purpose and status

**Purpose:** reconcile the two independent proposals into one coherent, reviewable documentation baseline, following `architecture/RECONCILIATION_INSTRUCTIONS.md`:

- **Claude Code** (`architecture/claude-code/`): used as the detailed working basis.
- **Spec Kit** (`specs/001-splitsmart-expense-sharing/`).

The ChatGPT cross-review (`architecture/CROSS_REVIEW_REPORT.md`) served as a checklist.

**Constraints followed:**

- The original proposals and authoritative sources were **not modified**.
- The package lives only under `architecture/final/` (F-10).
- Prepared 2026-11-05. **Readiness verdict: READY FOR HUMAN REVIEW** (§9).

## 2. Source precedence

1. `UserRequirement.md`: customer baseline.
2. `specs/001-splitsmart-expense-sharing/spec.md`: clarified FR, NFR, AC, assumptions, open questions (it governs unless it conflicts with item 1).
3. `.specify/memory/constitution.md`: engineering and governance constraints.
4. `architecture/CROSS_REVIEW_REPORT.md`: findings F-01–F-11.
5. The Claude Code and Spec Kit proposals, as candidate solutions.

Agreement between the two AI proposals is **not** customer approval. Conflicts are recorded in [reconciliation-matrix.md](reconciliation-matrix.md) and [open-questions-and-change-requests.md](open-questions-and-change-requests.md).

## 3. Labels used in this package

| Label in documents | Meaning (RECONCILIATION_INSTRUCTIONS §2 term) |
|---|---|
| **[CUSTOMER REQUIREMENT]** | CUSTOMER REQUIREMENT, from `UserRequirement.md` |
| **[CLARIFIED SPEC]** | CLARIFIED SPECIFICATION, from `spec.md` |
| **[CONSTITUTION]** | CONSTITUTION CONSTRAINT |
| **[ENGINEERING DECISION]** | ENGINEERING DECISION (PROPOSED). Always proposed; never approved by this package. |
| **[ASSUMPTION]** | ASSUMPTION |
| **[DEFERRED – APPROVAL REQUIRED]** | DEFERRED WITH APPROVAL REQUIRED |
| **[NEEDS CUSTOMER DECISION]** | NEEDS CUSTOMER DECISION |
| **[COND-A]** | Applies only to the web platform (recommended, baseline-compliant) |
| **[COND-B]** | Applies only if the customer gives documented approval for Swing. Does **not** satisfy NFR-001. |

**Status words** (traceability and verification): `PROPOSED`, `UNVERIFIED`, `BLOCKED`, `APPROVED`. `APPROVED` is used nowhere.

## 4. Scope

- **Full customer scope:** FR-001–FR-012, NFR-001–NFR-005, AC-001–AC-023, delivered primarily on the web.
- **Release 1:** FR-001–FR-011.
- **Release 2:** receipts (FR-003) and custom date-range reports (FR-012). R2 is **inside the planned delivery**; removing it needs customer approval (OQ-13).
- **Details:** [requirements.md §7](requirements.md#7-scope-and-phasing).

## 5. Reading order and document index

| # | Document | Content |
|---|---|---|
| 1 | [requirements.md](requirements.md) | Actors; FR, NFR, and AC with sources; business rules; assumptions; measurable NFR methods; suggested quality attributes (not approved); scope and phasing |
| 2 | [decisions.md](decisions.md) | Decision log D-01–D-41, owners, statuses; highlights unresolved customer decisions |
| 3 | [open-questions-and-change-requests.md](open-questions-and-change-requests.md) | OQ-1–OQ-13, owner decisions OD-1–OD-4, blockers, deviations, proposed change requests CR-01–CR-06 (not applied) |
| 4 | [architecture.md](architecture.md) | Context and boundary, platform gate, layers, workflows, security boundaries, financial policy, deployment, quality attributes, trade-offs, risks |
| 5 | [design.md](design.md) | Detailed design: classes and ports, money and allocation algorithm, validation, state machines, balances, payments, transactions and concurrency, audit, authorization matrix, errors, notifications, reports, web and Swing presentation, walk-throughs, tests |
| 6 | [data-model.md](data-model.md) | Entities, constraints, indexes, invariants, lifecycles, derived balances, Spec Kit reconciliation, proposed migrations |
| 7 | [contracts/application-services.md](contracts/application-services.md) | Shared service operations, validation and error codes, authorization, postconditions, transactions |
| 8 | [contracts/web-api.md](contracts/web-api.md) | REST API [COND-A] incl. recovery, invitations, receipts, history; Spec Kit reconciliation |
| 9 | [contracts/ui-workflows.md](contracts/ui-workflows.md) | Browser screens U1–U20, navigation, feedback, accessibility; Swing in a conditional section |
| 10 | [reconciliation-matrix.md](reconciliation-matrix.md) | RM-01–RM-40 comparison and dispositions; F-01–F-11 |
| 11 | [traceability.md](traceability.md) | FR, NFR, and AC ↔ design, contracts, diagrams, tests, release; backward maps; gaps |
| 12 | [verification-plan.md](verification-plan.md) | Unit, application, integration, concurrency, security, contract, E2E, UML, and performance tests; NFR-004 protocol; exit criteria |
| 13 | [quickstart.md](quickstart.md) | What exists vs. what is planned; proposed commands; diagram rendering; walk-throughs S1–S10 |

No separate `security.md` or `deployment.md` was added. Security is covered in architecture §6 and design §9 and §13. Deployment is in architecture §8.

## 6. Diagram index

All sources are in [diagrams/](diagrams/). See §8 for render status.

| Diagram | Type | Requirements | Described in |
|---|---|---|---|
| [use-cases.puml](diagrams/use-cases.puml) | Use case | FR-001–FR-012, NFR-002 | requirements §1; ui-workflows §3; design §9.2 |
| [components.puml](diagrams/components.puml) | Component (web, trust boundaries) | NFR-001, NFR-002 | architecture §2, §4, §6 |
| [components-swing-conditional.puml](diagrams/components-swing-conditional.puml) | Component (conditional Swing) | NFR-001 (not met) | architecture §3.2 |
| [classes-detailed.puml](diagrams/classes-detailed.puml) | Class (domain overview) | FR-003–FR-010, NFR-003 | design §1.1, §2–§7 |
| [classes-application.puml](diagrams/classes-application.puml) | Class (services, ports, repositories) | All FRs, NFR-002, NFR-005 | design §1.2 |
| [domain-model.puml](diagrams/domain-model.puml) | Domain / ER | FR-002–FR-010, NFR-003, NFR-005 | data-model §1–§5 |
| [sequence-expense-review.puml](diagrams/sequence-expense-review.puml) | Sequence | FR-003–FR-008 | design §8.2, §15.1–15.2 |
| [sequence-payment.puml](diagrams/sequence-payment.puml) | Sequence | FR-009, FR-010 | design §7 |
| [sequence-password-reset.puml](diagrams/sequence-password-reset.puml) | Sequence | FR-001 | design §13.2 |
| [states.puml](diagrams/states.puml) | State (expense) | FR-006, FR-007, NFR-005 | data-model §6.1; design §5.1 |
| [states-invitation.puml](diagrams/states-invitation.puml) | State (invitation, membership) | FR-002 | data-model §6.2–6.3 |
| [states-payment.puml](diagrams/states-payment.puml) | State (payment) | FR-010, NFR-005 | data-model §6.5 |
| [activities.puml](diagrams/activities.puml) | Activity (expense submission and review) | FR-003–FR-008 | design §15 |
| [activities-payment.puml](diagrams/activities-payment.puml) | Activity (payment settlement) | FR-009, FR-010 | design §7 |

**How to render** (needs Java and a PlantUML jar, or an IDE PlantUML extension):

```bash
java -jar plantuml.jar -checkonly architecture/final/diagrams/*.puml
java -jar plantuml.jar -tsvg -o out architecture/final/diagrams/*.puml
```

## 7. Unresolved gates

| Gate | Owner | Blocks |
|---|---|---|
| **OQ-1 platform** | Customer | All presentation and deployment work; AC-020 |
| OQ-2 review policy (safe default in place) | Customer | Nothing; CR-04 follows |
| OQ-13 release phasing | Customer | Any removal of receipts or FR-012 from the delivery |
| OQ-3–OQ-12 | Customer | Nothing (defaults in place) |
| OD-1–OD-4 | Project owner | Deployment; reproducible builds (OD-2) |
| Human review of this package | Reviewer | Treating anything as a baseline |

## 8. Validation record

Only checks listed as **executed** were actually run. All were run on 2026-11-05.

| Check | Status | Result |
|---|---|---|
| All 14 required documents and 8 required diagram types present | **Executed** | Present. 14 `.puml` files in total. |
| Every Markdown link and section anchor in `architecture/final/` resolves | **Executed** (script) | 0 broken |
| Every diagram is referenced in prose, and every referenced diagram exists | **Executed** (script) | 0 issues |
| FR-001–FR-012, NFR-001–NFR-005, AC-001–AC-023 present in requirements.md and traceability.md | **Executed** (script) | 12/12, 5/5, 23/23 |
| F-01–F-11 present in reconciliation-matrix.md | **Executed** (script) | 11/11 |
| Every D-, OQ-, and RM- ID referenced is defined | **Executed** (script) | 0 undefined (41 decisions, 13 questions, 40 matrix rows) |
| Error codes used in docs and diagrams are defined in application-services.md | **Executed** (script) | All defined. Remaining non-matches are file names, `Long.MAX_VALUE`, and the success status `INVITATION_RECORDED`. |
| Stale references from copied Claude documents (old module names, `rest-api.md`, `swing-ui.md`, old diagram names, MVP/full-scope wording) | **Executed** (script + fixes) | Fixed. Remaining hits are intentional provenance notes. |
| State-name consistency (no DRAFT, PendingReview, or SETTLED states) | **Executed** (script) | Consistent. Hits are method names or explicit "rejected" notes only. |
| PlantUML syntax check of all 14 diagrams (PlantUML 1.2025.10 jar from the VS Code "Plan UML" extension, run with JDK 25.0.2) | **Executed** | 14/14 pass (`-checkonly` exit 0) |
| Render all 14 diagrams to SVG and PNG | **Executed**; output in the session scratchpad, **not committed** | 14/14 rendered; no "Syntax Error" in output |
| Visual cross-check of rendered `use-cases`, `states`, `states-invitation` against the text | **Executed** (manual) | Consistent. Mismatches found and fixed during review: the `Payment` entity and `isReviewBlocked` were added to design §1.1/§5.2, and `listHistory` was assigned to `GroupService`. |
| Diagram cross-check against data model, contracts, matrix, and state machines for the other 11 diagrams | Partially executed (source-level review while authoring; not every render inspected) | No known inconsistencies |
| Allocation arithmetic in examples (design §3.4, quickstart S3/S5/S6/S7) | **Executed** (throwaway Java script in scratchpad) | Matches. The quickstart S5.3 example was **corrected** to 10.00/10.00/10.00. |
| Originals and authoritative sources unmodified (`git diff HEAD` on UserRequirement.md, specs/, .specify/, architecture/claude-code/, cross-review, instructions) | **Executed** | No changes |
| Application build, unit, integration, concurrency, security, E2E tests | **Not run.** No code exists. | — |
| NFR-004 performance benchmark | **Not run** | — |
| Java 21 / Maven verification | **Not possible here.** JDK 25.0.2 present; Maven absent (OD-2). | — |
| Human review (CONST VII) | **Pending** | — |

## 9. Readiness verdict

**READY FOR HUMAN REVIEW.**

**Not** "ready with approved conditions": no customer decision or human approval is recorded.

**Not** implementation-ready for any platform:

- OQ-1 is unresolved.
- Nothing is approved.
- No code, test, or benchmark exists.

After human review, platform-independent core work (domain, application, persistence, and their tests) may begin (D-03). Presentation work stays **BLOCKED** on OQ-1.
