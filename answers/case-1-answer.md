# Case 1: Cross-Domain Conversion Tracking

## Overview

**The journey:** `usehardal.com` → `apps.shopify.com/hardal` → Shopify app installed

**The core challenge:** The Shopify App Store is a third-party domain we don't control. This makes standard cross-domain tracking (GA4 `_gl` parameter, cookies) impossible for the App Store segment. Additionally, Shopify does not provide an `app/installed` webhook — the install signal comes from the OAuth callback (`afterAuth` hook).

**The solution:** Capture and store full attribution server-side *before* the user leaves usehardal.com. Use a first-party cookie on `.usehardal.com` as the attribution bridge, retrieved when the merchant completes installation and returns via OAuth callback to `app.usehardal.com`.

---

## Technical Architecture

### The Three Zones

```
┌─────────────────────┐     ┌────────────────────────────┐     ┌────────────────────────┐
│   usehardal.com     │     │  apps.shopify.com/hardal   │     │  app.usehardal.com     │
│   (Zone 1 — Ours)   │────▶│  (Zone 2 — Blind Zone)     │────▶│  (Zone 3 — Ours)       │
│                     │     │                            │     │                        │
│  • Capture click    │     │  • No control              │     │  • OAuth callback      │
│  • Store click_id   │     │  • UTM params lost         │     │  • Read cookie         │
│  • Set cookie       │     │  • No webhook on install   │     │  • Link install ↔ click│
│  • Redirect to AS   │     │  • Merchant clicks Add App │     │  • Fire conversion     │
└─────────────────────┘     └────────────────────────────┘     └────────────────────────┘
```

### Data Flow Diagram

```
User                  usehardal.com              Redis/DB           apps.shopify.com
 │                          │                       │                      │
 │── visits landing page ──▶│                       │                      │
 │   (with UTM params)      │                       │                      │
 │                          │                       │                      │
 │── clicks Shopify CTA ───▶│                       │                      │
 │                          │── JS captures UTMs    │                      │
 │                          │── generates click_id  │                      │
 │                          │── POST /api/track ───▶│                      │
 │                          │                       │── stores:            │
 │                          │                       │   click_id → attrs   │
 │                          │                       │   TTL: 30 days       │
 │                          │◀── 200 OK ────────────│                      │
 │                          │── sets cookie:        │                      │
 │                          │   hrd_click_id        │                      │
 │◀── redirect ─────────────│                       │                      │
 │                                                  │                      │
 │── navigates to App Store ──────────────────────────────────────────────▶│
 │                                                  │                      │
 │── clicks "Add app" ─────────────────────────────────────────────────────│
 │                                                  │    Shopify OAuth     │
 │                          │◀── OAuth callback ────────────────────────── │
 │                          │   (shop=x.myshopify.com)                     │
 │                          │── reads hrd_click_id cookie                  │
 │                          │── lookup attribution ─▶│                     │
 │                          │◀── {utm_source, ...} ──│                     │
 │                          │── stores: shop ↔ click_id                    │
 │                          │── fires server-side conversion event         │
 │                          │── completes OAuth, installs app              │
```

---

## Implementation

### Step 1: Frontend Click Capture (`usehardal.com`)

Add this script to the landing page. It intercepts the Shopify CTA click, reads UTM parameters from the current URL, and routes through a tracking endpoint before redirecting to the App Store.

