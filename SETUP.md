# Admin Panel — Setup Guide

This document explains how to replicate this entire system for a new client.
Read it fully before starting. At the end there is a checklist of questions to ask the client.

---

## System Architecture

```
Client Browser
    │
    ├── GitHub Pages  →  index.html (main website, public)
    │                →  admin/index.html (admin panel, password-protected)
    │                →  content.json (all site content, editable via admin)
    │
    └── Cloudflare Worker  →  reads Cloudflare Analytics (CF GraphQL API)
                           →  reads/writes KV (WhatsApp clicks, photo views, time events)
                           →  returns JSON to admin dashboard
```

**Key constraint:** The admin panel is a single static HTML file. There is no backend server.
All secrets (GitHub PAT, Cloudflare tokens) are handled via sessionStorage or Worker env vars — never in source code.

---

## Components

### 1. GitHub Repository
- Hosts the static site via **GitHub Pages** (branch: `main`, root `/`)
- Files: `index.html`, `content.json`, `admin/index.html`, images (`.webp`)
- The admin panel writes to `content.json` via the GitHub API (PAT required at login)

### 2. Admin Panel (`admin/index.html`)
- Password = **GitHub Personal Access Token (PAT)**
- Token stored in `sessionStorage` only (never localStorage, never source code)
- Repo hardcoded in the file: `const REPO = 'owner/repo-name';`
- Tabs: **Conteúdo** (content editor), **Dashboard** (CF analytics), **Integrações** (integrations)
- On publish: reads all form fields → builds `content.json` → commits via GitHub API

### 3. Cloudflare Worker
- URL stored in `localStorage` key `cf_worker_url` and in `content.json > integrations.worker_url`
- Receives `?days=N` (default 7), optional `?from=YYYY-MM-DD&to=YYYY-MM-DD`
- Queries Cloudflare GraphQL Analytics API
- Reads event counts from KV namespace (WhatsApp clicks, photo opens, time-on-site)
- Returns JSON: `{ data: { viewer: { zones: [...] } }, events: {...} }`

### 4. KV Namespace (`CACAMBAS_KV`)
- Stores event counters written by the main site's JS
- Keys: `events` (JSON object with `wa`, `photo`, `time` sub-keys)

### 5. `content.json`
All site content is stored here. The admin panel reads and writes this file.
Structure: `seo`, `hero`, `services`, `how`, `benefits`, `stats`, `contact`, `footer`, `integrations`

---

## Step-by-Step Setup for a New Client

### Step 1 — GitHub

1. **Create a new repository** (public, for GitHub Pages)
   - Copy all files from this repo: `index.html`, `admin/index.html`, `content.json`, `favicon.webp`, logo images
2. **Enable GitHub Pages**: Settings → Pages → Deploy from branch `main` → `/ (root)`
3. **Create a PAT** for the client (or they create one):
   - GitHub → Settings → Developer Settings → Personal Access Tokens → Tokens (classic)
   - Permissions needed: **`repo`** (full control of private repositories) or just `public_repo` for public repos
   - Name it clearly, e.g. `admin-panel-clientname`
4. **Edit `admin/index.html`**: change the `REPO` constant on line ~1491:
   ```javascript
   const REPO = 'github-username/repo-name';
   ```
5. **Edit branding**: update logo `src`, site name in `<title>`, sidebar name, and `CF_WORKER_DEFAULT` URL

### Step 2 — Cloudflare Worker

