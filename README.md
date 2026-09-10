# Common Paper plugin for Claude

Query, create, and manage your contracts in [Common Paper](https://commonpaper.com) from Claude Cowork and Claude Code, in plain English.

The plugin bundles two things:

- **The Common Paper MCP connector** (`https://api.commonpaper.com/mcp`). You sign in with your Common Paper account through OAuth, so there's no API key to copy or store.
- **A Common Paper skill** that teaches Claude how Common Paper works: agreement statuses, the Cover Page + Standard Terms model, safe defaults (every agreement starts as a draft), template parameters for each agreement type, the rules for porting contract text into custom terms without altering it, and a guided onboarding flow for new accounts.

For a few things the connector doesn't cover yet (filtering by company name or end date, uploading Word files as custom terms, and creating a brand-new account), the skill falls back to the Common Paper REST API.

## What you can do

- **Count and search:** "How many signed NDAs do we have?" / "Do we have a contract with Acme?"
- **Send agreements:** "Draft an NDA to jane@example.com from ben@company.com" (Claude creates a draft, shows you the link, and sends only when you confirm)
- **Manage agreements:** void, reassign the recipient, resend the signature email, download the PDF, get a shareable link
- **Financial questions:** "What's our largest CSA by deal value?"
- **Renewals:** "Which contracts expire in the next 90 days?"
- **Signers:** "Who signed the deal with Microsoft?"
- **Templates:** create and update NDA, CSA, DPA, PSA, Partnership, BAA, LOI, Software License, Design Partner, and Pilot templates
- **Custom agreements:** publish custom terms and build custom cover page templates
- **Onboarding:** "Set up my Common Paper account" walks through ten quick questions and creates a starter set of templates

## Example prompts

1. `How many signed contracts do we have, and how many are still waiting on the other side?`
2. `Do we have an active NDA with Northwind? If not, draft one to sam@northwind.com from me.`
3. `List every agreement that ends in the next 90 days as a table with counterparty, type, and end date.`
4. `Create a CSA template with Delaware law, a 1x fees liability cap, and net-30 annual invoicing.`
5. `Set up my Common Paper account.`

## Installation

### Claude Cowork

Install **Common Paper** from the plugin directory, then connect when prompted and sign in to Common Paper.

### Claude Code

Install from the plugin directory with `/plugin`, or add this repo as a marketplace directly:

```bash
claude plugin marketplace add CommonPaper/claude-plugin
claude plugin install commonpaper@commonpaper
```

Then run `/mcp`, choose `common-paper`, and sign in to Common Paper.

### Upgrading from the standalone skill

If you previously installed [CommonPaper/claude-skill](https://github.com/CommonPaper/claude-skill), you can remove it after installing the plugin. If the plugin ever needs an API key for the REST fallback, it copies a key saved by the old skill to `~/.config/commonpaper/cp-api-token` automatically.

## Requirements

- A [Common Paper](https://commonpaper.com) account
- Claude Cowork, or Claude Code
- For the REST fallback only: `curl` and network access to `api.commonpaper.com`. Custom terms and custom templates must be enabled for your organization to use those features.

## Security and privacy

- The connector uses OAuth. Claude acts as the Common Paper user who signed in and can only see that user's organization.
- Claude confirms before anything that sends an email or changes a live agreement (send, void, reassign, resend, invite), and creates agreements as drafts by default.
- If the REST fallback needs an API key, the plugin mints one for the signed-in user with the connector (or asks you for one), stores it at `~/.config/commonpaper/cp-api-token` with owner-only permissions, and never prints it in chat.
- Contract text ported into custom terms is kept character-for-character. Claude verifies the stored text after writing and never "fixes" typos or punctuation on its own.
- Privacy policy: https://commonpaper.com/privacy-policy/ (questions: privacy@commonpaper.com)

## Troubleshooting

| Problem | Fix |
|---|---|
| Claude says the Common Paper tools aren't connected | Cowork: Customize > Connectors > Common Paper > Connect. Claude Code: run `/mcp`, choose `common-paper`, and sign in. |
| An agreement you expect is missing | The API only covers agreements sent by your organization. Agreements you received from another organization aren't included. |
| "You've reached your plan limit" on a new account | Usually means your email address isn't verified yet. Verify it, then try again. |
| 404 on custom terms or custom templates | The feature isn't enabled for your organization. Contact Common Paper support. |
| REST fallback fails with a network error | Your environment can't reach `api.commonpaper.com` directly. Claude will fall back to what the connector supports. |
| Search by company name is slow on a large account | Without the REST fallback, Claude has to page through agreements. Allow network access to `api.commonpaper.com` so it can filter server-side. |

## Support

- Help center: [Common Paper and LLMs](https://help.commonpaper.com/en/articles/14627004-common-paper-and-llms)
- Bugs and feature requests: open an issue in this repository
- For account or connection problems, contact Common Paper support through the [help center](https://help.commonpaper.com) with the name of the client you're using and a screenshot of the error

## Plugin contents

```
.claude-plugin/plugin.json          Plugin manifest
.claude-plugin/marketplace.json     Lets this repo be added as a marketplace
.mcp.json                           Common Paper MCP connector
skills/commonpaper/SKILL.md         Core skill
skills/commonpaper/references/      REST API, templates, custom terms, onboarding details
```

## License

MIT
