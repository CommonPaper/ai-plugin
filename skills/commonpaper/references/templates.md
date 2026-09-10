# Templates and Attachments

This file documents template parameters by agreement type. The same field names apply whether you use the MCP tools or the REST API:

| Task | MCP tool | REST equivalent |
|---|---|---|
| List templates | `list-templates` (`queryParams.full`, `page[number]`, `page[size]`) | `GET /v1/templates` |
| Get one template | `get-template` | `GET /v1/templates/{id}` |
| Create a template | `create-template` (fields go in `body`) | `POST /v1/templates` |
| Update a template | `update-template` (`pathParams.id` + `body`) | `PATCH /v1/templates/{id}` |
| Archive a template | `archive-template` (restorable in the app) | none |
| List attachments | `list-attachments` | none |
| Upload an attachment | `upload-attachment_application_json` with `attachment.filename` set to a local PDF path; it returns a `curl_command` to run exactly as returned | `POST /v1/attachments` |
| Archive an attachment | `archive-attachment` | none |

The `curl` examples below are for the REST fallback (see `rest-api.md` for authentication). With the MCP tools, pass the same JSON object as the tool's `body`.

## Templates

```
GET   /v1/templates          — List all templates (paginated; default page size 15)
GET   /v1/templates/{id}     — Get single template
POST  /v1/templates          — Create a new template
PATCH /v1/templates/{id}     — Update an existing template
```

**Compact vs full list responses:** By default, the templates list includes shared attributes only (name, version, roles, `agreement_type`, etc.). Type-specific attributes and nested objects (such as `governing_law_region`, `template_csa_order_form`, or `template_psa_statement_of_work`) are excluded. Pass `full=true` to include those in the list response. Use `GET /v1/templates/{id}` when you only need one template's full configuration.

```
GET /v1/templates?full=true
```

**When to use `full=true` on list:**
- Inspecting order form defaults, fees, or type-specific settings across multiple templates
- Comparing template configurations without a separate GET per template

**When the compact list is enough:**
- Finding a template ID by type (`type == "template_nda"`)
- Listing template names for the user to pick from

Templates have a `type` field in the response indicating the agreement type: `template_nda`, `template_csa`, `template_dpa`, `template_design`, `template_psa`, `template_partnership`, `template_baa`, `template_loi`, `template_software_license`, `template_pilot`.

When creating an agreement, look up the appropriate template by matching the `type` field. For example, to send an NDA, find the template where `type == "template_nda"`.

### Creating a Template

```
POST /v1/templates
```

The `type` field is required and must be one of the **short names** (not the `template_*` response values):

| Short name | Agreement type |
|---|---|
| `NDA` | Mutual Non-Disclosure Agreement |
| `CSA` | Cloud Service Agreement |
| `DPA` | Data Processing Agreement |
| `Design` | Design Partner Agreement |
| `Pilot` | Pilot Agreement |
| `PSA` | Professional Services Agreement |
| `Partnership` | Partnership Agreement |
| `BAA` | Business Associate Agreement |
| `LOI` | Letter of Intent |
| `Software License` | Software License Agreement |

All other fields in the request body are optional and set the defaults pre-filled when agreements are created from the template.

**Common parameters (all types):**
- `name` — template name (required, human-readable)
- `governing_law_country` — defaults to "United States of America"
- `governing_law_region` — state/region (e.g., "Delaware", "California")
- `chosen_courts_country` — country for dispute resolution
- `chosen_courts_region` — state/region for dispute resolution
- `district_or_county` — district or county for courts (e.g., "Court of Chancery")
- `negotiations_allowed` — boolean (default: true)
- `sender_role` — label for your role (e.g., "Provider", "Company")
- `recipient_role` — label for counterparty role (e.g., "Customer", "Partner")
- `sender_last_to_sign` — boolean; when true, sender countersigns after recipient
- `default_signer_email` — pre-fill the signer's email on new agreements
- `include_additional_changes` — boolean; allow freeform additional terms
- `effective_date_type` — `"signature"` (on signing date) or `"date"` (specific date)

**NDA-specific parameters:**
- `term` — term length in years (integer; default: 1)
- `term_period_perpetual` — boolean; if true, term never expires
- `confidentiality_period` — confidentiality period in years (integer; default: 2)
- `confidentiality_period_perpetual` — boolean; perpetual confidentiality
- `purpose` — purpose statement (default: "Evaluating whether to enter into a business relationship…")
- `include_terms` — boolean; include standard CP terms on the agreement

**CSA versions**: Common Paper has two major CSA versions. The API defaults to the latest (currently v2.1). Pass `standard_version: "1.0"` or `"2.0"` to override. Most new accounts should use v2.

