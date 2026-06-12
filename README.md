# Helpdesky Integration Skill

A drop-in [Replit Agent skill](https://docs.replit.com) that teaches an AI agent how to integrate [Helpdesky](https://helpdesky.io) — a hosted help center and support platform — into any app.

Once installed, you can ask your agent things like *"add the Helpdesky help widget"*, *"add a contact form"*, *"add a support center"*, or *"write our help articles into Helpdesky"*, and it will know exactly how to do it.

## What it covers

- **Floating support widget** — site-wide search, AI answers, and chat in one script tag.
- **Contact form** — a lightweight "email us" form that creates support tickets.
- **Ticket center** — an authenticated area where logged-in users manage their own tickets (HMAC-signed).
- **Hosted help center** — link/deep-link to your Helpdesky-hosted knowledge base (subdomain, custom domain, or path-based).
- **Content API** — programmatically create, update, and sync help articles and categories.
- **Inbound email** — route an existing support inbox into Helpdesky.

It also includes a "which surface should I use" decision guide and a checklist of the one-time setup the owner must do in the Helpdesky dashboard (AI provider key, enabling the widget/messaging, allowed embed domains, custom domain, branding, publishing articles).

## Installation

### Recommended — one command

Install it straight from GitHub with the [`skills`](https://www.npmjs.com/package/skills) CLI (part of the [skills.sh](https://skills.sh) ecosystem; no global install needed):

```bash
npx skills add helpdesky/helpdesky-skill -a replit
```

- `-a replit` targets the Replit agent. Omit it to be prompted, or target other agents instead — e.g. `-a claude-code`, `-a cursor`, `-a codex`. Add `-y` for a non-interactive install.
- The CLI reads the `SKILL.md` at the repo root and installs it under `.agents/skills/`, where your agent discovers it automatically.

### Manual install (fallback)

Copy `SKILL.md` into your project at this exact path:

```
.agents/skills/helpdesky-integration/SKILL.md
```

The folder name (`helpdesky-integration`) must match, and the file must be named `SKILL.md`. Or pull it directly from the published repo:

```bash
mkdir -p .agents/skills/helpdesky-integration
curl -o .agents/skills/helpdesky-integration/SKILL.md \
  https://raw.githubusercontent.com/helpdesky/helpdesky-skill/main/SKILL.md
```

## Usage

After installing, ask your agent in plain language, for example:

- "Add the Helpdesky help widget to every page."
- "Add a Helpdesky contact form to our support page."
- "Add a ticket center to the logged-in account area."
- "Link to our Helpdesky help center from the nav."
- "Write our getting-started docs into Helpdesky and keep them synced."

The agent will ask you for the credentials it needs (your Helpdesk ID, and — depending on the feature — an HMAC secret or API key), which you copy from your Helpdesky dashboard.

## Credentials & security

- Your **Helpdesk ID** is public (it ships inside the embed script).
- Your **HMAC secret** (ticket center) and **API key** (content API) are secrets. The skill instructs the agent to store them as environment variables and never expose them in the browser — keep them that way.

This `SKILL.md` is safe to publish publicly: it contains only instructions, no secrets.

## License

[MIT](LICENSE) — use, adapt, and redistribute freely.