1. **Log in** to [dash.cloudflare.com](https://dash.cloudflare.com) → **Workers & Pages** → **Create**
2. **Create Worker**: give it a name (e.g. `analytics-clientname`), deploy the code below
3. **Create KV Namespace**: Workers & Pages → **KV** → Create namespace (e.g. `CLIENT_KV`)
4. **Bind KV to Worker**: Worker → Settings → Bindings → Add binding
   - Variable name: `CACAMBAS_KV`
   - Namespace: the one you just created
5. **Add environment variables**: Worker → Settings → Variables
   - `CF_API_TOKEN` — Cloudflare API Token (see below)
   - `CF_ZONE_ID` — Zone ID of the client's domain (see below)
6. **Custom domain (optional)**: Worker → Settings → Triggers → Custom Domains
   - e.g. `analytics.clientdomain.com.br`
   - Alternatively use the default `*.workers.dev` URL

#### How to get the Zone ID
Cloudflare Dashboard → select the domain → Overview page → right sidebar → **Zone ID** (copy it)

#### How to create the CF API Token
Cloudflare → My Profile → API Tokens → Create Token → **Custom Token**:
- Permission: **Zone → Analytics → Read**
- Zone Resources: **Include → Specific zone → [client domain]**

### Step 3 — Worker Code

Deploy this code to the Cloudflare Worker:

```javascript
export default {
  async fetch(request, env) {
    const headers = {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': '*',
      'Access-Control-Allow-Methods': 'GET, OPTIONS',
      'Access-Control-Allow-Headers': 'Content-Type',
    };

    if (request.method === 'OPTIONS') return new Response(null, { headers });

    const url   = new URL(request.url);
    const days  = parseInt(url.searchParams.get('days') || '7');
    const from  = url.searchParams.get('from');
    const to    = url.searchParams.get('to');

    const now   = new Date();
    const fmt   = d => d.toISOString().split('T')[0];
    const end   = to   ? new Date(to)   : now;
    const start = from ? new Date(from) : new Date(now.getTime() - (days - 1) * 86400000);
    const prevEnd   = new Date(end.getTime()   - days * 86400000);
    const prevStart = new Date(start.getTime() - days * 86400000);

    // IMPORTANT: httpRequestsAdaptiveGroups and webAnalyticsRumPageloadEventsAdaptiveGroups
    // only support a 1-day range. Both date_geq AND date_leq must be fmt(now).
    const query = `{
      viewer {
        zones(filter: { zoneTag: "${env.CF_ZONE_ID}" }) {
          current: httpRequests1dGroups(
            limit: ${days + 5}
            orderBy: [date_ASC]
            filter: { date_geq: "${fmt(start)}", date_leq: "${fmt(end)}" }
          ) {
            date
            sum { requests cachedRequests bytes }
            uniq { uniques }
          }
          prev: httpRequests1dGroups(
            limit: ${days + 5}
            orderBy: [date_ASC]
            filter: { date_geq: "${fmt(prevStart)}", date_leq: "${fmt(prevEnd)}" }
          ) {
            date
            sum { requests }
            uniq { uniques }
          }
          devices: httpRequestsAdaptiveGroups(
            limit: 10
            orderBy: [count_DESC]
            filter: { date_geq: "${fmt(now)}", date_leq: "${fmt(now)}" }
          ) {
            count
            dimensions { clientDeviceType }
          }
          os: webAnalyticsRumPageloadEventsAdaptiveGroups(
            limit: 10
            orderBy: [count_DESC]
            filter: { date_geq: "${fmt(now)}", date_leq: "${fmt(now)}" }
          ) {
            count
            dimensions { operatingSystem }
          }
        }
      }
    }`;

    let cfData = {};
    try {
      const cfRes = await fetch('https://api.cloudflare.com/client/v4/graphql', {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${env.CF_API_TOKEN}`,
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({ query }),
      });
      cfData = await cfRes.json();
    } catch (e) {
      return new Response(JSON.stringify({ error: e.message }), { status: 500, headers });
    }

    // Read events from KV
    let events = null;
    try {
      events = await env.CACAMBAS_KV.get('events', { type: 'json' });
    } catch (_) {}

    return new Response(JSON.stringify({ ...cfData, events }), { headers });
  },
};
```

### Step 4 — Admin Panel First Login

1. Go to `https://clientdomain.com/admin/`
2. Enter the GitHub PAT as password
3. Go to **Integrações** tab
4. Fill in **Analytics URL** with the Worker URL (e.g. `https://analytics.clientdomain.com.br/`)
5. Click **Salvar configurações** → this writes the Worker URL to `content.json`
6. Go to **Dashboard** tab to verify analytics are loading

### Step 5 — Cloudflare Security Settings (recommended for all clients)

In Cloudflare Dashboard → **Security** → **Bots**:
- **Bot Fight Mode** → **On**
- **Block AI Bots** → **On** → Scope: **Block on all pages**

---

## `content.json` Structure

```json
{
  "seo": {
    "title": "...",
    "description": "...",
    "keywords": "...",
    "og_image": "https://domain.com/share.webp",
    "schema_local": {
      "name": "Business Name",
      "phone": "+55 XX XXXXX-XXXX",
      "address": "...",
      "area": "...",
      "opens": "07:00",
      "closes": "18:00"
    },
    "faq": []
  },
  "hero": { "subtitle": "<p>...</p>" },
  "services": {
    "tag": "...", "title": "...", "title_highlight": "...",
    "subtitle": "...", "size": "...", "name": "...",
    "description": "<p>...</p>",
    "features": ["...", "..."]
  },
  "how": {
    "subtitle": "...",
    "steps": [
      { "title": "...", "description": "<p>...</p>" }
    ]
  },
  "benefits": {
    "subtitle": "...",
    "items": [
      { "title": "...", "description": "<p>...</p>" }
    ]
  },
  "stats": [
    { "value": 500, "suffix": "+", "label": "Clientes Atendidos" }
  ],
  "contact": {
    "tag": "...", "title": "...", "title_highlight": "...",
    "subtitle": "<p>...</p>",
    "cards": [
      { "type": "whatsapp", "display": "(XX) XXXXX-XXXX", "link": "55XXXXXXXXXXX" },
      { "type": "area", "text": "..." },
      { "type": "hours", "text": "...", "note": "..." }
    ],
    "wa_panel_label": "<p>...</p>",
    "wa_panel_sub": "<p>...</p>"
  },
  "footer": {
    "location": "...",
    "hours": "..."
  },
  "integrations": {
    "google_analytics": "G-XXXXXXXXXX",
    "meta_pixel": "",
    "meta_verify": "",
    "worker_url": "https://analytics.clientdomain.com.br/"
  }
}
```

