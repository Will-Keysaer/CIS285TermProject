# Spec Kit Prompt — SplitSmart Architecture and Design

Use this prompt with the repository’s Spec Kit planning workflow (normally `/speckit.plan`). It directs Spec Kit to create its architecture, research, and design artifacts without implementing application code.

# Prompt 1 — Spec Kit: Architecture, Research, and Design

**How to use:** In the repository working directory, invoke the project's Spec Kit planning workflow (normally `/speckit.plan`). Paste the prompt below as the planning instructions/arguments. Let Spec Kit use its own workflow and templates. Do not manually pre-create its expected output files.

```text
You are executing the Spec Kit implementation-planning and design workflow for the SplitSmart CIS 285 term project.

GOAL
Generate a complete, technically coherent architecture and design package based on the repository's authoritative customer requirements, clarified feature specification, and project constitution. Follow the repository's installed Spec Kit planning workflow and templates. Do not implement application code.

AUTHORITATIVE INPUTS — READ THESE FIRST
1. Customer requirements: `UserRequirement.md` (authoritative customer baseline).
2. Clarified feature specification: `specs/001-splitsmart-expense-sharing/spec.md`.
3. Project constitution: `.specify/memory/constitution.md`.
4. The repository's actual Spec Kit planning instructions, templates, and scripts.

Treat these documents according to this precedence:
- `UserRequirement.md` defines what the customer asked for.
- `spec.md` elaborates and clarifies the feature requirements, acceptance criteria, assumptions, and open questions, but does not override the customer baseline.
- `constitution.md` defines project engineering constraints and governance. It does not authorize changing customer requirements.

CRITICAL PLATFORM CONFLICT — DO NOT HIDE OR GUESS
The customer explicitly requires a primarily web-based product (`NFR-001` and `UserRequirement.md`). The constitution says Java Swing is only an implementation decision and a Swing-only desktop application does not satisfy that requirement. The specification's Open Question 1 asks whether the customer approves a Swing desktop MVP or requires the MVP itself to be web-based.

Do not silently choose Swing as the primary product, reinterpret “web-based,” or claim that a Swing-only implementation satisfies NFR-001. Do not assume customer approval exists. In the architecture, explicitly:
- Identify this as an unresolved product/platform decision and a potential planning gate.
- Present at least two viable alternatives: (A) a web-based product architecture that satisfies the customer baseline, and (B) a Swing desktop MVP only if documented customer approval changes the delivery scope/platform.
- Explain the consequences, trade-offs, technical implications, and requirement coverage of each alternative.
- Mark the decision as `NEEDS CUSTOMER DECISION` wherever appropriate.
- State what approval or requirement change is needed before implementation proceeds under the Swing alternative.
- Avoid designing an incoherent hybrid merely to make both choices appear resolved.

CONSTITUTIONAL AND TECHNICAL REQUIREMENTS
- Target Java 21 for Java application code and use Maven.
- Use JUnit 5 for automated tests and Maven verification.
- Keep GUI/presentation, application/business logic, and persistence/infrastructure concerns separated. Business rules and balance calculations must not live in Swing event handlers or view classes.
- Use `BigDecimal` for every monetary value and calculation. Never use `float` or `double` for money. Specify currency scale, rounding, and persistence boundaries.
- For percentage allocation, percentages must be nonnegative and total exactly 100%. Apply the largest-remainder method to distribute fractional cents, ensuring shares reconcile exactly to the expense total.
- Equal splits must also reconcile exactly to the smallest supported currency unit.
- Pending expenses must not affect balances. Approved expenses affect balances; rejected expenses do not.
- A payment recorded by its sender affects the balance immediately, as clarified in the specification. Preserve payment history and identify who recorded it and when.
- Respect group membership, manager permissions, invite acceptance, user privacy, and authorization boundaries.
- The payer is selected separately from participants, must belong to the group, and may be excluded from the expense allocation.
- Use PlantUML for architecture/design diagrams when a diagram communicates structure or important interactions. Keep diagrams consistent with the written design.
- AI-generated artifacts require human review; do not describe AI output as approved or verified merely because it was generated.

REQUIREMENT COVERAGE
Preserve all customer feature areas, including:
- Individual accounts, sign-in, and account recovery.
- Group creation and management, invitations, acceptance, and membership permissions.
- Expense creation with amount, date, description, payer, participants, equal/percentage allocation, and optional receipt image.
- Notifications when group expenses are submitted.
- Expense review/approval before balances change.
- Pairwise balances.
- Payment recording and balance adjustment.
- Expense, balance, and payment summaries and date-range/custom reports.
- Primary web-based delivery.

The specification identifies a proposed narrower Java Swing MVP subset and deferred items. Treat that as a proposal, not as permission to delete customer requirements. Distinguish clearly between:
1. Full customer product scope.
2. Candidate MVP scope.
3. Deferred scope.
4. Items blocked on customer clarification/approval.

WORKFLOW
1. Follow the installed Spec Kit planning skill exactly, including its pre-plan checks, setup script, templates, and any registered hooks. Do not bypass required workflow steps.
2. Read the entire clarified specification and constitution before making architecture decisions.
3. Identify unknowns, dependencies, integrations, and assumptions. Research relevant technical choices where the workflow supports it. Do not fabricate research or claim a source was consulted if it was not.
4. For unresolved customer questions, preserve them as explicit decisions or `NEEDS CLARIFICATION`; do not invent answers.
5. Generate the artifacts required by the installed planning workflow. In addition to the standard Spec Kit outputs, create a **standalone `architecture.md`** in the same feature-planning directory as `plan.md` (normally `specs/001-splitsmart-expense-sharing/`). This is an explicit project deliverable; do not assume that architecture decisions scattered across `plan.md` and other files are an adequate substitute. Do not replace or rename any standard Spec Kit artifact. At minimum, the expected design package should include:
   - `architecture.md` (**required standalone architecture proposal**): a cohesive, developer-facing description of the proposed system architecture. Include the platform alternatives and unresolved customer decision; system context and major components; layer/component responsibilities and dependencies; important runtime interactions and data flows; technology choices with rationale and alternatives; security, authorization, privacy, and persistence boundaries; deployment/runtime model where known; key architectural decisions, trade-offs, risks, and open questions. Clearly distinguish confirmed constraints, recommendations, conditional choices, and decisions requiring customer approval. Keep it consistent with `plan.md`, `research.md`, contracts, and diagrams.
   - `plan.md` (technical context, architecture approach, constitution checks, key decisions, and implementation plan)
   - `research.md` (decisions, rationale, alternatives, and sources where research was performed)
   - `data-model.md` (entities, fields, relationships, constraints, validation, and important state transitions)
   - `contracts/` (appropriate interface/API/UI contracts for the selected platform; if the platform decision is unresolved, document alternatives or clearly mark the contract as conditional rather than pretending one is approved)
   - `quickstart.md` (prerequisites, setup, build/run/test commands, and repeatable end-to-end validation scenarios)
   - PlantUML diagram source files when needed to communicate architecture, component boundaries, domain relationships, or important sequences.
6. Keep the artifacts consistent with the actual repository and installed templates. Do not invent existing source code, frameworks, infrastructure, databases, APIs, or integrations.
7. Trace architecture and design decisions to requirement IDs (FR-001–FR-012, NFR-001–NFR-005, AC-001–AC-023) wherever applicable. Include a traceability matrix or equivalent mapping.
8. Explain which decisions are confirmed by the source documents, which are engineering recommendations, and which require customer approval.
9. Run only the validation checks appropriate to generated documentation/design artifacts. Do not claim that the application builds or tests pass if no application implementation exists.
10. Do not implement application code, alter customer requirements, amend the constitution, or commit changes.

QUALITY EXPECTATIONS
- Be specific enough that a developer could use the artifacts to implement the design.
- Make `architecture.md` a readable standalone architecture proposal, not merely a summary or copy of `plan.md`; a reviewer should be able to understand and evaluate the proposed architecture from it.
- Explain responsibilities and boundaries for the major components/layers.
- Define domain concepts and invariants, especially expense allocation, approvals, balances, and payments.
- Describe data ownership, authorization, and privacy boundaries.
- Describe error handling and validation at the relevant boundaries.
- Identify persistence needs and consistency/transaction requirements without selecting unsupported infrastructure as fact.
- Include test strategy and requirement-to-test traceability.
- Identify risks, trade-offs, and unresolved questions.
- Avoid generic boilerplate and unsupported claims.
- Do not expand the MVP beyond what is justified by the specification.

FINAL REPORT
When finished, report:
- The branch and exact paths of generated artifacts.
- The architecture direction(s) considered and the unresolved platform decision.
- Any constitution check failures or unresolved gates.
- Any assumptions or questions requiring customer input.
- What validation was actually performed and its result.
- Confirmation that no implementation code was written and no commit was made.
```