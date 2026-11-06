# SplitSmart: Traceability (Reconciled Candidate)

Status: **Reconciled candidate. Pending human review.** This document maps requirements to the design in both directions.

- **Forward:** every FR, NFR, and AC → design sections, data and contract elements, UML diagrams, planned tests, and release.
- **Backward:** diagrams, design sections, and tests → requirements.
- **Design coverage is not verification.** No test exists or has run.

**Status values:**

| Status | Meaning |
|---|---|
| `PROPOSED` | Designed, unverified |
| `UNVERIFIED` | Depends on a measurement or test on the selected database |
| `BLOCKED` | Waits on a decision |
| `APPROVED` | Not used: nothing is approved |

**Release values:** R1, R2 (requirements §7); "R1/R2 [A]" means the item depends on Alternative A.

**Abbreviations:**

| Abbreviation | Document |
|---|---|
| D§ | [design.md](design.md) |
| DM§ | [data-model.md](data-model.md) |
| AS§ | [contracts/application-services.md](contracts/application-services.md) |
| API§ | [contracts/web-api.md](contracts/web-api.md) |
| UI | [contracts/ui-workflows.md](contracts/ui-workflows.md) screen IDs |
| VP | [verification-plan.md](verification-plan.md) test IDs |
| S# | [quickstart.md §4](quickstart.md#4-end-to-end-validation-scenarios) scenarios |

## 1. Functional requirements → design → verification

| Req | Design sections | Data / contracts | Diagrams | Planned tests | Release | Status | Under Swing [COND-B] |
|---|---|---|---|---|---|---|---|
| FR-001 | D§4.3, §13.2, §14; architecture §5.1 | DM§1.1–1.3; AS§2; API§3.1; UI U1–U5 | use-cases, sequence-password-reset, components | VP-A01, VP-A02, VP-S02, VP-S04; S1 | R1 | PROPOSED | ✅ via recovery codes |
| FR-002 | D§5.3–5.5, §8.1, §15.6–15.7 | DM§1.4–1.6; AS§3; API§3.2; UI U6–U8 | use-cases, states-invitation, domain-model, classes-detailed | VP-U11, VP-U12, VP-A03; S2, S9 | R1 | PROPOSED | ◐ same installation only |
| FR-003 | D§2, §4.2, §13.3, §15.1 | DM§1.7–1.9; AS§4; API§3.3, §3.8; UI U10–U11 | sequence-expense-review, activities, domain-model | VP-U01, VP-U06, VP-A04, VP-S03; S3, S4, S6, S8.2 | R1 (receipt: **R2**) | PROPOSED | ◐ receipt only if built |
| FR-004 | D§3 | DM§1.8; AS§4.1 (`previewAllocation`) | classes-detailed, sequence-expense-review | VP-U04, VP-U05; S3 | R1 | PROPOSED | ✅ |
| FR-005 | D§2.2, §3 | DM§1.8; AS§4.2 | classes-detailed, sequence-expense-review | VP-U03, VP-U04, VP-U05; S4, S5.3 | R1 | PROPOSED (precision OQ-10) | ✅ |
| FR-006 | D§5.1, §6.3 | DM§5 I-8; AS§4.3 | states, activities | VP-U07, VP-U09; S3.4 | R1 | PROPOSED | ✅ |
| FR-007 | D§5.1–5.2, §8.2 | DM§1.10; AS§4.1; API§3.3; UI U11–U12 | states, sequence-expense-review, activities | VP-U07, VP-U08, VP-A05, VP-C01, VP-C03; S5 | R1 (single level) | PROPOSED; policy NEEDS CUSTOMER DECISION (OQ-2) | ✅ |
| FR-008 | D§11 | DM§1.11; AS§6; API§3.5; UI U5, U16 | sequence-expense-review, use-cases | VP-A04; S3.3 | R1 (in-app) | PROPOSED; channels OQ-3 | ◐ same machine |
| FR-009 | D§6 | DM§1.13, §4; AS§5; API§3.4; UI U13, U17 | domain-model, classes-detailed | VP-U09, VP-I02; S5, S6 | R1 | PROPOSED | ✅ |
| FR-010 | D§7, §8 | DM§1.12; AS§5, §8; API§3.4; UI U14–U15 | sequence-payment, states-payment, activities-payment | VP-U10, VP-A06, VP-C02, VP-C04; S7 | R1 | PROPOSED; UNVERIFIED (concurrency) | ✅ |
| FR-011 | D§12.1 | AS§6; API§3.6; UI U18 | use-cases | VP-A07; S8.1 | R1 | PROPOSED | ✅ |
| FR-012 | D§12.2 | DM§1.14; AS§6; API§3.6; UI U19 | use-cases | VP-U13, VP-A07, VP-I03; S8.3–8.4 | **R2** | PROPOSED; export/filters OQ-5 | ◐ if built |

## 2. Nonfunctional requirements → design → verification

