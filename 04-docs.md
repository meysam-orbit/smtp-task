# Part 04 — Docs
# Task: Implement the SMTP Relay (smtp.orbit.devotel.io)

> Scope of this part: every documentation change required by the SMTP relay launch, applied to https://docs.orbit.devotel.io — the docs are the spec, with the carve-out in Notes below).
> Design decisions: [00-gates-and-design-decisions.md](00-gates-and-design-decisions.md). **Depends on:** gate G1 (the signed-mail/tracking note depends on the auto-skip verdict) and gate G2 (the test-mode paragraph depends on the confirmed event table).

## Documentation updates

- Rewrite the mixed-recipient fanout example to the RCPT-TO refusal model: suppressed addresses get `550` at `RCPT TO`, and `250` after `DATA` means every surviving recipient is accepted. The "(1/2 recipients)" post-DATA drop cannot stand — clients never parse reply text.
- Fix the protocol contradiction in the troubleshooting section (`335` → `530`), including the client-snippet commentary; note `334` is the actual SASL challenge code.
- Add a "Signed mail and tracking" note: open/click tracking alters the HTML (pixel injection, link rewriting, per sending-domain setting) — **tracking automatically skips DKIM/S/MIME/PGP-signed messages**, so signed mail keeps its signatures with no customer action; unsigned messages receive the standard tracking rewrites (including the HTML part's re-encode); Orbit's own DKIM always validates (applied after tracking).
- Document that pool-exhaustion refusals are temporary (`451 4.7.1`, retry later) while a transaction too large for the tenant's pool capacity is permanent (`552 5.7.28`, use fewer recipients).
- Document duplicate-recipient semantics: repeated `RCPT TO` for the same address in one transaction is accepted once — one record, one billed unit.
- Add a pricing line: "SMTP submissions bill at the standard email rate — the sending method doesn't affect price" (Postmark-style parity; no separate SMTP SKU).
- Add a Test-mode paragraph: test keys sandbox identically over SMTP (`test_sent`, normal `250`, no charges); live keys are unaffected by any header a client might send.
- The `535` troubleshooting checks add the Messaging-write scope requirement (see Permissions in [02-panel.md](02-panel.md)).
- Label the empty-body `554` as an Orbit product policy, not SMTP behavior.
- Add the attachment-size limit and the recipient-cap (≥ 100) to the connection profile section.
- Keep the six client snippets verbatim-compatible — they are the acceptance scripts.
- Document the AUTH capability advertisement per RFC 4954: `AUTH PLAIN LOGIN` appears in EHLO **only after STARTTLS**, never on the plaintext session.
- Document the single-visible-sender policy: multi-mailbox `From:` fields are refused `554` as an intentional product restriction (legal RFC 5322, restricted by policy).
- The Developer → SMTP auth card and the `535` troubleshooting checks state the permission requirement: the key must hold **Messaging write** scope.
- The Messaging → Write scope description in the key-creation dialog covers SMTP ("Create messages via the HTTP API and the SMTP relay").
- Add a "DNS requirements" section: cross-links Domain Setup, states explicitly that **no extra records are needed for SMTP**, and notes the Gmail/Yahoo bulk-sender posture applies equally to SMTP-originated sends (see [02-panel.md](02-panel.md)).
- Add a short "Webhooks" paragraph: events fire identically for SMTP submissions, payloads carry `ingress`, protocol-level rejections fire nothing — cross-linked to the webhooks section (see [03-webhooks.md](03-webhooks.md)).
- When live, remove the "not launched yet" callout from the SMTP relay docs page.

## Notes

- The docs are the spec, with one carve-out: the SMTP relay documentation page defines the connection profile, message mapping, reply semantics, and troubleshooting — implement to it, except where it contradicts the SMTP protocol (then fix the docs). Companion guide: the SMTP relay recipes guide.
- Reference pattern: Postmark's and SES's SMTP ingestion endpoints — existing API credential as the SMTP password, submissions join the normal send pipeline. The work is an SMTP edge + mapping layer in front of the existing email pillar, not a parallel product: **SMTP submission → shared email pipeline → existing delivery providers.** No inbound mail, no IMAP/POP3, no mailbox hosting.
