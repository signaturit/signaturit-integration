# Signaturit API Reference (condensed)

Source of truth: https://docs.signaturit.com/api/latest (single long anchor-navigated page).
This file is a condensed, structured summary for quick lookup. It was last verified 2026-09-03.
Anything marked **unconfirmed** or **not found in docs** was explicitly searched for and not found —
treat it as an open question to raise with the customer or verify with a fresh WebFetch of the docs
before relying on it, never assume a value.

If you need something not covered here (Contacts/Groups, Team Management, Certified Files upload,
Photo ID Validation, Massive Direct Access payload shape, exact v4 template limits, production `/v4`
base URL existence, rate limits, idempotency, webhook signature verification), WebFetch
https://docs.signaturit.com/api/latest directly and look for the relevant anchor
(e.g. `#contacts`, `#team-management`, `#massive-signing`) rather than guessing.

---

## 1. Authentication

- Called "OAuth2" by Signaturit but is actually a **static bearer access token** — not a
  client-credentials or authorization-code flow. There is no token exchange endpoint.
- Token is generated from the account dashboard (separate dashboards/accounts for sandbox and production).
- Required on every request:
  ```
  Authorization: Bearer [YOUR_ACCESS_TOKEN]
  ```
- Unconfirmed: whether tokens expire or rotate. Treat as long-lived static secrets — store in a
  secret manager, never in source or client-side code.

## 2. Environments

| | Sandbox | Production |
|---|---|---|
| Base URL (v3) | `https://api.sandbox.signaturit.com/v3` | `https://api.signaturit.com/v3` |
| Base URL (v4, templates only) | `https://api.sandbox.signaturit.com/v4` | unconfirmed — only sandbox `/v4` seen in docs |
| Credentials | separate sandbox account/token | separate production account/token |

- Switching environments = change base URL + swap access token. No other documented code changes.
- Sandbox behavioral differences from production (e.g. whether emails actually deliver, whether
  signatures are legally binding) are **not found in docs** — don't assume sandbox is a full no-op
  simulation; ask the customer or check empirically before relying on it for testing email delivery.

## 3. Core Resources & Endpoints

### Signature Requests (`/v3/signatures`)
- `GET /v3/signatures/count.json`
- `GET /v3/signatures.json` — params: `limit` (max 100), `offset` (default 0; `limit+offset` ≤ 10,000),
  `status`, `since`/`until` (YYYY-MM-DD), `ids` (comma-separated), `data` (custom field filter, e.g. `?crm_id=2445`)
- `GET /v3/signatures/{id}.json` — get one
- `POST /v3/signatures.json` — create. Key params:
  - `files` (required): PDF/DOC/DOCX documents to sign
  - `recipients` (required): array of signer objects — see Section 4
  - `name`: label for the signature request
  - `subject`, `body`: email subject/body (HTML allowed in body)
  - `delivery_type`: `email` (default) | `sms` | `url`
  - `expire_time`: expiration in days, 1–365
  - `branding_id`: custom branding to apply
  - `callback_url`: browser redirect URL after signing completes
  - `events_url`: webhook URL for event notifications
  - `data`: custom key/value pairs for later filtering (max 64 chars/field)
  - `templates`: array of template IDs or hashtags (e.g. `"#NDA"`)
  - `signing_mode`: `sequential` (default) | `parallel` — request-level, not per-recipient
  - `type`: `simple` | `advanced` (default) | `smart`
  - `reminders`: day(s) until automatic reminder
  - `cc`: array of copy-recipients (email/name) — not signers
  - `reply_to`: custom reply-to address
  - `require_file_attachment`, `require_photo`, `require_photo_id`: required uploads per document
  - `require_sms_validation`, `sms_code`: SMS-based extra verification
  - Response: `id`, `created_at`, `data`, `documents[]` (each with `id`, `email`, `name`, `status`,
    `file: {name, pages, size}`, `events[]`)
