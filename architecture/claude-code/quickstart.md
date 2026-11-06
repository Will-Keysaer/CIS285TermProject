# SplitSmart Quickstart (Claude Code proposal)

> **No application code exists yet.** Every build, run, and test command below is **PROPOSED** and **has not been executed**. They describe how the project *should* work once it is scaffolded according to [architecture.md](architecture.md). The only commands actually run while preparing this proposal are listed in §1 under "Verified".

## 1. Prerequisites

| Tool | Needed for | Status on the authoring machine (2026-11-05) |
|---|---|---|
| JDK **21** | All modules (constitution II) | **Not present.** `java -version` → OpenJDK **25.0.2** (Temurin). Install Temurin 21, or confirm that compiling with `--release 21` on JDK 25 is acceptable. |
| Maven 3.9+ (or Maven Wrapper `mvnw`) | Build and test | **Not found** on PATH. Recommend committing the Maven Wrapper when scaffolding. |
| PostgreSQL 16 or Docker | Alternative A local run / `-Ppostgres` tests | Not checked (`psql` not found; a legacy Docker Toolbox install was detected but not tested) |
| Node.js LTS | Only if the A front end is an SPA built via `frontend-maven-plugin` (that plugin can download Node itself) | Not checked |
| PlantUML (or an IDE plugin) | Render `diagrams/*.puml` | **Not found** on PATH |

**Verified** (actually executed during authoring): `java -version` and a PATH lookup for `java`, `mvn`, `plantuml`, `docker`, and `psql`. Nothing else.

## 2. Proposed project layout

```
pom.xml                         parent: <maven.compiler.release>21</maven.compiler.release>, JUnit 5 BOM
splitsmart-domain/              shared
splitsmart-application/         shared
splitsmart-persistence/         shared (Flyway: src/main/resources/db/migration)
splitsmart-web/                 Alternative A only
splitsmart-desktop/             Alternative B only - create ONLY after documented customer approval
```

Until OQ-1 is answered, scaffold only the three shared modules.

## 3. Proposed commands (not yet verified)

```bash
# Build everything and run all unit + integration tests (H2) - the constitution's "Maven verification"
./mvnw verify

# Core only (works before the platform decision)
./mvnw -pl splitsmart-domain,splitsmart-application -am verify

# Integration tests against real PostgreSQL via Testcontainers (requires Docker)
./mvnw verify -Ppostgres

# Alternative A - local PostgreSQL + run the web app
docker compose up -d postgres            # compose file to be added with splitsmart-web
./mvnw -pl splitsmart-web -am spring-boot:run -Dspring-boot.run.profiles=local
# then open https://localhost:8443 (or http://localhost:8080 in local profile)
# password-reset emails are printed to the console by the local EmailRecoveryChannel stub

# Alternative B [CONDITIONAL-B] - run the desktop app
./mvnw -pl splitsmart-desktop -am package
java -jar splitsmart-desktop/target/splitsmart-desktop.jar

# NFR-004 performance check (seeds 50 members / 10,000 records; reports p95)
./mvnw verify -Pperf -pl splitsmart-web

# Optional E2E browser smoke tests for Alternative A
./mvnw verify -Pe2e -pl splitsmart-web

# Render diagrams (requires PlantUML)
plantuml -tsvg architecture/claude-code/diagrams/*.puml
```

Windows (PowerShell): use `.\mvnw.cmd` in place of `./mvnw`.

## 4. End-to-end validation scenarios

Run these manually through the UI, or automate them (A: MockMvc/Playwright; B: presenter tests). Expected results trace to spec acceptance criteria.

**Test fixture:**

- Accounts **Alex** (alex@example.com), **Blake**, **Casey**, **Drew** (outsider).
- In automated tests, fixed UUIDs order the accounts Alex < Blake < Casey < Drew, so the largest-remainder tie-break is predictable.
- Group "Apartment 4B", USD, created by Alex. Self-approval off.

### S1: Accounts and privacy (AC-001, AC-002)
1. Register Alex, Blake, Casey, and Drew. Sign in as each. → Each sees only their own profile.
2. Alex signs out and requests a password reset (A) or uses a recovery code (B). → A: a reset link is printed to the console; the reset succeeds and the old password fails. B: the code works once only.
3. Request a reset for `nobody@example.com`. → Same response as for a real account (A).

