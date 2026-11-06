# SplitSmart: Decision Log (Reconciled Candidate)

Status: **Reconciled candidate. Pending human and customer review.** **No decision in this log is APPROVED.** Agreement between the Claude Code and Spec Kit proposals is **not** approval (RECONCILIATION_INSTRUCTIONS §1). Dates are given only where known. Every entry was drafted on 2026-11-05 and none has been decided.

**Status values:**

| Status | Meaning |
|---|---|
| `PROPOSED` | Engineering decision the team can adopt after human review (Constitution VII) |
| `NEEDS CUSTOMER DECISION` | The customer owns this choice. A safe default is given so design can continue. |
| `NEEDS OWNER DECISION` | The project owner or instructor owns this choice |
| `BLOCKED` | Downstream work cannot start until it is decided |
| `DEFERRED – APPROVAL REQUIRED` | Proposed to postpone work. The postponement itself needs approval. |

**Source abbreviations:**

| Abbreviation | Source |
|---|---|
| UR | `UserRequirement.md` |
| SPEC | `specs/001-splitsmart-expense-sharing/spec.md` |
| CONST | `.specify/memory/constitution.md` |
| XR | `architecture/CROSS_REVIEW_REPORT.md` |
| CC | `architecture/claude-code/` (R-xx = its research.md, DD-x = its design.md §17) |
| SK | `specs/001-splitsmart-expense-sharing/` (plan, research, architecture, data-model, contracts) |

Customer questions (OQ-n) and change requests are detailed in [open-questions-and-change-requests.md](open-questions-and-change-requests.md). The item-by-item comparison is in [reconciliation-matrix.md](reconciliation-matrix.md).

## 1. Unresolved customer decisions (highlighted)

| Decision | Topic | Question | Blocks |
|---|---|---|---|
| **D-01** | **Delivery platform** | **OQ-1: web (baseline) or Swing MVP?** | **All presentation and deployment work** |
| D-13 | Currencies | OQ-4 | Nothing (default USD) |
| D-14 | Percentage precision | OQ-10 | Nothing (default 2 dp) |
| D-15 | Amount ceiling | OQ-11 | Nothing (default 1,000,000) |
| D-21 / D-22 | Review policy and approval levels | OQ-2 | Nothing (safe default) |
| D-23 | Notification channels | OQ-3 | Nothing (in-app) |
| D-24 | Receipt limits and retention | OQ-12 | Nothing (default limits) |
| D-25 | Overpayment handling | OQ-7 | Nothing (reject) |
| D-26 | Removing members with balances | OQ-8 | Nothing (block) |
| D-30 | Editing and deleting expenses | OQ-6 | Nothing (pending-only edit) |
| D-33 | Release phasing / any deferral | OQ-13 | Any cut of FR-003 receipts or FR-012 from the delivered product |
| D-37 | Account deletion and retention | OQ-9 | Nothing (out of scope) |
| D-38 | Report export and filters | OQ-5 | Nothing (no export) |

## 2. Decision log