**CSA-specific parameters (both versions):**
- `general_cap_amount_type` — `"preceding"` (multiple of fees paid), `"qty"` (fixed dollar amount), or `"unlimited"`
- `general_cap_preceding_amount` — multiplier if type is `"preceding"` (e.g., `"1.0"` = 1× fees)
- `general_cap_amount` — fixed cap if type is `"qty"` (dollar amount as string)
- `include_covered_claims` — boolean; include indemnification coverage
- `include_covered_claims_include_provider_claims` — boolean
- `include_covered_claims_provider_claims` — text of provider IP indemnification
- `include_covered_claims_include_customer_claims` — boolean
- `include_covered_claims_customer_claims` — text of customer claims
- `include_increased_cap_amount` — boolean
- `include_increased_claims` — boolean
- `include_increased_claims_breach_of_privsec` — boolean
- `include_increased_claims_breach_of_confidentiality` — boolean
- `include_unlimited_claims` — boolean
- `include_unlimited_claims_indemnification_obligation` — boolean
- `include_unlimited_claims_breach_of_confidentiality` — boolean
- `include_security_policy` — boolean
- `include_security_policy_include_reasonable_efforts` — boolean
- `include_security_policy_include_annual_maintenance` — boolean; **must be set to `true` when enabling any certification flags below, or the certifications will not render on the agreement**
- `include_security_policy_soc2` — boolean
- `include_security_policy_soc2_type2` — boolean
- `include_security_policy_iso27001` — boolean
- `include_security_policy_penetration_testing` — boolean
- `include_acceptable_use_policy` — boolean
- `include_additional_warranties` — boolean
- `include_insurance_minimums` — boolean
- `include_insurance_minimums_include_general_liability` — boolean
- `include_insurance_minimums_general_liability_minimum` — string, e.g. `"1000000.0"`
- `include_insurance_minimums_general_liability_aggregate` — string
- `include_publicity_rights` — boolean
- `include_billing_workflow` — boolean
- `billing_workflow_type` — `"stripe"` or `"link"`

**v1-only CSA parameters:**
- `include_security_policy_hipaa` — boolean (v1 only; use BAA template for HIPAA compliance in v2)

**v2-only CSA parameters:**
- `include_security_policy_hitrust` — boolean
- `include_ai_addendum` — boolean; requires `include_ai_addendum_attachment_id` (upload PDF first)
- `include_ai_addendum_attachment_id` — UUID from `POST /v1/attachments`
- `framework_terms_type` — **required for v2** — `"new"` (standard CP terms) or `"description"` (custom)
- `include_dpa` — boolean; include DPA addendum (v2 only; requires `include_dpa_type`)
- `include_dpa_type` — `"description"` (inline text) or `"attachment"` (requires `include_dpa_attachment_id`)

**CSA order form (`template_csa_order_form_attributes`):**

- `cloud_service_description` — **required** — description of the cloud service
- `subscription_period` — integer (default: 1)
- `subscription_period_unit` — `"year(s)"` or `"month(s)"`
- `subscription_auto_renew` — `"days"` (rolling) or `"no"`
- `subscription_renewal_notice_days` — integer (default: 30)
- `include_sla` — boolean
- `include_product_support` — boolean
- `include_professional_services` — boolean
- `include_free_trial` — boolean
- `free_trial_days` — integer
- `include_max_users` — boolean
- `max_users` — integer
- `fees_currency` — `"USD"` (default)
- `fees_inclusive_of_taxes` — boolean
- `fees_include_fee_increase_upon_renewal` — boolean
- `fees_include_fee_increase_upon_renewal_fee` — percentage as decimal (e.g., `"5.0"` = 5%)

**v1 order form payment fields:**
- `payment_period` — integer days until payment due (e.g., 30)
- `payment_period_unit` — `"day(s)"`
- `invoice_period` — `"annually"` or `"monthly"`
- `affiliates_as_users` — boolean

**v2 order form payment fields:**
- `payment_process_type` — **required for v2** — `"automatic"` (Stripe recurring), `"invoice"`, or `"custom"`
- `payment_process_automatic_frequency` — required if `payment_process_type` is `"automatic"` (e.g., `"monthly"`, `"annually"`)
- `payment_process_type_invoice_type` — billing frequency, required if type is `"invoice"` (e.g., `"annually"`, `"monthly"`)
- `payment_process_type_invoice_amount` — net-X count (integer), required if type is `"invoice"` (e.g., `30` for net-30)
- `payment_process_type_invoice_duration` — net-X unit, required if type is `"invoice"` (e.g., `"day"`)
- `include_pilot_period` — boolean; enable a pilot/trial period before full subscription
- `include_pilot_period_duration_amount` — integer
- `include_pilot_period_duration_type` — e.g., `"day(s)"`, `"month(s)"`
- `include_pilot_period_include_fees` — boolean; whether pilot period has fees

**Important**: when `payment_process_type` is `"invoice"`, you MUST set all of `payment_process_type_invoice_type`, `payment_process_type_invoice_amount`, and `payment_process_type_invoice_duration`. Missing the amount/duration crashes the agreement preview when the template is used (the presenter calls `pluralize(nil, nil)` and raises). For net-30 annual billing, that's `{ invoice_type: "annually", invoice_amount: 30, invoice_duration: "day" }`.

