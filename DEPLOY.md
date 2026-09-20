# Mantisy — static site deploy (unbundled)

Upload the **contents** of this folder to the site root, so that
`https://mantisy.com/index.html`, `/support.js` and `/assets/…` all resolve.

```
index.html          the page (markup + logic, ~186 KB)
support.js          the runtime that renders it
kh-dict.js          Khmer strings, loaded deferred
assets/             logos, marks, flags, AEON logo
og-image.png        social preview
llms.txt            for AI crawlers — must sit at the root
robots.txt
sitemap.xml
.do/app.yaml        DigitalOcean App Platform spec
```

## Required host settings

- **index document**: `index.html`
- **catchall document**: `index.html` — clean URLs (`/services/pos`, `/work/aeon-online`)
  are handled by the in-page router, so unknown paths must serve the page rather than 404.

`.do/app.yaml` already sets both. Point it at your repo before deploying.

## Why paths are absolute

Every asset is referenced as `/assets/…`, not `assets/…`. With a catchall in
place, a relative path on `/services/pos` would resolve to
`/services/assets/…`, which the catchall answers with HTML instead of an
image. Absolute paths are what make deep links safe — keep them that way.

## The enquiry form

Unchanged: it posts to the DigitalOcean Function in `export/do-function/`.
The Telegram token stays in the Function's environment, never in this folder.

## Caching

Set a long `max-age` on `assets/`, `support.js` and `kh-dict.js`, and a short
one (or `no-cache`) on `index.html`, so a new deploy is picked up immediately
while the heavy files stay cached.

## What changed from the single-file build

- Dropped the unused Modernist design-system CSS and JS bundle — nothing on
  the page referenced them (that was the "203 KiB unused CSS" in Lighthouse).
- React is served from this origin (`/vendor/`) instead of unpkg.
- The Khmer font is fetched only when a visitor taps the Khmer flag.
- `kh-dict.js` is deferred.
- Images carry explicit `width`/`height`, and below-the-fold ones are lazy.


## One manual step: the React files

`index.html` loads React from `/vendor/`. Fetch both files once and upload
them alongside everything else:

```sh
mkdir -p vendor
curl -o vendor/react.production.min.js \
  https://unpkg.com/react@18.3.1/umd/react.production.min.js
curl -o vendor/react-dom.production.min.js \
  https://unpkg.com/react-dom@18.3.1/umd/react-dom.production.min.js
```

Why: on a throttled mobile connection unpkg cost ~2,400 ms of render-blocking
time — a separate DNS lookup and TLS handshake before 47 KB could even start
downloading. Served from mantisy.com the connection is already open.

If either file is missing the runtime falls back to unpkg on its own, so a
forgotten upload makes the site slow again but never broken. Verified.

Give `/vendor/` a long cache lifetime — the filenames are version-pinned.
