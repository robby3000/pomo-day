# Pomo/Day — PWA Implementation Plan

## Overview

Convert the deployed single-file app at `robby3000.github.io/pomo-day/` into a fully installable Progressive Web App. The app must be add-to-home-screen eligible on iOS and Android, work 100% offline after first load, and show a custom splash screen and app icon.

The single-file constraint makes this slightly unusual: the service worker and manifest cannot be inlined into `index.html` — they **must** be separate files served at the repo root. This is the one intentional exception to the no-external-files rule.

---

## Constraints & Context

- Single-page app: `index.html` is the entire app.
- Hosted via GitHub Pages at `robby3000.github.io/pomo-day/` — repo root maps to that URL.
- Fonts loaded from Google Fonts CDN (`fonts.googleapis.com` / `fonts.gstatic.com`).
- All app state in localStorage; no server-side calls.
- No build step, no bundler, no npm.
- AGENTS.md allows new files only for PWA artefacts (manifest, service worker, icons).

---

## Files to Create

| File | Purpose |
|---|---|
| `manifest.json` | Web App Manifest — install metadata |
| `sw.js` | Service Worker — offline caching |
| `icons/icon-192.png` | App icon 192×192 (required by manifest) |
| `icons/icon-512.png` | App icon 512×512 (required by manifest) |
| `icons/icon-maskable-192.png` | Maskable icon 192×192 (for Android adaptive icons) |
| `icons/icon-maskable-512.png` | Maskable icon 512×512 |

## Files to Edit

| File | Change |
|---|---|
| `index.html` | Add `<link rel="manifest">`, theme/apple meta tags, service worker registration script |

---

## Task Breakdown

### Task 1 — Create icons

Generate four PNG icons at the sizes above. Design: dark background (`#0d0d0d`), yellow accent (`#e8ff47`), minimal "PD" or tomato/timer glyph centred. The maskable variants must keep all content inside the safe zone (the inner 80% circle).

Tools: any raster editor or script (e.g. `canvas`-based Node script, Figma export, or `sharp` CLI). Commit the PNGs at `icons/`.

**Acceptance**: all four PNGs exist, are valid images, correct sizes.

---

### Task 2 — Create `manifest.json`

Create `manifest.json` at the repo root with the following fields:

```json
{
  "name": "Pomo / Day Planner",
  "short_name": "Pomo/Day",
  "description": "Pomodoro timer with daily task planning.",
  "start_url": "/pomo-day/",
  "scope": "/pomo-day/",
  "display": "standalone",
  "orientation": "portrait",
  "background_color": "#0d0d0d",
  "theme_color": "#e8ff47",
  "icons": [
    {
      "src": "/pomo-day/icons/icon-192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "/pomo-day/icons/icon-512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "/pomo-day/icons/icon-maskable-192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "maskable"
    },
    {
      "src": "/pomo-day/icons/icon-maskable-512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "maskable"
    }
  ]
}
```

**Note on `start_url` / `scope`**: GitHub Pages serves the repo at `/pomo-day/`. The `start_url` must match this path exactly or the browser will refuse to install. Confirm the live URL before finalising.