**CSA fees (v2)**: v2 CSAs support structured fee line items as an array under `fees_attributes`. Each fee has a `type` field:

| Fee type | Description |
|---|---|
| `Fees::Flat` | Fixed total amount (`cost`) |
| `Fees::Cost` | Per-unit pricing (`cost` × `quantity`) |
| `Fees::Metered` | Usage-based (`cost`) |
| `Fees::OneTime` | One-time non-recurring charge |
| `Fees::Discount` | Discount line item (`discount_type`: `"fixed_amount"` or `"percentage"`) |
| `Fees::Graduated` | Tiered pricing with tiers |
| `Fees::Included` | Included at no charge |
| `Fees::Text` | Free-form text description only |
| `Fees::Attachment` | Fee defined via uploaded PDF attachment |

Common fee fields: `description`, `cost` (decimal as string), `quantity` (integer), `stripe_product_id`.

**PSA-specific parameters:**
- `template_psa_statement_of_work_attributes` — nested object with PSA defaults

**Software License-specific parameters:**
- `template_software_license_order_form_attributes` — nested object with license defaults

**Partnership-specific parameters:**
- `template_partnership_business_terms_attributes` — nested object

**Pilot-specific parameters:**
- `product_description` — short description of the product being piloted (set this when creating, otherwise the template renders without context)
- `pilot_period_description` — text describing the pilot period and its terms
- `pilot_period_fees_description` — text used when the pilot has fees (paired with `fees_type`)
- `fees_type` — `"free"` (default) or paid variants
- `include_technical_support` — boolean
- `technical_support_description` — text shown when support is included

**Example: Create an NDA template**

```bash
curl -s -H "Authorization: Bearer $(cat ~/.config/commonpaper/cp-api-token)" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "NDA",
    "name": "Standard Mutual NDA",
    "governing_law_region": "Delaware",
    "chosen_courts_region": "Delaware",
    "term": 1,
    "confidentiality_period": 2,
    "purpose": "Evaluating a potential business relationship.",
    "negotiations_allowed": true
  }' \
  "https://api.commonpaper.com/v1/templates"
```

**Example: Create a CSA template**

```bash
curl -s -H "Authorization: Bearer $(cat ~/.config/commonpaper/cp-api-token)" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "CSA",
    "name": "Standard Cloud Service Agreement",
    "governing_law_region": "Delaware",
    "general_cap_amount_type": "preceding",
    "general_cap_preceding_amount": "1.0",
    "include_security_policy": true,
    "include_dpa": false,
    "template_csa_order_form_attributes": {
      "cloud_service_description": "SaaS platform for team collaboration",
      "subscription_period": 1,
      "subscription_period_unit": "year(s)",
      "fee_period": "year",
      "payment_period": 30,
      "payment_period_unit": "day(s)",
      "invoice_period": "annually"
    }
  }' \
  "https://api.commonpaper.com/v1/templates"
```

The response is a JSONAPI object. The new template's UUID is in `data.id`.

### Updating a Template

```
PATCH /v1/templates/{id}
```

Pass only the fields you want to change. All the same fields from the create endpoint are valid. Nested objects like `template_csa_order_form_attributes` are also accepted.

**Example: Update an NDA template's jurisdiction**

```bash
curl -s -H "Authorization: Bearer $(cat ~/.config/commonpaper/cp-api-token)" \
  -H "Content-Type: application/json" \
  -d '{
    "governing_law_region": "California",
    "chosen_courts_region": "California",
    "district_or_county": "Superior Court of Santa Clara County"
  }' \
  "https://api.commonpaper.com/v1/templates/{id}"
```

**Note**: There is no delete endpoint for templates. Templates can be renamed but not removed via the API.

## Attachments

```
POST /v1/attachments       — Upload a file attachment
```

Attachments are files (PDFs, policies, addenda) that can be referenced from templates. Upload uses `multipart/form-data`.

**Request fields (inside the `attachment` form object):**
- `pdf` — the file to upload (required)
- `name` — display name for the attachment (required)
- `description` — optional description
- `uploaded_by` — optional email of the uploader

**Example:**

```bash
curl -s -X POST \
  -H "Authorization: Bearer $(cat ~/.config/commonpaper/cp-api-token)" \
  -F "attachment[name]=Security Policy" \
  -F "attachment[description]=Our information security policy" \
  -F "attachment[uploaded_by]=legal@company.com" \
  -F "attachment[pdf]=@/path/to/security-policy.pdf" \
  "https://api.commonpaper.com/v1/attachments"
```

The response includes the attachment ID, which can then be referenced in template fields like `include_security_policy_attachment_id`, `include_dpa_attachment_id`, `include_acceptable_use_policy_attachment_id`, etc.
