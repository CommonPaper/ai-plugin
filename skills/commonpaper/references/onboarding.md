# Account Provisioning and Onboarding

**Which connection to use:** the Common Paper MCP connector is signed in as the user's existing Common Paper login, so it always acts on that user's current organization. Provisioning creates a *different* organization that only its new API key can reach. After provisioning, run everything for the new account (including Onboarding Mode) through the REST API with that key, as described in `rest-api.md` and `templates.md`. For onboarding an account the user is already signed in to, the MCP tools are the simpler path.

## Provisioning a New Account (Agentic Signup)

Use this flow when the user asks to create a brand-new Common Paper account from scratch via the API — phrases like "provision a new account", "create a fresh Common Paper account", "sign me up for Common Paper", or "create an agentic signup account for X". This is for net-new orgs, not for using an existing one.

The flow is two unauthenticated public endpoints. No prior API key is needed.

### Step 1: Mint a provision key

```bash
curl -s -X POST "https://api.commonpaper.com/v1/keys" \
  -H "Content-Type: application/json" \
  -d '{"contact_email":"YOUR_CONTACT_EMAIL","integrator_name":"YOUR_CLIENT_NAME"}'
```

`contact_email` is the operator/agent's email (a contact point if the key needs follow-up), not the email of the user being signed up. Throwaway domains (mailinator, tempmail, guerrillamail, 10minutemail, throwawaymail) are rejected with 422.

`integrator_name` is optional metadata for attribution. Set it to the client you're running in, in lowercase kebab-case, for example `claude-code`, `claude-cowork`, `cursor`, `grok-build`, `gemini-cli`, or `codex`.

The response is JSON with a one-time `key` (used as a Bearer token in Step 2), plus `key_last_four`, `contact_email`, and `integrator_name`. Capture the `key` — it's not shown again, and once redeemed it's revoked.

Rate limits: 5 req/minute and 50 req/day per IP. If the user is testing the flow repeatedly, expect 429s after the first few.

### Step 2: Redeem for an account

```bash
curl -s -X POST "https://api.commonpaper.com/v1/accounts" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_PROVISION_KEY" \
  -d '{"email":"NEW_USER_EMAIL","name":"NEW_USER_NAME","org_name":"NEW_ORG_NAME"}'
```

All three fields are required. The new user is created as an admin of the new org. The response includes `api_key` (the production-bucket key the agent uses from here on), `user_id`, and `organization_id`. We also queue up two emails to the new user's address: One is a verification email and the other is a password reset email.

Possible non-201 responses:
- **404 Not Found** — provision key doesn't exist
- **410 Gone** — provision key has already been redeemed (single-use; this is the most common confusion if you try to re-run Step 2 with a stale key)
- **409 Conflict** — email is already in use
- **422 Unprocessable Entity** — required field missing
- **502 Bad Gateway** — an upstream service failed; wait a moment and retry

### Step 3: Install the new API key for the rest of the skill

The new account's API key needs to be in `cp-api-token` for the skill to use it. If a token is already saved, back it up first.

```bash
if [ -f ~/.config/commonpaper/cp-api-token ]; then
  cp ~/.config/commonpaper/cp-api-token ~/.config/commonpaper/cp-api-token.bak
fi
printf '%s' 'NEW_API_KEY' > ~/.config/commonpaper/cp-api-token
chmod 600 ~/.config/commonpaper/cp-api-token
```

Validate with the standard test call (`GET /v1/agreements?page%5Bsize%5D=1`) and confirm 200 before continuing.

