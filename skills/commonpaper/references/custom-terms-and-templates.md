# Custom Terms and Custom Templates

| Task | MCP tool | REST equivalent |
|---|---|---|
| List terms (metadata only) | `list-custom-terms` | `GET /v1/custom_terms` |
| Get terms with full body | `get-custom-term` | `GET /v1/custom_terms/{id}` |
| Create text terms | `create-custom-term_application_json` | `POST /v1/custom_terms` |
| Publish a new text version | `create-custom-term-version_application_json` | `POST /v1/custom_terms/{id}/versions` |
| Rename terms | `update-custom-term` | `PATCH /v1/custom_terms/{id}` |
| Upload Word (docx) terms | not supported over MCP; use REST multipart | `POST /v1/custom_terms` with `term_type=file` |
| List / get custom templates | `list-custom-templates` / `get-custom-template` | `GET /v1/custom_templates[/{id}]` |
| Create a custom template | `create-custom-template` | `POST /v1/custom_templates` |
| Publish a new YAML version | `create-custom-template-version` | `POST /v1/custom_templates/{id}/versions` |
| Rename or relink terms | `update-custom-template` | `PATCH /v1/custom_templates/{id}` |
| Archive a custom template | `archive-custom-template` | none |

**Porting an existing contract: prefer the REST path.** The fidelity rules below say to never retype contract text into a payload and to build the payload with a script that reads the extracted source file. MCP tool arguments can't be filled from a file that way, so when the source is an existing contract, use the REST endpoints for the write if they're reachable (see `rest-api.md`). If only the MCP tools are available, tell the user the text has to pass through the model on its way to Common Paper, and run the verify-after-writing check against the body returned by `get-custom-term` before reporting success. Drafting new terms from scratch has no source to preserve, so the MCP tools are fine there.


Custom agreements are built from two pieces: **custom terms** (the standard terms document that follows the cover page) and a **custom template** (a YAML definition of the cover page form). Both must be enabled for the organization; the endpoints return 404 otherwise.

```
GET   /v1/custom_terms                — List terms (metadata only; body omitted because it can be huge)
GET   /v1/custom_terms/{id}           — Get one term including the full released body
POST  /v1/custom_terms                — Create terms (markdown body published as version 1)
POST  /v1/custom_terms/{id}/versions  — Publish a new released version (versions are immutable)
PATCH /v1/custom_terms/{id}           — Rename only

GET   /v1/custom_templates                — List templates
GET   /v1/custom_templates/{id}           — Get one template including its YAML definition
POST  /v1/custom_templates                — Create a template from a YAML definition
POST  /v1/custom_templates/{id}/versions  — Publish a new definition version
PATCH /v1/custom_templates/{id}           — Rename or swap the linked custom terms
```

Write endpoints require `user_email` or `user_id` identifying a user in the organization, recorded as the creator.

**Porting an existing contract into custom terms (IMPORTANT):** When the source is an existing contract (a Word document, PDF, or pasted text), the published body must be character-for-character identical to the source text. The structural markup described below — the nested ordered list and bolded titles — is the only thing you add.

- **Never correct anything in transit** — not typos, not grammar, not punctuation, not capitalization. Executed agreements incorporate these terms by reference, so a silent "improvement" is an unauthorized edit to a legal document that the user has no way to notice. If you spot something that looks wrong, collect it and present a list of proposed corrections AFTER publishing, for the user to accept or decline.
- **Watch for typographic characters that get silently flattened to ASCII.** A single Word contract routinely holds dozens of these. They survive extraction intact and are lost only when text is retyped by hand:

  | Source character | Codepoint | Flattens to | Codepoint |
  |---|---|---|---|
  | ’ (curly apostrophe) | U+2019 | ' | U+0027 |
  | “ ” (curly quotes) | U+201C / U+201D | " | U+0022 |
  | — (em dash) | U+2014 | - | U+002D |
  | … (ellipsis) | U+2026 | ... | U+002E ×3 |
  | non-breaking space | U+00A0 | regular space | U+0020 |

- **Never retype contract text into a request payload.** Extract the source to a file and send that file's exact contents programmatically (e.g., build the JSON payload with a script that reads the file), so no human transcription sits between extraction and upload.
- **Verify after writing, not just before.** Re-fetch the stored body with `GET /v1/custom_terms/{id}` and assert every source paragraph survives verbatim. A cheap pre-check: the per-character counts of each non-ASCII character must match between source and stored body. Full check — strip only the markup you added, then assert nothing from the source is missing:

  ```bash
  # source.txt = extracted source text; stored_body.md = re-fetched body
  stored="$(sed -E -e 's/\*\*//g' \
    -e 's/^[[:space:]]*([0-9]+|[a-z])\.[[:space:]]+//' stored_body.md \
    | tr -s '[:space:]' ' ')"

  awk 'BEGIN{RS=""}
  {
    n=split($0, L, "\n"); out=""
    for (i=1; i<=n; i++) { sub(/^[ \t]*([0-9]+|[a-z])\. +/, "", L[i]); out = out " " L[i] }
    gsub(/\*\*/, "", out); gsub(/[ \t]+/, " ", out)
    sub(/^ /, "", out); sub(/ $/, "", out)
    if (out != "") print out
  }' source.txt | while IFS= read -r p; do
    [[ "$stored" == *"$p"* ]] || { echo "MISSING FROM STORED BODY: ${p:0:80}..."; exit 1; }
  done && echo "all source paragraphs verified verbatim"
  ```