| Req | Design | Data / contracts | Diagrams | Planned tests | Release | Status | Under Swing |
|---|---|---|---|---|---|---|---|
| NFR-001 | architecture §3.1; D§13 | API (all); UI §1–§5 | components, use-cases | VP-E2E-1…10 | R1 [A] | **BLOCKED** (OQ-1) | ❌ **NOT MET** by definition |
| NFR-002 | D§9, §10.2, §13.1; architecture §6 | AS§1 (check order); API§2 (404 rule) | components (trust boundary), use-cases | VP-A08, VP-S01, VP-S02, VP-S04; S1, S2.5 | R1 | PROPOSED | ◐ DB file readable unless encrypted |
| NFR-003 | D§2, §3, §6; architecture §7 | DM§3, §5 I-1–I-4, I-13, I-15 | classes-detailed, domain-model | VP-U02, VP-U04, VP-U05, VP-U09, VP-I02, VP-ARCH | R1 | PROPOSED | ✅ |
| NFR-004 | D§6.1–6.2; architecture §9 | DM§2 indexes; API paging | none | VP-PERF (verification-plan §6) | R2 exit | **UNVERIFIED** (no measurement) | Measured separately |
| NFR-005 | D§5.1, §8.5, §11 | DM§1.10, §1.12, §1.15; API§3.7; UI U9, U11, U15, U20 | states, states-payment, domain-model | VP-A05, VP-A09; S5.3, S5.5, S7.1 | R1 | PROPOSED | ✅ |

## 3. Acceptance criteria → verification

| AC | Req | Design / contract | Diagram | Planned verification | Release | Status |
|---|---|---|---|---|---|---|
| AC-001 | FR-001 | D§13.2; API§3.1 | sequence-password-reset | VP-A01, VP-A02; S1.1–1.3 | R1 | PROPOSED |
| AC-002 | FR-001, NFR-002 | D§9 | components | VP-A08, VP-S01; S1.1, S2.5 | R1 | PROPOSED |
| AC-003 | FR-002 | D§5.3, §15.6; API§3.2 | states-invitation | VP-U12, VP-A03; S2.1–2.3 | R1 | PROPOSED |
| AC-004 | FR-002 | D§5.5, §15.7; DM I-18 | states-invitation | VP-U11, VP-A03, VP-A09; S2.4, S9 | R1 | PROPOSED |
| AC-005 | FR-003 | D§4.2, §15.1 | sequence-expense-review | VP-U06, VP-A04; S3 | R1 | PROPOSED |
| AC-006 | FR-003 | D§13.3; API§3.8 | use-cases | VP-S03; S8.2 | **R2** | PROPOSED |
| AC-007 | FR-004 | D§3 | classes-detailed | VP-U04; S3.2 | R1 | PROPOSED |
| AC-008 | FR-005 | D§3 | classes-detailed | VP-U04; S5.3 | R1 | PROPOSED |
| AC-009 | FR-003, FR-005 | D§4.2; AS§4.2 | activities | VP-U01, VP-U03, VP-U06; S4 | R1 | PROPOSED |
| AC-010 | FR-006 | D§5.1, §6.3 | states | VP-U07, VP-A04; S3.4 | R1 | PROPOSED |
| AC-011 | FR-007 | D§8.2 | sequence-expense-review | VP-A05; S5.2 | R1 | PROPOSED |
| AC-012 | FR-007 | D§8.2 | sequence-expense-review | VP-A05; S5.3 | R1 | PROPOSED |
| AC-013 | FR-008 | D§11 | sequence-expense-review | VP-A04; S3.3 | R1 | PROPOSED |
| AC-014 | FR-009 | D§6 | domain-model | VP-U09; S5.2, S6 | R1 | PROPOSED |
| AC-015 | FR-009, FR-010 | D§7 (overpayment rejected) | sequence-payment | VP-U10, VP-C02; S7.1–7.3 | R1 | PROPOSED; rule pending OQ-7 |
| AC-016 | FR-010 | D§7 | sequence-payment, states-payment | VP-A06; S7.1, S7.4 | R1 | PROPOSED |
| AC-017 | FR-011 | D§12.1 | use-cases | VP-A07; S8.1 | R1 | PROPOSED |
| AC-018 | FR-012 | D§12.2 | use-cases | VP-A07, VP-I03; S8.3 | **R2** | PROPOSED |
| AC-019 | FR-012 | D§12.2; DM§1.14 | use-cases | VP-U13; S8.4 | **R2** | PROPOSED |
| AC-020 | NFR-001 | architecture §3.1 | components | VP-E2E-10 | R1 [A] | **BLOCKED** (OQ-1); cannot pass under Swing |
| AC-021 | NFR-003 | D§3 | classes-detailed | VP-U04, VP-U05 | R1 | PROPOSED |
| AC-022 | NFR-004 | verification-plan §6 | none | VP-PERF; S10 | R2 exit | **UNVERIFIED** |
| AC-023 | NFR-005 | D§8.5; API§3.7 | states, states-payment | VP-A09; S5.3, S7.1 | R1 | PROPOSED |

