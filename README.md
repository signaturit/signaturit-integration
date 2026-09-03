# signaturit-integration

The official [Claude Code plugin](https://docs.claude.com/en/docs/claude-code/plugins) from
Signaturit (a Namirial company) that turns Claude into a Signaturit integration expert. Describe
the business workflow you want ("when a contract is approved in my CRM, send it through Signaturit
to two signers, listen for the webhook that tells me it's completed, download the signed document,
and store it back in the CRM") and it designs and implements the integration — without you needing
to know Signaturit's API concepts up front.

## What it does

Given a plain-language workflow description, the skill:

1. Understands the business requirement and asks only the minimum clarifying questions needed.
2. Maps it onto the correct Signaturit API resources and call sequence (signature requests,
   certified email/SMS, templates, webhooks).
3. Applies integration best practices by default: secure credential handling, sandbox/production
   separation, documented API limits, error-category-aware handling, bounded retries with backoff,
   idempotency guards against duplicate signature requests, webhook-vs-polling trade-offs, and
   observability.
4. Produces a structured deliverable: workflow understanding, integration design, API call
   sequence, working implementation code, configuration/secrets guidance, resilience decisions, and
   a sandbox validation checklist.

It treats [docs.signaturit.com/api/latest](https://docs.signaturit.com/api/latest) as the source of
truth via a bundled condensed reference (`skills/signaturit-integration/references/api-reference.md`),
and explicitly flags anything it can't confirm from the docs rather than guessing — this matters
because Signaturit's API has several genuinely undocumented areas (webhook signature verification,
absolute file size limits, rate limits, idempotency keys) that are easy to hallucinate a
plausible-sounding answer for.

## About Signaturit (a Namirial company)

Signaturit was founded in Barcelona in 2013 and grew from an e-signature provider into a
comprehensive digital transaction management platform. In 2026, Signaturit Group merged with
Namirial — a pan-European trust services provider — to form Europe's leading digital transaction
management platform: 1,400+ employees across 25+ offices, serving 250,000+ companies in 90+
countries across industries like financial services, insurance, healthcare, logistics, real estate,
and public administration. The combined platform covers e-signature, identity verification,
KYC/fraud prevention, digital wallets, certified communications, electronic invoicing, qualified
archival, and time-stamping, all compliant with eIDAS regulations. Learn more at
[signaturit.com/](https://www.signaturit.com/).

## Installation

This repo is a Claude Code **plugin** (see `.claude-plugin/plugin.json`), published to the
[Claude plugin directory](https://claude.ai/admin-settings/directory) — install it from there once
it's live.

To try it locally before that, add this repo as a plugin marketplace source and install from it:

```
/plugin marketplace add /path/to/signaturit-integration
/plugin install signaturit-integration
```

Once installed, the skill is available automatically — it triggers whenever you describe a
Signaturit-related integration or workflow, even without naming any Signaturit API concept
explicitly.

## Repo layout

```
signaturit-integration/                  (plugin root)
├── .claude-plugin/
│   └── plugin.json               # plugin manifest (name, version, author, etc.)
├── skills/
│   └── signaturit-integration/
│       ├── SKILL.md              # the skill itself: triggering description + workflow
│       ├── references/
│       │   └── api-reference.md  # condensed Signaturit API reference (source of truth: the docs)
│       └── evals/
│           └── evals.json        # test prompts + assertions used to validate the skill
├── README.md
└── LICENSE
```

## Evaluation

The skill was benchmarked against a no-skill baseline on 3 realistic integration requests (CRM
webhook flow, template-based single-signer NDA, batch sequential countersigning with size/order
constraints). With the skill, all assertion checks passed; without it, the baseline model
consistently invented undocumented behavior (a per-recipient signing-order field, webhook HMAC
verification, specific file-size limits) that don't exist in Signaturit's actual API. See
`skills/signaturit-integration/evals/evals.json` for the test prompts and assertions.

## License

MIT — see [LICENSE](LICENSE).
