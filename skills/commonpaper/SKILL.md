---
name: commonpaper
description: Query and manage contracts in Common Paper. Use when the user asks about their contracts, agreements, signers, NDAs, CSAs, renewals, deal values, or wants to create/send/void/reassign agreements. Also use when the user mentions "Common Paper" or "commonpaper", asks contract questions like "how many signed contracts do I have?" or "do I have an NDA with X?", wants to create or update agreement templates, custom terms, or custom templates, wants to onboard or set up a Common Paper account, or asks about uploading attachments.
---

# Common Paper

This plugin connects Claude (or Cursor) to Common Paper in two ways:

1. **The Common Paper MCP connector** (bundled in this plugin, OAuth sign-in, no API key). This is the default path for everything it supports.
2. **The Common Paper REST API over `curl`** (API key). This is the fallback for things the connector can't do. Its details live in `references/rest-api.md`.

## Security Rules

**CRITICAL - follow these at all times:**

1. **Never display an API key or token in chat output**, in code blocks, or in explanations, unless the user explicitly asks to see it. This includes keys returned by the `generate-api-key` tool or by account provisioning.
2. **Never pass a token as a literal in a shell command** except the one-time write to the credentials file. Every REST call reads it with command substitution (see `references/rest-api.md`).
3. **When showing the user a curl command**, replace the auth header with `-H "Authorization: Bearer $CP_TOKEN"` and URL-encode brackets as `%5B` / `%5D`.
4. **Sanitize user inputs** before putting them in URLs or filters. Strip or encode `&`, `=`, `#`, `?`, newlines, backticks, `$()`, and semicolons.
5. **Confirm before any action that emails someone or changes a live agreement** (send, void, reassign, resend, invite). A Cowork user may be watching a task they didn't start, so state exactly what will happen before doing it.

## Connecting

**Check for the MCP tools first.** Look for Common Paper tools such as `list-agreements`, `get-me`, and `create-agreement` (their full names carry a server prefix that differs depending on whether they came from this plugin or from the connector directory). Use whichever set is available; they are the same server.

- **If the tools are listed and work**, call `get-me` once to confirm who is signed in, and proceed.
- **If the tools are listed but return an authentication error**, the user needs to sign in. In Claude Code: run `/mcp`, pick `common-paper`, and authenticate. In Cowork or claude.ai: Customize > Connectors > Common Paper > Connect. In Cursor: open the Customize page, find `common-paper` under MCP servers, and connect. In any other client: connect the `common-paper` MCP server (`https://api.commonpaper.com/mcp`) from that client's MCP or connector settings.
- **If there are no Common Paper tools at all**, fall back to the REST API (`references/rest-api.md`), which asks for an API key from the Integrations tab at app.commonpaper.com.

Tokens are scoped to one organization. If the user can't find an agreement they expect, it was probably sent by another organization (they were the recipient), and the API only covers agreements sent by the signed-in organization.

## When to use MCP vs REST

| Need | Use |
|---|---|
| Counts, lists, and lookups by type, status, created date, or recipient email | MCP `list-agreements` |
| Search by company name, end date, signer, deal value, or any other field, or sorting | REST filters (`references/rest-api.md`), or MCP paging with client-side filtering for small accounts |
| Create, send, void, reassign, resend, download, share agreements | MCP |
| Templates, attachments, org settings, users, invitations | MCP |
| Custom terms from scratch, custom templates | MCP |
| Porting an existing contract's text into custom terms, or uploading Word (docx) terms | REST (see `references/custom-terms-and-templates.md`) |
| Provisioning a brand-new Common Paper account | REST (see `references/onboarding.md`) |

If REST is needed and the MCP connector is connected, get a key with the `generate-api-key` tool instead of asking the user for one. If `curl` to `api.commonpaper.com` fails with a network error, the environment can't reach the REST API. Say so and do what the MCP tools allow. The same applies if you can't run shell commands at all (for example in a chat-only client): REST isn't available, so say which part of the request needs it and do the rest with the MCP tools.

## MCP Tool Reference

All tools take their inputs as `queryParams`, `pathParams` (the `id`), and/or `body`.

**Agreements**