- `POST /v3/signatures/{signId}/reminder.json` — send reminder
- `PATCH /v3/signatures/{signId}/cancel.json` — cancel
- `DELETE /v3/signatures/{signatureId}` — delete
- `POST /v3/signatures/{id}/documents/{id}/signer` — change signer email (`email`, optional `name`)
- `POST /v3/signatures/{id}/name` — rename request (`name`)
- `GET /v3/signatures/{signatureId}/documents/{documentId}/sms_status.json` — SMS delivery status

### Certified Email (`/v3/emails`)
- `GET /v3/emails/count.json`, `GET /v3/emails.json` (params: `limit`, `offset`, `status`, `since`, `data`), `GET /v3/emails/{id}.json`
- `POST /v3/emails.json` — create. Params: `to`, `cc`, `bcc`, `subject`, `body`, `attachments` (PDF),
  `type`: `delivery` | `open_document` | `open_every_document`, `events_url`
- `GET /v3/emails/{id}/certificates/{id}/download/audit_trail`

### Certified SMS (`/v3/sms`)
- `GET /v3/sms/count.json`, `GET /v3/sms.json` (params: `limit`, `offset`, `status`, `since`), `GET /v3/sms/{id}.json`
- `POST /v3/sms.json` — create. Params: `recipients` (phone+name pairs), `body` (max 120 chars),
  `phone` must include country prefix
- `GET /v3/sms/{certifiedSmsId}/certificates/{certificateId}/sms_status.json`
- `GET /v3/sms/{id}/certificates/{id}/download/audit_trail`

### Templates
- `GET /v3/templates.json` — params: `limit` (up to 100), `offset`
- `GET /v4/templates` — sandbox base confirmed; production `/v4` unconfirmed. `limit`/`page` params
  (max limit conflicting across doc passes: seen both 10 and 100 — verify before relying on a specific number)
- Used in signature creation via `templates` param (IDs or hashtags like `"#NDA"`)

### Branding
- `GET /v3/brandings.json` (params: `limit` max 10, `offset`, `since`, `until`), `GET /v3/brandings/{id}.json`
- `POST /v3/brandings.json` — create (`name` max 24 chars), `PATCH /v3/brandings/{id}.json` — update
- Merge fields for custom templates: `{{sender_email}}`, `{{signer_name}}`, `{{signer_email}}`,
  `{{filename}}`, `{{logo}}`, `{{remaining_time}}`, `{{sign_button}}`, `{{validate_button}}`,
  `{{email_button}}`, `{{email_body}}`, `{{code}}`, `{{reason}}`, `{{dashboard_button}}`, `{{signers}}`
- Template types: `signatures_request`, `signatures_receipt`, `request_expired`, `pending_sign`,
  `document_canceled`, `emails_request`, `validation_request`, `pending_sign_bulk`, `signed_document`,
  `document_declined`, `request_expired_requester`, `sms_verify`, `sms_validate`

### Contacts / Groups / Team Management
- Listed in nav (Contacts: list/get/create/update/delete; Team Management: users/seats/groups/managers)
  but **full field-level schemas not confirmed** — WebFetch `#contacts` / `#team-management` before
  building against these.

### Subscriptions (account-level webhook registration, distinct from Event Hooks log)
- `GET /v3/subscriptions.json` (params: `limit` max 100, `offset`, `event`), `GET /v3/subscriptions/{id}.json`
- `POST /v3/subscriptions.json` — create (`url`, `events` array or `"*"`)
- `PATCH`/`DELETE /v3/subscriptions/{id}.json` — update/delete (detailed params unconfirmed)

### Other resources (paths captured, detail limited — verify before use)
- Enrollment & Devices: `GET /v3/account/enrollment-code`, `DELETE /v3/account/devices/{id}`
- Certified Files: `GET /v3/files/{id}.json`, `POST /v3/files.json`
- Photo ID Validation: `POST /v3/photoid/validate.json`
- Credits: `GET /v3/account/credits.json`
- Massive Direct Access (bulk signing): `POST /v3/massive-signing/recipients/bulk-request` (payload shape unconfirmed)
- Event Hooks (delivery log — see Section 6)

