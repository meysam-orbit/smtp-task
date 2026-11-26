# Part 03 — Webhooks
# Task: Implement the SMTP Relay (smtp.orbit.devotel.io)

> Scope of this part: the webhook behavior contract for SMTP-originated messages and the test-mode event table.
> Design decisions: [00-gates-and-design-decisions.md](00-gates-and-design-decisions.md). **Blocked by:** gate G2 (test-mode event confirmation) and gate G3 (bounce loop). Webhook tests live in the master test matrix, row "Webhooks" ([01-relay-core.md](01-relay-core.md)).

## Webhooks

**SMTP introduces no new webhook event types.** Events key on the message lifecycle, never on the ingress: an SMTP-originated message flows through the same email pillar as an HTTP send, so the existing email catalog fires identically. This task must not add new event types.

| Lifecycle moment | Webhook event | Fires for SMTP? |
|---|---|---|
| Durable acceptance (`250` after DATA) | — none — | Yes, same as HTTP (`202 queued` fires nothing; the accepted message exists, no event yet) |
| Handed to the outbound provider | `message.sent` | Yes — identical |
| Provider confirms delivery (DLR) | `message.delivered` | Yes — identical |
| Permanent failure / provider bounce | `message.failed` | Yes — identical, via the VERP return-path (see below) |
| Recipient engagement | `email.opened`, `email.clicked` | Yes — identical |
| Suppression-list change (bounce/complaint ingestion) | Suppression event payloads | Yes — identical |
| RCPT refusal (`550` suppressed, `452 4.5.3` over cap, `501` syntax) | — none — | **No** — no message record exists; same class as an HTTP 4xx validation error |
| AUTH failure (`535`/`454`), throttle (`451`/`452`), relay disabled (`554`), oversize (`552`) | — none — | **No** — transient protocol conditions, not message events |

**Test-mode event behavior (per event, Live SMTP vs Test SMTP):** the guarantee is *SMTP matches HTTP test mode exactly, event-for-event* — not this table's opinion. The table states the expected behavior derived from the documented platform rules (`test_sent` terminates at accept; no provider is called; no balance moves) and **must be confirmed against the actual HTTP Test Mode implementation** (backend-dependencies list in [00](00-gates-and-design-decisions.md), confirmed by gate G2) before coding:

| Event | Live SMTP | Test SMTP (expected — confirm vs HTTP) |
|---|---|---|
| `message.sent` | On provider hand-off | **Expected: no** — a `test_sent` message is never handed to a provider; if the HTTP path fires a test-specific `message.sent`, SMTP matches that |
| `message.delivered` | On delivery DLR | **No** — nothing delivers |
| `message.failed` | On permanent failure/bounce | **No** — nothing can fail in transit |
| `email.opened` / `email.clicked` | On real engagement | **No** — no delivered message to open or click |

The acceptance criterion is the match, not the expected column: whichever way the HTTP implementation resolves `test_sent` webhook behavior, SMTP-originated records behave identically.

Rules:

- **Bounce tracking closes the loop because of the envelope rule:** the client's `MAIL FROM` is stored, never wired (design decision 5 in [00](00-gates-and-design-decisions.md); worked example in [01-relay-core.md](01-relay-core.md)) — outbound mail carries Orbit's VERP return-path, so provider bounces return to Orbit and surface as `message.failed` plus suppression events on the customer's webhook. Had the client's address stayed on the wire, bounces would vanish to the client and Orbit would never fire failure events for SMTP traffic.
- **Payload:** the message object in webhook payloads carries the `ingress: api | smtp` field (additive, alongside the existing fields), so receivers can segment by source without a new subscription. Everything else in the payload is unchanged.
- **Configuration surface: none added.** Customers subscribe with the same webhook endpoints / event sinks they already have; SMTP needs no new subscription category, endpoint, or scope.
- **Docs note:** the SMTP relay docs page gains a short "Webhooks" paragraph — events fire identically for SMTP submissions, payloads carry `ingress`, protocol-level rejections fire nothing — cross-linked to the webhooks section. (→ [04-docs.md](04-docs.md))
- **Record model:** one record per surviving recipient (the HTTP multi-recipient fanout model). Every lifecycle event, message-list row, detail view, and billing unit belongs to a single recipient's record; a multi-recipient transaction is simply N records born from one SMTP session.