| Tool | Use |
|---|---|
| `list-agreements` | `queryParams`: `filter[agreement_type_eq]`, `filter[status_eq]`, `filter[created_at_gteq]` / `filter[created_at_lteq]` (ISO 8601 date-time), `filter[recipient_email_eq]` / `filter[recipient_email_cont]`, `full`, `page[number]`, `page[size]`. No other filters are accepted. |
| `get-agreement` | Full details for one agreement |
| `create-agreement` | Create from a standard, uploaded PDF, or custom template (see Creating an Agreement) |
| `send-agreement` | Send a draft (draft status only) |
| `void-agreement` | Void an agreement |
| `reassign-agreement` | New recipient: `body.agreement.recipient_name`, `recipient_email`, optional `add_recipient_as_cc`, `comments` |
| `resend-agreement-email` | Resend the last notification to the recipient |
| `download-agreement-pdf` | Returns a `curl_command`; run it exactly as returned to save the PDF (the URL is single-use and expires in 5 minutes) |
| `generate-shareable-link` | Shareable review link |
| `get-agreement-history` / `list-agreement-history` | Event history for one agreement / across the org |
| `list-agreement-types` / `list-agreement-statuses` | Valid type and status values, including custom types |

**Account**

| Tool | Use |
|---|---|
| `get-me` | The signed-in user |
| `list-organizations` / `get-organization` / `update-organization` | Org name, address, notice and nudge defaults |
| `list-users` / `get-user` | Members (`filter[email_eq]`, `filter[email_cont]`); use to fill in signer name and title |
| `create-invitation` / `list-invitations` / `get-invitation` | Invite teammates (`invitation.email`, `inviter_email`, `access`) |
| `generate-api-key` | Mint a REST key for the signed-in user (treat as a secret) |

**Templates, attachments, custom terms and templates:** see `references/templates.md` and `references/custom-terms-and-templates.md`.

## Answering Questions

List responses are JSONAPI: records are in `data[].attributes`, the total count is `meta.pagination.records`, and `data[].links.agreement_url` links to the agreement in the app. List calls return compact records unless `full=true`, which adds type-specific nested data such as `csa_order_form` fees.

- **"How many signed contracts do I have?"** `list-agreements` with `filter[status_eq]=signed` and `page[size]=1`, then read `meta.pagination.records`.
- **"How many NDAs did we send this quarter?"** Add `filter[agreement_type_eq]` and `filter[created_at_gteq]`. Use the type value as `list-agreement-types` returns it.
- **"Do I have a contract with {Company}?"** The connector can't filter by company. Use the REST `_cont` filters on `recipient_organization` and `sender_organization` (see `references/rest-api.md`). If REST isn't available, page through `list-agreements` with `page[size]=100` and match both organization fields case-insensitively. Check `meta.pagination.records` first and tell the user if the scan will be long.
- **"Do I have an active NDA with {Company}?"** Signed, type NDA, matching company, and not expired (`end_date` empty or on/after today).
- **"Who signed the deal with {Company}?"** Find the agreement, then report `sender_signer_name`, `sender_signer_email`, `recipient_name`, `recipient_email`.
- **"What's our largest CSA?"** Signed CSAs with `full=true`, sorted by `ai_gmv`. `ai_gmv` can be `"0"` or null even when fees exist, so also check `summary` and `csa_order_form.fees`.
- **"What renews in the next 90 days?"** Signed agreements with an `end_date` in the window, sorted by `end_date`. REST can filter this server-side; with MCP, page and filter client-side.

## Agreement Statuses

Prefer `display_status` over `status` when presenting results.

- **Signed / active:** `signed` (display "Completed")
- **Draft:** `draft`
- **In flight:** anything not `draft`, `signed`, `voided_by_sender`, or `declined_by_recipient`, for example `sent_waiting_for_initial_review`, `in_progress`, `sent_to_recipient_for_signature`, `waiting_for_sender_signature`, `signed_waiting_for_final_confirmation`
- **Closed without signing:** `voided_by_sender`, `declined_by_recipient`
- **Outside the platform:** `sent_manually`

`list-agreement-statuses` returns the full list. `references/rest-api.md` has a description of each.

