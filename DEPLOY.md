# Mantisy website — deploy notes

## 1 · Files to upload to the web root
- `index.html` — the whole site, one self-contained file
- `og-image.png` — link preview image
- `robots.txt`, `sitemap.xml`

`DEPLOY.md`, `.do/app.yaml`, `do-function/` and `telegram-proxy.worker.js`
are for you, not the server. Do not upload them.

## 2 · Clean URLs — one required setting
Paths like `/services/pos` are handled inside the page. The server must serve
`index.html` for unknown paths, or a visitor landing directly on
`/services/pos` gets a 404.

**DigitalOcean control panel:** App → Settings → your static site component →
**Catchall document** → `index.html` → Save.

(Or use `.do/app.yaml`: edit the repo name, then
`doctl apps update <app-id> --spec .do/app.yaml`.)

## 3 · Contact form → Telegram

### Why the token cannot just be locked to your domain
BotFather's `/setdomain` only restricts the **Telegram Login Widget**, not the
Bot API. A bot token is a full credential: anyone who reads it can post as your
bot, read what it receives, or hijack it with `setWebhook` — from any script,
anywhere. Telegram offers no origin restriction, and CORS cannot help because
browsers enforce CORS and attackers do not use browsers.

So the token must live somewhere the public cannot read it. You are already on
DigitalOcean, so use **DigitalOcean Functions** — same account, no new vendor,
and the free allowance (90,000 GiB-seconds/month) is far beyond what a contact
form uses.

### Step 1 — get a fresh token
The old token was shared in chat, so retire it:
1. **@BotFather** → `/mybots` → **@mantisybot** → **API Token** → **Revoke current token**
2. Copy the new token. It goes only into step 3.

### Step 2 — create the function
DigitalOcean console → **Functions** → **Create Namespace** (any name, region `sgp1`)
→ **Create Function** → runtime **Node.js 18**, name **submit**.

Open the editor, delete the sample, and paste all of
`do-function/packages/enquiry/submit/index.js`. **Save**.

### Step 3 — add the two variables
Same function → **Settings** → **Environment Variables** → add:

| Key | Value |
|---|---|
| `TG_TOKEN` | the new token from step 1 |
| `TG_CHAT` | `-983521664` |

**Save**, then **Deploy**.

### Step 4 — make it web-accessible and copy the URL
In the function's settings, enable **Web** (public HTTP access), then copy the
endpoint URL. It looks like:

    https://faas-sgp1-XXXX.doserverless.co/api/v1/web/fn-XXXX/enquiry/submit

### Step 5 — point the site at it
In `index.html`, find

    proxy: 'PASTE_YOUR_WORKER_URL_HERE'

and replace the placeholder with your function URL, keeping the quotes.
Or send me the URL and I will wire it in and rebuild.

### Step 6 — test
Submit the form on the live site. The enquiry should appear in
**Mantisy Contact**. If not, open the function's **Logs** and submit again —
the Telegram error will be printed there.

### Prefer the CLI?
`do-function/` is a ready doctl project:

    cd do-function
    export TG_TOKEN=... TG_CHAT=-983521664
    doctl serverless deploy .

### Cloudflare alternative
If you ever do use Cloudflare, `telegram-proxy.worker.js` is the same thing as
a Worker. Either one works; you only need one.

### Until step 5 is done
The form opens the visitor's mail client with the enquiry pre-filled to
business@mantisy.com, so no enquiry is lost in the meantime.

## Notes
- The allowed-origins list is at the top of the function. Add any other domain
  you serve the site from, or requests from it will be refused.
- Telegram group ids are negative — keep the minus sign.
- The site sends the enquiry as `text/plain` so there is no CORS preflight;
  do not change that unless your endpoint handles `OPTIONS`.
