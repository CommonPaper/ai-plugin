# Common Paper REST API Reference (fallback path)

Use this reference only when the Common Paper MCP tools can't do the job: the connector isn't connected, you need a filter or sort the MCP `list-agreements` tool doesn't expose (company name, end date, deal value, signer), you're uploading a Word file as custom terms, or you're provisioning a brand-new account. Everything here goes through `curl` in the shell, so it needs network access to `api.commonpaper.com` from the environment you're running in. If a request fails with a network or proxy error (not an HTTP status), tell the user the REST fallback isn't reachable here and do what you can with the MCP tools instead. If you can't run shell commands at all, the REST fallback isn't available; say which part of the request needs it.

## Authentication

The API uses Bearer token authentication. Only an API token is required — the organization is inferred from the token. No Organization ID header is needed.

The token lives in `~/.config/commonpaper/cp-api-token`, outside the plugin's install folder, so it survives plugin updates.

### Credential Loading

Before making any API call, check if a saved token exists (and migrate one saved by the older standalone skill):

```bash
mkdir -p ~/.config/commonpaper && chmod 700 ~/.config/commonpaper
if [ ! -s ~/.config/commonpaper/cp-api-token ] && [ -s ~/.claude/skills/commonpaper/cp-api-token ]; then
  cp ~/.claude/skills/commonpaper/cp-api-token ~/.config/commonpaper/cp-api-token
  chmod 600 ~/.config/commonpaper/cp-api-token
fi
test -s ~/.config/commonpaper/cp-api-token && echo "token present" || echo "no token"
```

**Do not `cat` the file or otherwise display the value to the user.**

If there is no token:

1. **If the Common Paper MCP connector is connected**, call its `generate-api-key` tool (with `api_key.description` such as `"Claude plugin REST fallback"`). It returns the key once for the signed-in user. Save it as described below without repeating it in chat.
2. **Otherwise**, ask the user: "What is your Common Paper API token? (You can generate one from your account's Integrations tab)"

### Credential Validation

Before saving the token, check its format:

- Must contain only alphanumeric characters and underscores

### Saving Credentials

Every REST call reads the token from the credentials file, so it has to be written there to use it. Before writing it, ask: "Would you like me to keep this token for future sessions? If not, I'll delete it when we're done."

Write the token to the credentials file, then test it with a lightweight API call:

```bash
mkdir -p ~/.config/commonpaper && chmod 700 ~/.config/commonpaper
printf '%s' 'THE_TOKEN' > ~/.config/commonpaper/cp-api-token && chmod 600 ~/.config/commonpaper/cp-api-token
curl -s -H "Authorization: Bearer $(cat ~/.config/commonpaper/cp-api-token)" "https://api.commonpaper.com/v1/agreements?page%5Bsize%5D=1" -o /dev/null -w "%{http_code}"
```

If the response is not `200`, inform the user that their token appears invalid and ask them to double-check.

If the user said not to keep the token, delete it when the task is done:

```bash
rm -f ~/.config/commonpaper/cp-api-token
```

### Making Requests

**All API requests MUST read the token from the credentials file via command substitution to avoid exposing it as a literal in the command.**

```bash
curl -s -H "Authorization: Bearer $(cat ~/.config/commonpaper/cp-api-token)" \
  "https://api.commonpaper.com/v1/agreements"
```

For POST/PATCH requests, add:
```bash
-H "Content-Type: application/json"
```

**Base URL**: `https://api.commonpaper.com/v1`

**IMPORTANT — URL encoding**: Always URL-encode square brackets in query parameters. Use `%5B` for `[` and `%5D` for `]`. Unencoded brackets will cause `zsh: no such file or directory` errors.

**Note**: While `$(cat ~/.config/commonpaper/cp-api-token)` is expanded by the shell before execution (so the token briefly appears in the process list), this is significantly better than having the literal token in the command text shown in chat. The token file is also chmod 600 so only the user can read it.

## Endpoints

### List Agreements (Primary endpoint for queries)

