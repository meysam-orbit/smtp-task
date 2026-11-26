# Part 02 — Panel
# Task: Implement the SMTP Relay (smtp.orbit.devotel.io)

> Scope of this part: everything user-facing in the Orbit panel — the Developer → SMTP surface, the Email message list/detail changes, credentials and permissions UX, and the customer-facing From/custom-domain product decisions.
> Design decisions and gates: [00-gates-and-design-decisions.md](00-gates-and-design-decisions.md). Panel behavior tests live in the master test matrix, rows "Panel surface" and "DNS / domains" ([01-relay-core.md](01-relay-core.md)).
> **Depends on:** backend dependency 1 (`ingress` field) from [00](00-gates-and-design-decisions.md); gate G1 verdicts affect some copy.

## Panel surface and product decisions (Orbit UI)

Benchmarked against the established email/CPaaS platforms (SendGrid, Mailgun, Postmark, SES): none of them models SMTP as a separate channel — it is a second ingress into one email product.

### Placement: Developer → SMTP is the single surface (no Email-channel tab)

The panel already ships **Developer → SMTP** (Developer | SMTP Relay) with the connection profile (host, ports 587/465/2525 with copy buttons, region note), the authentication card (`apikey` sentinel, password = any active API key, Manage-API-keys deep link), a pre-filled swaks test command, and a "Coming soon" callout. This page is the canonical SMTP surface — **do not add a second one**:

- **One config surface, no drift.** A duplicate tab on the Email channel page would show the same connection profile in two places — the same "two config surfaces" risk the architecture section rules out for the backend, applied to the UI.
- **Audience match.** The person wiring up a legacy SMTP system acts as an integrator; the docs page links straight here, and the Developer section is that audience's home.
- **Platform precedent.** Org-level credentials deliberately live in one home (API Keys in Settings, with a Developer redirect); Postmark and SendGrid likewise keep SMTP settings out of the operational message UI.
- **Taxonomy, unchanged.** The Channels grid is a *media* taxonomy — SMS, MMS, WhatsApp, Email, RCS, Viber, Telegram. An SMTP-sent message **is email**: it counts in Email's Sent/Delivered, appears in the Email message list, and shares Email providers, suppression, billing, and domains. No new card there either.

**What launch adds to the Developer → SMTP page** (all of it lives on this page):

- Remove the "Coming soon" callout.
- **Enable/disable toggle** — tenant-level relay switch. When disabled, keys still authenticate, but `MAIL FROM` is refused with `554 5.7.1 Relay access denied`, and the page states it.
- **Usage panel** — last-24h submissions/errors and a link to the Email message list pre-filtered by source = SMTP.
- **Domain status** — link (or embedded traffic lights) to the verified sending domains in the Sending authentication section, so an admin can confirm the domain is ready before touching the client config.

**What the Email channel page gets** (light entry points, not configuration):

- A small **"SMTP relay" link** inside the existing Sending authentication section, and the Email card on the Channels grid links to the Developer page — discovery without duplication.
- The **ingress badge + source filter stay on the message list** (message data, not SMTP config) per the subsection below.

**Usage panel and message-detail expansion.** The usage panel on the Developer → SMTP page shows, for the last 24h (with a 7-day trend): accepted submissions, refusals broken down by reply class (2xx / 4xx / 5xx), authentication failures, top reply codes (`530` / `535` / `550` / `554` / `452`), suppressed-recipient refusals, and active-session gauge. Under the **one-record-per-surviving-recipient** model (design decision 6 in [00](00-gates-and-design-decisions.md)), the Email message list shows one row per dispatched recipient — exactly like HTTP multi-recipient sends — and the message detail view for an SMTP-originated record shows: `ingress: smtp`, the **stored envelope sender** (`MAIL FROM`, traceability metadata), the client's EHLO hostname, the SMTP session ID, TLS protocol/cipher used, and the per-recipient decision context (accepted at RCPT TO, with the reply codes of any sibling refusals in the same transaction) — the traceability an admin needs for bounce and refusal debugging.

### Message list: ingress badge + source filter