## Creating an Agreement

1. **Look up the sender** with `list-users` to get name, title, and email.
2. **Find the template** with `list-templates` (match `type`, for example `template_nda`) or `list-custom-templates`.
3. **Confirm the details** with the user before creating anything.
4. **Create it as a draft** with `create-agreement`:
   - `body.template_id`, `body.owner_email` (or `owner_id`), `body.signer_email` (or `signer_id`, a member of the owner's org), and `body.draft: true`
   - `body.agreement.recipient_name` and `recipient_email` are required; add `recipient_organization` and `recipient_title` when known
   - Optional: `cc_users`, `agreement.message`, `agreement.test_agreement`, governing law fields, billing fields, expiration fields, and type-specific overrides such as `csa_attributes` or `csa_order_form_attributes`
   - For a custom template, put cover page answers in `agreement.custom_field_values`, keyed by the template's YAML field keys (governing law answers go there too)
5. **Show the user the draft link** (`links.agreement_url`) and ask whether to send it.
6. **Send only after they confirm**, with `send-agreement`.

**Always default to `draft: true`.** Only send immediately when the user explicitly says so, and restate that the email goes out right away before calling the tool.

`test_agreement: true` sends a real email and looks identical to a live agreement, but it's marked not legally binding and doesn't count against the monthly limit. It's a good first send for a new template.

**Introducing Common Paper to a counterparty:** agreements use a Cover Page + Standard Terms structure. For counterparties new to that model, suggest an `agreement.message` such as "This is a Common Paper standard agreement. The terms are published at https://commonpaper.com/standards. Only the cover page is specific to this deal." Common Paper also publishes a one-page explainer, ["Why We Use Common Paper Standard Agreements"](https://commonpaper.com/wp-content/uploads/2022/05/Why-We-Use-Common-Paper-Standard-Agreements.pdf).

## Other Write Operations

- **Void:** always confirm first.
- **Reassign:** requires the new recipient's name and email.
- **Resend:** resends the last notification; confirm the recipient first.
- **Templates:** confirm the type and key settings, apply sensible defaults, confirm, then create. CSA templates need `template_csa_order_form_attributes.cloud_service_description`. Full parameters are in `references/templates.md`.
- **Custom terms and custom templates:** read `references/custom-terms-and-templates.md` before writing. **Never alter contract text when porting it into custom terms**: no typo fixes, no quote or dash substitutions. Collect suspected errors and propose them after publishing.

## Onboarding and Provisioning

- **Onboarding Mode** ("set up my account", "onboard", "create all my templates"): a short Q&A, a proposed plan, then template creation. Read `references/onboarding.md` and follow it closely, including the "Not legal advice" disclaimer at the start and explicit confirmation before creating anything.
- **Provisioning** ("create a new Common Paper account", "sign me up for Common Paper"): a two-step REST flow that creates a new org and API key. Also in `references/onboarding.md`. The MCP connector stays signed in to the user's existing org, so work on the new org goes through REST with the new key.

## Presenting Results

- Use tables or short summaries. Give exact counts from `meta.pagination.records`.
- Agreement lists: Agreement Type, Counterparty (`recipient_organization`), Status (`display_status`), Effective Date, End Date.
- Signer queries: Name, Email, Title, Organization.
- Format currency amounts properly. Sort date-based reports chronologically.
- Include `links.agreement_url` when the user might want to open the agreement.
- The `summary` attribute is a human-readable summary of key terms, useful for quick overviews.

## Error Handling

- **401 / auth error:** MCP: the user needs to sign in again (see Connecting). REST: the key is invalid or revoked.
- **400:** check the request shape. Field value errors name the field (for example a select value that isn't one of the options).
- **403 or "You've reached your plan limit":** on a new account this almost always means the user's **email isn't verified**. Check `email_verified` with `list-users` and ask them to verify. Test agreements bypass plan limits but still need a verified email.
- **404:** the resource doesn't exist, belongs to another org, or (for custom terms and custom templates) the feature isn't enabled for the org.
- **422:** invalid filter or body, or an action not allowed in the current status (for example sending a non-draft).
