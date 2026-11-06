# SplitSmart: Technical Decisions and Research (Claude Code proposal)

Status: **draft, needs human review.** Companion to [architecture.md](architecture.md).

## Research method and evidence

- **No external research was performed for this proposal.** No web searches, benchmarks, or vendor documentation were consulted during authoring. There are therefore **no citations**.
- The decisions below rest on (a) the three authoritative inputs (`UserRequirement.md`, `spec.md`, `constitution.md`), (b) inspection of this repository, and (c) general software-engineering knowledge. Category (c) is **not independently verified evidence**. Library versions and feature claims marked "verify" must be confirmed against official documentation when implementation starts.
- **Repository facts observed on 2026-11-05:**
  - No application source code and no `pom.xml`.
  - Spec Kit scaffolding (`.specify/`, `.github/skills/`) and the spec directory, which contains only `spec.md` and `checklists/`.
  - On the authoring machine, `java -version` reports **OpenJDK 25.0.2 (Temurin)**. `mvn` and `plantuml` were **not found** on the PATH.
- No performance figures in this package are measured. Any statement about expected performance is labeled as an expectation.

## Decision log

Each entry gives the decision, its rationale, the alternatives considered, and its status. Status values: **Constraint** (fixed by the constitution), **Proposed** (this proposal's choice, open to change), **Conditional** (depends on OQ-1), **Needs customer input**.

### R-01 Delivery platform
- **Decision:** Present two unblended alternatives (A: web, B: Swing desktop) and recommend **A**.
- **Rationale:** NFR-001 and constitution VIII. Only A satisfies the customer baseline.
- **Alternatives:** Choose B outright, which is not allowed without documented approval. Build a hybrid, which is rejected (R-02).
- **Status:** **Needs customer input (OQ-1). Blocking for presentation and infrastructure work.**

### R-02 Swing variant
- **Decision:** If B is approved, build it as a **standalone** desktop app with an embedded DB.
- **Rationale:** This is the cheapest path that is honest about its limits. A Swing client on top of a networked server needs most of A's server work (API, authentication, hosting, TLS) and **still** fails NFR-001.
- **Alternatives:**
  - (i) Swing client + REST server. Rejected as a blend that costs more than either alternative.
  - (ii) Swing app connecting directly to a shared remote database. Rejected because DB credentials would sit on every client and authorization could be bypassed.
- **Status:** Conditional.

### R-03 Platform-independent core first
- **Decision:** Build `splitsmart-domain` and `splitsmart-application` (pure Java 21, JUnit 5) before the platform decision.
- **Rationale:** Constitution III requires this separation anyway. All business rules are identical under A and B, so this work is never wasted.
- **Alternatives:** Wait for OQ-1 before any code. That wastes schedule.
- **Status:** Proposed.

### R-04 Web stack (A)
- **Decision:** Spring Boot 3.x (Spring MVC, Spring Security, Spring Data JPA), packaged as a single executable JAR **[PROPOSAL]**. Browser client: a same-origin single-page app (TypeScript; framework left to the team) built into the JAR during `mvn package` (for example with `frontend-maven-plugin`).
- **Rationale:**
  - Mainstream Java 21-compatible stack (verify the exact supported version).
  - Integrated security, transactions, and testing (MockMvc).
  - A JSON API gives a clean, contract-testable presentation boundary.
  - A same-origin setup avoids CORS and keeps cookie sessions simple.
- **Alternatives:**
  - **Thymeleaf server-rendered pages**: all-Java, lower complexity, secure defaults. A strong fallback if the team lacks front-end experience. The REST contract would then become controller/form handlers with the same validation, authorization, and errors.
  - Jakarta EE / Quarkus / Micronaut: viable, but less common in course settings.
  - A separate front-end deployment: adds CORS and deployment complexity.
- **Status:** Conditional (A). Front-end choice is decision D-6.

### R-05 Database
- **Decision:**
  - A: **PostgreSQL** **[PROPOSAL]**.
  - B: **H2 file mode** **[PROPOSAL]**.
  - Tests: H2 by default, plus a PostgreSQL profile.
- **Rationale:** Relational integrity (FKs, unique constraints, `NUMERIC`), `SELECT … FOR UPDATE`, and mature Java support.
- **Alternatives:** MySQL/MariaDB (equivalent for this use); SQLite (weaker concurrency and `NUMERIC` semantics); document stores (poor fit for relational, ledger-like data).
- **Status:** Proposed.

### R-06 Persistence technology
- **Decision:** Spring Data JPA (Hibernate) with **Flyway** migrations shared by A and B.
- **Rationale:**
  - Optimistic `@Version` support.
  - Schema versioning.
  - One persistence module for both alternatives.
  - Balance aggregation uses explicit JPQL/SQL queries, not entity graphs.
- **Alternatives:** Spring JDBC/jOOQ (more explicit SQL, more code); Liquibase instead of Flyway (equivalent).
- **Status:** Proposed.

### R-07 Authentication and recovery
- **Decision:**
  - Email + password (fixed by spec clarification).
  - BCrypt hashing.
  - A: server session cookie + CSRF; recovery by emailed single-use token (hash stored, 30 min expiry).
  - B: recovery by one-time codes issued at registration.
- **Rationale:** The spec fixes email/password. A server session suits a same-origin SPA and is simpler and safer than browser-stored JWTs. B has no reliable out-of-band channel.
- **Alternatives:**
  - JWT bearer tokens: unnecessary for a single same-origin client.
  - Argon2id: also good; choose whichever is easier to configure.
  - OAuth/social login: not requested.
  - Security questions: weak; rejected.
- **Status:** Proposed. Email verification at sign-up is decision D-4.

### R-08 Money representation
- **Decision:**
  - `BigDecimal` with the currency's default fraction digits.
  - Reject excess input precision; never round input.
  - Persist as `NUMERIC(19,4)`.
  - Compare with `compareTo`.
- **Rationale:** Constitution IV and spec FR-003/NFR-003.
- **Alternatives:**
  - `long` minor units: exact, but conflicts with the explicit `BigDecimal` constraint at the domain boundary. It is used **internally** inside the allocation algorithm only.
  - Joda-Money/JSR-354 (Moneta): adds a dependency for little gain.
- **Status:** Constraint (`BigDecimal`) + Proposed (details).

### R-09 Allocation algorithm
- **Decision:** Largest remainder over integer minor units. Tie-break by ascending participant `accountId`. Equal split = largest remainder with equal weights.
- **Rationale:**
  - The spec requires largest remainder.
  - A deterministic tie-break makes results reproducible and testable.
  - Running equal splits through the same algorithm keeps one code path.
- **Alternatives:**
  - Tie-break by submission order: reproducible only if order is stored and stable.
  - Give the remainder to the payer: not the largest-remainder method.
  - Banker's rounding per share: does not reconcile.
- **Status:** Proposed (the tie-break needs human sign-off; the customer is unlikely to care).

### R-10 Balance storage
- **Decision:** Derive balances on read. No materialized balance table in the MVP.
- **Rationale:** One source of truth, reconciliation by construction (NFR-003), and no lost-update risk. The expected cost at NFR-004 scale is small, but this **must be measured**.
- **Alternatives:**
  - Materialized pair-balance table updated transactionally: faster reads, but a drift risk. Kept as the fallback.
  - Event-sourced ledger: overengineered here.
- **Status:** Proposed.

### R-11 Concurrency control
- **Decision:**
  - Conditional status updates plus optimistic `version` on expenses.
  - A pessimistic group-row lock for approvals, payments, and membership removal.
  - Idempotency keys for create operations.
- **Rationale:** Prevents double approval, approve/reject races, and concurrent over-settlement. Contention is negligible for groups of ≤ 50.
- **Alternatives:** `SERIALIZABLE` isolation (works, but adds retry handling everywhere); per-pair lock rows (finer-grained, unnecessary).
- **Status:** Proposed.

### R-12 Default review policy
- **Decision:** Managers approve, one approval, no self-approval unless the group enables it.
- **Rationale:** The spec assumes a single review step. Self-review defeats the purpose of the review. A group setting avoids deadlock in small groups.
- **Alternatives:** Any other member may approve; all participants must approve; multi-level approval (data model allows it later).
- **Status:** **Needs customer input (OQ-2).**

### R-13 Notifications
- **Decision:** In-app inbox. Rows are written in the same transaction as the triggering event. A `NotificationPublisher` port allows email or push later.
- **Rationale:** The spec assumes in-app notifications. Writing in the same transaction guarantees no lost notifications without needing a message broker.
- **Alternatives:** Email (needs SMTP and opt-outs); WebSocket/SSE live push (UX nicety, more complexity); message broker (overengineered).
- **Status:** **Needs customer input (OQ-3)** for channels.

### R-14 Receipt storage (A)
- **Decision:**
  - File system on a private volume behind a `ReceiptStorage` port.
  - JPEG and PNG only, ≤ 5 MB, magic-byte check, EXIF stripped.
  - Metadata in the DB.
- **Rationale:** Simple, cheap, and keeps the DB small.
- **Alternatives:** DB `BYTEA` (simple backups, DB bloat); S3-compatible object storage (better for cloud hosting; the port makes it a swap).
- **Status:** Proposed. Deferred in the candidate MVP.

### R-15 Overpayment
- **Decision:** Reject a payment larger than the sender's current outstanding debt to the recipient in that group.
- **Rationale:** AC-015 forbids the balance becoming negative solely because of an overpayment. Rejection is the simplest compliant rule.
- **Alternatives:** Allow it and record a credit (reverse debt), which arguably violates AC-015; cap the amount silently, which surprises users.
- **Status:** Proposed (D-1).

### R-16 Member removal
- **Decision:** Block removal or leaving while the member has a non-zero balance or pending involvement in the group. The creator cannot be removed, and the last manager cannot leave.
- **Rationale:** Prevents orphaned debts. AC-004 needs history to stay attributable, which is satisfied because records keep account IDs.
- **Alternatives:** Allow it with read-only carry-over of balances.
- **Status:** Proposed (D-2).

### R-17 API error format (A)
- **Decision:** RFC 9457 Problem Details with a stable `code` and `errors[]`.
- **Rationale:** A standard, machine-readable format that maps one-to-one onto domain `ErrorCode`s.
- **Alternatives:** An ad-hoc error JSON.
- **Status:** Proposed.

### R-18 Architecture enforcement
- **Decision:** Maven multi-module dependency direction + ArchUnit tests (no `double`/`float` money fields, no domain → framework imports, no presentation → persistence imports).
- **Rationale:** Makes constitution III and IV checkable in `mvn verify`.
- **Alternatives:** Code review only.
- **Status:** Proposed.

### R-19 Identifiers
- **Decision:** UUIDs for all externally visible IDs.
- **Rationale:** Non-guessable, and safe to generate on the client for idempotency keys.
- **Alternatives:** Sequential IDs (enumerable; leak counts).
- **Status:** Proposed.

### R-20 Test tooling
- **Decision:**
  - JUnit 5 (constraint).
  - Optional additions: jqwik (property tests), ArchUnit, Spring MockMvc, Testcontainers (PostgreSQL profile), Playwright for Java (A E2E), AssertJ-Swing (B smoke).
  - A JUnit/HTTP or Gatling harness for NFR-004.
- **Rationale:** Covers each layer through Maven.
- **Alternatives:** Mockito-heavy unit tests instead of in-memory fakes. Fakes are preferred for readable application tests.
- **Status:** Proposed. Confirm versions at implementation time.

## Unknowns and decisions requiring customer input

| ID | Question | Why it matters | Default used here |
|---|---|---|---|
| OQ-1 | Web vs. Swing MVP | Determines the presentation and infrastructure build; Swing does not meet NFR-001 | Recommend web; Swing only with written approval |
| OQ-2 | Approver rules, self-approval, multi-level | ReviewPolicy | Managers; no self-approval; single level |
| OQ-3 | Notification channels | Infrastructure adapters | In-app only |
| OQ-4 | Currencies / exchange rates | Money scale, aggregation | One fixed currency per group; no conversion |
| OQ-5 | Report export, extra filters | ReportService scope | No export; filters listed in architecture §4.6 |
| OQ-6 | Edit/delete expenses; history | State machine, audit | Pending editable by submitter; approved/rejected immutable |
| D-1 | Overpayment handling | Payment rule | Reject |
| D-2 | Removing members with balances | Membership rule | Block |
| D-3 | Account deletion and retention | Privacy vs. history | Out of scope |

## Internal unknowns (team or owner, not customer)

- **JDK:** the authoring machine has JDK 25, not 21. Install JDK 21, or confirm that compiling with `--release 21` on JDK 25 is acceptable under constitution II.
- **Maven:** not installed on the authoring machine. Install it, or add the Maven Wrapper when the project is scaffolded.
- **Hosting target for A** (D-5) and front-end technology (D-6).
