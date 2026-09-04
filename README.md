# ForSale-check — RFC 10023 domain checker extension

🧪 Experimental ⚠️
(developed with AI)

> [!NOTE]
> While this works on Google Chrome, we noticed that it does not on the Chromium instance we tested. This might be a local problem on our end. We're looking into this.

Screenshot:

<img src="/screenshot.png" alt="Screenshot" width="300">

## forsale-check-extension.zip

A minimal Chrome extension (Manifest V3) that checks whether the domain of
the site you're currently on has a `_for-sale` TXT record, as defined in
**RFC 10023** (July 2026, author M. Davids, SIDN Labs). It checks live as
you browse: the toolbar icon itself acts as a status lamp, no click needed.

## The toolbar "lamp"

The icon has four states, set per-tab as you navigate:

| Color  | Meaning                                   |
|--------|--------------------------------------------|
| Grey   | Not checked yet (e.g. an internal browser page) |
| Blue   | Checked — no `_for-sale` record found       |
| Bright green | Checked — domain **is** marked for sale     |
| Red    | The DNS check itself failed (see popup for details) |

Click the icon any time for the full detail (which record fields were
found, and — if a redirect was involved — the status of both the domain
you started at and the one you landed on).

## How it works

Browser extensions don't have a built-in DNS resolver — you can't just do
a TXT lookup from JavaScript. This extension works around that with
**DNS-over-HTTPS (DoH)**: a plain `fetch()` to Cloudflare's public DoH
endpoint (`cloudflare-dns.com/dns-query`), which returns the answer as JSON.

**On redirects and why it's only ever a 2-point chain:** 
A navigation that bounces through several redirects before landing is only
ever seen here as *where it started* and *where it ended*, not every
intermediate hop. For "example.nl redirects to example.org" — the case
this was built for — that's exactly what's needed; a multi-hop chain would
show only the first and last domain.

Clicking the icon opens the popup, which reads the result the background
worker already computed for that tab (instant, no extra request) and shows
each domain checked with its status, plus any RFC 10023 tag content
(`fcod=`, `ftxt=`, `furi=`, `fval=`) parsed per the ABNF in RFC 10023 §2.1.
If no live result exists yet for some reason, the popup falls back to
checking the current tab's domain itself.

Lookups are cached for 10 minutes per domain in `chrome.storage.session`
(cleared when the browser closes — nothing is written to disk), so
revisiting the same site repeatedly during a session doesn't repeat the
DoH request every time.

## A note on permissions

This extension requests:

- **`webNavigation`** — to observe the URL of each top-level navigation.
  This is the only Chrome API that reports the URL for *every* site without
  needing a per-site host permission, which is exactly why it's the right
  fit here — but be aware Chrome shows it to users as **"Read your browsing
  history"** in the install prompt. That warning is broader-sounding than
  what the code actually does (it never records a history, never sends URLs
  anywhere except the registrable domain used to build a single DoH query,
  and nothing is persisted beyond the current browser session) — but there
  is no narrower built-in API for "current navigation's URL only". Worth
  stating plainly in the Chrome Web Store listing's permission
  justification field, since reviewers will ask about it.
- **`activeTab`** — lets the popup read the URL of the tab the user just
  clicked into, without granting standing access to any tab.
- **`storage`** — for the short-lived, session-only cache described above.
- **`host_permissions: ["https://cloudflare-dns.com/*"]`** — the DoH
  resolver endpoint. A single named host is one of the least broad
  permission types Chrome offers (compare to `<all_urls>`).

No content-script or page-content access of any kind is requested — the
extension never sees anything on the pages you visit beyond their URL.

## Installing on Chromium (unpacked, for local testing)

Download the ZIP-file, unpack it, then...

1. Open `chrome://extensions`.
2. Turn on **Developer mode** (top right).
3. Click **Load unpacked**.
4. Select th containing `manifest.json`.
5. The icon appears in the toolbar, starting grey. Browse normally — it
   turns blue or bright green once a page has been checked. Click any time for
   details.
6. Pin it to the toolbar.

After editing the code: go to `chrome://extensions` and click the ↻ icon
to reload. Manifest changes (e.g. permissions) always require a reload. If
`chrome://extensions` shows a red "Errors" button, click it — it gives the
exact file/line, which is far more useful than the one-line summary shown
at the top of the page.

## Files

- `manifest.json` — configuration (MV3, permissions, popup, background worker, icons)
- `background.js` — live check on navigation, toolbar lamp state, session cache writes
- `shared.js` — PSL resolution, DoH lookup, RFC 10023 parsing (used by both `background.js` and `popup.js`)
- `popup.html` / `popup.css` / `popup.js` — the UI shown when you click the icon
- `icons/` — `icon*.png` (brand icon, used on the extensions/store page), `lamp-*-*.png` (toolbar status lamp, one set of 16/32/48px per state)
- `libs/psl.js` — bundled Public Suffix List library (MIT license, see `libs/psl-LICENSE.txt`)

## Known limitations

- Only Cloudflare is used as a DoH resolver; there's no fallback if it's
  unreachable (e.g. blocked by a corporate firewall or security software).
- The in-progress navigation's start URL is held in memory in the service
  worker between `onBeforeNavigate` and `onCommitted`. If Chrome
  terminates the worker in that window (it can do this after ~30s idle),
  the redirect's origin is lost and only the final URL gets checked — a
  rare edge case, since that window is normally well under a second.
- As explained above, only the first and last domain of a redirect chain
  are checked — not every intermediate hop.
- No badge/lamp state for anything beyond the top-level frame — iframes
  are intentionally ignored, matching the "just the visited URL, nothing
  else" design goal.
- Sub domains may go wrong. For example `dynamic.testdns.nl` will be seen as `testdns.nl`, just as `www.testdns.nl` would.

## Disclaimer
Although developed with care, this experimental tool is provided “as is”, without guarantees of functionality or availability. Use it at your own risk. No liability is accepted. We may withdraw it at any time without notice.