### S2: Group and invitations (AC-003, AC-004)
1. Alex creates the group and invites Blake and Casey. → Blake and Casey are **not** members yet.
2. Blake accepts; Casey accepts. → Both are listed as MEMBER.
3. Blake (MEMBER) tries to invite Drew. → 403 `FORBIDDEN_ROLE`.
4. Drew requests `GET /groups/{id}` or any expense in it. → **404**, not 403.

### S3: Equal split with rounding (AC-005, AC-007, AC-021)
1. Blake submits $100.00, "Groceries", EQUAL. Payer **Alex**. Participants Alex, Blake, Casey.
2. → Shares Alex 33.34 / Blake 33.33 / Casey 33.33. The extra cent goes to the lowest account ID, which is Alex. Σ = 100.00. Status **PENDING**.
3. Alex and Casey have an `EXPENSE_SUBMITTED` notification. Blake (the submitter) does not (AC-013).
4. Group balances are unchanged, all zero (AC-010).

### S4: Validation (AC-009)
Submit each of the following; each must be rejected with the listed code and nothing saved:
- `0.00` → `AMOUNT_NOT_POSITIVE`
- `10.005` → `AMOUNT_PRECISION_EXCEEDED`
- PERCENTAGE 60 / 30 → `PERCENTAGES_NOT_100`
- PERCENTAGE 110 / −10 → `PERCENTAGE_NEGATIVE` (+ `PERCENTAGE_OUT_OF_RANGE`)
- Drew as participant → `PARTICIPANT_NOT_MEMBER`
- Empty participants → `PARTICIPANTS_REQUIRED`

All applicable field errors are returned together.

### S5: Review (AC-010 – AC-012, AC-023; OQ-2 default)
1. Blake tries to approve their own S3 expense. → `FORBIDDEN_ROLE` (Blake is not a manager).
2. Alex (manager, not the submitter) approves. → APPROVED. Balances: Blake owes Alex 33.33; Casey owes Alex 33.33. (Alex's own 33.34 share creates no debt.)
3. Blake submits $30.00 PERCENTAGE 33.33 / 33.33 / 33.34 (payer Casey; participants Alex, Blake, Casey). Alex rejects it with a reason. → REJECTED. Balances unchanged. History shows the REJECTED badge and the reason.
4. Two browser sessions approve and reject the same pending expense at once. → Exactly one succeeds; the other gets 409.

### S6: Payer not a participant (FR-003)
1. Casey submits $50.00 EQUAL, payer **Casey**, participants Alex and Blake only. Alex approves.
2. → Alex owes Casey 25.00; Blake owes Casey 25.00. Net with the earlier debts: Casey owes Alex 33.33 − 25.00 = 8.33.

### S7: Payments (AC-015, AC-016)
1. Blake records a payment of $33.33 to Alex. → Blake–Alex balance becomes 0.00 immediately. History shows recordedBy = Blake with a timestamp.
2. Blake records a $1.00 payment to Alex. → `NOTHING_OWED`.
3. Blake records a $30.00 payment to Casey while owing 25.00. → `PAYMENT_EXCEEDS_OUTSTANDING` with outstanding 25.00.
4. Alex tries to record a payment "from Blake". → `PAYMENT_NOT_SENDER`.
5. Repeat step 1's request with the same `Idempotency-Key`. → The same payment is returned; there is no second payment.

### S8: Summary and reports (AC-017 – AC-019)
1. Each member opens their summary. → The figures match S3 – S7. Drew sees nothing from the group.
2. *(Full scope)* Run a report from 2026-11-01 to 2026-11-05, both types. → It includes only records dated in that range, including records dated exactly on the boundaries.
3. *(Full scope)* Run a report with `to < from`. → `INVALID_DATE_RANGE`, and no report is shown.

### S9: Membership removal (AC-004, D-2)
1. Alex tries to remove Casey while Casey has a non-zero balance. → `MEMBER_HAS_OUTSTANDING_BALANCE`.
2. Settle all balances, then remove Casey. → Casey is no longer listed. Earlier expenses still show Casey as a participant. Casey can still view the records Casey was party to, but no other group records.

### S10: Performance (AC-022, NFR-004)
Run `-Pperf`. → A report of p95 latency per view. Pass if ≥ 95% of the specified views complete in ≤ 3 s. The outcome is unknown until measured.

## 5. Definition of done for verification

- `./mvnw verify` passes. This includes domain rounding tests, balance and reconciliation tests, authorization tests, and ArchUnit rules.
- S1 – S9 pass for the chosen alternative. S10 has been measured and recorded.
- A human has reviewed the AI-generated code, tests, and these documents (constitution VII).