**Coverage:**

| Set | Mapped to design and a planned test | Verified |
|---|---|---|
| FR-001–FR-012 | 12 / 12 | 0 / 12 |
| NFR-001–NFR-005 | 5 / 5 | 0 / 5 |
| AC-001–AC-023 | 23 / 23 | 0 / 23 |

## 4. Gaps, deferrals, and unverified items

| Item | Gap | Why it stays in full scope | Action |
|---|---|---|---|
| FR-003 receipt image / AC-006 | Planned for R2, not R1 | UR: "an optional receipt image" | Deliver in R2. Any cut needs OQ-13. |
| FR-012 / AC-018, AC-019 | Planned for R2 | UR: "custom reports with specified date ranges and other parameters" | Same |
| FR-007 multi-level approval | Single level only | UR: "could involve multiple levels… depending on the needs of the group" | OQ-2 |
| FR-008 other channels | In-app only | UR names no channel; SPEC OQ-3 | OQ-3 |
| NFR-001 / AC-020 | Blocked by the platform decision. **Not met under Swing.** | Customer baseline | OQ-1 |
| NFR-004 / AC-022 | Not measured | Planning target | Run VP-PERF |
| All FR / NFR / AC | No executed tests | n/a | Implement after OQ-1 and human review |

## 5. Success criteria (SC-001–SC-006)

| SC | Verification |
|---|---|
| SC-001 (90% of first-time users complete their first tasks unassisted) | Usability session on U2, U6–U8, and U10 [COND-A]. **Not designed in detail; planned for R1 exit.** |
| SC-002 (all split tests reconcile) | VP-U04, VP-U05 |
| SC-003 (balances = approved allocations − payments) | VP-U09, VP-I02 |
| SC-004 (95% of views ≤ 3 s) | VP-PERF |
| SC-005 (100% of authorization tests block outsiders) | VP-A08, VP-S01 |
| SC-006 (every feature area linked to a requirement and a criterion) | This document, §1–§3 |

## 6. Backward traceability

### 6.1 Diagrams → requirements

| Diagram | Requirements | Design / contract sections |
|---|---|---|
| use-cases.puml | FR-001–FR-012, NFR-002 | design §9.2; ui-workflows §3 |
| components.puml | NFR-001, NFR-002, CONST III | architecture §2, §4, §6; design §1 |
| components-swing-conditional.puml | NFR-001 (not met), CONST VIII | architecture §3.2; ui-workflows §6 |
| classes-detailed.puml | FR-003–FR-010, NFR-003 | design §1.1, §2, §3, §5–§7 |
| classes-application.puml | FR-001–FR-012, NFR-002, NFR-005 | design §1.2; application-services |
| domain-model.puml | FR-002–FR-010, NFR-003, NFR-005 | data-model §1, §2, §5 |
| sequence-expense-review.puml | FR-003–FR-008, NFR-002, NFR-005 | design §8.2, §15.1–15.2; web-api §3.3 |
| sequence-payment.puml | FR-009, FR-010, AC-015, AC-016 | design §7, §8.4 |
| sequence-password-reset.puml | FR-001, AC-001 | design §13.2; web-api §3.1 |
| states.puml | FR-006, FR-007, NFR-005 | data-model §6.1; design §5.1 |
| states-invitation.puml | FR-002, AC-003, AC-004 | data-model §6.2–6.3; design §5.3–5.5 |
| states-payment.puml | FR-010, NFR-005 | data-model §6.5; design §7 |
| activities.puml | FR-003–FR-008 | design §15.1–15.2 |
| activities-payment.puml | FR-009, FR-010 | design §7 |

### 6.2 Design sections → requirements

| design.md section | Requirements |
|---|---|
| §1 Structure | CONST II, III |
| §2 Money | FR-003, FR-005, NFR-003, CONST IV |
| §3 Allocation | FR-004, FR-005, NFR-003 |
| §4 Validation | FR-001–FR-005, FR-010, FR-012 |
| §5 State machines | FR-002, FR-006, FR-007 |
| §6 Balances | FR-009, NFR-003 |
| §7 Payments | FR-010 |
| §8 Transactions | FR-007, FR-010, NFR-003, NFR-005 |
| §9 Authorization | NFR-002, FR-002, FR-011 |
| §10 Errors | NFR-002 |
| §11 Notifications | FR-008 |
| §12 Reports | FR-011, FR-012 |
| §13 Web [COND-A] | NFR-001, FR-001, FR-003 |
| §14 Swing [COND-B] | none (NFR-001 not met) |
| §15 Walk-throughs | all FRs |
| §16 Tests | all |

### 6.3 Tests → requirements

The requirement for every test ID is listed in its row in [verification-plan.md](verification-plan.md) §2–§7.
