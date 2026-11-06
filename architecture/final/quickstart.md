# SplitSmart Quickstart (Reconciled Candidate)

> **What exists today:** documentation and PlantUML sources only. There is **no** `pom.xml`, no `src/`, and no application code (repository inspected 2026-11-05).
>
> **Everything below is PLANNED. Every build, run, and test command is PROPOSED and has not been executed.** The only commands actually run during reconciliation were documentation and diagram checks ([README.md §8](README.md#8-validation-record)).

## 1. Prerequisites

| Tool | Needed for | Observed on the authoring machine |
|---|---|---|
| JDK **21** | Compile and test (CONST II) | **Not present.** OpenJDK 25.0.2 (Temurin) is installed. OD-2: install JDK 21, or approve `--release 21` on JDK 25. |
| Maven 3.9+ or the Maven Wrapper | Build (CONST II) | **Not found** on PATH. Commit `mvnw` when scaffolding. |
| PostgreSQL 16 or Docker | [COND-A] run; `-Ppostgres` concurrency tests | Not checked |
| PlantUML (jar + Java) or an IDE plugin | Render `diagrams/*.puml` | A PlantUML 1.2025.10 jar exists in the VS Code "Plan UML" extension storage, and was used for the checks in README §8 |
| A modern browser | [COND-A] E2E, AC-020 | n/a |

## 2. Planned project layout (not created)

```
pom.xml                               Java 21, JUnit 5 BOM, Spring Boot 3.x [COND-A], Flyway, ArchUnit
src/main/java/splitsmart/domain/       shared core
src/main/java/splitsmart/application/  shared core
src/main/java/splitsmart/persistence/  shared core
src/main/java/splitsmart/web/          [COND-A] controllers, security, adapters
src/main/java/splitsmart/desktop/      [COND-B] only after documented approval
src/main/resources/db/migration/       Flyway scripts (data-model.md §9)
src/main/resources/static/             [COND-A] browser client (D-06)
src/test/java/...                      JUnit 5 tests (verification-plan.md)
```

Until OQ-1 is answered, scaffold only the shared core packages (D-03).

## 3. Planned commands (PROPOSED, not run)

```bash
./mvnw verify                                   # all unit + H2 integration tests (CONST "Maven verification")
./mvnw verify -Ppostgres                        # concurrency/integration on PostgreSQL (F-07)
docker compose up -d postgres                   # [COND-A] local DB (compose file to be added)
./mvnw spring-boot:run -Dspring-boot.run.profiles=local   # [COND-A] then open http://localhost:8080
                                                # local profile prints password-reset links to the console
./mvnw verify -Pe2e                             # [COND-A] browser E2E (S1–S9)
./mvnw verify -Pperf                            # NFR-004 protocol (verification-plan.md §6)
```

On Windows PowerShell, use `.\mvnw.cmd`.

**Render diagrams** (this works today, given Java and a PlantUML jar):

```bash
java -jar plantuml.jar -checkonly architecture/final/diagrams/*.puml   # syntax check
java -jar plantuml.jar -tsvg -o out architecture/final/diagrams/*.puml # render SVGs to diagrams/out/
```

## 4. End-to-end validation scenarios

**Fixture:**

- Accounts **Alex**, **Blake**, **Casey**, **Drew** (outsider). In automated tests, fixed UUIDs order them Alex < Blake < Casey < Drew so tie-breaks are predictable.
- Group "Apartment 4B", USD, created by Alex. Self-approval off (default policy, OQ-2).

### S1: Accounts, recovery, privacy (AC-001, AC-002; F-02)

1. Register all four accounts. Each sees only their own profile.
2. Alex requests a reset. The link is printed to the console (local). The reset succeeds, the old password fails, and the link cannot be used twice. The link stops working after 30 minutes (test with a fixed clock).
3. A reset request for `nobody@example.com` gets the same response as for a real account.

### S2: Groups and invitations (AC-003, AC-004; F-03)

1. Alex creates the group and invites Blake and Casey. Neither is a member yet.
2. Drew tries `POST /me/invitations/{Blake's invitationId}/accept` → **404**. Only the invitee can accept.
3. Blake accepts. A second accept → `INVITATION_NOT_PENDING`. Casey accepts.
4. Blake (a MEMBER) tries to invite Drew → 403 `FORBIDDEN_ROLE`.
5. Drew opens `GET /groups/{id}` → **404**.

### S3: Equal split, notification, pending (AC-005, AC-007, AC-010, AC-013, AC-021)

1. Blake submits $100.00 "Groceries", EQUAL. Payer **Alex**. Participants Alex, Blake, Casey.
2. Shares are Alex 33.34, Blake 33.33, Casey 33.33. Status **PENDING**. The `reviewBlocked` flag is false, because Alex is a manager and is not the submitter.
3. Alex and Casey receive `EXPENSE_SUBMITTED` notifications that include "pending review". Blake does not.
4. Group balances are all zero (the expense is not approved yet).
5. History shows an `EXPENSE_SUBMITTED` event.

### S4: Validation (AC-009)

Each of the following is rejected with the listed code, and nothing is saved. All applicable errors are returned together.

| Input | Code |
|---|---|
| `0.00` | `AMOUNT_NOT_POSITIVE` |
| `10.005` | `AMOUNT_PRECISION_EXCEEDED` |
| `1E+2` | `AMOUNT_FORMAT_INVALID` |
| PERCENTAGE 60/30 | `PERCENTAGES_NOT_100` |
| PERCENTAGE 110/−10 | `PERCENTAGE_NEGATIVE` + `PERCENTAGE_OUT_OF_RANGE` |
| Drew as participant | `PARTICIPANT_NOT_MEMBER` |
| No participants | `PARTICIPANTS_REQUIRED` |

### S5: Review (AC-008, AC-011, AC-012, AC-023; F-04)

1. Blake tries to approve the S3 expense → `FORBIDDEN_ROLE`.
2. Alex approves it. Blake owes Alex 33.33. Casey owes Alex 33.33. Alex's own share creates no debt.
3. Blake submits $30.00 PERCENTAGE 33.33/33.33/33.34, payer Casey, participants Alex, Blake, Casey. Exact shares are 9.999 / 9.999 / 10.002. Floors total 29.98, so two cents remain, and they go to the two largest remainders. Result: 10.00 / 10.00 / 10.00. Alex rejects it with a reason. Balances are unchanged. History shows REJECTED with the reason.
4. Two sessions approve and reject the same pending expense at once. Exactly one succeeds; the other gets 409.
5. Blake edits a pending expense while Alex still has the old version open. Alex's approval → `CONCURRENT_MODIFICATION`. The audit log shows the before/after edit.

### S6: Payer not a participant (FR-003)

1. Casey submits $50.00 EQUAL, payer Casey, participants Alex and Blake only. Alex approves.
2. Alex owes Casey 25.00. Blake owes Casey 25.00. Net: Casey owes Alex 33.33 − 25.00 = **8.33**.

### S7: Payments (AC-015, AC-016; F-07)

1. Blake pays Alex $33.33. The Blake–Alex balance is **0.00 immediately**. History shows recorded by Blake, with a timestamp.
2. Blake records another $1.00 to Alex → `NOTHING_OWED`.
3. Blake pays Casey $30.00 while owing 25.00 → `PAYMENT_EXCEEDS_OUTSTANDING` (outstanding 25.00).
4. Alex records a payment "from Blake" → `PAYMENT_NOT_SENDER`.
5. Step 1 is retried with the same `Idempotency-Key` → the same payment is returned; no new row.
6. Blake pays Casey $10.00, then $10.00 again on the same date → `POSSIBLE_DUPLICATE_PAYMENT`. Confirming records it. Balance 5.00.

### S8: Summary, receipts, and reports (AC-006, AC-017–AC-019)

1. Each member's summary matches S3–S7. Drew sees nothing from the group.
2. *(Release 2)* Blake attaches a PNG receipt to a pending expense. Alex downloads it. Drew → 404. A 6 MB file → 413. A GIF renamed `.jpg` → 415.
3. *(Release 2)* A report for 2026-11-01 to 2026-11-05 includes only records dated in that range, **including records dated on the boundaries**.
4. *(Release 2)* A report with `to < from` → `INVALID_DATE_RANGE`, and no report is shown.

### S9: Membership removal and former members (AC-004; OQ-8)

1. Alex tries to remove Casey while Casey has a non-zero balance → `MEMBER_HAS_OUTSTANDING_BALANCE`.
2. After everything is settled, Alex removes Casey. Earlier expenses still name Casey. Casey can view only the records Casey was party to.

### S10: Performance (AC-022)

Run the protocol in [verification-plan.md §6](verification-plan.md#6-performance-protocol-nfr-004--ac-022). **The outcome is unknown until measured.**

## 5. Definition of done

1. `./mvnw verify` and `-Ppostgres` pass on JDK 21.
2. S1–S9 pass for the platform approved under OQ-1.
3. S10 has been measured and recorded.
4. All PlantUML sources render.
5. A human has reviewed all AI-generated artifacts (CONST VII).