---

## Dashboard Cache Behavior

- Cache stored in `localStorage` with keys `cf_cache_7d`, `cf_cache_30d`, `cf_cache_90d`
- **Hard expiry**: 2 hours — after that, always re-fetches
- **Background recheck**: after 5 minutes, re-fetches silently in background
- **Cache-first display**: always shows cached data immediately, then shows yellow "Atualizando dados…" banner during background refresh
- **Error handling**: if refresh fails, shows red banner "Limite da API atingido" and keeps showing last cached data

---

## Security Rules

| What | Where | Value |
|---|---|---|
| GitHub PAT | `sessionStorage['gh_token']` | Cleared on logout / page close |
| CF API Token | Worker env var `CF_API_TOKEN` | Never in source code |
| CF Zone ID | Worker env var `CF_ZONE_ID` | Never in source code |
| Worker URL | `localStorage['cf_worker_url']` + `content.json` | Acceptable (not a secret) |

---

## Questions to Ask the Client Before Setup

Ask the client (or gather yourself) before starting:

### Business Info
- [ ] Business name
- [ ] Phone number (WhatsApp, with country+area code, e.g. `5519993277969`)
- [ ] Service area / city
- [ ] Business hours (open/close times)
- [ ] Address (if applicable)
- [ ] Years of experience, number of clients served, other stats for the hero counters

### Domain & Hosting
- [ ] Domain name (e.g. `clientdomain.com.br`) — is it already on Cloudflare?
- [ ] GitHub username (will you create the repo, or does the client have an account?)
- [ ] Desired Worker subdomain (e.g. `analytics.clientdomain.com.br`) — or use the default `.workers.dev` URL?

### Branding
- [ ] Logo file (`.webp` preferred, or `.png`/`.svg`)
- [ ] Social share image (OG image, 1200×630px recommended)
- [ ] Brand colors (if they want to customize the yellow `#F5C400` accent)

### Content
- [ ] Hero subtitle / tagline
- [ ] Service name, size, description, feature list
- [ ] How it works — 3 steps
- [ ] Benefits (4 items recommended)
- [ ] FAQ items (optional)
- [ ] Footer text

### Integrations (all optional)
- [ ] Google Analytics Measurement ID (format: `G-XXXXXXXXXX`)
- [ ] Meta Pixel ID (format: numeric)
- [ ] Meta domain verification tag (if needed)

### Cloudflare (you set this up, client doesn't need to know)
- [ ] Cloudflare account access (or client shares Zone ID + creates API Token)
- [ ] CF Zone ID (found in Cloudflare Dashboard → domain → Overview → right sidebar)
- [ ] CF API Token (create via My Profile → API Tokens → Custom Token → Zone Analytics Read)

---

## Files to Change When Cloning for a New Client

| File | What to change |
|---|---|
| `admin/index.html` line ~1491 | `const REPO = 'owner/repo-name';` |
| `admin/index.html` line ~1552 | `const CF_WORKER_DEFAULT = 'https://analytics.newdomain.com.br/';` |
| `admin/index.html` sidebar | Business name in sidebar header |
| `admin/index.html` logo `src` | Path to new logo file |
| `admin/index.html` `<title>` | Page title |
| `content.json` | All content (can be edited via admin panel after setup) |
| `index.html` | SEO meta tags, schema markup, GA script tag |

---

## Common Errors and Fixes

| Error | Cause | Fix |
|---|---|---|
| `zone cannot request time range wider than 1d` | `httpRequestsAdaptiveGroups` or `webAnalyticsRumPageloadEventsAdaptiveGroups` queried with a range > 1 day | Change both `date_geq` and `date_leq` to `fmt(now)` in Worker code |
| `Sem dados. Verifique se CF_ZONE_ID está configurado` | Wrong or missing `CF_ZONE_ID` in Worker env vars | Check Worker → Settings → Variables |
| `401` from GitHub API | PAT expired or wrong repo in `REPO` constant | Regenerate PAT; verify `REPO = 'owner/repo'` |
| Dashboard shows spinner forever | No cache + Worker error | Fix Worker first; cache builds on first successful fetch |
| Admin panel shows blank after deploy | `content.json` missing or malformed | Check GitHub repo for valid `content.json` |
