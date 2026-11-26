# Part 01 — Relay Core
# Task: Implement the SMTP Relay (smtp.orbit.devotel.io)

> Scope of this part: the SMTP edge implementation — message integrity boundary, session/reply mapping, acceptance and durability, sender identity, security controls, rate limits, test mode, attachments, operations, logging, and the **master test matrix**.
> Design decisions and blocking gates: [00-gates-and-design-decisions.md](00-gates-and-design-decisions.md). **Blocked by gates G1–G4.**

**Verification tags.** Every substantive claim below carries one of three tags so implementation never confuses decided behavior with unverified assumption:

- **[VERIFIED-DOCS]** — confirmed against the published docs or the live panel.
- **[VERIFY-BACKEND]** — cannot be confirmed without the backend repository; gated by a pre-implementation verification gate (see [00](00-gates-and-design-decisions.md)). The stated behavior is the *requirement*, to be reconciled with the actual implementation before coding.
- **[PROPOSED]** — new behavior decided by this plan; implementable as written.

### Message integrity — the preservation boundary

The relay is **content-preserving, not byte-identical**: the sender-authored content must survive unmodified except for an exhaustive, staged list of transformations. **The original RFC 5322/MIME representation flows through the pipeline as-is**: parsing and validation operate on a copy and must never feed a re-serialized, reconstructed, or "normalized" message back into the send path — the bytes that leave Orbit are the bytes that arrived, minus the listed removals and plus the listed additions. Demonstrating this byte-preservation through the existing pipeline is part of gate G1; if the pipeline cannot demonstrate it, building that guarantee is a required backend change. The list below enumerates the **actual** send-time behavior of the existing email pipeline (from the documented email channel: open tracking injects a 1×1 pixel, click tracking rewrites links — both configured **per sending domain** — plus DKIM signing; anything else found in the backend implementation must be added to this list before coding, per the backend-dependencies list).

**Preserved (the invariant):** every sender-authored header (order preserved), the complete MIME tree structure, part boundaries, transfer encodings, and body bytes — **subject to the one honest exception below**. After transport, the message the recipient sees must be the message the client submitted, except where the exception applies: **when tracking (stages 6–7) is enabled and the message is unsigned, the HTML part is decode→modify→re-encode** — its transfer encoding and bytes change, by necessity. That is the only re-encoding the relay ever performs; everything else stays byte-identical.

**Transformation stages, in order** (the table is closed; the only thing that may amend it is the transformation-enumeration verification gate below, and only by explicit reconciled additions):

| Stage | # | Transformation | Content-altering? | Gating | Source |
|---|---|---|---|---|---|
| Edge (pre-persistence) | 1 | Remove the `Bcc:` header | No (removes a private header) | Always | [VERIFIED-DOCS] (message-mapping rules) |
| Edge (pre-persistence) | 2 | Insert `From:` from the tenant's From-email defaults — **only when the header is absent**; addition-only, never replacement | No (adds a missing header) | Only when absent | [PROPOSED] (implements the From rule; pipeline parity confirmed in dependency 6) |
| Edge (pre-persistence) | 3 | Prepend the `Received:` trace header | No (adds a header) | Always | [VERIFIED-DOCS] (RFC 5321 §3.8.2) |
| Edge (pre-persistence) | 4 | Synthesize `Date:` and `Message-ID:` **only when absent** | No (adds missing transport headers — `Date:` required by RFC 5322; `Message-ID:` a SHOULD) | Only when absent | [PROPOSED] — the edge guarantees transport headers exist so the pipeline never has to |
| Edge | 5 | Persist the message record (record body field prefers the HTML part) | Record-only — never on the wire | Always | [VERIFIED-DOCS] (message-mapping rules) |
| Pipeline (outbound, shared with HTTP) | 6 | Open tracking — inject a 1×1 pixel into the HTML part. **This stage is the documented exception to the no-re-encoding invariant**: injecting into an encoded part requires decode → modify → re-encode, so when tracking is enabled the HTML part's transfer encoding may legitimately change (e.g., quoted-printable ↔ base64) and its bytes change. When tracking is disabled the part ships byte-identical. **Signed parts are skipped automatically:** tracking must detect DKIM/S/MIME/PGP-signed content (`DKIM-Signature` headers, `application/(x-)pkcs7-mime` / PGP parts) and skip pixel/link injection for those messages — a customer who signs must not silently lose signatures or be told to disable tracking per domain | **Yes — alters the HTML body (decode/re-encode)** | Per sending-domain tracking setting; auto-skipped for signed messages | [VERIFIED-DOCS] (Engagement Tracking section) + [PROPOSED] signed-part skip |
| Pipeline (outbound, shared with HTTP) | 7 | Click tracking — rewrite links to Orbit redirects (unrewritable links ship untracked). Same re-encoding exception and same automatic signed-part skip as stage 6 | **Yes — alters the HTML body** | Per sending-domain tracking setting; auto-skipped for signed messages | [VERIFIED-DOCS] (Engagement Tracking section) + [PROPOSED] signed-part skip |
| Pipeline (outbound, shared with HTTP) | 8 | One-click `List-Unsubscribe` header handling — **parity rule**: SMTP-originated messages receive exactly the treatment HTTP-originated messages receive, nothing more and nothing less. **No-injection-when-present rule:** if the client supplied its own `List-Unsubscribe` (and `List-Unsubscribe-Post`) header, the pipeline must **not inject a second one** — the client's header wins and stage 8 is a no-op for that message; Orbit injects only when the header is absent. Two `List-Unsubscribe` headers on one message are an RFC 5322 violation and confuse one-click handling | Header-additions only (if the pipeline injects and the header is absent) | [VERIFY-BACKEND] — the docs mention the one-click flow as a suppression reason but never document header injection; the enumeration gate decides whether this row exists, and whether the backend already honors an existing client header | [VERIFY-BACKEND] |
| Pipeline (outbound, shared with HTTP) | 9 | Orbit `DKIM-Signature` — **the final Orbit-side step: every Orbit-side header transformation (stages 1–8) precedes signing**, so Orbit's signature always covers the message exactly as the receiver sees it | No (adds a header; signs what is there) | Always | [VERIFIED-DOCS] (Domain Setup signing) |
| Provider | 10 | Provider-side additions are outside Orbit's control | Out of scope | — | Inherent |

