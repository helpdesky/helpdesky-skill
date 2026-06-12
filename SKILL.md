---
name: helpdesky-integration
description: Integrate Helpdesky (help center, support widget, contact form, ticket center, and docs) into this app. Use when the user wants to add a help/support widget, add a knowledge base or help center, add a "contact us"/contact form, let users contact support or see their support tickets, link to documentation, auto-write or sync help articles, or route their support email into Helpdesky. Triggers on phrases like "add a help widget", "embed Helpdesky", "add a support center", "add a contact form", "write help docs", or "link to our help articles".
---

# Helpdesky Integration

This skill teaches you to integrate **Helpdesky** (https://helpdesky.io) — a hosted help-center and support platform — into the app you are building. Everything here uses embed scripts, public endpoints, the hosted help center, and the content REST API that already exist; you never build or host any of it.

Helpdesky serves the embed scripts with `Access-Control-Allow-Origin: *`, so they work from any domain. The customer's domain must still be added to the **Allowed Embed Domains** list in their Helpdesky dashboard (**Settings → Messages**) for the messaging endpoints (contact form, ticket center) to accept requests.

---

## Step 1 — Gather credentials (ask the customer)

Ask the customer only for what the chosen surface needs, then store each value as a **project secret / environment variable — never hardcode keys, and never ship the HMAC secret or API key to the browser.**

| Surface | What to ask for | Where they find it in the Helpdesky dashboard |
|---|---|---|
| Widget | **Helpdesk ID** (a UUID) | **Embeds → Widget** (shown in the embed snippet) |
| Contact form | **Helpdesk ID** | **Embeds → Contact Form** |
| Ticket center | **Helpdesk ID** + **HMAC secret** | **Embeds → Ticket Center** (generate/copy the HMAC secret there) |
| Writing/syncing articles | **API key** (starts with `hdh_`) | **Settings → API Integration** |
| Hosted help center | **helpdesk slug** (and custom domain, if any) | **Settings** |
| Inbound email | **slug** + **6-digit code** | **Embeds → Email Forwarding** |

The Helpdesk ID is **not** a secret (it ships in the public embed). The **HMAC secret** and **API key are secrets** — keep them server-side only.

The base URL is `https://helpdesky.io` everywhere below.

---

## Step 2 — Which surface should I use?

| Surface | What it is | Use it when |
|---|---|---|
| **Floating widget** | Site-wide bubble with search, AI answers, and chat. Anonymous self-service on any public page. | You want help available everywhere with one script tag. |
| **Contact form** | A simple "email us" form that creates a support ticket. No login required. | You want a lightweight contact/support form on a page. |
| **Ticket center** | An authenticated area **inside the customer's logged-in account** where a known, signed-in user sees all their past tickets/conversations and starts new ones. Requires email + HMAC signature. | You have logged-in users and want them to manage their own support history. |
| **Hosted help center** | The public, SEO-indexed knowledge base, hosted by Helpdesky at a subdomain or custom domain. | You want a docs/help site. Just link to it — no code to build. |
| **Content API** | REST API to create/update/delete categories and articles. | You want to programmatically write or keep help articles in sync. |
| **Inbound email** | Forward the customer's existing support inbox into Helpdesky as conversations. | They already have a support@ address. Dashboard setup, not code. |

You can combine these. A common setup is: floating widget on every page + hosted help center linked from the nav + ticket center inside the account area.

---

## Dashboard setup the customer must do (in the Helpdesky UI)

Several features silently do nothing until the owner configures them in the Helpdesky dashboard. **You (the agent) cannot do these** — they require the customer's login. Walk them through whichever apply to the surfaces you're adding, and tell them exactly which menu to open.

| Setting | Where in the dashboard | Needed for | Required? |
|---|---|---|---|
| **AI provider API key** (OpenAI / Anthropic / Google / etc.) | **Settings → AI Settings** | The widget's "Ask AI" tab, AI semantic search, and AI article generation. The customer brings their own key — there is no platform fallback. | Required for any AI feature |
| **Enable Widget** | **Embeds → Widget** | The floating widget to appear at all | Required for the widget |
| **Enable messaging** | **Messages** page (click the **Enable messaging** button) | Contact form & ticket center to actually send/receive messages | Required for contact form + ticket center |
| **Allowed Embed Domains** | **Settings → Messages** | Restricting which sites may embed; add the customer's site domain (leave empty = any domain allowed) | Recommended for embeds |
| **HMAC secret** | **Embeds → Ticket Center** | Generating the secret used to sign ticket-center users | Required for ticket center |
| **Helpdesk slug & name** | **Settings → General** | The help-center URL — path-based `helpdesky.io/help/{slug}` always works; subdomain / custom domain are optional | Required |
| **Custom domain** | **Settings → Custom Domain** | Serving the help center on the customer's own domain; shows DNS/SSL verification status (CNAME → `origin.helpdesky.io`) | Optional, often plan-gated |
| **Branding** (logo, favicon, colors, theme, layout: Default / Sidebar / Docs) | **Branding** page | Look & feel of the help center and embeds | Optional |
| **Remove "Powered by Helpdesky"** | **Settings → General** | White-labeling public pages | Optional, plan-gated |
| **Publish articles** | **Articles** page | Anything to show in the help center and widget/AI search | Required for useful content |
| **Notifications** | **Settings → Notifications** | Choosing who gets emailed about new messages | Recommended with messaging |
| **Data Sources** | **Settings → Data Sources** | Extra URLs for the AI to learn from, improving AI answers | Optional |

Key dependencies to call out to the customer:
- The widget's **AI answers won't work without an AI provider key** in **Settings → AI Settings** — the "Ask AI" tab simply stays hidden until a key is added.
- The **contact form and ticket center won't process messages unless messaging is enabled** — the customer goes to the **Messages** page and clicks **Enable messaging** — and (if you've listed any) the site domain is in **Allowed Embed Domains**.
- Articles must be **published** to appear in the help center, widget search, or AI answers.