- **If terms are already published with drift**, publish a corrected version via `POST /v1/custom_terms/{id}/versions`, and tell the user the flawed version stays in the version history since versions are immutable.

**Markdown formatting for terms bodies (IMPORTANT):** Common Paper renders custom terms with legal numbering generated from nested ordered lists, not from text. Structure the entire terms document as ONE continuous nested numbered list:

- Top-level list items are the sections. They render with bold numbers ("1.", "2."). Bold the section title: `1. **Services & Restrictions**`
- Second-level items are the clauses, rendering as "1.1", "1.2". Bold the clause title at the start (`1. **Performing Services.** ...`); the renderer underlines it.
- Third-level items render with letters ("a.", "b."), so in-text cross-references like "Section 4.2(a)" resolve to the third level.
- Indent 3 spaces per nesting level.
- Do NOT use `## heading` lines with flat lists under them. Headings break the numbering scheme and the lists restart at 1 instead of reading 1.1, 1.2.

Example shape:

```markdown
1. **Services & Restrictions**

   1. **Performing Services.** Contractor will perform...
   2. **Non-Solicitation.** During the term...

2. **Term & Termination**

   1. **Term.** This Agreement will commence...
   2. **Termination.**
      1. Either party may terminate...
      2. Company may immediately terminate...
```

**Creating text terms:**

```bash
curl -s -X POST "https://api.commonpaper.com/v1/custom_terms" \
  -H "Authorization: Bearer $(cat ~/.config/commonpaper/cp-api-token)" \
  -H "Content-Type: application/json" \
  -d '{"name": "Acme Standard Terms", "body": "1. **Services**\n\n   1. **Performing Services.** ...", "user_email": "user@company.com"}'
```

Publishing a new version takes the same `body` shape at `POST /v1/custom_terms/{id}/versions`. The new version immediately becomes the one new agreements use.

**File terms (uploaded Word documents):** send `multipart/form-data` with `term_type=file` and a `docx` file part (curl needs the `@` prefix) to the same create and versions endpoints. The docx converts to a pdf asynchronously; poll the term's `conversion_status` attribute (`processing`, `ready`, `failed`) after uploading.

**Creating a custom template:** send a `name` and a YAML `definition` describing the cover page. Field inputs include text, textarea, markdown, date, number, currency, radio, select, multiselect, email, states, address, rate, fixed, and header. **When converting a document into a template, create a section for every input the contract collects** (parties, descriptions, dates, rates, payment terms, and so on); do not drop or merge inputs the document asks for. The exception is anything the standard signature block already captures — party legal names, signatory name/title/date, and notice-of-legal-notice addresses are handled automatically by Common Paper's agreement and signing workflow for every template type, so never create custom fields for them even when the source document has blanks in its signature or notice section. **Default every section to `required: true`** so it always appears in the agreement; only make a section optional (which adds an include checkbox on the form, `included: false` to default it unchecked) when the user asks for a section to be optional or the source document marks it as removable. An `import: governing_law` section adds the standard governing law picker and takes `defaults` for `governing_law_region` and `chosen_courts_region`. Field keys must be unique across the whole template; a duplicate key anywhere in the YAML is rejected as a validation error.

```yaml
title: Consulting Agreement
terms_label: Statement of Work
parties:
  sender: Company
  recipient: Contractor
sections:
  - heading: Services
    required: true
    fields:
      services:
        label: Description of services
        input: textarea
        required: true
  - import: governing_law
    required: true
    defaults:
      governing_law_region: Delaware
      chosen_courts_region: Delaware
```

**Linking terms to a template:** pass `custom_term_id` when creating or updating the template; the terms must belong to the same organization, and an empty string unlinks them. Without a link, agreements created from the template render with no terms.

**Template versions record their terms:** each template version stores the `custom_term_id` it was published with (returned on version responses along with `version_number`; template responses report `current_version_number`). Changing a template's terms via PATCH publishes a new version recording the swap — pass `user_id` to record who made it. Publishing a definition identical to the current version with the same linked terms is a no-op: the current version comes back with a 200 instead of a 201 and nothing new is created. So after any template write, read `current_version_number` to confirm whether a version was actually published.

**Creating agreements from a custom template:** use the normal `POST /v1/agreements` with the template's UUID as `template_id`, and put cover page answers in `agreement.custom_field_values` keyed by the YAML field keys. Governing law answers also go inside `custom_field_values` (as `governing_law_region`, `chosen_courts_region`), not as top-level agreement attributes.

**Field values are validated against the schema.** An invalid value returns a 400 naming the field: select, radio, and multiselect values must be one of the field's `options` (match the option text exactly); number and currency values must be numeric; date values must be `YYYY-MM-DD` (they render as long dates like September 1, 2026); email values must be valid addresses. Currency values keep exactly the precision entered, including sub-cent amounts, and textarea values keep their line breaks.
