# SplitSmart: Reconciliation Matrix

Status: **Reconciled candidate. Pending human review.** Item-by-item record comparing **Claude Code** (`architecture/claude-code/`), **Spec Kit** (`specs/001-splitsmart-expense-sharing/`), and the **source requirements** (UR, SPEC, CONST), with the final disposition.

- **Dispositions:** **Adopt** (taken as is), **Modify** (taken with changes), **Reject**, **Defer** (postponed, approval required), **Open** (needs a customer or owner decision).
- **Starting point:** Claude's design is the working basis, per RECONCILIATION_INSTRUCTIONS §2. "Adopt (Claude)" means Claude's version was kept after comparison, not by default.

## 1. Item-by-item matrix

| RM | Item | Claude Code | Spec Kit | Source requirement | Disposition | Evidence / reason | Decision · Test |
|---|---|---|---|---|---|---|---|
| RM-01 | Platform | A web recommended / B Swing conditional (`architecture.md` §2) | Same (`plan.md`, `architecture.md` "Unresolved Product Decision") | UR "primary web-based"; NFR-001; AC-020; CONST VIII; SPEC OQ-1 | **Open** (web documented as primary) | Both agree; agreement is not approval | D-01 · VP-E2E-10 |
| RM-02 | Swing form | Standalone + embedded DB; client/server rejected | Not specified ("same domain model/services") | CONST VIII | Adopt (Claude), conditional | Honest about limits | D-02 |
| RM-03 | Build order | Core first | Phase 1 domain, then services, then UI (`plan.md`) | CONST III | Adopt (both agree) | | D-03 |
| RM-04 | Build structure | 5 Maven modules | One `src/` with packages | CONST II, III | **Modify:** one module + ArchUnit | XR §4: Claude "may be over-specified for a term project" | D-04 · VP-ARCH |
| RM-05 | Browser client | TypeScript SPA via frontend-maven-plugin; Thymeleaf fallback | Framework-independent "browser-based pages and API-driven interactions" | NFR-001 | **Modify:** static HTML/CSS/JS, no Node build | Avoids an extra toolchain while keeping the REST contract | D-06, OD-3 |
| RM-06 | Web framework | Spring Boot 3.x | Spring Boot 3.x recommended | none (engineering) | Adopt (both agree) | Label: engineering decision | D-05 |
| RM-07 | Database | PostgreSQL / H2 | PostgreSQL / H2 | none | Adopt (both agree) | | D-07 · VP-I01 |
| RM-08 | Persistence | JPA + Flyway; explicit aggregate SQL | Repositories/DAO | none | Adopt (Claude) | More concrete | D-08 |
| RM-09 | Authentication | Session + CSRF; BCrypt | "session token or authenticated user context" | SPEC clarification: email + password | Adopt (Claude) | Same-origin session is simplest and safe | D-09 · VP-S02 |
| RM-10 | Account recovery | Reset-token API and flow | UI "Recover account" only; **no API** | FR-001, AC-001 | **Adopt (Claude)**; fills the SK gap | XR **F-02** | D-10 · VP-A02, sequence-password-reset.puml |
| RM-11 | Money representation | `Money` value object, string input, `UNNECESSARY` | `BigDecimal`, strings in JSON | CONST IV; FR-003 | Adopt (Claude) + SK JSON strings | Both agree on `BigDecimal` and strings | D-12 · VP-U01–U02 |
| RM-12 | Currency scales | {0,2,3}, default USD | `currency_scale` stored per group; client-supplied | SPEC Assumptions; OQ-4 | **Open** (default {0,2,3}); SK stored/client scale **rejected** | XR **F-06**; avoid two sources of truth | D-13 · OQ-4 |
| RM-13 | Percentage precision | ≤ 2 dp (basis points) | Not specified | FR-005 (unspecified) | **Open** (default 2 dp) | XR **F-06** | D-14 · OQ-10 |
| RM-14 | Amount ceiling | 1,000,000 | none | none | **Open** (default 1,000,000) | XR **F-06** | D-15 · OQ-11 |
| RM-15 | Allocation algorithm and tie-break | Integer largest remainder; UUID-string tie-break | Largest remainder; `allocation_order` column | FR-005, NFR-003, AC-021 | Adopt (Claude); SK `allocation_order` **rejected** | Deterministic without extra state | D-16, D-17 · VP-U04–U05 |
| RM-16 | Balance storage | Derived; materialized fallback | "derived view or query", `BalanceView` with ID/timestamp | FR-009, NFR-003 | Adopt (both agree on derived); SK IDs dropped | One source of truth | D-18 · VP-I02 |
| RM-17 | Overpayment | Reject | "not negative solely due to overpayment… beyond agreed semantics" | AC-015 | **Open** (default reject) | XR **F-07** | D-25 · VP-U10, VP-C02, OQ-7 |
| RM-18 | Concurrency and transactions | Group row lock, conditional updates, READ COMMITTED | "transactions around approval/payment changes" | FR-007, FR-010 | **Modify:** Claude design, plus correctness must be shown on the selected DB | XR **F-07** | D-19 · VP-C01–C04 |
| RM-19 | Idempotency and duplicates | Key + request hash; 24 h duplicate confirmation | none | SPEC edge case "recorded twice" | Adopt (Claude) | | D-20 · VP-C04, S7 |
| RM-20 | Audit history | Review decisions and immutable payments only | "audit records" repository (`plan.md`) | NFR-005; AC-004; OQ-6 | **Adopt (SK concept), modified:** append-only `audit_event` | Keeps edit history while OQ-6 is open | D-31 · VP-A09 |
| RM-21 | Invitations | Separate `Invitation` with ID; accept/decline by invitee | `GroupMembership.invite_status`; accept via `/members/{accountId}/accept` | FR-002, AC-003 | **Adopt (Claude)**; SK model **rejected** | XR **F-03** | D-32, D-39 · VP-U12, S2 |
| RM-22 | Review policy | Managers; no self-approval by default; `reviewBlocked` | "authorized approver" (unspecified) | FR-007; OQ-2 | **Open** (Claude default as safe conditional design) | XR **F-04** | D-21, D-22 · VP-U08, OQ-2 |
| RM-23 | Receipts | Secure upload pipeline; full scope, deferred in MVP | `receipt_reference`; storage and retention to be decided | FR-003, AC-006 | **Modify:** Claude pipeline; planned in Release 2; retention **Open** (SK question adopted) | XR **F-05** | D-24 · VP-S03, OQ-12 |
| RM-24 | Scope and phasing | Candidate MVP defers receipts and FR-012 | "future MVP slice" | Full UR scope; SPEC candidate MVP | **Modify:** R1/R2 both inside the delivery; any cut = **Defer (approval required)** | XR **F-09** | D-33 · OQ-13 |
| RM-25 | Notifications | In-app, same transaction; recipient rules | In-app; status pending/delivered/read/dismissed | FR-008, AC-013; OQ-3 | Adopt (Claude); SK delivery statuses dropped | No delivery channel in scope | D-23 · VP-A04 |
| RM-26 | Reports | Summary + custom; inclusive business dates; `/me/reports` | Group-scoped `/reports/summary`, `/reports/custom?accountId=`; persisted `report_requests` | FR-011, FR-012, AC-017–019 | Adopt (Claude); SK persisted requests and free `accountId` **rejected** | Privacy; nothing requires saving reports | D-38 · VP-A07, VP-I03 |
| RM-27 | Error format | RFC 9457 + codes | `{success, error{code,…}}` envelope | none | Adopt (Claude) | Standard | D-28 · VP-K01 |
| RM-28 | API shape | `/api/v1`, invitation-ID endpoints, approval/rejection with version | Unversioned; `approverAccountId` in the body | NFR-002 | Adopt (Claude); SK actor-in-body **rejected** | The actor must come from the session | web-api §5 |
| RM-29 | Expense states | PENDING / APPROVED / REJECTED | Draft → Submitted → PendingReview → Approved / Rejected | FR-006, FR-007, NFR-005 | Adopt (Claude) | SK calls Draft an implementation detail and treats Submitted ≡ PendingReview | states.puml |
| RM-30 | Former-member visibility | Read-only party records | "group-scoped… restricted to authorized members" | FR-001, NFR-002, AC-004 | Adopt (Claude) | Keeps one's own history | D-27 · VP-A08 |
| RM-31 | Payer vs. participants | Separate; may or may not participate | Separate; "may be excluded" | FR-003 (clarified) | Adopt (both agree) | Prompt ambiguity resolved by SPEC (UD-5) | VP-U06, S6 |
| RM-32 | Member removal | Blocked with balance or pending involvement | Not specified | SPEC edge case | **Open** (default block) | | D-26 · VP-U11, OQ-8 |
| RM-33 | Performance | Indexes, paging, harness; unmeasured | Target stated; query strategy | NFR-004, AC-022 | **Modify:** add a defined benchmark protocol | XR **F-08** | D-34 · verification-plan §6 |
| RM-34 | Reproducibility | JDK 25 observed; commands labeled proposed | `mvn clean verify` etc., labeled planning | CONST II, V | **Modify:** commands labeled PROPOSED; OD-2 | XR **F-11** | OD-2 |
| RM-35 | Organization | `architecture/claude-code/` | `specs/001-…/` | none | Adopt: originals left in place; final under `architecture/final/` | XR **F-10** | README |
| RM-36 | UI contract and accessibility | Swing presenter contract only | Browser screens, dashboard, accessible errors, generic login error | NFR-001, NFR-002 | **Modify:** SK screens + Claude error mapping; a11y as a suggestion (QA-S1) | Complementary | contracts/ui-workflows.md |
| RM-37 | Expense edit policy | Submitter edits pending; approved immutable | "Edited while pending" edge case only | SPEC OQ-6 | **Open** (default) | | D-30 · OQ-6 |
| RM-38 | Password policy | 12–128 characters | Length + complexity | none | Adopt (Claude) as engineering decision | Simpler | D-40 |
| RM-39 | Identifiers | UUID | UUID or surrogate | none | Adopt (Claude) | | D-29 |
| RM-40 | Account status / deletion | Lockout fields; deletion out of scope | `status` incl. "deleted" | none | **Open** (deletion OQ-9); lockout adopted | | D-37 |