## 4. Participants / Signers Model

Signers go in the `recipients` array on signature-request creation. Each object:
- `email` (required unless method is sms/device)
- `name` (required)
- `phone` (required if delivery via SMS; include country code, e.g. `"34555667788"`)
- `type`: `signer` (default) | `validator` | `qualified`
- `method`: `email` | `sms` | `device`
- `device_id`: required only when `method` is `device`
- `subject`, `body`: per-recipient override of email/SMS content
- `widgets`: array of signature-field placements for that recipient

**Signing order**: controlled only by request-level `signing_mode`: `sequential` (default) or `parallel`.
The docs explicitly state the `recipients` array index does **not** determine order, and there is
**no per-recipient order field** — you cannot express an arbitrary custom signing order, only
"everyone in sequence" or "everyone at once." Unconfirmed whether sequential order follows array order.

`cc` recipients are separate from `recipients` — copy-only, not signers.

## 5. Status / Lifecycle

Per-document status values: `in_queue`, `ready`, `signing`, `completed`, `expired`, `canceled`,
`declined`, `error`.

- `GET /v3/signatures/{id}.json` returns current status per document.
- `GET /v3/signatures.json?status=...` filters by status.
- No lightweight status-only endpoint beyond fetching the full resource (except SMS's `sms_status.json`).

## 6. Webhooks / Events

Three related but distinct mechanisms:

**a) Per-request `events_url`** — set on signature/email/SMS creation. Signaturit POSTs on each
lifecycle event. Default content-type `application/x-www-form-urlencoded`; append `.json` to the URL
to receive JSON instead. Example payload:
```json
{
  "created_at": "2015-02-25T13:38:33+0000",
  "document": {
    "email": "john@signaturit.com",
    "events": [{"type": "email_processed", "created_at": "2014-10-30T09:58:01+0000"}],
    "file": {"name": "contract.pdf", "pages": 5, "size": 72218},
    "id": "29109781-f42d-11e4-b3d4-0aa7697eb409",
    "name": "John",
    "status": "ready"
  },
  "type": "reminder_email_processed"
}
```
Event types (signatures): `email_processed`, `email_delivered`, `email_bounced`, `email_deferred`,
`reminder_email_processed`, `reminder_email_delivered`, `sms_processed`, `sms_delivered`,
`document_opened`, `document_signed`, `document_completed`, `audit_trail_completed`,
`document_declined`, `document_expired`, `document_canceled`, `photo_added`, `voice_added`,
`file_added`, `photo_id_added`, `expiration_extended`.
Event types (certified email/SMS): `email_processed`, `email_delivered`, `email_bounced`,
`email_deferred`, `documents_opened`, `document_opened`, `document_downloaded`,
`certification_completed`, `sms_processed`, `sms_delivered`.

**b) `Subscriptions`** — account-level webhook registration independent of a single request
(`POST /v3/subscriptions.json` with `url` + `events`). Relationship to per-request `events_url`
(whether both fire, precedence) is **not clarified in docs**.

**c) `Event Hooks`** — a delivery/audit log of webhook call attempts, not the subscription config:
- `GET /v3/event-hooks` — params: `limit` (max 100), `page`, `date` (range object), `status`
  (HTTP codes array), `method`, `search`. Fields: `id`, `status_code`, `url`, `method`,
  `event_type`, `created_at`.
- `POST /v3/event-hooks/{eventHookId}/retry` — manually retry a failed delivery (requester/admin only).

**Security**: the docs do **not** mention any HMAC/shared-secret/cryptographic verification
mechanism for inbound webhooks. Tell customers webhook authenticity cannot be cryptographically
verified per the docs — mitigate with IP allowlisting if Signaturit publishes sending IPs, and/or
by treating the webhook only as a trigger to re-fetch authoritative status via
`GET /v3/signatures/{id}.json` rather than trusting its body outright.

