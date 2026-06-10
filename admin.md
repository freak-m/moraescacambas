# Admin Panel — How It's Built

## Overview

The admin panel is a single static HTML file (`admin/index.html`) with no backend server.
It talks directly to GitHub's API to read and write site content, and to a Cloudflare Worker to fetch analytics.

---

## Authentication

- **Login screen** asks for a GitHub Personal Access Token (PAT) as the password
- The PAT is stored in `sessionStorage['gh_token']` — cleared when the tab closes or the user logs out
- On page load, `autoLogin()` checks if a token already exists in sessionStorage and skips the login screen
- The repo is hardcoded as `const REPO = 'freak-m/moraescacambas'` (~line 1491)

---

## How Content Editing Works

1. On login, `loadContent()` calls `GET /repos/{REPO}/contents/content.json` via GitHub API
2. The JSON is decoded from base64 and parsed
3. `populateForm(content)` fills all input fields and rich-text editors (Quill.js) with the data
4. The live preview iframe (`index.html` in an `<iframe>`) is updated via `postMessage` as the user types
5. On **Publicar**, `collectForm()` reads all fields back into a JSON object, then `PUT /repos/{REPO}/contents/content.json` commits the new file to GitHub
6. GitHub Pages rebuilds automatically (usually takes ~30 seconds)

### Rich Text Fields
Uses **Quill.js 1.3.7** (CDN). Fields that use Quill: hero subtitle, service description, how-it-works steps, benefits, contact subtitle/panels.

### Image Uploads
Images are base64-encoded in the browser and committed directly to the repo via GitHub API (`PUT /repos/{REPO}/contents/{filename}`). The public URL becomes `https://moraescacambas.com.br/{filename}`.

---

## Tabs

| Tab | ID | What it does |
|---|---|---|
| Conteúdo | `tab-content` | All site content editing (SEO, hero, services, etc.) |
| Dashboard | `tab-dashboard` | Cloudflare analytics charts |
| Integrações | `tab-integrations` | GA ID, Meta Pixel, Worker URL |

`NO_PREVIEW_TABS = new Set(['dashboard', 'integrations'])` — these two tabs hide the preview iframe.

---

## Dashboard (Cloudflare Analytics)

### Flow
1. On tab open, `initDashboard()` reads `localStorage['cf_worker_url']` (or defaults to `https://analytics.moraescacambas.com.br/`)
2. `loadDashboardData()` checks localStorage for cached data (`cf_cache_7d`, `cf_cache_30d`, `cf_cache_90d`)
3. **Cache-first**: if cache exists, renders immediately, then refreshes in background
4. Yellow banner "Atualizando dados…" shows during background refresh
5. Red banner shows if refresh fails (keeps displaying last cached data)

### Cache rules
- Hard expiry: **2 hours** (`CACHE_MAX_MS`)
- Background recheck: **5 minutes** (`CACHE_CHECK_MS`)
- Cache keys: `cf_cache_7d`, `cf_cache_30d`, `cf_cache_90d`

### Worker (`fetchCfAnalytics`)
- Calls `GET {workerUrl}?days=N`
- Worker queries **Cloudflare GraphQL Analytics API** using `CF_API_TOKEN` and `CF_ZONE_ID` (env vars, never in source code)
- Returns: `{ data: { viewer: { zones: [...] } }, events: {...} }`

### Important Cloudflare API constraints
- `httpRequests1dGroups` — supports multi-day ranges (used for daily visits chart)
- `httpRequestsAdaptiveGroups` — **1-day range only** — both `date_geq` and `date_leq` must be today's date
- `webAnalyticsRumPageloadEventsAdaptiveGroups` — **1-day range only** — same constraint

### KV Events
The main `index.html` writes events to the Worker's KV namespace (`CACAMBAS_KV`) when users:
- Click the WhatsApp button (`wa` counter)
- View the service photo (`photo` counter)
- Spend time on the page (`time` events)

The dashboard reads these counters from the Worker response (`json.events`).

---

## Cloudflare Worker

Deployed separately at `analytics.moraescacambas.com.br` (Cloudflare Workers).

**Environment variables (set in Worker settings, never in code):**
- `CF_API_TOKEN` — Cloudflare API token with Zone Analytics Read permission
- `CF_ZONE_ID` — Zone ID of moraescacambas.com.br

**KV Binding:**
- Variable name: `CACAMBAS_KV`
- Namespace: bound in Worker settings

---

## Integrations Tab

| Field | Stored where | Used for |
|---|---|---|
| Google Analytics ID (`G-XXXXXXX`) | `content.json > integrations.google_analytics` | GA4 script injected in `index.html` |
| Meta Pixel ID | `content.json > integrations.meta_pixel` | Meta Pixel script |
| Meta domain verify | `content.json > integrations.meta_verify` | `<meta>` tag in `<head>` |
| Analytics Worker URL | `localStorage['cf_worker_url']` + `content.json > integrations.worker_url` | Dashboard data fetching |

---

## Security

- **GitHub PAT**: `sessionStorage` only. Never localStorage, never source code.
- **CF API Token**: Worker env var only. Never in the HTML file or content.json.
- **CF Zone ID**: Worker env var only.
- **Worker URL**: localStorage + content.json (acceptable — not a secret).
- `<meta name="robots" content="noindex, nofollow">` on the admin page prevents search engine indexing.

---

## Key Constants (in `admin/index.html`)

```javascript
const REPO             = 'freak-m/moraescacambas';   // GitHub repo
const API              = 'https://api.github.com';
const CONTENT_PATH     = 'content.json';
const CF_WORKER_KEY    = 'cf_worker_url';             // localStorage key
const CF_WORKER_DEFAULT = 'https://analytics.moraescacambas.com.br/';
const CACHE_MAX_MS     = 2 * 60 * 60 * 1000;         // 2h cache hard expiry
const CACHE_CHECK_MS   = 5 * 60 * 1000;              // 5min background recheck
```

---

## File Structure

```
/
├── index.html          ← Public website (reads content.json at runtime)
├── content.json        ← All editable content (committed by admin panel)
├── admin/
│   └── index.html      ← Admin panel (this system)
├── favicon.webp
├── moraescacambaslogo.webp
├── share.webp          ← OG image
└── SETUP.md            ← Guide for replicating this for new clients
```