This table plus the enumeration gate **closes the boundary**: the implementer never "discovers" transformations during coding — gate G1 (in [00](00-gates-and-design-decisions.md)) reconciles the backend's actual send-time behavior against this table *before* implementation starts, and any extra transformation is either added here explicitly (with content-altering classification and gating) or disabled for SMTP-originated messages.

**Client-signature validity — corrected and precise:** header *additions* and *removals* affect signature families differently, so the guarantee is stated per family:

- **Orbit's own DKIM signature always validates** — it is applied last (stage 9), after every Orbit-side header transformation.
- **Client DKIM** (RFC 6376) signs the headers listed in its `h=` tag plus the body hash — and an **oversigned** header (listed more than once in `h=`) breaks the signature on *any* addition *or* removal of that header type, not just modifications. So the guarantee accounts for every Orbit-side header change: header *additions* whose name is **not protected by the client's `h=` list** (Received, Orbit's DKIM, From-insertion when absent) do not break it; additions or removals of a header **in** `h=` break it — including the signed-`Bcc:` removal edge case and the added-`DKIM-Signature` case (clients that oversign `dkim-signature` — a known hardening practice — will see their signature broken by Orbit's added signature; inherent to any relay that signs). One precision the guarantee must carry: **RFC 6376 explicitly allows a signer to include headers that do not exist at signing time**, precisely to protect against their later addition. Therefore, if a client protects an absent `Date:`, `Message-ID:`, or (via the pipeline's stage-8 parity handling) `List-Unsubscribe:` in `h=`, Orbit's later insertion of that header **breaks the signature**. Content alteration breaks the body hash: **stages 6–7 (tracking) break client DKIM whenever they touch a signed part** — which is why tracking **auto-skips signed messages** (stage 6–7 gating): a DKIM-signed or S/MIME/PGP-signed submission ships untracked and its signature survives regardless of the domain's tracking setting. (If the backend cannot implement reliable signed-part detection, the fallback is the HTTP-parity behavior — tracking applies and breaks client signatures, documented — but auto-skip is the required behavior for SMTP, where signed mail is far more common.)
- **S/MIME / PGP** sign the *content octets of the signed MIME part(s)* — outer headers (Received, added DKIM, Bcc removal, absent-From insertion, oversigning concerns) are irrelevant to them. They break **only when stages 6–7 alter the bytes of the signed part** — which the auto-skip prevents; with the skip in place they always survive.
- Net rule: **tracking is the single content-altering gate, and it exempts signed messages automatically** — signed messages (client DKIM, S/MIME, PGP detected) ship untracked, so client signatures survive and Orbit's signature validates; unsigned messages with tracking enabled receive the pixel/link rewrites (a decode/re-encode of the HTML part — the documented exception to no-re-encoding). Header-name edge cases (oversigned `dkim-signature`, absent-header protection in `h=`) remain as described above. Docs must say this; tests must cover it, including oversigned, absent-header-protected, and auto-skipped-signed messages.

Forbidden: header rewriting or reordering (except the listed additions), address rewriting, MIME re-encoding or part reordering (sole exception: the stage 6–7 tracking re-encode of the unsigned HTML part), body transcoding, dropping or "normalizing" any sender header, HTML/plain-text selection on the wire.

**Missing-From insertion (stage 2):** when the client sends no `From:` header, the tenant's From-email defaults supply it. The edge inserts the default as a normal `From:` header **before persistence**, so the stored record and the wire message both carry it — the pipeline downstream never sees a headerless message. The insertion is addition-only, with the same RFC 6376 caveat as any insertion: a signature that protects the absent `from` name via `h=` breaks when the header appears. Such a signature is already invalid under RFC 6376's requirement that `From:` be signed and present, so nothing valid is lost — but the guarantee is stated generally: insertions are safe only for header names the client's `h=` list does not protect.

### SMTP session and response mapping

Session shape: `EHLO` (plaintext session advertises `SIZE 26214400`, `8BITMIME`, `PIPELINING`, `ENHANCEDSTATUSCODES` — **no AUTH**) → (`STARTTLS` → **client must re-EHLO**; the post-TLS EHLO now additionally advertises **`AUTH PLAIN LOGIN`** — per RFC 4954, password-based mechanisms are never advertised on a plaintext session, and the capability list can change after TLS) → `AUTH PLAIN|LOGIN` → `MAIL FROM` (null sender `<>` valid) → `RCPT TO`… → `DATA` → reply → `QUIT`. `RSET` and `NOOP` are accepted at any state; multiple transactions per connection are allowed.

**AUTH extension rules (RFC 4954):** only mechanisms actually implemented are advertised (`PLAIN`, `LOGIN`); an unsupported mechanism → `504`; a second `AUTH` after successful authentication → `503`; `AUTH` during an open mail transaction (between `MAIL FROM` and the `DATA` reply) → `503`.