- The message record gains **`ingress: api | smtp`** (extensible later to campaigns/builder/agents).
- List row: a small **"SMTP" / "API" badge** beside the Subject — the table already carries six columns, so no dedicated column (this is Postmark's activity-feed pattern: an inline "via SMTP" tag with filtering).
- A **Source filter** (All / API / SMTP) joins the existing direction filter row.
- Message detail: show the ingress source and the stored envelope sender (`MAIL FROM`) — the pair admins need for bounce debugging.

### Credentials: existing API keys, scoped per system

- SMTP authentication = existing org API keys: username `apikey`, password = an active key (SendGrid's exact model). **No new credential entity, no separate SMTP passwords, no "connections" management page in v1.**
- Scope: any key holding **`messages:write`** can authenticate SMTP — the same entitlement the HTTP send endpoint requires; SMTP adds no privilege. A dedicated `smtp:send` scope is deferred; the API Keys page's existing permission-scope filter already lets orgs find SMTP-capable keys.
- **SMTP authentication updates the key's Last Used timestamp + IP**, exactly like API calls — the connection-activity signal comes for free.
- Documented usage pattern (goes in the docs page and the Developer → SMTP page): **one key per source system** (`crm-prod`, `jenkins`, `erp-staging`), scoped and IP-restricted; never one shared org-wide key; never per-person keys. The SMTP audience is systems, not people — legacy CRMs, ERPs, ticketing/monitoring tools, CI/CD servers, scanners, internal relays — each configured once by an admin or developer.
- **IP allowlists are enforced on SMTP AUTH exactly as on HTTP:** a key restricted to specific addresses authenticates only from those addresses; a restriction violation is a `535` credential failure (single-code semantics preserved) [VERIFY-BACKEND: enforcement parity — the key model carries per-key IP restrictions; confirm the SMTP edge consults them].
- Per-domain SMTP credentials (the Mailgun model) are a possible v2; not needed at launch.

### Permissions: where SMTP appears in key creation

The Create-API-key dialog groups 38 scopes into resource groups (Messaging, Contacts, Voice, Video, Numbers, Webhooks, **SMPP Credentials**, …), each with a None/Read/Write triad plus expandable individual scopes. SMTP send **reuses the existing Messaging → Write scope** — it performs the same action as `POST /api/v1/messages/email` (create a message, billed identically), and one capability maps to one scope. No new permission group is added:

- **Why no "SMTP" group or `smtp:send` scope in v1:** the relay has no separate credential to manage (that design decision is the whole point — contrast the **SMPP Credentials** group, which exists because SMPP *has* its own credentials to manage), and an ingress split would double the send scope for a distinction orgs rarely need on day one. If demand appears later, an individual `smtp:send` scope under Messaging (via the existing "Show individual scopes" expander) is the natural extension point.
- **Visibility fixes (the real gap):** without these, a key creator has no way to learn that SMTP is governed by Messaging → Write, and a Contacts-only key would fail with an inscrutable `535`:
  - The **Messaging → Write scope description** becomes "Create messages via the HTTP API and the SMTP relay" (same text on the individual `messages:write` scope).
  - The **Developer → SMTP auth card** says "Password: any active API key **with Messaging write access**" — not just "any active API key"; the Manage-API-keys deep link (which already carries `action=create`) can preselect the Messaging group.
  - The **docs troubleshooting section** for `535` gains a check: "the key must hold Messaging write scope." (→ [04-docs.md](04-docs.md))
- **Mode semantics:** the authenticating key's mode (Live vs Test) determines the submission's behavior — a `dv_test_sk_…` key yields sandbox semantics (logged, not delivered), a `dv_live_sk_…` key bills and delivers; the edge does not consult the panel's UI mode switch. (Behavioral detail: [01-relay-core.md](01-relay-core.md), Test mode over SMTP.)

### From: handling and custom domains (product decisions)

- **From: is per-message** — it comes from each message's headers and cannot be static (one CRM sends receipts from `billing@` and notifications from `noreply@`). The relay validates, never pins:
  - From: present and its domain is a verified sending domain → send as addressed.
  - From: header absent → fall back to the tenant's **From-email defaults** (Senders → From-email defaults) — this tab is already the platform-wide fallback ("fallback from-name, from-address, and reply-to for every email send that doesn't override them"), satisfying the v2 fallback rule.
  - From: domain unverified → refused `554`, exactly as on the HTTP path.
- **Custom domains are inherited, nothing new to build:** verified domains (Senders → Domains, per-domain SPF/DKIM/DMARC) apply identically to SMTP submissions because both ingresses enter the same pipeline. Orbit DKIM-signs outbound with the verified domain's key; a client-signed message also passes through with its own DKIM intact (per the integrity boundary in [01-relay-core.md](01-relay-core.md)) — both signatures coexist, same as SendGrid/Mailgun/Postmark.

### DNS and custom-domain setup for SMTP (customer-facing flow)

**Customers publish zero SMTP-specific DNS records.** The relay hostname `smtp.orbit.devotel.io` lives in Orbit-operated DNS — nothing in the customer's zone points at it. Sending authorization is domain-based and shared with the HTTP path, so a customer configuring SMTP sets up exactly the records the Email channel already requires:

| Record | Type | Purpose | Why SMTP cares |
|---|---|---|---|
| DKIM | TXT | Email signing key | Orbit signs SMTP-originated sends with this per-domain key |
| SPF | TXT | Sender authorization | Authorizes Orbit's relay IPs to send as the customer's domain — SMTP and HTTP share the same outbound IPs |
| DMARC | TXT | Delivery policy | Customer's policy; alignment is satisfied via DKIM/Return-Path for both ingresses |
| Return-Path | CNAME | Bounce subdomain | Makes Orbit's controlled VERP return-path DMARC-aligned for SMTP-sent mail |

The flow reuses the existing Domain Setup mechanism end-to-end: customer registers the domain under **Senders → Domains**, then **creates or updates** the required records in their own DNS zone — **update, never duplicate**: SPF is *merged* (add Orbit's include mechanism to the existing SPF record — a domain must carry exactly one SPF record, so a second one would void both), and DMARC is *updated in place* if the domain already publishes one (one DMARC record per domain); DKIM and Return-Path records are typically new additions. Orbit validates with the per-record traffic-light view (Verify DNS button + daily drift re-validation), and the domain flips to verified. From that moment SMTP submissions from that domain behave identically to HTTP sends — same IPs, same signing keys, same authentication results.

- **Out-of-the-box option:** tenants that never verify a custom domain send from the platform domain (`orbit.devotel.io`, SPF/DKIM/DMARC Valid, Managed by Devotel) with the From-email defaults — over SMTP exactly as over HTTP.
- **SMTP page shows domain status:** the Developer → SMTP page links to (or embeds) the verified-domain list with per-record traffic lights from the Sending authentication section, so an admin wiring up a legacy system can see at a glance whether the domain is ready before touching the client config.
- **Unverified domain over SMTP** → refused `554` with an actionable reply/diagnostic pointing at the domain's card ("domain not verified — publish the DKIM/SPF/DMARC records in Senders → Domains").
- **Docs page** gains a "DNS requirements" section: cross-links Domain Setup, states explicitly that **no extra records are needed for SMTP**, and notes the Gmail/Yahoo bulk-sender posture applies equally to SMTP-originated sends. (→ [04-docs.md](04-docs.md))