## 2. Cross-review findings F-01 – F-11

| Finding | Severity | Disposition | How it is addressed | Decision / evidence | Status |
|---|---|---|---|---|---|
| **F-01** platform | Critical / gate | **Open**, with web as the documented primary path | architecture §3; requirements §7; Swing only in [COND-B] sections; NFR-001/AC-020 marked ❌ under B in traceability | D-01, OQ-1, CR-06 (conditional); VP-E2E-10 | **BLOCKED** pending the customer |
| **F-02** recovery | High | **Resolved in design** | Request and completion endpoints; 30 min single-use hashed token; uniform responses; session invalidation; recovery codes for B | D-10; web-api §3.1; design §13.2; sequence-password-reset.puml; VP-A02, VP-S02; S1 | PROPOSED, UNVERIFIED |
| **F-03** invitations | High | **Resolved in design** | Separate Invitation entity with ID; invitee-only accept/decline; lazy 14-day expiry; double-accept prevented by version; role changes; removal guards; history attribution | D-32, D-39, D-26; data-model §1.6; states-invitation.puml; VP-U12, VP-A03; S2, S9 | PROPOSED, UNVERIFIED |
| **F-04** review | High | **Open** (customer), with a safe default | Managers decide; no self-approval unless enabled; `reviewBlocked`; single level with `level` reserved; version-checked race handling | D-21, D-22, OQ-2, CR-04; VP-U08, VP-C01; S5 | NEEDS CUSTOMER DECISION |
| **F-05** receipts | Medium | **Resolved in design**; limits and retention **Open** | Upload/download endpoints, magic bytes, size and pixel limits, re-encoding, random keys, authorized download, retention rule | D-24, OQ-12; web-api §3.8; design §13.3; VP-S03; S8 | PROPOSED; retention NEEDS CUSTOMER DECISION |
| **F-06** money limits | Medium | **Relabeled**; limits **Open** | `BigDecimal`, scales, 2 dp, ceiling, and tie-break separated into constitution / spec / engineering / customer-decision items | design §2.0; D-12–D-17; OQ-4, OQ-10, OQ-11; VP-U01–U05 | Limits NEEDS CUSTOMER DECISION; algorithm PROPOSED |
| **F-07** payments and concurrency | Medium | **Resolved in design; verification required** | Sender-only; immediate derived balance; overpayment rejected (OQ-7); idempotency and duplicate confirmation; group lock; READ COMMITTED; approve/pay/edit races; tests on the selected DB | D-19, D-20, D-25; design §7–8; VP-C01–C04 | PROPOSED, **UNVERIFIED** |
| **F-08** performance | Medium | **Protocol defined; not measured** | Dataset, environment, load, views, timing boundary, pass rule, evidence | D-34; verification-plan §6; CR-03 | **UNVERIFIED** |
| **F-09** scope | Medium | **Resolved**: full scope retained | Receipts and FR-012 planned in Release 2 within the delivery; any cut needs approval | D-33, OQ-13; requirements §7; traceability §4 | DEFERRED – APPROVAL REQUIRED (for any cut only) |
| **F-10** organization | Low | **Resolved** | Originals untouched; final package only under `architecture/final/` | README §1; git status check (README §8) | Done |
| **F-11** reproducibility | Low | **Resolved in documentation** | Java 21 requirement; JDK 25 observation; Maven absent; every command labeled PROPOSED | OD-2; quickstart §1, §3; verification-plan §1 | PROPOSED; environment NEEDS OWNER DECISION |

## 3. Cross-cutting checks (RECONCILIATION_INSTRUCTIONS §5 item 12)

| Check | Where enforced | Test |
|---|---|---|
| Group privacy; non-member → 404 | design §9; architecture §6 | VP-A08, VP-S01 |
| Former members see only their party records | D-27; design §9.2 | VP-A08 |
| Payer can be a member **without** being a participant | BR-05; design §4.2 | VP-U06; S6 |
| Pending and rejected never affect balances; approved do | BR-07; design §6.3 | VP-U07, VP-U09, VP-A05; S3, S5 |
| Audit and history survive edits and membership changes | D-31; I-18 | VP-A09; S5.5, S9 |
| Notifications target the **other** active members | BR-13; design §11 | VP-A04; S3.3 |
| Date-range reporting respects permissions | D-38; design §12 | VP-A07, VP-I03; S8 |