---

## Widget embed

One script tag, anywhere in the page (ideally before `</body>`). It injects a floating launcher with Shadow-DOM isolation, so it won't clash with the host app's styles.

```html
<script src="https://helpdesky.io/widget.js" data-helpdesk-id="YOUR_HELPDESK_ID"></script>
```

- `data-helpdesk-id` (required) — the helpdesk UUID.
- Appearance (position, colors, AI on/off, etc.) is configured in the dashboard, not via attributes.

---

## Contact form embed

A target `<div>` plus a script tag. The form renders inside the div (Shadow-DOM isolated).

```html
<div id="hdh-contact-form"></div>
<script src="https://helpdesky.io/contact-form.js"
  data-helpdesk-id="YOUR_HELPDESK_ID"></script>
```

Optional attributes:
- `data-align="center|left|right"` — alignment within its container (default `center`).
- `data-max-width="560px"` — max form width (default `560px`).
- `data-container="hdh-contact-form"` — custom target div id (must match the `<div>` id).

The customer's domain must be in the **Allowed Embed Domains** list (**Settings → Messages**), or the submit request is rejected with a CORS/"Origin not allowed" error.

The dashboard's copy snippet appends a cache-busting `?v=N` (e.g. `contact-form.js?v=5`) to the script URL — it's optional, and the plain `contact-form.js` URL works identically.

---

## Ticket center embed (authenticated)

The ticket center shows a signed-in user their own tickets. It authenticates the user with an **HMAC-SHA256 signature of their email**, computed **server-side** with the HMAC secret. The secret must never reach the browser.

### Server-side signing recipe

**Normalize the email to `email.trim().toLowerCase()` first, then use that exact same normalized string for BOTH the signature and the `data-email` attribute.** The server lowercases the email before verifying, and a case- or whitespace-mismatch between what you signed and what you put in `data-email` is the #1 cause of "Verification failed". Output is a hex digest.

Node.js:
```js
import crypto from "crypto";

// HELPDESKY_HMAC_SECRET is a server-only secret/env var
const email = userEmail.trim().toLowerCase();
const signature = crypto
  .createHmac("sha256", process.env.HELPDESKY_HMAC_SECRET)
  .update(email)
  .digest("hex");
// render the embed with this SAME `email` in data-email and `signature` in data-signature
```