**Acceptance**: `manifest.json` validates at [web.dev/measure](https://web.dev/measure) or Chrome DevTools → Application → Manifest with no errors.

---

### Task 3 — Create `sw.js` (service worker)

The service worker must:

1. **Pre-cache on install** — cache `index.html` and the Google Fonts stylesheets/font files so the app works fully offline after first visit.
2. **Serve from cache on fetch** — use a **cache-first** strategy for all cached assets; fall back to network.
3. **Update on new deploy** — use a versioned cache name (`pomo-day-v1`). On `activate`, delete old cache versions.
4. **Handle Google Fonts** — cache both the CSS API response and the actual `.woff2` files. Use a separate `pomo-day-fonts-v1` cache with a stale-while-revalidate strategy so fonts update eventually but never block load.

```js
// sw.js — structure outline
const CACHE = 'pomo-day-v1';
const FONT_CACHE = 'pomo-day-fonts-v1';

const PRECACHE_URLS = [
  '/pomo-day/',
  '/pomo-day/index.html',
  '/pomo-day/manifest.json',
  '/pomo-day/icons/icon-192.png',
  '/pomo-day/icons/icon-512.png',
];

self.addEventListener('install', event => {
  event.waitUntil(
    caches.open(CACHE).then(cache => cache.addAll(PRECACHE_URLS))
  );
  self.skipWaiting();
});

self.addEventListener('activate', event => {
  event.waitUntil(
    caches.keys().then(keys =>
      Promise.all(
        keys
          .filter(k => k !== CACHE && k !== FONT_CACHE)
          .map(k => caches.delete(k))
      )
    )
  );
  self.clients.claim();
});

self.addEventListener('fetch', event => {
  const url = new URL(event.request.url);

  // Google Fonts — stale-while-revalidate
  if (url.hostname === 'fonts.googleapis.com' || url.hostname === 'fonts.gstatic.com') {
    event.respondWith(staleWhileRevalidate(event.request, FONT_CACHE));
    return;
  }

  // Everything else — cache-first
  event.respondWith(cacheFirst(event.request, CACHE));
});

async function cacheFirst(request, cacheName) {
  const cached = await caches.match(request);
  if (cached) return cached;
  const response = await fetch(request);
  const cache = await caches.open(cacheName);
  cache.put(request, response.clone());
  return response;
}

async function staleWhileRevalidate(request, cacheName) {
  const cache = await caches.open(cacheName);
  const cached = await cache.match(request);
  const networkFetch = fetch(request).then(response => {
    cache.put(request, response.clone());
    return response;
  });
  return cached || networkFetch;
}
```

**Version bumping**: when `index.html` is updated, increment `CACHE` to `pomo-day-v2`, etc., so users get the new version on next visit.

**Acceptance**: Chrome DevTools → Application → Service Workers shows the SW registered and active; Network tab shows cached responses on reload with DevTools offline mode enabled.

---

### Task 4 — Edit `index.html`: add manifest link + meta tags

Inside `<head>`, after the existing `<meta viewport>` line, add:

```html
<!-- PWA manifest -->
<link rel="manifest" href="/pomo-day/manifest.json">

<!-- Theme colour (browser chrome on Android) -->
<meta name="theme-color" content="#e8ff47">

<!-- iOS home screen support -->
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
<meta name="apple-mobile-web-app-title" content="Pomo/Day">
<link rel="apple-touch-icon" href="/pomo-day/icons/icon-192.png">
```

**Why these tags**: iOS Safari does not use `manifest.json` for install metadata — it reads its own proprietary meta tags. Without them the app title defaults to the page URL and the icon defaults to a screenshot.

**Edit precision**: insert the block as a group immediately after `<meta name="viewport" ...>` (currently line 5). Do not touch any other line.

**Acceptance**: on iOS Safari, "Add to Home Screen" shows correct title "Pomo/Day" and the icon. Status bar in standalone mode renders translucent over the dark background.

---

### Task 5 — Edit `index.html`: register service worker

At the very bottom of `<script>`, after the final init line (`Notification.requestPermission()`), add:

```js
// ── SERVICE WORKER ──
if ('serviceWorker' in navigator) {
  window.addEventListener('load', () => {
    navigator.serviceWorker.register('/pomo-day/sw.js').catch(err => {
      console.warn('SW registration failed:', err);
    });
  });
}
```

**Why `window.addEventListener('load', ...)`**: defers registration until after the page is interactive, so the SW install/pre-cache fetch doesn't compete with initial render.

**Edit precision**: insert only these lines, immediately after the `Notification.requestPermission()` block, before the closing `</script>` tag.

**Acceptance**: DevTools → Application → Service Workers shows `sw.js` as registered. No console errors.

---

## Verification Checklist

Run through these after all tasks are complete:

- [ ] Chrome DevTools → Application → Manifest: no errors, all icons resolved, `start_url` matches live URL.
- [ ] Chrome DevTools → Application → Service Workers: status "activated and is running".
- [ ] Lighthouse PWA audit (DevTools → Lighthouse → Progressive Web App): score passes installability checks.
- [ ] DevTools Network → toggle "Offline" → reload: app loads fully from cache.
- [ ] On Android Chrome: "Add to Home Screen" prompt appears or is manually triggered; icon and splash screen are correct.
- [ ] On iOS Safari: "Add to Home Screen" works; icon, title, and standalone mode are correct.
- [ ] Fonts render correctly both online and offline (cached).
- [ ] After deploying a cache-version bump, old cache is deleted on activate and new content is served.

---

## PWA Installability Requirements (reference)

For Chrome to show the install prompt automatically, all must be true:

1. Served over HTTPS ✓ (GitHub Pages)
2. Has a valid `manifest.json` with `name`, `short_name`, `start_url`, `display: standalone`, and at least one 192×192 icon
3. Has a registered service worker with a `fetch` handler
4. Has not been dismissed by the user in the last 90 days

---

## Deployment Order

1. Commit icons (`icons/` folder)
2. Commit `manifest.json`
3. Commit `sw.js`
4. Commit updated `index.html` (meta tags + SW registration)
5. Push to `main` → GitHub Pages redeploys
6. Open `robby3000.github.io/pomo-day/` in Chrome, run Lighthouse PWA audit

Steps 1–4 can be batched into a single commit if preferred.

---

## Cache Version Management (ongoing)

After any future update to `index.html`:

1. In `sw.js`, increment `const CACHE = 'pomo-day-v1'` → `'pomo-day-v2'` (etc.)
2. Commit both files together.

This ensures returning users pick up the new HTML on next visit rather than being stuck on the cached version indefinitely.