```javascript
(function () {
  'use strict';

  /**
   * Generates a UUID v4 (random).
   * Used as a unique identifier for each click event.
   */
  function generateUUID() {
    return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, function (c) {
      var r = (Math.random() * 16) | 0;
      var v = c === 'x' ? r : (r & 0x3) | 0x8;
      return v.toString(16);
    });
  }

  /**
   * Reads UTM parameters and referrer from the current page context.
   */
  function getAttributionData() {
    var params = new URLSearchParams(window.location.search);
    return {
      utm_source:   params.get('utm_source')   || '',
      utm_medium:   params.get('utm_medium')   || '',
      utm_campaign: params.get('utm_campaign') || '',
      utm_content:  params.get('utm_content')  || '',
      utm_term:     params.get('utm_term')     || '',
      referrer:     document.referrer          || '',
      page_url:     window.location.href,
      timestamp:    new Date().toISOString(),
    };
  }

  /**
   * Intercepts click on the Shopify App CTA.
   *
   * The selector targets any <a> element pointing to the /shopify page
   * or directly to the Shopify App Store listing. Adjust as needed.
   */
  function attachClickHandler() {
    var selectors = [
      'a[href="/shopify"]',
      'a[href*="apps.shopify.com/hardal"]',
    ];

    selectors.forEach(function (selector) {
      document.querySelectorAll(selector).forEach(function (el) {
        el.addEventListener('click', function (event) {
          event.preventDefault();

          var clickId = generateUUID();
          var attribution = getAttributionData();

          // Send attribution data to our tracking endpoint.
          // The endpoint stores the data and issues the redirect to the App Store.
          var trackingUrl = '/api/track/shopify-redirect';
          var body = JSON.stringify({ click_id: clickId, attribution: attribution });

          // Using fetch with keepalive so the request completes even during navigation.
          fetch(trackingUrl, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: body,
            keepalive: true,
          })
            .then(function (res) {
              if (res.redirected) {
                window.location.href = res.url;
              } else {
                // Fallback: navigate directly to App Store if our endpoint is unreachable.
                window.location.href = 'https://apps.shopify.com/hardal';
              }
            })
            .catch(function () {
              // Graceful degradation: tracking failed, but the user journey continues.
              window.location.href = 'https://apps.shopify.com/hardal';
            });
        });
      });
    });
  }

  if (document.readyState === 'loading') {
    document.addEventListener('DOMContentLoaded', attachClickHandler);
  } else {
    attachClickHandler();
  }
})();
```

---

### Step 2: Server-Side Tracking Endpoint

This is a Next.js Route Handler (or equivalent Node.js endpoint). It receives the click attribution data, stores it in Redis, sets a first-party cookie, and redirects to the Shopify App Store.

`**app/api/track/shopify-redirect/route.ts**`

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { Redis } from '@upstash/redis'; // or any Redis client

const redis = Redis.fromEnv();

const SHOPIFY_APP_STORE_URL = 'https://apps.shopify.com/hardal';
const COOKIE_NAME = 'hrd_click_id';
const COOKIE_TTL_SECONDS = 30 * 24 * 60 * 60; // 30 days
const REDIS_TTL_SECONDS = 30 * 24 * 60 * 60;  // 30 days

