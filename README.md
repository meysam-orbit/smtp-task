# SMTP Relay Implementation Plan — split for review

Master document (source of truth, reviewed by senior): `../smtp-relay-implementation-plan.md`

| Part | File | Scope | Blocked by |
|---|---|---|---|
| 00 | `00-gates-and-design-decisions.md` | Context, design decisions, architecture, verification gates G1–G4, backend dependencies, master "Done when" list | — (gates are the first work) |
| 01 | `01-relay-core.md` | SMTP edge: integrity boundary, session/reply mapping, durability, security, rate limits, test mode, ops, master test matrix | G1–G4 |
| 02 | `02-panel.md` | Developer → SMTP surface, ingress badge/filters, credentials & permissions UI, From/custom-domain product decisions | Backend dependency 1 (`ingress` field); G1 (docs copy depends on gate verdicts) |
| 03 | `03-webhooks.md` | Webhook behavior contract, test-mode event table | G2 (event table confirmation), G3 (bounce loop) |
| 04 | `04-docs.md` | Documentation updates against docs.orbit.devotel.io | G1, G2 outcomes (several notes depend on gate verdicts) |

Rules that keep the parts from drifting:

1. **One master acceptance list** — "Done when" lives only in part 00.
2. **One master test matrix** — lives only in part 01. Parts 02–04 add no separate matrices.
3. Cross-part contracts (reply codes, `ingress` field, one-record-per-recipient, atomic admission codes) are stated as design decisions in part 00 and referenced elsewhere — never restated with different wording.
