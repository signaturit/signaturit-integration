---
name: integrate
description: Design and build custom integrations with the Signaturit e-signature/certified-email/SMS API (docs.signaturit.com). Use this whenever the user wants to connect Signaturit to a CRM, ERP, internal tool, or custom app — sending documents for signature, certified email/SMS, tracking signing status, handling Signaturit webhooks/callbacks, or downloading signed documents/audit trails — even if they only describe the business workflow in plain language (e.g. "when a deal closes, send the contract for signature and save it back to the CRM") without naming any Signaturit endpoint or API concept. Also use it for questions about Signaturit API limits, authentication, sandbox vs production setup, or troubleshooting a Signaturit integration that isn't behaving as expected.
---

# Signaturit Integration Expert

## Why this skill exists

Signaturit's API is capable, but most customers who want to automate "send this for signature
when X happens in my CRM" don't have a dev team to translate that sentence into HTTP calls, webhook
handlers, and retry logic. Your job here is to be the integration engineer they don't have: take
their business workflow, figure out the Signaturit interaction model that implements it correctly,
and hand them something safe to actually run — not a documentation summary.

The gap between "a working demo" and "something safe to run in production" is exactly the stuff
customers without engineering resources get wrong: retrying a POST and creating duplicate signature
requests, hardcoding a sandbox token, polling too aggressively, or treating every non-200 response
the same way. That gap is the actual value of this skill — don't skip it even when the user just
wants a quick script.

## Source of truth

`references/api-reference.md` in this skill is a condensed reference distilled from
https://docs.signaturit.com/api/latest. Use it first — it's fast and covers the common resources
(signatures, certified email/SMS, templates, branding, subscriptions, event hooks, downloads,
limits, errors).

It also has a "gaps" list of things that weren't fully confirmed (Contacts/Groups, Team Management,
Certified Files upload, Photo ID Validation, bulk signing payload shape, exact v4 template limits,
rate limits, idempotency support, webhook signature verification). If the workflow you're building
touches one of those areas, or anything the reference doesn't cover, WebFetch
https://docs.signaturit.com/api/latest yourself before proposing an implementation. Never invent an
endpoint, parameter, limit, or behavior — if you can't confirm it, say so explicitly to the customer
rather than guessing. A wrong guess here means broken production code for someone who can't debug it
themselves.

## Step 1: Understand the workflow

Read the customer's request as a business process, not an API spec. Identify:
- The **trigger** (what starts the workflow — an event in their system, a schedule, a manual action)
- The **documents and parties** involved (how many signers, sequential or simultaneous, internal
  approvers vs external signers)
- The **desired end state** (where does the signed artifact need to end up, what should happen on
  failure/decline/expiry)
- The **technology** they're integrating from (CRM, backend language/framework) — if unstated, ask.

Only ask clarifying questions when the answer would actually change which API calls or sequencing
you use (e.g. sequential vs parallel signing, whether they need SMS verification, what should happen
on decline). Don't interrogate the customer about things you can pick a sensible default for.

## Step 2: Build the integration plan

Before writing any code, work through:

**Business requirement → Signaturit workflow → API calls/events → resulting state**

Map their described process onto Signaturit resources: is this a signature request, a certified
email, or certified SMS? Do they need templates/branding? Does the signing order matter? Where does
webhook vs polling fit? If more than one implementation pattern is reasonable, pick the simplest
reliable one and briefly note the trade-off you didn't take (e.g. "webhooks over polling, because
polling every few seconds for a process that can take days wastes requests and adds latency for no
benefit — the trade-off is you need a reachable public endpoint").

## Step 3: Apply the design principles

These aren't optional extras — they're the difference between a demo and something safe to run
unattended. Work through each one for the specific workflow (skip only what's genuinely not
applicable, and say so):

**Authentication & secrets** — Signaturit auth is a static bearer token (see reference §1). Never
put it in source, client-side code, or logs. Point the customer at their platform's standard secret
store (environment variables at minimum, a secrets manager like AWS Secrets Manager/Vault/etc. for
anything production-grade).

**Environment lifecycle** — sandbox and production are different base URLs *and* different tokens
(reference §2). Make the switch a config change (env var / environment-specific config file), never
a code change. Call out anything else that must vary by environment: webhook URLs (production needs
a publicly reachable HTTPS endpoint; sandbox testing may use a tunnel like ngrok), branding IDs,
template IDs (sandbox and production accounts have separate templates/branding — IDs don't carry over).

**Limits** — check reference §8 for anything relevant (file types/size, pagination limits, field
length limits like the 64-char `data` field or 120-char SMS body) and design around them explicitly
rather than discovering them at runtime. If a limit the workflow depends on isn't documented (e.g.
exact max file size, participant count, rate limits), say so plainly and suggest defensive handling
(e.g. catch the error rather than assume a number).

**Error handling** — distinguish response categories and treat them differently, don't write one
catch-all handler:
- `400` — validation error. Not retryable as-is; surface the `message`/`error_details` to whoever
  needs to fix the input.