export async function POST(req: NextRequest) {
  let body: { click_id: string; attribution: Record<string, string> };

  try {
    body = await req.json();
  } catch {
    // Malformed request — redirect to App Store anyway, tracking skipped.
    return NextResponse.redirect(SHOPIFY_APP_STORE_URL, { status: 302 });
  }

  const { click_id, attribution } = body;

  if (click_id && attribution) {
    const redisKey = `hrd:click:${click_id}`;
    const payload = {
      ...attribution,
      created_at: new Date().toISOString(),
    };

    // Store attribution data server-side. Even if the cookie is blocked,
    // we have a record we can attempt to match probabilistically later.
    try {
      await redis.set(redisKey, JSON.stringify(payload), { ex: REDIS_TTL_SECONDS });
    } catch {
      // Non-fatal — proceed with redirect even if Redis write fails.
      console.error('[tracking] Redis write failed for click_id:', click_id);
    }
  }

  // Redirect to the App Store with the cookie set.
  const response = NextResponse.redirect(SHOPIFY_APP_STORE_URL, { status: 302 });

  if (click_id) {
    response.cookies.set(COOKIE_NAME, click_id, {
      domain: '.usehardal.com',       // Shared across marketing site and app subdomain
      path: '/',
      httpOnly: true,
      secure: true,
      sameSite: 'lax',               // 'lax' allows the cookie to be sent on cross-site navigations
      maxAge: COOKIE_TTL_SECONDS,
    });
  }

  return response;
}
```

**Redis key structure:**

```
Key:   hrd:click:{uuid}
Value: {
  "utm_source": "google",
  "utm_medium": "cpc",
  "utm_campaign": "shopify-launch",
  "utm_content": "hero-cta",
  "utm_term": "",
  "referrer": "https://www.google.com/",
  "page_url": "https://usehardal.com/?utm_source=google&utm_medium=cpc",
  "timestamp": "2026-03-19T10:24:00.000Z",
  "created_at": "2026-03-19T10:24:01.234Z"
}
TTL:   30 days
```

---

### Step 3: OAuth Callback — Attribution Linking

When a merchant installs the Hardal Shopify app, Shopify completes the OAuth flow by redirecting to `app.usehardal.com/auth/shopify/callback`. This is where attribution is linked to the install.

`**app/api/auth/shopify/callback/route.ts**` (simplified — assumes Shopify OAuth is otherwise handled)

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { Redis } from '@upstash/redis';

const redis = Redis.fromEnv();

const COOKIE_NAME = 'hrd_click_id';

interface ShopifyOAuthCallbackParams {
  shop: string;       // e.g. "merchant-name.myshopify.com"
  code: string;       // Authorization code, exchanged for access token
  state: string;      // CSRF nonce — must be validated before proceeding
  hmac: string;       // Shopify HMAC signature for request validation
}

export async function GET(req: NextRequest) {
  const { searchParams } = new URL(req.url);
  const shop  = searchParams.get('shop')  ?? '';
  const code  = searchParams.get('code')  ?? '';
  const state = searchParams.get('state') ?? '';
  const hmac  = searchParams.get('hmac')  ?? '';

  // 1. Validate HMAC and state (CSRF check) — required Shopify security step.
  //    (Implementation omitted for brevity; use a Shopify library like @shopify/shopify-api)
  const isValid = await validateShopifyOAuth({ shop, code, state, hmac });
  if (!isValid) {
    return NextResponse.json({ error: 'Invalid OAuth callback' }, { status: 400 });
  }

  // 2. Exchange code for access token — standard Shopify OAuth step.
  const accessToken = await exchangeCodeForToken(shop, code);

  // 3. Read the attribution cookie set when the merchant clicked the CTA.
  const clickId = req.cookies.get(COOKIE_NAME)?.value;

  if (clickId) {
    const redisKey = `hrd:click:${clickId}`;

    try {
      const attributionRaw = await redis.get<string>(redisKey);

      if (attributionRaw) {
        const attribution = JSON.parse(attributionRaw);

        // 4. Store the install-to-click association.
        const installKey = `hrd:install:${shop}`;
        await redis.set(installKey, JSON.stringify({
          click_id:     clickId,
          shop:         shop,
          utm_source:   attribution.utm_source,
          utm_medium:   attribution.utm_medium,
          utm_campaign: attribution.utm_campaign,
          utm_content:  attribution.utm_content,
          utm_term:     attribution.utm_term,
          referrer:     attribution.referrer,
          page_url:     attribution.page_url,
          clicked_at:   attribution.timestamp,
          installed_at: new Date().toISOString(),
        }));

        // 5. Fire a server-side conversion event.
        //    This sends the install event with full attribution to your analytics stack.
        await fireConversionEvent({
          event:        'app_installed',
          shop:         shop,
          click_id:     clickId,
          utm_source:   attribution.utm_source,
          utm_medium:   attribution.utm_medium,
          utm_campaign: attribution.utm_campaign,
          referrer:     attribution.referrer,
          value:        249,          // Minimum plan value ($249/mo)
          currency:     'USD',
        });
      }
    } catch (err) {
      // Attribution linking is non-blocking — log and continue with install.
      console.error('[attribution] Failed to link install to click:', err);
    }
  } else {
    // No cookie found — record as organic/direct install.
    await recordDirectInstall(shop);
  }

  // 6. Complete the OAuth flow (save token, redirect to app dashboard).
  await saveAccessToken(shop, accessToken);
  return NextResponse.redirect(`https://app.usehardal.com/dashboard?shop=${shop}`);
}

/**
 * Fires a server-side conversion event to Hardal's own analytics pipeline
 * and optionally to GA4 Measurement Protocol or Meta Conversions API.
 */
async function fireConversionEvent(event: Record<string, unknown>) {
  // Option A: Hardal's own signal endpoint
  await fetch('https://cm5v9x9f80003tivvx7nr2bjh-signal.usehardal.com/event', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      name:       event.event,
      properties: event,
      website_id: 'cm5v9x9f80003tivvx7nr2bjh',
    }),
  });

  // Option B: GA4 Measurement Protocol (server-side)
  // await fetch(`https://www.google-analytics.com/mp/collect?measurement_id=${GA4_ID}&api_secret=${GA4_SECRET}`, {
  //   method: 'POST',
  //   body: JSON.stringify({
  //     client_id: event.click_id,
  //     events: [{ name: 'app_installed', params: { utm_source: event.utm_source, value: event.value } }]
  //   })
  // });

  // Option C: Meta Conversions API
  // await sendMetaCAPIEvent({ event_name: 'Purchase', custom_data: { value: event.value, currency: 'USD' } });
}