**Tell the user** the new account is provisioned, mention that the new user will receive verification and password reset emails, and offer to restore the original token from the backup at the end of the session (or whenever they're done with the new account).

### What's immediately usable vs. blocked

After Step 2, the new account is ready for most read and setup work without further action:
- Templates: create, update, list
- Org info: read, update
- Users: read

What's still gated until the new user clicks the email verification link:
- `POST /v1/agreements/{id}/send` — and any agreement create that omits `draft: true` (since that immediately sends). Surfaces as 403 with a "plan limit" message, which is misleading — it's actually the verification gate.
- `test_agreement: true` bypasses plan limits but still requires verification before send.

Draft creation itself (`POST /v1/agreements` with `draft: true`) is not gated by verification — drafts can be created on an unverified account. Templates, org reads/writes, and user reads also work pre-verification, so Onboarding Mode (below) runs fine on a fresh account. Only the send step is blocked.

### Chaining with Onboarding Mode

A common pattern is "provision a new account and then onboard it." Run Steps 1–3 above, then transition straight into Onboarding Mode below using the new account's token. The fresh org will have zero templates, so the full Q&A is appropriate.

## Onboarding Mode

Onboarding mode guides a brand-new user through setting up their entire Common Paper account from scratch. Trigger it when the user asks to "set up my account", "onboard", or "create all my templates".

**IMPORTANT — Legal disclaimer**: Always display this at the start of onboarding:

> ⚠️ **Not legal advice.** The suggestions here are common starting points based on your answers, not legal recommendations. Have a qualified attorney review your final templates before use.

### Phase 1: Account setup

Confirm you can reach the right Common Paper account: the MCP connector for an existing account, or the saved REST token for a freshly provisioned one. Then fetch the company name with the `list-organizations` tool or `GET /v1/organizations` (`data[0].attributes.name`) — do not ask the user for it. Use this name to prefix template names in Phase 4. Then explain what onboarding will do: ask a few quick questions about their business, then create a sensible set of contract templates tailored to their answers.

### Phase 2: Q&A — show all questions, answer one at a time

Display all questions upfront so the user can see the full scope, then ask them to answer one at a time starting with the first. This lets them see how much work is involved without requiring a single long response.

Keep it conversational and brief — the user should be able to answer in under 5 minutes. Do **not** ask about legal specifics; derive those from their answers.

```
Here's what I'll need to know to set up your templates. I'll ask you one at a time — let's start with the first:

1. What does your product or service do? (1–2 sentences)
2. What state or country's law do you want governing your agreements? (This is a legal preference, not necessarily where you operate. Don't suggest a specific jurisdiction — let the user decide.)
3. What's the email of the person who will typically send and sign agreements? (Must be an existing user in your Common Paper account — probably the email you signed up with)
4. Do you sell software as a service (SaaS), on-premise/embedded software, or both?
5. Do you offer professional services? (implementation, consulting, custom dev)
6. Do you have a free trial or pilot program? (If yes, also ask: do you want this as a **standalone Pilot template**, **embedded in the CSA** as a pilot period before the subscription kicks in, or **both**?)
7. Do any of your customers work in healthcare, or does your product process health records?
8. Do you have customers in Europe, or does your product process personal data from EU residents?
9. Does your product use AI or machine learning in a way that's visible to your customers?
10. Do you have security certifications? (e.g., SOC 2, ISO 27001, HITRUST — or none yet)

I have your company name from your Common Paper account already — let's start with the first question: what does your product or service do?
```

After each answer, acknowledge it briefly and ask the next question. Once all answers are collected, move to Phase 3.

### Phase 3: Interpret answers and propose a plan

Based on the user's answers, decide which templates to create and what key settings to apply. Use this mapping as your guide — apply judgment, don't apply it mechanically:

**Always create:** NDA (nearly every company needs one)

**Create CSA if:** they sell SaaS or any subscription software

**Create Software License if:** they sell on-premise or embedded software

**Defer DPA to the wrap-up checklist** if they have EU customers or process personal data on behalf of customers. Do not create a DPA template during onboarding — the API alone can't capture the configuration these need (subprocessors, transfer mechanisms, attachment, etc.). Note in the plan that DPA setup needs a dedicated flow and add it to the post-onboarding checklist.

**Defer BAA to the wrap-up checklist** if they mention healthcare customers or processing health records. Same reason — BAA configuration is too thin via the API alone. Note in the plan and surface it in the post-onboarding checklist.

**Scope of the deferral**: this only applies to the auto-onboarding flow, where defaults are inferred from a generic Q&A. If the user explicitly asks to create a DPA or BAA template (e.g., "make me a BAA template") outside onboarding, fall back to the standard Create Template flow and create it — collect the parameters from them directly.

**Create PSA if:** they offer professional services, implementation, or consulting

**Create Design Partner if:** they have a pilot or early-access program with design partners

**Create Pilot template if:** they want a standalone pilot agreement (or "both"). When creating, set `product_description` to a one-line description of what's being piloted (derived from their answer to Q1). If they only want it embedded in the CSA, skip this template.

**Embed pilot in CSA if:** they want it bundled with the subscription. Set `include_pilot_period: true`, `include_pilot_period_duration_amount`, `include_pilot_period_duration_type` (e.g., `"day(s)"`), and `include_pilot_period_include_fees: false` (unless they said the pilot has fees) on the CSA template.

**Create Partnership if:** they work with resellers, referral partners, or co-sell partners

**Create LOI if:** they mention large enterprise deals, M&A, or complex pre-contract negotiations (otherwise skip)

**Key settings to extrapolate:**

- **Governing law / courts**: Use the state they named.
- **Liability cap (CSA)**: Default to 1× fees paid in the preceding 12 months — standard for most SaaS.
- **DPA on CSA**: Do NOT set `include_dpa: true` on the CSA template — the reference requires a URL or attachment to be meaningful, and leaving it blank creates a broken addendum. Instead, create a standalone DPA template. Note in the plan that customers needing a DPA will sign that as a separate agreement.
- **AI addendum on CSA**: Requires a PDF attachment uploaded first via the `upload-attachment_application_json` tool or `POST /v1/attachments`. If the user has an AI addendum PDF during onboarding, upload it and set `include_ai_addendum: true` with `include_ai_addendum_attachment_id` pointing to the returned attachment ID. If not, skip it and include it in the wrap-up to-do list.
- **Security policy on CSA**: Enable `include_security_policy: true` if they have any certifications. **Also set `include_security_policy_include_annual_maintenance: true`** — without this flag, the certifications below won't render on the generated agreement. Then set the relevant cert flags based on what they listed: `include_security_policy_soc2`, `include_security_policy_soc2_type2`, `include_security_policy_iso27001`, `include_security_policy_hitrust`, `include_security_policy_penetration_testing`, etc. Note: HITRUST is a certification framework; HIPAA is a regulatory requirement — treat them separately.
- **Payment terms**: Default to net-30 (30 days), annual invoicing. Adjust to monthly if they mentioned monthly billing.
- **Free trial**: Enable on CSA order form if they said yes.
- **Default signer email**: Use the email from question 3 (signer) on all templates.
- **Negotiations**: Default to `true` (allowed) for all types except DPA and BAA, which default to `false`.

**Before creating anything**, present the plan clearly and wait for explicit confirmation. Do not proceed until the user says yes — a Cowork user may be watching without having initiated the request themselves.

```
Based on your answers, here's what I'm going to create in your Common Paper account:

Templates:
- ✓ NDA — mutual, 1-year term, 2-year confidentiality, [State] law
- ✓ CSA — 1× preceding fees liability cap, net-30, annual billing[, + AI addendum][, SOC 2/HITRUST security policy]
- ✓ DPA — [if applicable]
- ✓ PSA — [if applicable]
- (skipping LOI — not needed based on your answers)
...

Key settings applied to all:
- Governing law: [State]
- Default signer: [email]
- Negotiations allowed: [yes/no per type]

Ready to create these — does this look right, or would you like to change anything first?
```

**Do not create any templates until the user explicitly confirms.**

### Phase 4: Create the templates

Before creating, verify the signer email with the `list-users` tool (`filter[email_eq]`) or `GET /v1/users` to confirm it exists in the account. If it doesn't match any user, ask the user to correct it — `default_signer_email` must be a valid account member.

**Required nested fields by type** — these must be included or the API will return an error:
- **CSA**: `template_csa_order_form_attributes.cloud_service_description`
- **Software License**: `template_software_license_order_form_attributes.software_description`
- **PSA**: `template_psa_statement_of_work_attributes.services_description`
- **Pilot**: set `product_description` so the template has context

**CSA invoice payment trap**: when v2 CSA uses `payment_process_type: "invoice"`, you must set all three of `payment_process_type_invoice_type`, `payment_process_type_invoice_amount`, and `payment_process_type_invoice_duration` inside `template_csa_order_form_attributes`. Missing the amount/duration will not error on template create, but previewing an agreement built from the template will fail. For net-30 annual: `{ invoice_type: "annually", invoice_amount: 30, invoice_duration: "day" }`.

**AI addendum**: requires uploading a PDF via `POST /v1/attachments` first, then setting `include_ai_addendum: true` and `include_ai_addendum_attachment_id` on the CSA template. Do not attempt to enable it without a valid attachment ID.

Create templates one at a time and show progress as each is created. Report each template's name and ID as it's created.

**Org ID**: Do not rely on the users endpoint for the org ID — it may return null for new accounts. Instead, read `attributes.organization_id` from any template response.

### Phase 5: Wrap-up

After all templates are created, show a summary table with direct links to each template in the app. Template app URLs follow this pattern based on type:

| Response type | App URL path |
|---|---|
| `template_nda` | `/organizations/{org_id}/nda/{id}` |
| `template_csa` | `/organizations/{org_id}/csa/{id}` |
| `template_dpa` | `/organizations/{org_id}/dpa/{id}` |
| `template_psa` | `/organizations/{org_id}/psa/{id}` |
| `template_baa` | `/organizations/{org_id}/baa/{id}` |
| `template_loi` | `/organizations/{org_id}/loi/{id}` |
| `template_pilot` | `/organizations/{org_id}/pilot/{id}` |
| `template_design` | `/organizations/{org_id}/design/{id}` |
| `template_partnership` | `/organizations/{org_id}/partnership/{id}` |
| `template_software_license` | `/organizations/{org_id}/software_license/{id}` |

Base URL: `https://app.commonpaper.com`

Present the summary like this:

| Template | Link | Status |
|---|---|---|
| NDA | https://app.commonpaper.com/organizations/{org_id}/nda/{id} | ✓ Created |
| CSA | https://app.commonpaper.com/organizations/{org_id}/csa/{id} | ✓ Created |
| ... | ... | ... |

**Template naming**: Prefix with the company name fetched in Phase 1, e.g., "Acme Corp NDA" or "Acme Corp Cloud Service Agreement".

Then display what **cannot be done via the API** and must be completed manually:

**Here's what to do next in the app (app.commonpaper.com):**

1. **Invite team members** — Add colleagues to your organization (Settings → Members).
2. **Upload your logo** — Add company branding for agreements (Settings → Branding). Requires a Startup plan or higher.
3. **Connect Slack** — Get agreement notifications and updates directly in Slack (Settings → Integrations).
4. **Set up Stripe** — If using Stripe billing workflows, connect your account (Settings → Integrations).
5. **Configure webhooks** — Set up real-time event webhooks (Settings → Integrations).
6. **Add your AI addendum** — If you have an AI addendum PDF, you can upload it via the skill and link it to your CSA template.
7. **Preview each template** — Confirm the generated language looks correct before sending live agreements.
8. **Set up your DPA** *(only if EU customers / personal data processing was indicated)* — DPA configuration (subprocessors, transfer mechanisms, attachments) needs a dedicated Q&A this onboarding flow doesn't cover yet. Set the template up in the app for now, or come back to the skill once a DPA-specific flow exists.
9. **Set up your BAA** *(only if healthcare customers / PHI was indicated)* — Same situation as the DPA: the right defaults require a dedicated Q&A this onboarding doesn't cover yet.

Only include items 8 and 9 in the actual checklist when the user's onboarding answers triggered them.

**Want to test before going live?** Set `test_agreement: true` on your first send. It sends a real email to the recipient and looks identical to a live agreement, but is marked as not legally binding and doesn't count against your monthly limit. Once you're confident everything looks right, send a live version (without `test_agreement`).

Finally, always end onboarding with:

> **Want a lawyer to review your templates?** Common Paper works with a legal partner who can review and refine your contracts.

Look up the org ID from `attributes.organization_id` on any template response (not users — that field may be null for new accounts), then provide the direct link:

`https://app.commonpaper.com/organizations/{org_id}/legal_support`
