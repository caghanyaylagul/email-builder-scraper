# Visual Email Composer

A Chrome Extension (Manifest V3, React + TypeScript) that turns any ecommerce page into a responsive HTML email newsletter with one click.

Two capture modes:

- **Current page** — extract products from the tab you're viewing right now.
- **Multiple URLs** — paste a list of URLs; the extension visits each one in parallel and combines the products into a multi-section "roundup" newsletter.

---

## 1. Product concept summary

Marketers and ecommerce operators spend hours assembling promotional emails: pulling product images, copying prices, formatting tables, testing in Litmus. Visual Email Composer collapses that loop into one click — the extension reads the page DOM, extracts structured product data, ranks the best six picks, and renders a 600px responsive email built from tables and inline CSS that survives every major email client.

The MVP is local-only (no account, no backend). The architecture is structured so a hosted dashboard, AI copywriting, and ESP integrations (Revotas, Mailchimp, Klaviyo, Brevo) can be added without rewriting the extension.

## 2. Recommended architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                       Active tab (any URL)                         │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ content.js                                                   │  │
│  │   - listens for EXTRACT_REQUEST                              │  │
│  │   - runs lib/extractor.ts against document + window.location │  │
│  │   - returns CapturedData                                     │  │
│  └──────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘
        ▲                                              ▲
        │ chrome.tabs.sendMessage                      │ chrome.storage.local
        │                                              │
┌──────────────────┐                          ┌──────────────────────┐
│ popup (React)    │                          │ background worker    │
│  - Capture btn   │  chrome.storage.local    │  - install hook      │
│  - Preview       │ ◀───────────────────────▶│  - capture counter   │
│  - Copy/Download │                          │  - quota groundwork  │
│  - Send test ✦   │                          └──────────────────────┘
└──────────────────┘                                     │
        │                                                │
        │ chrome.tabs.create(preview.html)               │
        ▼                                                │
┌──────────────────┐  reads stored html via chrome.storage.local
│ preview (React)  │ ◀──────────────────────────────────┘
│  - iframe srcdoc │
│  - desktop/mob   │
└──────────────────┘
```

**Why this shape**

- Pure modules in `src/lib/` (extractor, emailGenerator, types) are framework- and runtime-agnostic. They can later run in a Node Lambda for an "Email by API" product.
- The popup never talks to the page directly — it goes through the content script via `chrome.tabs.sendMessage`. Everything is message-passing, so swapping the popup for a side panel or full dashboard is trivial.
- The preview lives in its own tab. The popup closes the moment Chrome loses focus, so a full-screen preview has to be a real tab loaded from `chrome-extension://…`.

## 3. Chrome Extension file structure

```
email-builder-scraper/
├── manifest.json                       # MV3 manifest
├── package.json                        # vite + react + ts
├── tsconfig.json
├── vite.config.ts                      # popup + preview + background build
├── vite.content.config.ts              # IIFE bundle for content.js
├── scripts/
│   └── gen-icons.mjs                   # generates icon16/48/128.png
├── public/
│   └── icons/                          # generated PNG icons
└── src/
    ├── lib/
    │   ├── types.ts                    # shared types + StoredCapture shape
    │   ├── messaging.ts                # chrome.runtime/tabs helpers
    │   ├── extractor.ts                # Document → CapturedData (pure)
    │   ├── multiCapture.ts             # batch fetch + DOMParser + extract
    │   └── emailGenerator.ts           # single + multi-source HTML
    ├── content/
    │   └── contentScript.ts            # runs in page, calls extractor
    ├── background/
    │   └── serviceWorker.ts            # MV3 background module
    ├── popup/
    │   ├── index.html
    │   ├── main.tsx
    │   ├── App.tsx                     # mode tabs, capture + batch UI
    │   └── popup.css
    └── preview/
        ├── index.html
        ├── preview.tsx
        └── preview.css
```

### Why two Vite configs?

The content script and the popup both import `src/lib/extractor.ts`. By default Rollup hoists shared modules into a separate chunk and rewrites both entries to `import` it at runtime. That breaks MV3 content scripts, which run in an isolated world and cannot perform runtime ES imports. To keep `content.js` self-contained, the content script is built with `vite.content.config.ts` using `lib.formats: ['iife']`, which inlines all transitive deps into a single file. The main `vite.config.ts` handles the popup, preview, and background worker.

## 4. Manifest V3 configuration

See [`manifest.json`](./manifest.json). Key choices:

- `manifest_version: 3` — required for new Chrome Web Store listings.
- `permissions: ["activeTab", "scripting", "storage", "tabs"]` — minimal set. `scripting` lets us inject `content.js` on demand into tabs that were already open at install time.
- `host_permissions: ["<all_urls>"]` — the extension is useful on any site, so we can't narrow this.
- `content_scripts` runs `content.js` at `document_idle` so ecommerce pages have finished hydrating.
- `web_accessible_resources` exposes `src/preview/index.html` so the popup can open it via `chrome.runtime.getURL(...)`.
- Background runs as a `type: "module"` service worker.

## 5. React popup code

See [`src/popup/App.tsx`](./src/popup/App.tsx). It has two modes toggled by tabs at the top:

### Current page mode

Click **Capture this page** → the popup messages the content script in the active tab → the extractor runs against `document` and returns structured product data → the email HTML is generated and cached in `chrome.storage.local`.

### Multiple URLs mode

Paste any number of URLs (one per line; commas and tabs also accepted) → the popup calls `batchCapture()` from [`src/lib/multiCapture.ts`](./src/lib/multiCapture.ts) → a worker pool of size 4 fetches each URL via `fetch()` with `credentials: 'omit'` and a 15s timeout, parses the HTML with `DOMParser`, and runs the same extractor against the parsed document.

The popup shows live progress as each URL completes: a progress bar, a per-URL list with success/failure icons, and the number of products detected. After all URLs settle, the multi-source generator stitches them into a single newsletter with one section per source. URLs that returned no products or failed to fetch are surfaced as "N of M URLs were skipped" inside the email and as a `failures` count on the summary card.

| Button         | Purpose                                                                                 |
| -------------- | --------------------------------------------------------------------------------------- |
| Capture this page | Extract from the active tab (single-page mode).                                      |
| Capture N URLs    | Run the batch fetcher against the URL list (multi mode).                             |
| Preview        | Open `src/preview/index.html` in a new tab.                                            |
| Copy HTML      | Write generated HTML to the clipboard.                                                  |
| Download .html | Save as `{brand}-newsletter.html`.                                                      |
| Send test ✦    | Placeholder for the future Pro feature.                                                 |
| Capture another | Reset to the idle state without losing the saved capture in storage.                   |

The popup rehydrates on open from `chrome.storage.local`, so closing/reopening it doesn't lose your last capture.

**Caveat**: the batch runs inside the popup's JS context. If you close the popup mid-batch, in-flight fetches are aborted. The UI surfaces this with "Keep this popup open until all URLs finish." Moving the batch into the service worker is a planned follow-up.

## 6. Content script — ecommerce extraction

See [`src/lib/extractor.ts`](./src/lib/extractor.ts) (pure module) and [`src/content/contentScript.ts`](./src/content/contentScript.ts) (Chrome glue).

Extraction priorities, in order:

1. **schema.org Product JSON-LD** — most reliable. Walks any `<script type="application/ld+json">`, recurses through nested `ItemList`/`@graph` structures, and pulls `name`/`image`/`url`/`offers.price`.
2. **DOM heuristics** — finds elements matching common product-card selectors (`.product-card`, `.product-item`, `[data-product-id]`, `li.product`, `article.product`, etc.) and falls back to any `<a>` containing both an `<img>` and a price-like text node. Supports Shopify, WooCommerce, BigCommerce, Magento, and most custom builds.
3. **Open Graph + favicon chain** — for logo, hero, brand name fallback.

Other guarantees:

- All `src`/`href` values are normalized to absolute URLs against `window.location.href`. Relative URLs would break the email the moment it leaves the user's machine.
- Products are de-duped first by URL (stripped of query string), then by image URL.
- Products are ranked by completeness (image + name + price + URL → top), then the top 6 are returned.
- Discount badges, original-price markers, and sale labels are detected via class-name heuristics (`[class*="sale" i]`, etc.).

## 7. Email HTML generation function

See [`src/lib/emailGenerator.ts`](./src/lib/emailGenerator.ts). Two public functions:

- `generateEmailHtml(data)` — single-page capture; one product grid with up to 6 products.
- `generateMultiSourceHtml(sources)` — multi-URL roundup; one section per successful source with up to 4 products each, sharing a single header (from the first source) and a single footer. Skipped/failed URLs surface as a small note inside the email.

Both share an internal `wrapEmailShell(...)` helper so client resets, the mobile media query, the Outlook VML preamble, and the footer stay in sync across both layouts.

Highlights of the produced HTML:

- 600px-wide table layout (Outlook ignores most modern CSS).
- Inline styles on every element (Gmail strips `<style>` on quoted replies).
- A single `<style>` block in `<head>` for the responsive `@media` rule that collapses the two-column product grid to one column under 620px.
- VML conditional comments for bulletproof CTAs in Outlook for Windows.
- System font stack with web-safe fallbacks.
- Plain-text preheader (hidden div) so inbox previews aren't blank.
- Discount badge, price block, and CTA per product cell.