```
GET /v1/agreements
```

Paginated — default page size is 15 (override with `page[size]`).

**Compact vs full list responses:** By default, the list response omits type-specific nested attributes (such as `csa`, `csa_order_form`, `dpa`, `baa`, and other agreement-type sub-models). Pass `full=true` to include those nested attributes in the list response. Use `GET /v1/agreements/{id}` when you only need one agreement's full details.

```
GET /v1/agreements?full=true
```

URL-encoded: `full=true` (no brackets). Combine with filters and pagination, e.g. `filter%5Bstatus_eq%5D=signed&full=true&page%5Bsize%5D=100`.

**When to use `full=true` on list:**
- You need nested fee/term data from multiple agreements in one pass (e.g., `csa_order_form.fees`, `csa.include_covered_claims`)
- You're building a report that requires type-specific fields without fetching each agreement individually

**When the compact list is enough:**
- Counting agreements (`meta.pagination.records`)
- Listing counterparties, statuses, dates, and summary fields
- Most search/filter questions about who signed what

Returns JSONAPI format with pagination metadata:
```json
{
  "data": [ { "id": "...", "type": "agreement", "attributes": { ... }, "links": { ... } } ],
  "meta": { "pagination": { "current": 1, "next": 2, "last": 5, "records": 127 } },
  "links": { "self": "...", "next": "...", "last": "..." }
}
```

**Key response fields:**
- `meta.pagination.records` — total number of matching agreements (use this for counts instead of paginating through all results)
- `data[].attributes` — agreement fields
- `data[].attributes.display_status` — human-friendly status (e.g., "Completed", "Waiting for counterparty") — prefer this over `status` when presenting to users
- `data[].links.agreement_url` — direct link to the agreement in the Common Paper app

### Get Single Agreement

```
GET /v1/agreements/{id}
```

### Create Agreement

```
POST /v1/agreements
```

**IMPORTANT**: The create endpoint does NOT use JSONAPI format. It uses a flat structure with these top-level keys:

```json
{
  "owner_email": "owner@company.com",
  "template_id": "uuid-of-template",
  "draft": false,
  "agreement": {
    "agreement_type": "NDA",
    "sender_signer_name": "Jane Smith",
    "sender_signer_title": "CEO",
    "sender_signer_email": "jane@company.com",
    "recipient_email": "recipient@example.com",
    "recipient_name": "John Doe"
  }
}
```

**Required fields:**
- `owner_email` — email of the user who will own this agreement (must be a user in the org)
- `template_id` — UUID of the template to use (get from `GET /v1/templates`)
- `agreement.recipient_email` — recipient's email address
- `agreement.recipient_name` — recipient's name (REQUIRED — will error without it)

**Top-level options:**
- `draft` — boolean (top-level, NOT nested under `agreement`). When `true`, the agreement is created with status `draft` and is **not** sent to the recipient. Send it later via `POST /v1/agreements/{id}/send`. When `false` or omitted, the agreement is created and sent immediately.

**Default to draft mode.** When creating an agreement on the user's behalf, always pass `draft: true` unless the user explicitly says to send it right away. Then tell the user the draft was created, give them the agreement URL to review, and ask whether to send it. Only call `POST /v1/agreements/{id}/send` after they confirm.

**Optional but recommended fields:**
- `agreement.sender_signer_name`, `agreement.sender_signer_title`, `agreement.sender_signer_email`
- `agreement.recipient_organization`, `agreement.recipient_title`
- `agreement.agreement_type` — NDA, CSA, etc.
- `agreement.test_agreement` — set to `true` to mark as a test; still sends a real email and requires a real recipient, but is noted as not legally binding and does not count against monthly limits
- `cc_users` — array of email addresses to CC on the agreement (must be users in the org)

