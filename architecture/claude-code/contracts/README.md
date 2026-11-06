# SplitSmart Interface Contracts (Claude Code proposal)

Status: **draft, needs human review.**

| Contract | Applies to | Status |
|---|---|---|
| [application-services.md](application-services.md) | **Both alternatives.** The platform-independent boundary between presentation and the core (`splitsmart-application`). | Proposed. Not blocked by OQ-1. |
| [rest-api.md](rest-api.md) | **Alternative A (web) only.** HTTP/JSON API at `/api/v1`. | **Conditional on OQ-1 = web** (the recommended alternative). |
| [swing-ui.md](swing-ui.md) | **Alternative B (Swing) only.** Screens, presenters, and their calls into application services. | **[CONDITIONAL-B]:** only with documented customer approval. Does **not** satisfy NFR-001. |

How the operations behave internally (validation order, transactions, locking, algorithms) is described in [../design.md](../design.md).

Rules shared by every contract:

- Validation, authorization, and state rules live in the application and domain layers. They are defined **once** in `application-services.md`. The presentation contracts only map those rules to HTTP or Swing.
- Error codes are a single shared enumeration (`application-services.md` §7).
- Money crosses every boundary as `BigDecimal`. In JSON it is a **decimal string**.
- The actor's identity comes from the authenticated session, never from request input.