## 8. Preview page / component

See [`src/preview/preview.tsx`](./src/preview/preview.tsx). Renders the generated HTML in an `<iframe srcdoc="…">` at either desktop (680px frame) or mobile (380px frame) widths, with Copy and Download buttons on a sticky toolbar.

## 9. Copy and download HTML

- **Copy** uses `navigator.clipboard.writeText(html)` — available in both the popup and preview because both run from `chrome-extension://` origins.
- **Download** creates a `Blob` of MIME type `text/html`, gives it an object URL, programmatically clicks an `<a download>`, and revokes the URL afterward.

## 10. Notes for future monetization

The MVP intentionally has no auth, no telemetry, no backend. It does have:

- A persistent `captureCount` in `chrome.storage.local`, so a free-tier quota (e.g., 5/month) can be enforced client-side at any time.
- Pure modules in `src/lib/` that can be lifted into a hosted backend without changes.
- A placeholder **Send test ✦** button to anchor the Pro upsell.

Planned tiers:

| Tier   | Features                                                                                 |
| ------ | ---------------------------------------------------------------------------------------- |
| Free   | 5 captures / month, basic template                                                       |
| Pro    | Unlimited captures, AI subject lines, AI product descriptions, brand kit detection       |
| Team   | Shared brand templates, library of saved newsletters, role-based access                  |
| Agency | Per-client workspaces, white-label export, REST export endpoint                          |

Integration roadmap:

- One-click send to **Revotas**, Mailchimp, Klaviyo, Brevo (OAuth + REST).
- Save captured newsletters to a hosted dashboard.
- AI module sits behind a single `/enrich` endpoint that takes `CapturedData` and returns subject + intro + product blurbs.
- Brand kit detection: persist `brandColor`, `accentColor`, fonts, logo per origin so subsequent captures auto-style.

---

## Install & run locally

### Prerequisites

- Node.js 20+
- Google Chrome (or any Chromium browser — Edge, Brave, Arc all work)

### Build

```bash
npm install
npm run build
```

This produces a `dist/` folder containing `manifest.json`, `background.js`, `content.js`, the popup, the preview, and `icons/`.

### Load the unpacked extension

1. Open `chrome://extensions/`.
2. Toggle **Developer mode** on (top right).
3. Click **Load unpacked**.
4. Select the `dist/` folder produced by `npm run build`.
5. Pin **Visual Email Composer** to the toolbar.

### Test it

Try any of these (most have JSON-LD or clean product grids — good for first runs):

- A Shopify storefront, e.g. `https://www.allbirds.com/collections/mens`
- A WooCommerce demo store, e.g. `https://demo.woothemes.com/storefront/`
- Any Amazon, Etsy, or generic ecommerce homepage

#### Single-page capture

1. Navigate to the page.
2. Click the extension icon → leave **Current page** tab selected → click **Capture this page**.
3. After ~1 second the summary card appears with detected products.
4. Click **Preview** to see the rendered email in a new tab.
5. Click **Copy HTML** to paste into your ESP, or **Download .html** to save it.

#### Multi-URL capture

1. Click the extension icon → switch to the **Multiple URLs** tab.
2. Paste one URL per line, for example:
   ```
   https://www.allbirds.com/collections/mens
   https://www.allbirds.com/collections/womens
   https://www.allbirds.com/collections/sale
   ```
3. Click **Capture N URLs**. The progress bar fills as each URL settles; per-URL status appears beneath it.
4. **Keep the popup open** until all URLs finish (closing it cancels in-flight fetches).
5. Once done, the summary card shows aggregate stats; **Preview** opens the multi-section newsletter.

Sites that fail to extract (no products found, HTTP errors, timeouts, or JS-only rendering) are reported per-URL and skipped in the final email.

### Dev loop

```bash
npm run dev
```

Runs `vite build --watch`. After changing source files, click the refresh icon for the extension on `chrome://extensions/` — Chrome doesn't auto-reload MV3 extensions, but pages with the content script will pick up the new build on next navigation.

### Troubleshooting

- **"No products were detected on this page."** — The page either has no products, or uses a non-standard markup that the heuristics miss. Open DevTools on the page and check whether `document.querySelectorAll('[class*="product"]').length` returns anything; if it does, file an issue with the URL so we can add a selector.
- **Logo missing.** — The site doesn't have an OG site_name, application-name, or recognizable header logo. Brand falls back to the hostname.
- **Outlook for Windows renders weirdly.** — Test in [Litmus](https://litmus.com) or [Email on Acid](https://www.emailonacid.com); the VML CTA fallback handles button rendering but Outlook desktop still ignores `border-radius` on most elements (expected).
#   e m a i l - b u i l d e r - s c r a p e r  
 