| ID | Topic | Source(s) | Options | Recommendation | Rationale | Owner / approval authority | Status | Impact | Affected artifacts |
|---|---|---|---|---|---|---|---|---|---|
| D-01 | Delivery platform | UR "primary web-based"; SPEC NFR-001, AC-020, OQ-1; CONST VIII; CC R-01; SK plan/research §1; XR F-01 | (A) Web. (B) Swing desktop MVP, only with documented approval. (Hybrid rejected.) | **A, web** | Only A satisfies NFR-001 and AC-020. Swing-only "does not satisfy this requirement" (SPEC NFR-001). | **Customer** | **NEEDS CUSTOMER DECISION; BLOCKED** for presentation and deployment | Decides the presentation layer, deployment, and the AC-020 outcome | All; architecture §3; web-api; ui-workflows |
| D-02 | Form of the Swing alternative, if approved | CC R-02 | Standalone with embedded DB; Swing client plus server; Swing on a shared remote DB | Standalone | A client/server Swing app needs most of A's server and still fails NFR-001. A shared remote DB puts credentials on every client. | Team (only if D-01 = B) | PROPOSED (conditional) | Limits multi-device use (single installation) | architecture §3.2; ui-workflows §6 |
| D-03 | Build order | CC R-03; XR §9 item 6 | Core first; wait for D-01 | Build the platform-independent core first, after human review | The core is identical under A and B, and CONST III requires the separation anyway | Team | PROPOSED | Lets work start before D-01 | quickstart; verification-plan |
| D-04 | Build structure | CC §3.1 (5 modules); SK plan "Project Structure" (one `src/`); XR §4 "may be over-specified" | 5 Maven modules; **1 module with layered packages + ArchUnit** | One module | Less build complexity for a course project. Layer rules stay enforced by tests (CONST III). | Team | PROPOSED | Simpler POM. Can be split into modules later. | design §1; architecture §4 |
| D-05 | Web framework (A) | CC R-04; SK plan Technical Context | Spring Boot 3.x; Jakarta EE; Quarkus/Micronaut | Spring Boot 3.x (pin the exact version when implementation starts) | Both proposals agree. Integrated security, transactions, and MockMvc. | Team | PROPOSED (COND-A) | Dependencies | architecture §4; quickstart |
| D-06 | Browser client technology (A) | CC R-04 (TypeScript SPA via frontend-maven-plugin; Thymeleaf fallback); SK ui-contracts (framework-independent) | TypeScript SPA; **static HTML/CSS/JS modules, no Node build**; Thymeleaf | Static HTML/CSS/JS | Keeps the REST contract and avoids Node tooling. Thymeleaf is a valid alternative if the team prefers all-Java. | Team (owner may override, OD-3) | PROPOSED (COND-A) | Front-end effort; keeps `mvn verify` self-contained | design §13; ui-workflows |
| D-07 | Database | CC R-05; SK plan Storage | PostgreSQL; MySQL; H2 only | PostgreSQL (A production), H2 (tests and B) | Both proposals agree. Relational integrity, `NUMERIC`, row locks. | Team | PROPOSED | Concurrency tests must run on the selected DB (D-19) | data-model; verification-plan |
| D-08 | Persistence technology | CC R-06 | JPA + Flyway; JDBC/jOOQ | Spring Data JPA + Flyway; explicit SQL for balance and report aggregates | Optimistic versions and versioned migrations | Team | PROPOSED | | design §1.3; data-model §9 |
| D-09 | Authentication | SPEC clarification (email + password); CC R-07; SK web-api login | Session cookie + CSRF; JWT bearer tokens | BCrypt; server session (`HttpOnly; Secure; SameSite=Lax`); CSRF double-submit (A) | Email + password is a clarified requirement. A session suits a same-origin client. | Team (mechanism); SPEC fixes email + password | PROPOSED | | design §13; web-api §1 |
| D-10 | Account recovery | SPEC FR-001, AC-001; XR **F-02**; CC DD-10 | A: emailed single-use token. B: one-time recovery codes. Security questions rejected. | A: 256-bit token, SHA-256 hash stored, 30 min expiry, single use, uniform responses, sessions invalidated, token in URL fragment. B: 8 codes. | Fills the Spec Kit gap. Avoids account enumeration. | Team | PROPOSED | SMTP dependency (A) | design §13.2; web-api §3.1; sequence-password-reset.puml |
| D-11 | Email verification at registration | CC architecture §11 D-4 | Verify first; no verification | Not in Release 1; revisit (OD-4) | Not required by SPEC. It would reduce enumeration and recovery-to-wrong-address risk. | Project owner | NEEDS OWNER DECISION | Registration flow | open-questions OD-4 |
| D-12 | Money representation and parsing | CONST IV; SPEC FR-003; CC R-08, DD-3, DD-4; SK research §3 | `BigDecimal` value object; long minor units throughout; JSR-354 | `Money` (`BigDecimal` at currency scale); plain-decimal string input; reject excess precision; `RoundingMode.UNNECESSARY` only; `NUMERIC(19,4)` | The constitution requires `BigDecimal`. Rejecting rather than rounding input keeps values exactly as entered. | Team | PROPOSED | | design §2; data-model §3 |
| D-13 | Supported currencies and scales | SPEC Assumptions, OQ-4; CC M2; XR **F-06** | Any ISO-4217; scales {0,2,3}; USD only | One currency per group, fixed after the first expense; scales {0,2,3}; default USD | Covers common currencies without exchange rates | **Customer** | **NEEDS CUSTOMER DECISION** (default in place) | Validation; reports per currency | design §2; data-model §1.4 |
| D-14 | Percentage precision | SPEC FR-005 (no precision given); CC AS-2; XR **F-06** | 2 dp; 4 dp; whole numbers | 2 dp (basis points) | Familiar to users. Integer arithmetic. The algorithm generalizes. | **Customer** | **NEEDS CUSTOMER DECISION** | Input validation only | design §2.2 |
| D-15 | Maximum amount | CC AS-10; XR **F-06** | None; 1,000,000; customer value | 1,000,000 major units (configurable constant) | Catches typos and bounds arithmetic | **Customer** | **NEEDS CUSTOMER DECISION** | Validation only | design §2.2 |
| D-16 | Allocation algorithm | SPEC FR-005, NFR-003, AC-021; CC R-09, DD-1; SK research §8 | Largest remainder on integer minor units; `BigDecimal` division with rounding | Integer basis points / minor units with largest remainder; Σ asserted | The spec requires largest remainder. Integer remainders are exact. | Team (method fixed by SPEC) | PROPOSED | | design §3 |
| D-17 | Tie-break for equal remainders | SPEC silent; CC DD-2; SK `allocation_order` | Account-ID string order; stored allocation order; payer first | Ascending lower-case UUID string | Deterministic and reproducible in Java and SQL without extra state | Team | PROPOSED | Test fixtures need fixed UUIDs | design §3.3 |
| D-18 | Balance storage | CC R-10; SK research "Persistence"; XR §6 | Derived; materialized; both | Derived on read; materialized table only as a measured fallback | Both proposals agree. One source of truth (NFR-003). | Team | PROPOSED | NFR-004 must be measured | design §6; data-model §1.13 |
| D-19 | Concurrency control | CC R-11, DD-5, DD-6; XR **F-07** | Group row lock + conditional updates (READ COMMITTED); SERIALIZABLE; per-pair locks | Group row lock for approvals, payments, and membership changes; version-checked conditional updates; group-first lock order | Prevents double approval, approve/reject races, over-settlement, and stale approval | Team | PROPOSED, **UNVERIFIED** until tests pass on the selected DB | Concurrency integration tests | design §8 |
| D-20 | Idempotency and duplicate payments | CC DD-7; XR F-07; SPEC edge case "recorded twice" | Idempotency key + request hash; possible-duplicate confirmation; none | Both | Safe retries plus a guard against honest mistakes | Team | PROPOSED | | design §7, §8.4 |
| D-21 | Review policy | SPEC OQ-2, Assumptions (single review step); CC DD-12; XR **F-04** | Managers only; any other member; all participants | Default: ACTIVE managers approve; no self-approval unless the group enables `selfApprovalAllowed`; `reviewBlocked` flag when the submitter is the only manager | Review must be possible in every group. Managers can designate another manager or enable self-approval. | **Customer** | **NEEDS CUSTOMER DECISION** (safe default) | ReviewPolicy only | design §5.2; states.puml |
| D-22 | Approval levels | UR "multiple levels of approval or a single review process"; SPEC Assumptions | Single; multi-level | Single level; `ReviewDecision.level` reserved | The spec assumes a single step for the initial workflow | **Customer** | NEEDS CUSTOMER DECISION (OQ-2) | Data model ready | data-model §1.10 |
| D-23 | Notifications | SPEC FR-008, Assumptions, OQ-3; CC R-13; SK research | In-app; email; push | In-app inbox written in the same transaction; `NotificationPublisher` hook for later channels | Spec assumption | **Customer** (channels) | NEEDS CUSTOMER DECISION (OQ-3) | | design §11 |
| D-24 | Receipt images | UR "optional receipt image"; SPEC FR-003, AC-006; CC R-14, DD-11; SK plan Assumptions; XR **F-05** | Filesystem; DB blob; object storage | `ReceiptStorage` port with private filesystem; JPEG/PNG; magic bytes; ≤ 5 MB; re-encode; authorized download | Secure by construction. The port allows object storage. | Team (mechanism); **Customer** (limits, retention) | PROPOSED; limits and retention NEEDS CUSTOMER DECISION (OQ-12) | Release 2 | design §13.3; web-api §3.8 |
| D-25 | Overpayment | SPEC AC-015; CC R-15; XR F-07 | Reject; record as credit; silently cap | Reject with `PAYMENT_EXCEEDS_OUTSTANDING` | Simplest reading that satisfies AC-015 | **Customer** (confirm) | NEEDS CUSTOMER DECISION (OQ-7; default in place) | | design §7 |
| D-26 | Member removal with a balance | SPEC edge case "member leaves… unsettled balances"; CC R-16 | Block; allow with read-only carry-over | Block while N ≠ 0 or pending involvement | Prevents orphaned debts | **Customer** | NEEDS CUSTOMER DECISION (OQ-8) | | design §5.5 |
| D-27 | Former-member visibility | SPEC FR-001 ("own… information"), NFR-002, AC-004 | No access; read-only party records; full group | Read-only access to records they were party to | Keeps one's own history (FR-001) without exposing the rest of the group | Team | PROPOSED | Authorization matrix | design §9 |
| D-28 | Error format (A) | CC R-17; SK web-api envelope | RFC 9457; custom envelope | RFC 9457 + stable `code` | Standard and machine-readable | Team | PROPOSED | | web-api §2 |
| D-29 | Identifiers | CC R-19 | UUID; sequential | UUID | Not guessable. Client-generated idempotency keys. | Team | PROPOSED | | data-model |
| D-30 | Editing expenses | SPEC OQ-6, edge case "edited while pending" | Pending editable by submitter; approved immutable; deletion | Submitter may edit while PENDING; approved and rejected are immutable; no deletion | Least risky default. Audit keeps the history (D-31). | **Customer** | NEEDS CUSTOMER DECISION (OQ-6) | | design §5.1 |
| D-31 | Audit log | SK plan "audit records"; SPEC NFR-005, AC-004, OQ-6 | Append-only audit table; none | Append-only `audit_event` in the same transaction | Reviewability and history of edits | Team | PROPOSED (new in reconciliation) | One table, one test | design §8.5; data-model §1.15 |
| D-32 | Invitation model | SPEC FR-002, AC-003; XR **F-03**; SK `GroupMembership.invite_status` | Separate Invitation entity; membership state | Separate `Invitation` with ID, invitee, expiry; invitee-only acceptance | Fixes the Spec Kit identity gap | Team | PROPOSED | | data-model §1.6; states-invitation.puml |
| D-33 | Release phasing | SPEC "Proposed Java Swing MVP Subset"; XR **F-09**; RECONCILIATION_INSTRUCTIONS §2 | R1 core + R2 receipts/reports inside the delivered product; cut R2 | R1 = FR-001–FR-011 (single review, in-app, summaries); R2 = FR-003 receipts + FR-012. **Both are in the planned delivery.** | Full scope is preserved. Phasing is not deletion. | **Customer** (any cut) | **DEFERRED – APPROVAL REQUIRED** for any removal from the delivered product (OQ-13) | Schedule | requirements §7; traceability |
| D-34 | Performance verification | SPEC NFR-004, AC-022, Assumptions; XR **F-08** | Defined benchmark protocol; none | Protocol in verification-plan §6 | No measurement exists. A protocol makes the claim testable. | Team (protocol); Customer (only if the target changes) | PROPOSED, **UNVERIFIED** | Perf harness | verification-plan §6 |
| D-35 | Test tooling | CONST V; CC R-20 | JUnit 5 (required) + ArchUnit, MockMvc, Testcontainers, jqwik, Playwright | JUnit 5 required; others optional and justified per test | Constitution | Team | PROPOSED | | verification-plan |
| D-36 | Hosting (A) | CC D-5 | Course VM; PaaS; local demo | Undecided | No impact on the architecture | Project owner | NEEDS OWNER DECISION (OD-1) | Deployment only | architecture §8 |
| D-37 | Account deletion and data retention | SPEC silent; CC D-3 | Out of scope; anonymize | Out of scope | History must stay attributable (AC-004) | **Customer** | NEEDS CUSTOMER DECISION (OQ-9) | | open-questions |
| D-38 | Report semantics | SPEC FR-011/012, AC-017–019, OQ-5 | — | Inclusive business-date range; ≤ 5 years; filters: group, record types, expense statuses, only-mine; per-currency totals; no export | Matches AC-018/019 | Team; **Customer** (export and filters) | PROPOSED; export NEEDS CUSTOMER DECISION (OQ-5) | | design §12 |
| D-39 | Invitation expiry and unregistered invitees | CC DD-8, DD-9 | 14 days, lazy; scheduler; never expire | 14 days, evaluated lazily; no auto-attach on later registration | No background jobs; no silent binding | Team | PROPOSED | | design §5.3 |
| D-40 | Password policy | CC (≥ 12 chars); SK ui-contracts (length + complexity) | Length-based; complexity rules | 12–128 characters, no composition rules | Proposed engineering choice. Simpler for users. | Team | PROPOSED | | design §4.3 |
| D-41 | Rate limiting and enumeration posture | CC §5.3; SK ui-contracts (generic login failure) | — | Generic login error; uniform recovery and invitation responses; registration reveals "email unavailable" (documented trade-off); 5 failures / 15 min | Both proposals agree on generic errors | Team | PROPOSED | | design §13.1 |

## 3. Project-owner decisions (not customer)

| ID | Topic | Default |
|---|---|---|
| OD-1 | Hosting environment for A (D-36) | Local demo until decided |
| OD-2 | JDK: install JDK 21, or allow JDK 25 with `--release 21` (F-11) | Install JDK 21 for verification |
| OD-3 | Override of D-06 (front-end technology) | Static HTML/JS |
| OD-4 | Email verification at registration (D-11) | Not in Release 1 |