Python:
```python
import hmac, hashlib, os

email = user_email.strip().lower()
signature = hmac.new(
    os.environ["HELPDESKY_HMAC_SECRET"].encode(),
    email.encode(),
    hashlib.sha256,
).hexdigest()
```

PHP:
```php
$email = strtolower(trim($userEmail));
$signature = hash_hmac('sha256', $email, getenv('HELPDESKY_HMAC_SECRET'));
```

### Embed markup

Render this only for logged-in users, injecting the **same normalized email** you signed (the `email.trim().toLowerCase()` value, not the raw input) and the server-computed signature:

```html
<div id="hdh-ticket-center"></div>
<script src="https://helpdesky.io/ticket-center.js"
  data-helpdesk-id="YOUR_HELPDESK_ID"
  data-email="USER_EMAIL"
  data-signature="HMAC_SIGNATURE"></script>
```

Optional: `data-align`, `data-max-width` (default `864px`), `data-container` (default `hdh-ticket-center`).

The customer's domain must be in **Allowed Embed Domains** (**Settings → Messages**) for the ticket-center endpoints to respond. As with the contact form, the dashboard snippet appends an optional cache-busting `?v=N` (e.g. `ticket-center.js?v=5`); the plain `ticket-center.js` URL works the same.

---

## Framework placement notes