| Scenario | Reply |
|---|---|
| AUTH / MAIL FROM before STARTTLS (TLS-only config) | `530 5.7.0 Must issue a STARTTLS command first` |
| MAIL FROM without authentication | `530 5.7.0 Authentication required` |
| Permanent credential failure (wrong username, revoked/rotated/invalid key, key without Messaging write scope — one code for all) | `535` |
| Key-lookup backend temporarily unavailable | `454` (temporary — clients keep retrying) |
| AUTH with an unsupported / unadvertised mechanism | `504` (mechanism not recognized — RFC 4954) |
| AUTH when already authenticated, or AUTH during an open mail transaction | `503` (bad sequence — RFC 4954) |
| Command line longer than 512 octets (including CRLF) | `500 Line too long` |
| AUTH attempt threshold exceeded (failed attempts per source IP) | `454 4.7.1` — temporary, with documented cool-down (see Limits; never `535`, which reads as "credentials permanently bad") |
| RCPT TO: recipient on the suppression list | `550` — refused at RCPT TO, where the client can see it |
| RCPT TO: transaction recipient cap exceeded (cap ≥ 100) | `452 4.5.3` |
| RCPT TO before MAIL FROM, or DATA before RCPT TO | `503` (bad command sequence) |
| Malformed address syntax | `501` |
| Message exceeds the advertised SIZE | `552` |
| Empty / unparseable body — **Orbit product rule**, not SMTP | `554` |
| Malformed message / unverified sender | `554` |
| DATA with zero surviving recipients (every RCPT TO was refused) | `554` — transaction refused; DATA is never accepted into the pipeline (RFC 5321 §3.3 permits `503` or `554`; Orbit uses `554`) |
| Unknown command | `500` / `502` |
| Rate-pool exhaustion (pool temporarily cannot fit every surviving recipient — atomic, decided before the body upload) | `451 4.7.1` at DATA — temporary, retry the whole transaction later; never a partial accept |
| Transaction needs more units than the pool can ever hold (small tenant override, recipients × units > capacity) | `552 5.7.28` — permanent; shrink the transaction (fewer recipients), retrying cannot succeed |
| Message-level consent failure (marketing/transactional, decided after DATA) | Whole transaction refused before `250` — permanent `554` (consent required) or temporary `451` (consent state unknown), never a subset accept |
| Connection-level limit, idle/command timeout, or shutdown | `421` (closing channel) |
| Temporary internal failure | `451` |
| Tenant-level relay disabled (key valid, SMTP switch off) | `554 5.7.1 Relay access denied` at MAIL FROM |
| Durable acceptance (≥1 surviving recipient) | `250` — every surviving envelope recipient is accepted |
| RSET / NOOP | `250` |

Rules:

- **Permanent (5xx) vs temporary (4xx) must mirror the application's error semantics** — 4xx invites retry, 5xx tells the client to stop. Map from the HTTP error catalog: `INVALID_RECIPIENT` → `550` at RCPT TO; `CHANNEL_NOT_CONFIGURED` → `554` after DATA; `RATE_LIMITED` → `451 4.7.1`; validation → `552`/`554`.
- `530`, not 3xx, for pre-TLS AUTH: on an AUTH command a 3xx reply is the SASL continuation challenge — RFC 4954's challenge code is `334` — and it would instruct the client to send credentials on the plaintext channel. The current docs' `335` is not a real code: fix the docs, don't honor them.
- After STARTTLS, discard all plaintext-session knowledge — including any commands the client pipelined behind STARTTLS; capabilities are renegotiated by the post-TLS EHLO.
- Every reply carries enhanced status codes (`ENHANCEDSTATUSCODES` is advertised) so clients can distinguish permanent from temporary conditions programmatically.

### Acceptance semantics and durability

Three stages, with SMTP replies meaning one thing:

1. **Per-recipient decisions at RCPT TO** — syntax (`501`), sequencing (`503`), suppression (`550`), **consent and every other recipient-specific eligibility check (`550`)**, transaction cap (`452 4.5.3`): decided at the command where the client can observe the result. The tenant is known after AUTH, so no recipient-specific gate needs post-DATA guesswork — nothing that can reject an individual recipient may wait for `DATA`.
2. **Transaction-level acceptance at DATA** — sender verification, **message-level consent** (marketing/transactional — unknowable at RCPT TO; failure refuses the whole transaction before `250`), **atomic rate-pool admission** (decided before the body upload starts), body gate (`554`): the same gates the HTTP path applies, same codes.
3. **Delivery** — asynchronous through the existing pipeline; `250` means *Orbit has accepted custody of the surviving recipient set*, never "recipient got it."

**Zero-surviving-recipient rule:** when every `RCPT TO` was refused, there is no transaction. The client's `DATA` command is answered `554` (no recipients — transaction refused): no message record, no persistence, no billing, no webhooks. Per-recipient refusals already told the client *which* addresses failed and *why* at `RCPT TO` — the `554` at `DATA` only closes the dead transaction. (RFC 5321 §3.3 allows `503` or `554` here; Orbit standardizes on `554` for rejection semantics.)

**Atomic rate-pool admission:** admission is decided once per transaction — **at the `DATA` command, before the message body is read** (the surviving-recipient count is known after the RCPT phase, so the upload is never spent on a transaction that will be refused) — and before durable acceptance. If the shared pool cannot accommodate **all** surviving recipients, the entire transaction is refused with `451 4.7.1` at `DATA` — Orbit never accepts a subset of recipients after `DATA`, which would violate the no-post-DATA-drops rule. A refused transaction consumes no quota; the client retries the whole transaction later.

- **Never-fits is permanent, not retryable:** if a transaction needs more units than the pool can *ever* hold in one admission (a small per-tenant override, or surviving recipients × units exceeding pool capacity by construction), `451` would tell the client to retry something that can never succeed. That case is refused with a **permanent `552 5.7.28 Transaction exceeds the tenant's rate-pool capacity`** — the client must shrink the transaction (fewer recipients), not retry it.
- **Fairness (no starvation):** atomic admission can starve large transactions while a stream of small HTTP sends keeps draining the pool. Admission therefore uses **aging/fair-share ordering** (e.g., a per-tenant FIFO lane with oldest-wait priority, or reserving a fraction of the pool headroom for the oldest waiting transaction): a transaction that has been refused `451` repeatedly gets priority as capacity frees, bounded so small sends still flow. The concrete mechanism is part of the G4 reservation design; the requirement — **no transaction waits indefinitely while smaller ones keep winning** — is tested (test matrix: Rate limiting).