// Stubs — implement with @shopify/shopify-api
declare function validateShopifyOAuth(params: ShopifyOAuthCallbackParams): Promise<boolean>;
declare function exchangeCodeForToken(shop: string, code: string): Promise<string>;
declare function saveAccessToken(shop: string, token: string): Promise<void>;
declare function recordDirectInstall(shop: string): Promise<void>;
```

---

## Edge Cases


| Scenario                                                                  | What Happens                                                     | Mitigation                                                                                                                                                                                                             |
| ------------------------------------------------------------------------- | ---------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Cookie blocked (Safari ITP, Firefox, ad blockers)**                     | `hrd_click_id` cookie is not set or is cleared before install    | (1) Server-side Redis entry still exists. (2) Attempt probabilistic match on OAuth callback by IP + User-Agent + time window. (3) Accept partial attribution loss — cookie blocking is an inherent browser constraint. |
| **User clicks CTA but installs days later**                               | Cookie may expire if browser clears it, but Redis entry survives | Cookie `maxAge` set to 30 days; Redis TTL is also 30 days. Most installs happen within 7 days of click. Adjust TTL based on observed sales cycle.                                                                      |
| **User installs from a different device or browser**                      | Cookie is device-specific — no match at OAuth callback           | Post-install onboarding email with a "How did you hear about us?" question provides a manual fallback for high-value installs.                                                                                         |
| **Direct install (user navigates to App Store without clicking our CTA)** | No cookie, no Redis entry                                        | Recorded as `utm_source: "direct"` or `"(none)"` — same as organic dark social. These installs show in Shopify Partner Dashboard as direct.                                                                            |
| **Merchant reinstalls after uninstalling**                                | OAuth callback fires again with same shop domain                 | Check if `hrd:install:{shop}` already exists in Redis. If yes, preserve the original attribution and log the reinstall timestamp separately.                                                                           |
| **Multiple UTM sources (multi-touch)**                                    | Only the last click before install is tracked by default         | Store all click events in a list (`hrd:clicks:{session_id}`) for full multi-touch visibility. Default reporting uses last-touch model.                                                                                 |
| **Network error on tracking POST**                                        | `fetch` fails, user never receives cookie                        | JavaScript falls back to direct redirect to App Store — user journey is uninterrupted but attribution is lost for that click. Acceptable tradeoff for UX.                                                              |
| **Shopify App Store strips query parameters**                             | Any UTM params added directly to the App Store URL are lost      | This is expected and why we capture attribution BEFORE the redirect, not after. Our approach is immune to this.                                                                                                        |


---

## What This Does NOT Cover (Known Limitations)

1. **No `app/installed` webhook from Shopify** — Shopify does not send a webhook on installation. Attribution linking happens at the OAuth callback. This is confirmed in Shopify's API documentation.
2. **We cannot track behavior inside the Shopify App Store** — Time spent on the listing, screenshot views, and review reads are inside Shopify's domain. The `surface_type` and `surface_detail` parameters Shopify provides describe in-store navigation only (e.g., came from search vs. home), not external campaigns.
3. **GA4 cross-domain linking cannot bridge to apps.shopify.com** — GA4's `_gl` parameter only works between domains where both sides run the GA4 SDK. We don't control the App Store domain.

---

## Summary


| Phase          | Location                        | Data Captured                             | Method                                 |
| -------------- | ------------------------------- | ----------------------------------------- | -------------------------------------- |
| **Click**      | `usehardal.com`                 | UTM params, referrer, page URL, timestamp | First-party JS → server-side POST      |
| **Store**      | Redis (server)                  | `click_id → full attribution`, 30-day TTL | Before redirect, on our infrastructure |
| **Cookie**     | `.usehardal.com`                | `click_id` only (the rest is server-side) | `httpOnly`, `secure`, `sameSite=lax`   |
| **App Store**  | `apps.shopify.com`              | Nothing — blind zone                      | N/A                                    |
| **Install**    | `app.usehardal.com`             | `shop` domain + `click_id` from cookie    | OAuth callback → Redis lookup          |
| **Conversion** | Hardal signal / GA4 / Meta CAPI | Full attribution linked to shop install   | Server-side event fire                 |