**Billing fields:**
- `agreement.include_billing_workflow` — set to `true` to enable a billing workflow after signing
- `agreement.billing_workflow_type` — `"link"` or `"stripe"`
- `agreement.payment_link_url` — payment URL shown after signing; requires `include_billing_workflow: true` and `billing_workflow_type: "link"`
- `agreement.include_automated_payment_reminders` — boolean; sends automatic payment reminders when true
- `agreement.include_billing_info` — boolean; enables billing contact fields
- `agreement.billing_name` — billing contact name; used when `include_billing_info` is true
- `agreement.billing_email` — billing contact email; used when `include_billing_info` is true

**Governing law fields:**
- `agreement.governing_law_country` — country whose laws govern the agreement (e.g., `"United States of America"`)
- `agreement.governing_law_region` — state or region whose laws govern the agreement (e.g., `"Delaware"`)
- `agreement.chosen_courts_country` — country where disputes will be resolved
- `agreement.chosen_courts_region` — state or region where disputes will be resolved
- `agreement.district_or_county` — district or county for dispute resolution

**Notice email fields:**
- `agreement.sender_notice_email_address` — email for legal notices to the sender
- `agreement.recipient_notice_email_address` — email for legal notices to the recipient

**Framework terms fields:**
- `agreement.framework_terms_type` — `"new"` (standard) or `"description"` (custom)
- `agreement.framework_terms_description` — custom framework terms; only used when `framework_terms_type` is `"description"`

**Other fields:**
- `agreement.manual_send` — set to `true` to mark the agreement as manually sent outside the platform
- `agreement.message` — optional plain-language note included when the agreement is sent to the recipient

### Presenting an agreement to a counterparty

Common Paper agreements use a **Cover Page + Standard Terms** structure: the cover page holds the deal-specific terms (parties, term, fees, governing law), and incorporates the published, version-numbered standard terms by reference. 

For counterparties unfamiliar with this model, use `agreement.message` to include a short note with the agreement email, e.g. "This is a Common Paper standard agreement. The terms are published at https://commonpaper.com/standards. Only the cover page is specific to this deal."