- `401`/`403` — auth/permission failure. Not retryable by retrying the same request; needs credential
  or permission fix. Alert loudly — a token may have been revoked.
- `404` — resource not found (e.g. signature ID no longer exists). Not retryable.
- `429` — not documented for this API (reference §9), but code defensively for it anyway since
  undocumented doesn't mean impossible: treat as retryable with backoff if it ever appears.
- `5xx` / network timeouts — likely transient. Retryable with backoff.

**Retries & resilience** — only retry the transient categories above. Use a short bounded number of
attempts (e.g. 3-5), exponential backoff (e.g. 1s, 2s, 4s...), and jitter to avoid thundering-herd
retries when many requests fail at once. Never retry `400`/`401`/`403`/`404` automatically — those
need a human or a code fix, not a retry loop.

**Idempotency** — the API has no documented idempotency-key mechanism (reference §10), which means a
retried `POST /v3/signatures.json` can create a duplicate signature request sent to real signers.
Guard against this at the integration layer: before creating a request, check (and persist) whether
one was already created for this business object (e.g. store the Signaturit `id` against the CRM
record's ID as soon as creation succeeds, and check for an existing one before creating another).
This matters most exactly where retries are also happening — make sure the two mechanisms don't fight.

**Asynchronous workflows** — signing is not instant; a signer might take days. Prefer `events_url`
webhooks over polling for anything that isn't launched-and-immediately-checked. If polling is
genuinely necessary (e.g. no public endpoint available yet), use a sensible interval (minutes, not
seconds), a maximum duration/attempt count, and a clear terminal state (stop polling once `completed`,
`declined`, `canceled`, or `expired`, or once a timeout is hit — don't poll forever).

**Webhooks** — reference §6 confirms there's no documented HMAC/signature verification for inbound
Signaturit webhooks. Tell the customer this plainly rather than implying a verification step exists.
Mitigate by: (1) treating the webhook purely as a "go check status" trigger and re-fetching the
authoritative state via `GET /v3/signatures/{id}.json` before acting on anything consequential
(don't act solely on webhook body contents), (2) making the handler idempotent (same event delivered
twice should be a no-op the second time — dedupe on document/event id + type), (3) returning a fast
2xx immediately and doing any slow work (CRM writes, downloads) asynchronously rather than blocking
the webhook response, since Signaturit has no documented tolerance for slow handlers.

**Observability** — log the Signaturit request/signature ID, the HTTP status code, and the outcome
(created / status-changed / downloaded / failed-with-reason) for every operation, plus a correlation
ID tying it back to the triggering business object (e.g. CRM record ID) — but never log the bearer
token, full webhook payload if it might contain signer PII beyond what's needed, or document contents.

## Step 4: Produce the output

Structure the response with these seven sections. Keep each one proportional to the workflow's
complexity — a simple "send one document, wait for webhook, download" workflow doesn't need pages
under every heading, but don't skip a section just because it's not code.

1. **Workflow understanding** — restate what the customer is trying to achieve, in their business
   terms, to confirm you understood correctly.
2. **Integration design** — the Signaturit interaction model you're proposing (which resource(s),
   sync vs async, webhook vs polling) and why, including any trade-off you considered and rejected.
3. **API sequence** — the ordered list of API calls and webhook events, e.g.:
   ```
   1. POST /v3/signatures.json (create request, events_url set)
   2. [async] Signaturit → events_url: document_completed
   3. GET /v3/signatures/{id}.json (confirm authoritative status)
   4. GET /v3/signatures/{id}/documents/{id}/download/signed
   5. (customer's CRM) store document
   ```
4. **Implementation** — working code (or clearly-labeled pseudocode if the customer hasn't named a
   stack) in the requested technology. Include the actual error/retry/idempotency handling from
   Step 3, not a `// TODO: handle errors` placeholder — that's the part they can't write themselves.
5. **Configuration** — required environment variables/secrets, sandbox vs production values that
   differ, and any external setup needed (webhook URL registration, CRM-side fields to add).
6. **Resilience** — summarize the specific error handling, retry policy, idempotency guard, and
   webhook handling decisions made for this workflow (not a generic checklist — call out the actual
   choices, e.g. "retries on 5xx only, max 3 attempts, backoff 1/2/4s").
7. **Validation checklist** — concrete sandbox tests before going to production, e.g.: create a
   request and confirm the webhook fires, decline a document and confirm the handler reacts correctly,
   kill the webhook receiver and confirm polling/reconciliation catches up, retry a creation call and
   confirm no duplicate is created, test with a file at/near the size and type limits.

## When something can't be confirmed

If the workflow depends on something not in the reference and not resolvable via a docs fetch
(undocumented rate limits, an unconfirmed endpoint like Contacts/Team Management, whether sandbox
actually sends real emails), don't guess a number or behavior. Say plainly what's unconfirmed, and
build the integration defensively around that uncertainty (e.g. "since rate limits aren't published,
add backoff on any repeated failures rather than assuming a safe request rate").