- **Plain HTML** — paste the script tag(s) before `</body>`. For the ticket center, compute the signature in a server-side template (PHP, etc.) and inject `data-email`/`data-signature`.
- **React / Vite (SPA)** — inject the widget/contact-form script once on mount, e.g. in a top-level component:
  ```jsx
  useEffect(() => {
    const s = document.createElement("script");
    s.src = "https://helpdesky.io/widget.js";
    s.setAttribute("data-helpdesk-id", import.meta.env.VITE_HELPDESK_ID);
    document.body.appendChild(s);
    return () => { s.remove(); };
  }, []);
  ```
  For the **ticket center**, the HMAC signature must come from your backend (an API route that signs the logged-in user's email); the SPA fetches `{ email, signature }` and then injects the ticket-center script with those attributes. Never put the HMAC secret in `VITE_`-prefixed env vars — those ship to the browser.
  - **SPA mount order (contact form & ticket center):** these two scripts look up their target `<div>` once, the instant they run, and silently do nothing if it's missing (no retry/observer). So the `<div id="hdh-contact-form">` / `<div id="hdh-ticket-center">` must already be in the DOM **before** the script executes. Render the `<div>` first, then inject the script from a post-mount `useEffect` — don't place a literal `<script>` tag in JSX (React won't execute those). The widget needs no container div, so it can be injected anywhere.
- **Next.js** — put the widget/contact-form tag in a Client Component or `next/script`. For the ticket center, sign in a Server Component / Route Handler / `getServerSideProps` using a server-only env var (no `NEXT_PUBLIC_` prefix), then pass `email` + `signature` into the rendered script tag.

---

## Hosted help center (link to it — don't build it)

The customer's help center is **already live and hosted by Helpdesky**. You only need to link to it from the app; there is no help-center code to write.

- **Path-based (canonical, always works):** `https://helpdesky.io/help/{slug}`. This is what Helpdesky uses by default and what its homepage links to — prefer it unless the customer has a custom domain.
- **Custom domain (branded, optional):** the customer sets it in **Settings → Custom Domain** and adds a **CNAME** record pointing to `origin.helpdesky.io` (Cloudflare for SaaS fallback origin). Once verified it serves at `https://{custom-domain}`. You cannot automate the DNS step — just tell the customer to do it.
- **Subdomain (`https://{slug}.helpdesky.io`):** the server does resolve helpdesks by subdomain, but this only works if the customer's `{slug}.helpdesky.io` actually resolves (wildcard DNS). Don't assume it's live — confirm with the customer, or just use the path-based URL.

Add a "Help" or "Docs" link in the app's nav/footer pointing to whichever URL matches the customer's setup.

---

## Deep-link to a specific article

Link from contextual places inside the app (e.g. a "Learn more" link next to a feature) straight to the relevant article. Use the URL pattern that matches the customer's setup:

- Path-based (always works): `https://helpdesky.io/help/{slug}/{article-slug}`
- Custom domain: `https://{custom-domain}/{article-slug}`
- Subdomain (only if `{slug}.helpdesky.io` resolves): `https://{slug}.helpdesky.io/{article-slug}`

The `{article-slug}` is auto-generated from the article title (and visible in the dashboard / returned by the content API).

---

## Auto-write & sync help articles (Content API)

Use the REST API to create, update, and keep articles in sync — e.g. to generate docs for the app you are building and update them as the app changes. Authenticate every request with the `X-API-Key` header (keep the key server-side).

- **Base URL:** `https://helpdesky.io/api/v1`
- **Auth header:** `X-API-Key: hdh_your_api_key`
- **Content type:** `application/json` (except image upload, which is multipart)

### Categories
- `GET /categories` — list.
- `POST /categories` — body: `name` (required); optional `description`, `icon`. Slug auto-generated. (`order` can only be set later via `PATCH`.)
- `PATCH /categories/:id` — update `name` / `description` / `icon` / `order`.
- `DELETE /categories/:id` — remove.

### Articles
- `GET /articles` (optional `?categoryId=`) — list.
- `POST /articles` — body: `title` (required), `content` (required, **Markdown**); optional `excerpt`, `categoryId`, `slug`, `published`, `numberHeadings`, `hiddenFromWidget`.
- `PATCH /articles/:id` — update any of the above.
- `DELETE /articles/:id` — remove.

### Images
- `POST /images` — multipart form with an `image` field (≤ 5 MB; JPG / PNG / GIF / WebP / SVG). Returns `{ url, filename, size, contentType }`.
- Embed the returned `url` in article content. **Prefer an HTML `<img>` tag so you can size the image** — this is the same format the dashboard editor emits:
  ```html
  <img src="/api/images/uploads/.../screenshot.png" width="400" />
  ```
  Markdown `![alt](url)` also works but always renders at the image's full original size (no width/height control). Both formats can be mixed in the same article.

### Rich article formatting (Markdown extras)
Article `content` is Markdown. Beyond standard syntax (headings, **bold**, *italic*, lists, links, `code`, fenced code blocks, blockquotes, and GitHub-style tables), Helpdesky renders these extras on the help center and widget — use them so generated docs look polished instead of plain-text:

- **Callouts** — a **blockquote** whose first line is `[!info]`, `[!warning]`, or `[!danger]` (the `>` markers are required — a plain paragraph won't render as a callout):
  ```
  > [!warning]
  > Back up your data before continuing.
  ```
- **Buttons** — a link with a `{.btn}` (filled) or `{.btn-outline}` modifier; optionally add `.nofollow` and/or `.same-tab` (open in the same tab):
  ```
  [Get started](https://app.example.com/signup){.btn}
  [Read the guide](/help/acme/setup){.btn-outline .same-tab}
  ```
- **Action cards** — a link with `{.card ...}`; optional `icon="…"`, `desc="…"`, `sameTab`, `nofollow`. Two or more consecutive cards form a grid:
  ```
  [Quick start](/help/acme/quick-start){.card icon="rocket" desc="Up and running in 5 minutes"}
  [Watch a demo](/help/acme/demo){.card icon="play" desc="2-minute video tour"}
  ```
  Valid `icon` values: `user-plus`, `play`, `book-open`, `rocket`, `settings`, `mail`, `download`, `link`, `info`, `check`, `sparkles`, `code` (an unknown name falls back to `info`).
- **YouTube embeds** — put a YouTube link on its own line (or as a plain Markdown link); `youtube.com/watch?v=…`, `youtu.be/…`, and `youtube.com/embed/…` auto-embed as a responsive player.

### Example: create a category + published article

```js
const BASE = "https://helpdesky.io/api/v1";
const headers = {
  "X-API-Key": process.env.HELPDESKY_API_KEY, // server-side secret
  "Content-Type": "application/json",
};

// 1. Category
const cat = await fetch(`${BASE}/categories`, {
  method: "POST", headers,
  body: JSON.stringify({ name: "Getting Started", description: "Set-up help" }),
}).then(r => r.json());

// 2. Article (Markdown content), published so it appears
const article = await fetch(`${BASE}/articles`, {
  method: "POST", headers,
  body: JSON.stringify({
    title: "How to create your first project",
    content: "## Welcome\n\n1. Click **New Project**\n2. Name it\n3. Done!",
    categoryId: cat.data.id,
    published: true,
  }),
}).then(r => r.json());

console.log("Created:", article.data.title, "→", article.data.slug);
```

### Keeping docs in sync
To revise docs as the app evolves, store each article's returned `id` and call `PATCH /articles/:id` to update `content`/`title`, or `DELETE /articles/:id` to remove. Same pattern for categories.

### Notes
- Articles must be **`published: true`** to appear in the help center and widget.
- **Slugs auto-generate** from the title (or pass an explicit `slug`); changing a published article's slug creates a redirect from the old one.
- **Plan-based article limits** are enforced — `POST /articles` returns HTTP 403 with code `ARTICLE_LIMIT_REACHED` when the limit is hit; surface that to the customer (they need to upgrade).
- **Rate limit: 60 requests/minute per API key.** Every response carries `RateLimit-Limit`, `RateLimit-Remaining`, and `RateLimit-Reset` headers; exceeding it returns HTTP 429 `{ "error": "Rate limit exceeded. Please try again later." }`. When seeding or syncing many articles, pace requests to stay under 60/min (watch `RateLimit-Remaining`) and back off / retry after `RateLimit-Reset` on a 429.
- Authoritative, always-current reference lives at **https://helpdesky.io/docs/api**.

---

## Inbound email support channel (dashboard setup, not code)

The customer can turn their existing support inbox into Helpdesky conversations by forwarding mail to:

```
{slug}-{code}@inbox.helpdesky.io
```

The 6-digit `{code}` is shown on the **Embeds → Email Forwarding** dashboard page and can be regenerated there. Forwarded emails become conversations in the inbox.

This is configured in the customer's email provider (e.g. a forwarding rule on their support@ address) — **you cannot perform the forwarding setup**. Walk the customer through it and point them to the **Embeds → Email Forwarding** page for the exact address and code.

---

## Troubleshooting

- **Ticket center "Verification failed" / signature rejected.** Almost always an email-normalization mismatch: the value you signed must equal `data-email` after the server lowercases it. Sign and send the same `email.trim().toLowerCase()` value. To debug, use the dashboard's **HMAC Verification Tool** (**Embeds → Ticket Center**): paste an email, generate a test signature, and compare it to what your server produces — enter the email already trimmed + lowercased so it matches your server-side normalization.
- **Contact form / ticket center renders nothing in an SPA.** The script looks up its `<div>` once and bails silently if it's missing. Ensure the `<div id="hdh-…">` exists before the script runs (render the div first, inject the script in a post-mount `useEffect`).
- **Embed request rejected (CORS / "Origin not allowed"), or messages never arrive.** The site's domain isn't in **Allowed Embed Domains** (**Settings → Messages**), or messaging isn't enabled — open the **Messages** page and click **Enable messaging**.
- **Widget shows but has no "Ask AI" tab.** No AI provider key is configured (**Settings → AI Settings**) — the AI tab stays hidden until the customer adds their own key.
- **Content API returns 429.** You exceeded 60 requests/minute — slow down and retry after `RateLimit-Reset`.

## Verification checklist before you finish

- Helpdesk ID / API key / HMAC secret are stored as secrets, not hardcoded, and the HMAC secret + API key never reach the browser.
- The customer has done the required dashboard setup for the surfaces you added (see "Dashboard setup the customer must do") — e.g. **Enable Widget**, click **Enable messaging** on the **Messages** page, and add an **AI provider key** if AI answers are expected.
- The customer's domain is added to **Allowed Embed Domains** if you embedded the contact form or ticket center.
- Ticket-center signatures are computed server-side over the **trimmed + lowercased** email, and `data-email` carries that same normalized value.
- Help-center and deep-link URLs match the customer's actual setup — default to path-based `helpdesky.io/help/{slug}` unless they have a custom domain.
- Articles you create via the API are `published: true` if they should be visible.