Common Paper also publishes a one-page explainer, ["Why We Use Common Paper Standard Agreements"](https://commonpaper.com/wp-content/uploads/2022/05/Why-We-Use-Common-Paper-Standard-Agreements.pdf), which can be linked or attached separately if more context is needed.

### Agreement Actions

```
POST  /v1/agreements/{id}/send          — Send a draft agreement to the recipient
PATCH /v1/agreements/{id}/void          — Void an agreement
PATCH /v1/agreements/{id}/reassign      — Reassign recipient
PATCH /v1/agreements/{id}/resend_email  — Resend signature email
GET   /v1/agreements/{id}/history       — Get agreement history
GET   /v1/agreements/{id}/download_pdf  — Download PDF
GET   /v1/agreements/{id}/shareable_link — Get shareable link
```

**Send**: Only works on agreements in `draft` status (returns 422 otherwise). No request body required.


### Other Endpoints

```
GET /v1/organizations       — List organizations (returns the caller's org)
GET /v1/organizations/{id}  — Get single organization
GET /v1/users               — List organization users
GET /v1/users/{id}          — Get single user
GET /v1/agreement_types     — List all agreement types
GET /v1/agreement_statuses  — List all agreement statuses
GET /v1/agreement_history   — List agreement histories (filterable; paginated; default page size 15)
```

### Organizations

The token is scoped to a single organization. `GET /v1/organizations` returns that org as a single-element list. Useful attributes include `name` (the company name), `street_address`, `city`, `state`, `zipcode`, `country`, and `onboarded`. Use this endpoint to fetch the company name rather than asking the user for it.

### Users

The `/v1/users` endpoint returns user details including `name`, `email`, and `title`. Use this to look up sender information when creating agreements — the user may only provide an email, and you can fill in name/title from the user record.

## Filtering (Ransack)

The API supports filtering via query parameters using ransack predicates. The syntax is `filter[field_predicate]=value`.

### Predicates

| Predicate | Meaning | Example |
|-----------|---------|---------|
| `_eq` | Equals | `filter[status_eq]=signed` |
| `_not_eq` | Not equal | `filter[status_not_eq]=draft` |
| `_cont` | Contains (case-insensitive) | `filter[recipient_organization_cont]=microsoft` |
| `_i_cont` | Contains (case-insensitive, explicit) | `filter[recipient_organization_i_cont]=google` |
| `_gt` | Greater than | `filter[effective_date_gt]=2024-01-01` |
| `_lt` | Less than | `filter[end_date_lt]=2025-12-31` |
| `_gteq` | Greater than or equal | `filter[end_date_gteq]=2025-01-01` |
| `_lteq` | Less than or equal | `filter[end_date_lteq]=2025-12-31` |
| `_present` | Field is not null/empty | `filter[end_date_present]=true` |
| `_blank` | Field is null/empty | `filter[end_date_blank]=true` |
| `_in` | In a list | `filter[status_in][]=signed&filter[status_in][]=sent_to_recipient_for_signature` |
| `_null` | Is null | `filter[end_date_null]=true` |
| `_not_null` | Is not null | `filter[end_date_not_null]=true` |

### Combining Filters

Chain multiple filter params: `filter[status_eq]=signed&filter[agreement_type_eq]=NDA`

### Filterable Fields on Agreements

All agreement columns are filterable, including nested model fields prefixed with the model name. Key fields:

**Identity & Type:**
- `agreement_type` — NDA, CSA, DPA, PSA, Design, Partnership, BAA, LOI, Software License, Amendment, Pilot (or custom)
- `status` — see Agreement Statuses below
- `description`

**Parties:**
- `sender_signer_name`, `sender_signer_email`, `sender_signer_title`
- `sender_organization`
- `recipient_name`, `recipient_email`, `recipient_title`
- `recipient_organization`

**Dates:**
- `effective_date` — when the agreement takes effect
- `end_date` — expiration date
- `sent_date` — when it was sent
- `all_signed_date` — when fully executed
- `sender_signed_date`, `recipient_signed_date`

**Terms:**
- `term` — term length (integer, in units of term_period)
- `term_period_perpetual` — boolean
- `purpose`
- `governing_law_country`, `governing_law_region`

**Financial:**
- `ai_gmv` — gross contract value (available on all agreement types, useful for sorting by deal size)
- `ai_arr` — annual recurring revenue
- `csa_order_form_total_contract_value` — total contract value (CSA-specific)
- `csa_order_form_subscription_fee` — subscription fee amount
- `csa_order_form_payment_period` — payment frequency

**Roles (nested):**
- `agreement_roles_user_email` — filter by user email in any role

**Other nested models:**
- `csa_*`, `csa_order_form_*`, `dpa_*`, `design_partner_*`, `psa_*`, `psa_statement_of_work_*`, `partnership_*`, `partnership_business_terms_*`, `baa_*`, `loi_*`

## Agreement Statuses

The `status` field contains the internal status. The `display_status` field contains a human-readable version. Prefer `display_status` when presenting results to users.

Internal statuses:
- `draft` — Not yet sent
- `sent_waiting_for_initial_review` — Sent, awaiting first review
- `sent_waiting_for_sender_review` — Waiting for sender to review
- `sent_waiting_for_recipient_review` — Waiting for recipient to review
- `in_progress` — Active negotiation
- `sent_to_recipient_for_signature` — Sent for recipient signature
- `sent_to_sender_signer_for_signature` — Sent for sender signature
- `waiting_for_sender_signature` — Awaiting sender signature
- `waiting_for_recipient_signature` — Awaiting recipient signature
- `signed` — Fully executed (display_status: "Completed")
- `signed_waiting_for_final_confirmation` — Signed, pending confirmation
- `declined_by_recipient` — Recipient declined
- `voided_by_sender` — Voided
- `sent_manually` — Sent outside the platform

**Signed/active** = `status_eq=signed`
**In-flight** = any status that is not `draft`, `signed`, `voided_by_sender`, or `declined_by_recipient`

## Pagination

These list endpoints are paginated and share the same defaults:
- `GET /v1/agreements`
- `GET /v1/templates`
- `GET /v1/agreement_history`

`GET /v1/agreements` and `GET /v1/templates` also accept `full=true` to return type-specific nested attributes in list responses (see those endpoint sections above). Default list responses are compact.

They return pagination metadata in `meta.pagination`:
- `records` — total number of matching results (use this for counts!)
- `current` — current page number
- `next` — next page number (null if on last page)
- `last` — last page number

Use `page[number]=N&page[size]=M` query params (URL-encoded: `page%5Bnumber%5D=N&page%5Bsize%5D=M`). Default page size is 15.

**For counting**: Use `page%5Bsize%5D=1` and read `meta.pagination.records` — no need to paginate through all results.

**For complete lists**: Paginate through all pages when the user wants a full list. Check for `meta.pagination.next` and keep fetching until it's null.

## Common Query Patterns

In all examples below, `$AUTH` refers to `-H "Authorization: Bearer $(cat ~/.config/commonpaper/cp-api-token)"`.

### "How many signed contracts do I have?"

```bash
curl -s $AUTH \
  "https://api.commonpaper.com/v1/agreements?filter%5Bstatus_eq%5D=signed&page%5Bsize%5D=1" | jq '.meta.pagination.records'
```

### "Do I have any contracts with {Company}?"

Use `_cont` for case-insensitive partial matching on `recipient_organization`. Also check `sender_organization` in case the user's org is the recipient.

### "Do I have an active NDA with {Company}?"

Combine agreement type, status, and company filters. An "active" NDA is signed and not expired:

```
filter[agreement_type_eq]=NDA&filter[status_eq]=signed&filter[recipient_organization_cont]={Company}&filter[expired_eq]=false
```

### "Who was the signer on the {Company} account?"

Find agreements with that company and extract signer info from attributes: `sender_signer_name`, `sender_signer_email`, `recipient_name`, `recipient_email`.

### "Largest CSA deal by total contract amount?"

Fetch all signed CSAs and sort client-side by `ai_gmv` with `jq`:

```bash
curl -s $AUTH \
  "https://api.commonpaper.com/v1/agreements?filter%5Bagreement_type_eq%5D=CSA&filter%5Bstatus_eq%5D=signed&page%5Bsize%5D=100" \
  | jq '[.data[] | {counterparty: .attributes.recipient_organization, gmv: (.attributes.ai_gmv // "0" | tonumber), summary: .attributes.summary}] | sort_by(-.gmv)'
```

Note: `ai_gmv` may be `"0"` or null for some agreements even if fees exist — check the `summary` field and nested fee data (`csa_order_form.fees`) for the full picture. Nested fee data is not included in the default list response; pass `full=true` on the list call or fetch individual agreements with `GET /v1/agreements/{id}`.

### "Upcoming renewal dates?"

```bash
curl -s $AUTH \
  "https://api.commonpaper.com/v1/agreements?filter%5Bstatus_eq%5D=signed&filter%5Bend_date_gteq%5D=$(date +%Y-%m-%d)&sort=end_date&page%5Bsize%5D=100"
```

Present results as a table with columns: Agreement Type, Counterparty, End Date, Term.

## Error Handling

- **400 Bad Request**: Check the request body schema — the create endpoint uses a specific format (see Create Agreement above), not JSONAPI.
- **401 Unauthorized**: Token is invalid or expired. Ask the user to check their API token.
- **403 Forbidden** / "You've reached your plan limit": On a new account this almost always means the user's **email is not verified** rather than an actual plan limit. Check `email_verified` on `GET /v1/users` and tell the user to verify their email before trying again. Test agreements bypass plan limits but still require a verified email.
- **404 Not Found**: The agreement or resource doesn't exist.
- **422 Unprocessable Entity**: Invalid filter or request body. Check the filter syntax.

## Write Operations

### Create Template

To create a template:

1. **Confirm the type** — ask the user which agreement type they want (NDA, CSA, DPA, etc.)
2. **Collect parameters** — ask for name and any key settings (jurisdiction, term, etc.) or apply sensible defaults
3. **For CSA only** — `cloud_service_description` is required; ask the user for it if not provided
4. **Confirm details** with the user before creating
5. **POST to `/v1/templates`**

The response includes the new template ID in `data.id`. Show the template name and ID to the user.

### Update Template

To update an existing template:

1. **Identify the template** — list templates if the user didn't specify which one, then confirm
2. **Confirm the changes** before patching
3. **PATCH to `/v1/templates/{id}`** with only the fields being changed

### Create Agreement

To create an agreement:

1. **Look up the sender** using `GET /v1/users` to get their name, title, and email
2. **Look up the template** using `GET /v1/templates` and find the one matching the desired type (e.g., `type == "template_nda"`). The compact list is sufficient for ID lookup; use `full=true` or `GET /v1/templates/{id}` only if you need to read template defaults before creating.
3. **Confirm details** with the user — be explicit about what you're going to do before you do it (a Cowork user may be watching without having initiated the request)
4. **POST to `/v1/agreements`** with `draft: true` by default:

```bash
curl -s -H "Authorization: Bearer $(cat ~/.config/commonpaper/cp-api-token)" \
  -H "Content-Type: application/json" \
  -d '{
    "owner_email": "owner@company.com",
    "template_id": "uuid-of-template",
    "draft": true,
    "agreement": {
      "agreement_type": "NDA",
      "sender_signer_name": "Jane Smith",
      "sender_signer_title": "CEO",
      "sender_signer_email": "jane@company.com",
      "recipient_email": "recipient@example.com",
      "recipient_name": "John Doe"
    }
  }' \
  "https://api.commonpaper.com/v1/agreements"
```

5. **Show the user the draft URL** (`data.links.agreement_url`) and ask whether to send it
6. **If they confirm, send it** via `POST /v1/agreements/{id}/send`

**Always default to `draft: true`.** Only set `draft: false` (or omit it) if the user has explicitly said to send the agreement immediately without review. When sending immediately, restate that the email will go out right away before posting.

### Send Draft Agreement

```
POST /v1/agreements/{id}/send
```

Sends a previously created draft. The agreement must be in `draft` status — sending an already-sent agreement returns 422. No request body needed.

```bash
curl -s -X POST -H "Authorization: Bearer $(cat ~/.config/commonpaper/cp-api-token)" \
  "https://api.commonpaper.com/v1/agreements/{id}/send"
```

### Void Agreement

Always confirm with the user before voiding.

### Reassign Recipient

Requires `recipient_email` and `recipient_name` in the request body.

### Resend Email

Simple PATCH to `/v1/agreements/{id}/resend_email`.

## Notes

- **NEVER expose the API token in chat output or as a literal in commands**
- **Never alter contract text when porting it into custom terms** — no corrections, no quote or dash substitutions; see "Porting an existing contract into custom terms" under Custom Terms and Custom Templates
- **Always URL-encode square brackets** in query parameters — use `%5B` for `[` and `%5D` for `]`. Unencoded brackets cause zsh errors.
- The `_cont` predicate is the best choice for company name searches since it handles partial and case-insensitive matching
- When a user asks about a company, search both `recipient_organization` and `sender_organization` fields since either party could be the counterparty
- For "active" or "current" agreements, filter on `status_eq=signed` combined with `expired_eq=false` or `end_date_gteq={today}`
- The API returns JSONAPI format for reads — data is in `response.data[].attributes`
- List endpoints for agreements and templates return compact payloads by default; pass `full=true` when you need nested type-specific attributes in list responses
- The API uses a **different format for creates** — NOT JSONAPI. Use `owner_email`, `template_id`, and `agreement` as top-level keys.
- Use `jq` for parsing JSON responses in curl commands
- When showing users the curl command you used, replace the auth header with `$CP_TOKEN` placeholder and ensure brackets are URL-encoded
- Use `display_status` instead of `status` when presenting results to users
- The `summary` field on agreements contains a human-readable summary of key terms — useful for quick overviews