**Durability boundary:** one message record per surviving recipient — the fanout model the HTTP path already uses — is durably persisted, ideally via the existing transactional/outbox mechanism, before the `250` reply is written to the socket. A crash between `250` and delivery never loses a record; a crash before `250` loses nothing the client believes was accepted.

**Raw-message storage (fanout must not multiply the payload):** the message content is stored **once per transaction as a single content blob** (the preserved RFC 5322/MIME representation); each per-recipient record **references** the blob (blob id + its per-recipient decisions) — never a per-recipient copy. A 100-recipient, 25 MiB submission stores 25 MiB once, not 2.5 GB. The blob and all recipient records are written **atomically** (one transactional write, or blob-first with records referencing it and orphan-blob cleanup on refusal): there is no state where records exist without their blob or a blob exists with half its records. The 25 MiB budget counts the blob once against the message-size limit; storage/billing implications of large blobs are a platform concern, not per-recipient multiplication.

**Duplicates and idempotency:** SMTP has no transaction identifier — a retry after a lost `250` is a brand-new transaction, and the server's internal ID is unknowable to the client. The documented contract is therefore **at-least-once**: the client may re-send and Orbit may deliver twice. Content-based dedupe (envelope + `Message-ID` + body hash within a time window) is optional hardening with a real false-positive risk — broken clients reuse `Message-ID`s, and a dedupe hit silently drops a legitimate send. A repeated `Message-ID` header alone is never treated as proof of a duplicate.

### Sender identity rules

- `MAIL FROM` (envelope sender) and `From:` (header sender) are different concepts: envelope drives bounce handling; header is the visible identity and what verified-sender enforcement checks.
- **One From rule, consistent with the HTTP pipeline.** The HTTP endpoint treats `from` as optional ("must resolve to a verified sender *when supplied*"), and the tenant's **From-email defaults** supply it when omitted. SMTP follows exactly:
  - `From:` header present → validated against the tenant's verified sending domains; unverified → `554`. A verified envelope sender never launders an unverified header sender.
  - `From:` header absent → the tenant's **From-email defaults** (Senders → From-email defaults) supply the visible sender, identical to an HTTP send without `from`.
  - `MAIL FROM` is **never** the visible sender and never a fallback for it — it is envelope-only metadata.
- The client's `MAIL FROM` is stored **as traceability metadata** and **never placed on the wire as the Return-Path** — outbound messages carry Orbit's controlled VERP bounce address, and bounce correlation runs through that return-path identifier (unless backend verification establishes another mechanism — see the backend-dependencies list in [00](00-gates-and-design-decisions.md)). With this routing, bounces return to Orbit for tracking and suppression, and SPF evaluates Orbit's own sending infrastructure.
- Multiple **separate** `From:` header fields → `554` (malformed). A single `From:` field containing **multiple mailboxes** is legal RFC 5322 when a `Sender:` field is present — Orbit deliberately restricts sends to **one visible sender**, so a multi-mailbox `From:` also gets `554`, documented as an intentional product restriction rather than a malformed-input case.

#### Worked example — envelope, Return-Path, SPF, DKIM, DMARC

Client submits over SMTP (its domain `customer.com` is a verified sending domain, with the four Domain-Setup records published):

| Field | Client sends (inbound session) | Orbit stores | Orbit puts on the wire (outbound to provider) |
|---|---|---|---|
| Envelope sender | `MAIL FROM:<reports@customer.com>` | `stored_envelope_sender = reports@customer.com` | `MAIL FROM:<bmb+msg123@bounce.customer.com>` — Orbit's VERP bounce address on the customer's return-path subdomain |
| Visible sender | `From: Alerts <alerts@customer.com>` | record `from` = alerts@customer.com | unchanged — `From: Alerts <alerts@customer.com>` |
| Recipients | `RCPT TO:<ops@example.net>` | surviving recipient set | `RCPT TO:<ops@example.net>` |

How the checks resolve for the outbound message:

- **SPF** is evaluated against the *Return-Path* domain: `bounce.customer.com` is the customer's return-path CNAME (published during Domain Setup) pointing at Orbit's bounce infrastructure, whose SPF includes Orbit's sending IPs → pass. **Verify against the Domain Setup implementation**: the docs table names the record but not its target — confirm the actual CNAME target host, and that its zone carries both the **SPF include authorizing Orbit's sending IPs** and the **MX routing bounces back to Orbit**. The plan's promise ("no extra records for SMTP") holds only if those two properties are already part of the standard Return-Path record.
- **DKIM**: Orbit signs with the verified domain's selector (`d = customer.com`) using the DKIM key published in Domain Setup → pass. If the client also signed its own message (its DKIM header is preserved per the integrity boundary, subject to the tracking caveat), both signatures travel; each receiver validates what it knows.
- **DMARC — stated explicitly:** alignment is satisfied by **DKIM**: `d = customer.com` matches the `From:` domain exactly, so DMARC passes under both relaxed (`adkim=r`, the default) and strict (`adkim=s`) modes. SPF alignment is **relaxed-only**: `bounce.customer.com` shares the organizational domain `customer.com`, which satisfies `aspf=r` but **not** `aspf=s` (strict SPF would require the return-path to be exactly `customer.com`, which VERP bounce routing cannot do). A policy with `aspf=s` still passes on the strength of the aligned DKIM signature — DMARC needs one aligned pass, and DKIM guarantees it. This is why the envelope rule (Orbit's VERP return-path) is safe for every DMARC policy the customer is likely to publish.
- **Bounce routing**: if the provider later reports a failure, the DLR/bounce returns to `bmb+msg123@bounce.customer.com`, which MX-points back at Orbit — Orbit matches the embedded message ID, marks the record failed, fires `message.failed`, and applies suppression — the loop from the Webhooks section ([03-webhooks.md](03-webhooks.md)), closed.

If the tenant never verified a custom domain, the same flow runs on the platform domain (`orbit.devotel.io`, platform-signed, From-email defaults supplying the visible sender).

### Security controls

| Control | Requirement |
|---|---|
| TLS | Required on all ports; no plaintext AUTH (`530`) |
| Post-STARTTLS state | All plaintext commands buffered before the handshake are discarded (command injection) |
| Line endings | Bare LF in commands or DATA rejected (SMTP smuggling) |
| Authentication | API key only; username `apikey`; `535` for permanent failures, `454` for temporary ones |
| Anonymous relay / open relay | Disabled; unauthenticated transactions rejected (`530 5.7.0`) |
| Connection limits | Global and per-IP; abusive sessions closed (`421`) |
| Concurrent sessions (pre-AUTH) | Bounded per IP and globally — applies before authentication, where IP is the only identity available |
| Authentication throttling | Brute-force resistant (delay/lockout per source) |
| Session timeouts | Idle and per-command timeouts (`421` on expiry) |
| Message size | Explicit configurable maximum, advertised via SIZE → `552` |
| Recipient count | Explicit per-transaction cap, at least 100 (RFC 5321) → `452 4.5.3` |
| Secrets | API keys and credentials never logged |

**MIME parsing is an attack surface** — enforce in v1: maximum message size, maximum MIME nesting depth, maximum header size, maximum attachment count/size, and parsing time/memory caps. Pathological input is rejected (`554`), not merely observed.

**Internal (edge ↔ application) security** — the SMTP edge is a privileged internal service; its channel to the application is hardened like production infrastructure:

| Control | Requirement |
|---|---|
| Service authentication | The edge authenticates to the application with a dedicated service credential (mTLS or signed service token) — never a tenant API key, never anonymous internal calls |
| Transport | Internal calls over TLS (or an otherwise isolated private network path); internal endpoints are not publicly routable |
| Tenant binding | The tenant is fixed once at AUTH (from the validated key) and carried as read-only context on every internal call — no endpoint accepts a caller-supplied tenant id, so cross-tenant access is structurally impossible |
| Authorization | The key-validation service returns scope + mode + tenant id with the validation result; the edge enforces Messaging-write before accepting a transaction and never performs an internal call for a key that failed validation |
| Revocation propagation | **Revocation/rotation events from the key service are mandatory** — the edge subscribes and evicts immediately, so a revoked key fails on the next SMTP AUTH attempt with no service restart (the acceptance criterion). **Revocation must also close live sessions:** on the revocation event the edge terminates every open session authenticated with that key (`421`), and as a consistency fallback each `MAIL FROM` on an open session re-validates the cached key — a long-lived session carrying many transactions must never keep sending after its key is revoked. The ≤ 60 s TTL exists only as a consistency fallback for a missed event, never as the primary mechanism |
| Secret hygiene | The edge stores only the key's id and a hash/prefix for logging — never the full key; validation happens via the key service |

**Limits — proposed defaults (configurable; ops confirms before launch):**

| Limit | Default | Reply when exceeded |
|---|---|---|
| Message size (RFC 1870 measurement) | **26,214,400 octets (25 MiB)** — message octets including CRLF pairs, **excluding the terminating dot and dot-stuffing**; this exact number is used for the EHLO `SIZE` advertisement, the client's `MAIL FROM … SIZE=` declaration, server-side DATA enforcement, and attachment encoded-size accounting | `552` |
| Attachments per message | 20 | `552` |
| Attachment payload size | 20 MB **of encoded MIME part size** (≈15 MB decoded when base64) — measured on the same RFC 1870 basis as the message limit: encoded attachments + body + headers must jointly fit the 26,214,400-octet budget, and the attachment gate catches attachment-heavy messages before full upload | `552` |
| MIME nesting depth | 10 | `554` |
| Header block total | 256 KB | `554` |
| RFC 5322 message-header line | 998-char guidance; accepted up to the DATA-line ceiling below — no separate 32 KB rejection | (see DATA lines) |
| DATA text line | RFC 5321 guidance: 1,000 octets including CRLF (excluding dot-transparency); Orbit accepts up to a configurable ceiling (default 10,000 octets) — real HTML mail routinely exceeds the RFC floor, so the floor is guidance, not a rejection trigger; beyond the ceiling → `552` [PROPOSED] | `552` |
| SMTP command line | 512 octets including CRLF (RFC 5321) | `500 Line too long` |
| SMTP reply line | ≤ 512 octets including CRLF — server-side invariant; Orbit's own replies always conform | — |
| Recipients per transaction | 100 (the RFC floor; raise per tenant if needed) | `452 4.5.3` |
| Idle session timeout | 5 minutes | `421` |
| Per-command timeout | 2 minutes | `421` |
| Concurrent sessions (post-AUTH) | Bounded **per authenticated key** (default 10 per key) and per tenant — hosted CRMs sending for many tenants from one address share an IP and must not be squeezed by a per-IP post-AUTH cap | `421` |
| AUTH attempts | Sliding-window throttle per source IP **and per key**: 5 failed attempts per minute. **Success does NOT reset the failure counter** — a window of failures with an occasional success still counts as an attack pattern; the counter decays only with time (each failure ages out after the window). The per-IP cool-down applies only to *unauthenticated* guessing (pre-AUTH failures); once a key **authenticates successfully**, that key's sessions are exempt from further per-IP cool-downs — so one bad client behind shared NAT does not lock out the office, while a credential-stuffing attempt cannot launder itself with one valid login | `454 4.7.1 Too many authentication attempts — retry later`, with a 15-minute default cool-down documented in the reply text; see NAT note below |
| Message rate | Shared per-tenant pool, 200 msg/min default (no separate SMTP quota) | `451 4.7.1` |

**NAT note (scoped, not blanket):** the per-IP AUTH throttle is a *pre-authentication* control — it counts failed AUTH attempts per source IP in a sliding window and success never resets it (an attacker with one valid key cannot reset the counter by logging in). Legitimate traffic behind shared NAT is protected **not** by resetting on success but by scoping: after a key authenticates successfully, its sessions proceed regardless of the IP's failure history, and the post-AUTH session cap is per key (not per IP). Worst case for an office behind NAT is that *new* AUTH attempts from that IP are slowed while an attacker is actively guessing from it — already-authenticated sessions are unaffected. A persistent `454` after a cool-down indicates a genuinely stuck client, not a credential problem — which is exactly the semantic `535` would have gotten wrong.

**DATA-transfer timeout:** the 2-minute per-command timeout governs protocol commands (EHLO, MAIL, RCPT, the DATA command line itself). The message-body transfer following the `354` reply is governed by a **separate DATA-transfer policy**: the declared/measured `SIZE` (upload refused `552` the moment the octet count exceeds the advertised maximum) plus a configurable transfer timeout scaled to size — a 25 MiB upload must not be killed by the command timeout, and an endless trickle must not hold a session forever (default: transfer budget of N octets/second floor or M minutes for a full-size message, ops-confirmed).

### Rate limits and retries

- Shared per-tenant pool with HTTP sends (default 200 msg/min on email; per-tenant overrides apply).
- **Counting is unambiguously per surviving recipient** — the same fanout unit the HTTP multi-recipient path uses: one transaction with N surviving `RCPT TO` consumes N pool units; RCPT-refused recipients consume none. (Not per SMTP transaction, not per connection.)
- Rejected (suppressed/invalid/unverified-sender) messages consume no quota.
- Pool exhaustion → `451 4.7.1` with retry guidance — decided at the `DATA` command, **before the body upload starts**, so the client never uploads a message that is refused after the transfer; the recipient cap → `452 4.5.3`; a transaction that can never fit the pool → permanent `552 5.7.28`; connection-level throttling → `421`. Never a temporary rejection for a permanent condition.
- **Atomic reservation mechanism (engineering design requirement, backend-owned):** admission is an atomic reservation against the shared HTTP/SMTP pool, not a read-then-increment: (1) count the surviving recipients; (2) check the never-fits bound (transaction > pool capacity → permanent `552 5.7.28`); (3) atomically reserve that many units (check-and-set); (4) reservation failure → the whole transaction is refused `451 4.7.1`; (5) persist all recipient records and the outbox work durably; (6) commit the reservation with the durable acceptance, or release it if persistence fails. Concurrent HTTP and SMTP submissions must **provably** not oversubscribe the shared pool — verification gate G4, whose design deliverable also includes the fairness/aging rule (no starvation).
- **Billing mirrors the HTTP multi-recipient model exactly.** The HTTP endpoint fans a multi-recipient send out per recipient (`to`/`cc`/`bcc` arrays within one request's rate limit; per-recipient outcomes in the 207 multi-status response — a `succeeded` row per dispatched recipient, failed rows never billed). SMTP is the same contract in protocol form: each surviving `RCPT TO` becomes one dispatched recipient — one message record, one billed unit at the standard email rate, one pool unit — and each RCPT-refused recipient is the failed-row equivalent: never dispatched, never billed. Duplicate envelope recipients are deduplicated (one record, one unit — see design decision 6 in [00](00-gates-and-design-decisions.md)). Example: one transaction with three `RCPT TO`, one suppressed (refused `550` at RCPT) → two records, 2 × the email rate, 2 pool units, zero for the suppressed address. There is **no separate SMTP price** (see Notes in [04-docs.md](04-docs.md)).

### Test mode over SMTP

Test mode follows the platform's key-prefix rule (SMTP has no equivalent of the `X-Test-Mode` header, and none is added — matching the HTTP rule that a key's prefix overrides request headers):

- A `dv_live_sk_…` key is **always live** — provider called, delivered, billed.
- A `dv_test_sk_…` key is **unconditionally sandbox**: the same gates run (sender verification, suppression, recipient cap — identical outcomes, so clients test real behavior), but the message **terminates at `test_sent`**: no provider call, no delivery, **no balance movement**, and the message record carries the test-mode marker, exactly like an HTTP test send.
- **The SMTP reply for an accepted sandbox submission is a normal `250`** — indistinguishable on the wire from a live acceptance, so client configuration and retry logic behave identically in both modes. Sandbox-ness is server-side (record status), never a protocol-reply difference.
- **Webhooks:** per the test-mode event table in [03-webhooks.md](03-webhooks.md) — delivery-lifecycle events (`message.delivered`, `message.failed`) do not fire for test messages, and the `message.sent` behavior for `test_sent` records must match the HTTP Test Mode implementation exactly (backend confirms; see the backend-dependencies list in [00](00-gates-and-design-decisions.md)).
- **Rate pool still applies** to test submissions (they share the per-tenant pool) — sandboxing must not become a free-traffic loophole.

### Attachments and MIME mapping

- Attachments map into the same attachment model as the HTTP endpoint (filename, content type, content) — decide the size cap explicitly and enforce it at the SMTP layer (`552`).
- The wire message is never rebuilt: `multipart/alternative`, `multipart/mixed`, and `multipart/related` structures pass through untouched (see Message integrity). Part selection (HTML over plain text) applies only to the stored message record's body, matching HTTP selection so the two paths can't drift in what the record shows.
- Unsupported content — an empty or unparseable body → `554`, labeled as an Orbit product rule (SMTP permits an empty body; this gate is a deliberate platform choice).

### Deployment and operations

- DNS: a DNS-only (TCP-passthrough) record for `smtp.orbit.devotel.io` to the edge listener — an HTTP reverse-proxy/CDN proxy path does not carry arbitrary SMTP TCP traffic. Verify resolution from public resolvers before launch.
- TLS: certificate provisioning and automated renewal for all three ports.
- Network: firewall rules limited to the SMTP ports; health-check endpoint for the load balancer / supervisor.
- Lifecycle: graceful shutdown with connection draining on deploy (in-flight sessions get `421` and clients retry); bounded startup; documented restart policy.
- Observability: metrics for submissions, replies by code (especially `530`/`535`/`454`/`550`/`554`/`4xx`), active sessions, AUTH failures, parse rejections, queue depth; alerting on error-rate spikes and edge liveness.

### Session correlation and logging

Every SMTP connection gets a **session ID** (UUID) at accept time; every accepted message carries it, so support can walk from a panel message record to the exact protocol session and back.

- **Per-session summary log** (written at connection close): session ID, peer IP, EHLO hostname, TLS protocol/cipher, AUTH result with the **key id** (never the key), command/reply counts, total bytes in/out, duration, and the closing condition (QUIT / timeout / limit / error).
- **Live active-session gauge** — a runtime counter, distinct from the close-time log: incremented when a connection is accepted, decremented on **every** close path (QUIT, timeout, error, graceful shutdown), and reconciled after process restart. The usage panel's active-session figure reads this gauge; the summary log remains the historical record.
- **Per-transaction log** (one per accepted message): session ID, tenant id, message ID, envelope sender, per-recipient decisions with reply codes, durable-persistence timing relative to the `250`, and the final reply.
- **Privacy:** logs carry metadata and reply codes only — never API keys, AUTH payloads, message bodies, or header contents beyond the addressing fields above.
- **Retention and access** follow the platform's standard log pipeline; the edge emits structured events (not free-form lines) so the usage-panel counters in [02-panel.md](02-panel.md) are aggregations of these logs.

### Test matrix (master — single source; parts 02–04 add no separate matrices)

| Category | Essential tests |
|---|---|
| Protocol | EHLO/HELO, STARTTLS + re-EHLO, implicit TLS (465), AUTH PLAIN/LOGIN, bad sequencing (503s), RSET, NOOP, null sender, multiple transactions per connection, EHLO advertises SIZE/8BITMIME/PIPELINING/ENHANCEDSTATUSCODES |
| SMTP extensions (behavioral, not just advertised) | **PIPELINING** (RFC 2920): grouped commands accepted with replies returned in order — and the RFC's rules honored: EHLO and DATA are the **final commands in their respective groups**, and the client waits for the `354` DATA reply before transmitting the message body. **8BITMIME**: high-bit body bytes accepted and relayed unharmed. **SIZE**: EHLO advertises the actual numeric maximum; a client-supplied `SIZE=` larger than the limit is refused up front, and oversize DATA → `552` without accepting the message. **ENHANCEDSTATUSCODES**: every reply carries the dotted sub-code (`5.7.0`, `4.5.3`, `4.7.1`, …). Rule: an extension is advertised only when its actual behavior is implemented **and** these tests pass |
| Authentication | Valid, invalid, revoked, rotated keys; key without Messaging write scope → `535`; **key's IP allowlist enforced on SMTP AUTH exactly as on HTTP** (violation → `535`); test-mode key yields sandbox semantics; **revocation blocks the next AUTH attempt with no service restart (revocation-event eviction mandatory, TTL ≤ 60 s fallback)**; **revocation closes a live authenticated session mid-stream (`421` before its next MAIL FROM) — a connected client cannot keep sending after revocation; a second transaction on the same session with a still-valid key succeeds**; backend-down returns `454`, not `535`; failed-AUTH threshold per source IP returns `454` with documented cool-down (sliding window per IP+key, **success never resets the counter** — four guesses + one login + repeat stays throttled; successfully authenticated sessions exempt from the pre-AUTH cool-down — shared-NAT safe without laundering guessing); **EHLO advertises `AUTH PLAIN LOGIN` only after STARTTLS** (plaintext EHLO never advertises password mechanisms); unsupported mechanism → `504`; re-AUTH after success → `503`; AUTH mid-transaction → `503` |
| Authorization | No cross-tenant access via any header/envelope trick |
| Message integrity | DKIM-signed message verifies at the receiver; S/MIME/PGP parts intact; calendar invite intact; `Bcc:` stripped; `List-Unsubscribe` and `X-` headers preserved; `Received:` trace header added |
| Message parsing | Plain text, HTML, multipart/alternative, multipart/mixed, inline images, attachments; empty body → `554` (product rule) |
| Sender | Verified, unverified, missing `From:`, malformed/multiple separate `From:` fields, **multi-mailbox single `From:` field → 554 (intentional single-sender product policy)**, envelope different from header; null envelope sender stored correctly |
| Recipients | Suppressed → `550` at RCPT TO; **consent and every recipient-specific eligibility gate resolved before the RCPT TO reply** (no post-DATA drops of any kind); over-cap → `452 4.5.3`; invalid syntax → `501`; **duplicate `RCPT TO` for the same address → `250` but one record and one billed unit (envelope dedupe)**; DATA with zero surviving recipients → `554`, nothing created or billed |
| Message-level consent | Marketing/transactional consent gate resolved after DATA, before `250`: consent-required message to a consent-missing recipient set → whole transaction refused (permanent/temporary per reason), never a subset accept; consent-compliant transaction → normal `250` |
| Atomic rate admission | Transaction whose surviving recipients exceed the remaining pool → whole transaction refused `451 4.7.1` at DATA **before the body upload**, no quota consumed, never a subset accepted after DATA; transaction larger than the pool's total capacity → permanent `552 5.7.28`; **fairness: a large transaction repeatedly refused `451` under concurrent small-send load eventually admits (aging works — no indefinite starvation)** |
| Billing fanout | One transaction, N surviving RCPT TO → N records, N × email rate, N pool units; RCPT-refused recipients create nothing and bill nothing (mirrors the HTTP 207 per-row model) |
| Protocol integrity | Sender DKIM signature validates at the receiver after relay — **signed messages are auto-skipped by tracking regardless of the domain's tracking setting** (no per-domain disabling needed); with tracking enabled on an *unsigned* message, the only content deltas are the open pixel and rewritten links (stages 6–7, including the decode/re-encode of the HTML part) and Orbit's own signature validates; **S/MIME/PGP signatures always intact via the auto-skip** (outer header changes don't touch signed-part octets); header diff between submission and delivery shows only stages 1–10 of the reconciled transformation table; **a client-supplied `List-Unsubscribe` header is never duplicated by pipeline injection** |
| MIME representation | Original RFC 5322/MIME byte representation preserved end-to-end outside the allowed stages: part boundaries, transfer encodings, and CRLF sequence identical (parse-as-copy demonstrated per gate G1); no reconstruction or normalization anywhere in the send path; **with tracking disabled the HTML part ships byte-identical; with tracking enabled only the HTML part is decode→re-encode (documented exception), all other parts byte-identical** |
| Raw-message storage | 100-recipient × large-message transaction stores **one content blob** (byte size ≈ the message size, not × recipients) with per-recipient records referencing it; blob + records written atomically (no record-without-blob, no partial fanout state); refused transactions leave no orphaned blob (or cleanup proven) |
| Line lengths | Command line > 512 octets → `500`; every reply line ≤ 512 octets; DATA text lines within the configured ceiling (default 10,000 octets) accepted, beyond → `552`; boundary values tested at exactly the limit, one octet over, and one octet under; header block at 256 KB accepted, over → `554` |
| Client DKIM oversigning | Messages signed with oversigned `h=` lists (e.g., `h=from:from:subject:date:message-id:dkim-signature`): signature break points match the documented per-family guarantee — added `DKIM-Signature` breaks an oversigned `dkim-signature` signature; **absent-header insertions break signatures that protect the absent name via `h=`** (RFC 6376 allows signing nonexistent headers precisely to guard against their later addition — documented behavior, not a bug); `Bcc:` removal breaks only signed-`bcc` |
| Durability | Crash/restart right after `250` — message survives; outbox redelivery |
| Idempotency | Client retries after a lost `250` reply — at-least-once holds; dedupe window behavior if enabled |
| Billing | Accepted vs refused recipients; refused recipients not billed |
| Rate limiting | Pool exhaustion → `451 4.7.1` (before upload); never-fits → `552 5.7.28`; recipient cap → `452 4.5.3`; shared HTTP/SMTP pool; refused messages consume no quota; fairness under concurrent small-send load (large transaction not starved); duplicate RCPT TO bills one unit |
| Sessions & concurrency | Pre-AUTH concurrent session cap per IP (`421`); post-AUTH concurrent session cap per key (hosted-CRM pattern: many tenants behind one IP, distinct keys, all proceed); revocation mid-session closes the session |
| Security | Open-relay attempts, AUTH brute force, oversized message, deeply nested/pathological MIME, bare-LF smuggling attempt, plaintext injection behind STARTTLS |
| Reliability | Edge restart under load, provider failure after acceptance, interrupted connection mid-DATA, draining on deploy |
| Integration | Same message records, tracking, and logs as HTTP sends |
| Panel surface | Developer → SMTP page renders connection profile, toggle, usage panel (reply-class breakdown, AUTH failures, top codes, active sessions), and domain-status links; message detail shows ingress, stored envelope sender, EHLO hostname, session ID, TLS info, per-recipient outcomes; Email page shows only the Sending-authentication cross-link; ingress badge + source filter on the message list; key Last Used updates on SMTP AUTH; missing From: falls back to From-email defaults; disabled-relay refusal (554 5.7.1) shown and documented |
| DNS / domains | Verified domain sends via SMTP with no extra records; unverified domain → 554 with domain-card pointer; domain traffic lights linked from the Developer → SMTP page; daily drift regression surfaces the same as on HTTP; **Return-Path CNAME target verified (G3)**: SPF evaluation passes for the bounce domain, and a forced bounce to an undeliverable address returns to Orbit's bounce MX and marks the record `message.failed` |
| Release gate | From an external vantage: public DNS resolution on two resolvers; TCP 587/465/2525 reachable; certificate valid and matching `smtp.orbit.devotel.io` on every port; STARTTLS completes on 587 and 2525; implicit TLS completes on 465 — all green before launch |
| MUA compatibility | Thunderbird sends over 587 (STARTTLS) and 465 (implicit TLS) with a saved password; message arrives intact in real external mailboxes (consumer Gmail, Outlook.com) with DKIM = pass in Gmail "Show original", no spam-foldering, Bcc not exposed, calendar invite opens; attachments intact |
| Test mode | Test key → normal `250`, record terminates `test_sent`, no provider call, no billing; **webhook behavior per the confirmed event table (gate G2)**; live key unaffected; suppression and cap gates behave identically in both modes |
| Webhooks | Matched **live** API vs SMTP sends fire the same events (`message.sent`/`delivered`/`failed`, `email.opened`/`clicked`); payloads carry `ingress`; **test-mode SMTP webhook behavior matches HTTP test-mode event-for-event** (per the confirmed event table); RCPT refusals, AUTH failures, throttles, and disabled-relay refusals fire nothing; SMTP bounce → `message.failed` via the VERP return-path; suppression events fire identically |

Every acceptance criterion in the master "Done when" list ([00-gates-and-design-decisions.md](00-gates-and-design-decisions.md)) must trace to at least one test here.