**Automatic retry/redelivery**: not documented. The manual retry endpoint implies failed deliveries
are logged, but there's no documented automatic backoff schedule — design webhook handlers assuming
delivery is best-effort and reconcile via polling as a backstop for critical workflows.

**Post Message events** (embedded/iframe signing widget, not server-side webhooks): `ready`, `signed`,
`declined`, `completed` — posted to the parent window, include document/signature identifiers.

## 7. Downloading Documents

- `GET /v3/signatures/{id}/documents/{id}/download/signed` — final signed document
- `GET /v3/documents/{id}/download/uploaded` — original unsigned document
- `GET /v3/signatures/{id}/documents/{id}/download/sent` — sent version (with widgets, pre-signing)
- `POST /v3/signatures/{id}/documents/{id}/generate/audit_trail` — generate audit trail (async — generate before download)
- `GET /v3/signatures/{id}/documents/{id}/download/audit_trail` — download audit trail/certificate PDF
- `GET /v3/signatures/{id}/documents/{id}/download/attachments` — recipient-uploaded attachments
- Certified Email: `GET /v3/emails/{id}/certificates/{id}/download/audit_trail`
- Certified SMS: `GET /v3/sms/{id}/certificates/{id}/download/audit_trail`

## 8. Limits & Constraints

Only figures actually found in the docs are stated as fact; everything else is marked accordingly.

- **Pagination**: `limit` max 100 for signatures/emails/SMS lists; `limit+offset` ≤ 10,000.
  Brandings: `limit` max 10. Templates v3: up to 100. Templates v4: conflicting figures seen
  (10 vs 100) — **verify directly before depending on this number**.
- **File types** accepted for signature documents: PDF, DOC, DOCX.
- **File size**: Asolute max file size 15MB.
- **Custom `data` field**: max 64 chars/field.
- **Branding name**: max 24 chars.
- **Expiration (`expire_time`)**: 1–365 days.
- **SMS body**: max 120 chars.
- Each signature document must include at least one signature widget.
- **Rate limits**: not documented anywhere. Do not assume a specific requests/sec number — design
  clients to back off gracefully on `5xx`/timeouts rather than assuming a known ceiling.
- **Participant/recipient count limits**: not found in docs.

## 9. Error Handling

- Standard HTTP status + JSON body, e.g.:
  ```json
  { "status_code": 400, "message": "Invalid recipient list", "error_details": { "invalid email": "nonvalidemail" } }
  ```
  `error_details` appears optional/context-dependent — don't assume it's always present.
- Documented codes: `400` (validation), `401` (unauthorized), `403` (forbidden), `404` (not found),
  `500` (server error). `409`, `422`, `429` are **not documented** — treat any unexpected status
  conservatively (log and surface, don't silently swallow).
- Docs note error handling differs per official SDK/language — the raw HTTP API is the common
  denominator.

## 10. Other Integration Notes

- **Idempotency**: no idempotency-key mechanism documented. `POST /v3/signatures.json` retries
  (e.g. after a timeout) can create duplicate signature requests — the integration itself must guard
  against this (see SKILL.md's idempotency guidance).
- **Pagination style differs by version**: v3 uses `limit`/`offset`; v4 templates use `limit`/`page`.
- **Versioning**: path-based (`/v3`, `/v4`). Only templates have a confirmed v4 path; everything else
  is v3. No deprecation timeline found for v3.
- Postman collection: `https://api.postman.com/collections/38300093-69be9d3a-be87-4988-8d38-26425066b3f0`
- Support: support@signaturit.com
- **Widget validators** (form fields on documents): `email`, `phone`, `zip` (Spain), `dni` (Spain),
  `age` (18+), `iban` (Spain) — Spain/EU-centric; no documented non-Spanish equivalents.
- **Bulk signing**: `POST /v3/massive-signing/recipients/bulk-request` exists; payload shape unconfirmed.
