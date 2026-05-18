---
name: chrome-extension-debug
description: Systematically debug a Chrome Manifest V3 extension. Covers the six most common failure categories: service worker termination, message passing failures, auth/PKCE flow breakage, content script injection failures, access gate / subscription check failures, and API call failures. Includes DevTools console snippets ready to paste. Trigger on "debug my Chrome extension", "extension not working", "service worker terminated", "content script not injecting", "extension auth broken", or /chrome-extension-debug.
---

# Chrome MV3 Extension Debugging Guide

A systematic, category-by-category approach to diagnosing broken Chrome Manifest V3 extensions. Work through **Step 0** first to establish which context you are debugging in, then jump to whichever category matches the symptom.

---

## Step 0 — Open the right DevTools context

MV3 extensions have four separate JavaScript execution contexts. Bugs are often invisible because you are watching the wrong one.

| Context | How to open DevTools | What lives here |
|---|---|---|
| **Service worker** | `chrome://extensions` → find your extension → click **"Service worker"** link | Background logic, message handlers, alarm handlers, `chrome.storage` reads, auth token refresh, API calls made from the background |
| **Content script** | Open the page where the script runs → F12 → **Console** → change the context dropdown from "top" to your extension's content script | DOM manipulation code, `window.postMessage`, `chrome.runtime.sendMessage` calls initiated from the page |
| **Popup** | Right-click the extension toolbar icon → **Inspect** (must be done while popup is open, before it closes) | Popup UI logic, quick storage reads, message sends to service worker |
| **Options page** | Navigate to `chrome-extension://YOUR_EXTENSION_ID/options.html` → F12 | Settings UI, full storage read/write, auth initiation |

> **Find your extension ID:** `chrome://extensions` → enable Developer mode → your ID appears under the extension name. It looks like `abcdefghijklmnopqrstuvwxyz123456`. Replace every occurrence of `YOUR_EXTENSION_ID` in the snippets below with your actual ID.

After opening the correct context, run the **starter diagnostic** (see the bottom of this guide) to dump storage state before proceeding.

---

## Category 1 — Service Worker Terminated (the MV3 30-second problem)

### Symptoms
- Extension works immediately after reloading it at `chrome://extensions`, then stops working after ~30 seconds of inactivity
- `chrome.runtime.sendMessage` from a content script or popup gets `Error: Could not establish connection. Receiving end does not exist`
- `chrome://extensions` shows the service worker as **"Stopped"** or **"Inactive"**
- Console logs disappear mid-operation — the worker died between log statements

### Why this happens
MV3 service workers are **event-driven** and Chrome terminates them after they become idle (roughly 30 seconds with no active events). Unlike MV2 background pages, they cannot stay resident. Any in-memory state — caches, pending promises, open connections — is wiped on termination.

### Diagnostics

**1. Confirm the worker died:**
```
chrome://extensions → your extension → "Service worker" link
```
If it shows "No service worker" or the link is grayed out, the worker terminated. Clicking the link re-registers it.

**2. Check the worker lifetime log:**
Open the service worker DevTools → Network tab → look for a gap in requests that coincides with failure.

**3. Look for state stored only in memory:**
```js
// Paste in Service Worker console
// Any variable declared at module scope that is NOT backed by chrome.storage
// will be undefined after a restart — check your code for patterns like:
// let cachedUser = null; (module-level, refilled lazily)
// If this prints null/undefined when you expected a value, it was wiped
console.log('Module-level cache check — add your own variable names here');
```

### Fixes

| Root cause | Fix |
|---|---|
| In-memory cache reset on wake | Persist the cache to `chrome.storage.local` and rehydrate on every event handler entry |
| Long async chain keeps worker alive past its deadline | Break the chain; use `chrome.alarms` to re-trigger work instead of one giant awaited promise |
| `chrome.runtime.connect` port used for keep-alive | Ports do not prevent termination in MV3. Remove keep-alive hacks; redesign for stateless wake-up |
| Worker wakes but re-initialization is async and slow | Wrap every message handler entry point in an `await ensureInitialized()` guard that reads from storage |

**Minimal keep-alive guard pattern:**
```js
// service-worker.js
let initialized = false;

async function ensureInitialized() {
  if (initialized) return;
  // Re-hydrate from storage on every cold start
  const data = await chrome.storage.local.get(['authToken', 'userProfile' /* your keys */]);
  // ... restore in-memory state from data ...
  initialized = true;
}

chrome.runtime.onMessage.addListener((msg, sender, sendResponse) => {
  // IMPORTANT: return true to keep the message channel open across await
  (async () => {
    await ensureInitialized();
    // ... handle msg ...
    sendResponse({ ok: true });
  })();
  return true;
});
```

---

## Category 2 — Message Passing Failures

### Symptoms
- `Error: Could not establish connection. Receiving end does not exist`
- `Error: The message port closed before a response was received`
- Content script sends a message; service worker never logs receiving it
- Messages work in one tab but not another
- `chrome.runtime.connect` port disconnects immediately

### 2a — "Receiving end does not exist"

This means no listener matched the message at the time it was sent.

**Checklist:**
1. Is the service worker alive? (See Category 1)
2. Does the service worker actually call `chrome.runtime.onMessage.addListener`? Search your background script for this.
3. Did you call `sendMessage` *before* the listener was registered? (Race at startup)
4. Are you sending to a specific tab ID that no longer exists?

**Diagnose with:**
```js
// In Service Worker console — verify the listener is registered
// (Chrome doesn't expose listener count, but you can add a debug log)
chrome.runtime.onMessage.addListener((msg, sender, sendResponse) => {
  console.log('[SW] Received message:', msg, 'from:', sender.tab?.id);
  // your handler ...
});
```

### 2b — "Port closed before response was received"

You returned `undefined` from the listener (synchronous return), but tried to call `sendResponse` asynchronously. Fix: return `true` from the listener to signal async response.

```js
// WRONG — port closes immediately
chrome.runtime.onMessage.addListener((msg, sender, sendResponse) => {
  fetchSomething().then(result => sendResponse(result)); // too late
});

// CORRECT — return true keeps the port open
chrome.runtime.onMessage.addListener((msg, sender, sendResponse) => {
  fetchSomething().then(result => sendResponse(result));
  return true; // <-- this is the fix
});
```

### 2c — `externally_connectable` not configured (web page → extension messages)

If a web page is trying to send messages directly to your extension via `chrome.runtime.sendMessage(YOUR_EXTENSION_ID, ...)`, the extension must declare the origin in `manifest.json`:

```json
"externally_connectable": {
  "matches": ["https://your-app-domain.com/*"]
}
```

Without this, the call is silently dropped. Check:
```js
// In the webpage's DevTools console — if this throws, externally_connectable is the issue
chrome.runtime.sendMessage('YOUR_EXTENSION_ID', { type: 'ping' }, response => {
  if (chrome.runtime.lastError) {
    console.error('externally_connectable error:', chrome.runtime.lastError.message);
  } else {
    console.log('Response:', response);
  }
});
```

### 2d — Sender validation rejecting messages

Your service worker may be validating `sender.origin` or `sender.tab.url` and silently dropping unexpected senders. Log the sender to confirm:

```js
chrome.runtime.onMessage.addListener((msg, sender, sendResponse) => {
  console.log('[SW] sender.origin:', sender.origin);
  console.log('[SW] sender.tab?.url:', sender.tab?.url);
  console.log('[SW] sender.id:', sender.id); // sending extension's ID
  // If your code checks sender.origin and the value is unexpected, that's the bug
});
```

### 2e — Wrong tab ID

Messages sent to a specific tab with `chrome.tabs.sendMessage(tabId, ...)` fail silently if the tab was navigated, closed, or if you're using a stale tab ID from storage.

```js
// Verify the tab exists before messaging it
const tabs = await chrome.tabs.query({ active: true, currentWindow: true });
console.log('Current tab:', tabs[0]?.id, tabs[0]?.url);
```

---

## Category 3 — Auth / PKCE Flow Failures

> This section applies to any extension using OAuth2 PKCE — whether via AWS Cognito, Auth0, Okta, a custom server, or another provider.

### Symptoms
- Auth popup opens but never redirects back
- Redirect back happens but the extension never receives the code
- Storage has no token after what appeared to be a successful login
- Token is present but API calls return 401 — token may be expired or malformed
- Auth works in dev but fails in production (redirect URI mismatch)

### 3a — Inspect what is actually in storage

```js
// Paste in Service Worker console
(async () => {
  const all = await chrome.storage.local.get(null);
  const authKeys = Object.entries(all).filter(([k]) =>
    /token|auth|session|user|sub|access|refresh|code|pkce|verifier|state/i.test(k)
  );
  console.log('All storage keys:', Object.keys(all));
  console.log('Auth-related keys:', Object.fromEntries(authKeys));

  // Decode any JWTs found
  authKeys.forEach(([k, v]) => {
    if (typeof v === 'string' && v.split('.').length === 3) {
      try {
        const payload = JSON.parse(
          atob(v.split('.')[1].replace(/-/g, '+').replace(/_/g, '/'))
        );
        console.log(`Decoded JWT at "${k}":`, payload);
        console.log(`  Expires: ${payload.exp ? new Date(payload.exp * 1000).toISOString() : 'no exp'}`);
        console.log(`  Issued:  ${payload.iat ? new Date(payload.iat * 1000).toISOString() : 'no iat'}`);
        const nowSec = Date.now() / 1000;
        if (payload.exp) {
          console.log(`  Status: ${payload.exp > nowSec ? 'VALID' : '*** EXPIRED ***'}`);
        }
      } catch (e) { /* not a standard JWT */ }
    }
  });
})();
```

### 3b — PKCE verifier / state mismatch

The PKCE `code_verifier` and `state` are generated at the start of the flow and must survive until the callback. In MV3, the service worker may have terminated between the redirect and the callback processing.

```js
// Check whether the PKCE ephemeral state is still present
// (adjust key names to match what your auth library uses)
const pkceKeys = await chrome.storage.local.get(['pkce_verifier', 'oauth_state', 'code_verifier']);
console.log('PKCE state in storage:', pkceKeys);
// If these are null/undefined after a failed auth attempt, the SW terminated mid-flow
// Fix: write verifier/state to storage BEFORE starting the redirect
```

**Fix pattern — persist PKCE state before redirect:**
```js
async function startPKCEFlow() {
  const codeVerifier = generateCodeVerifier(); // your existing function
  const state = generateState();
  
  // Write to storage BEFORE starting the redirect — the SW may die and restart
  await chrome.storage.local.set({ pkce_code_verifier: codeVerifier, pkce_state: state });
  
  const codeChallenge = await deriveCodeChallenge(codeVerifier);
  const authUrl = buildAuthUrl({ codeChallenge, state }); // your existing function
  
  await chrome.tabs.create({ url: authUrl });
}
```

### 3c — Redirect URI not registered

The callback URL must exactly match what is registered with your OAuth provider. For extensions the URL is:
```
https://YOUR_EXTENSION_ID.chromiumapp.org/
```
(Note the trailing slash — providers are often sensitive to this.)

To get the exact URL Chrome will use:
```js
// Paste in Service Worker console
console.log('Extension redirect URI:', chrome.identity.getRedirectURL());
// Compare this to what is registered in your OAuth provider's allowed redirect URIs
```

If you use `chrome.identity.launchWebAuthFlow`, the `redirect_url` parameter must match exactly, including trailing slash.

### 3d — Token present but API returns 401

The token exists but may be:
- **Expired** — check `exp` claim (see 3a snippet above)
- **Wrong audience** — `aud` claim may not match your API's expected audience
- **Wrong scope** — check `scope` or `scp` claim
- **Being sent with the wrong header name** — some APIs expect `Authorization: Bearer <token>`, others want a custom header

```js
// Decode your access token and inspect every claim
(async () => {
  const { accessToken } = await chrome.storage.local.get('accessToken'); // adjust key name
  if (!accessToken) { console.log('No accessToken in storage'); return; }
  const parts = accessToken.split('.');
  if (parts.length !== 3) { console.log('Not a JWT — may be an opaque token'); return; }
  const payload = JSON.parse(atob(parts[1].replace(/-/g,'+').replace(/_/g,'/')));
  console.log('Access token payload:', payload);
  // Look for: aud, scope/scp, exp, iss, sub, and any custom claims your API checks
})();
```

### 3e — Silent refresh loop / refresh token missing

```js
// Check refresh token presence
(async () => {
  const data = await chrome.storage.local.get(null);
  const refreshKeys = Object.entries(data).filter(([k]) => /refresh/i.test(k));
  console.log('Refresh tokens:', Object.fromEntries(refreshKeys));
  // If empty: your auth flow may not be requesting offline_access or the refresh_token scope
})();
```

---

## Category 4 — Content Script Not Injected

### Symptoms
- Extension popup says it is active but nothing happens on the page
- `chrome.tabs.sendMessage` from popup gets "Receiving end does not exist" even though the tab matches your `content_scripts` URL patterns
- DevTools Sources panel does not show your extension's content script under the page
- Script works on page reload but not on navigation within a SPA

### 4a — Verify the script is loaded

```
Page DevTools → Sources → (top of left sidebar) → Content scripts
```

Find your extension in that list. If your script file appears there, it is injected. If it does not appear at all, move to 4b.

### 4b — Check your manifest `matches` pattern

The most common cause: the URL pattern in `manifest.json` does not match the current page URL.

Common mistakes:

| Pattern in manifest | URL that fails to match | Why |
|---|---|---|
| `https://example.com/*` | `https://www.example.com/page` | Missing `www` subdomain |
| `https://app.example.com/*` | `https://app.example.com` (no trailing slash) | Chrome normalizes to a trailing slash, but the pattern still works — double-check with the API |
| `*://example.com/*` | `https://example.com/` | This should work; if it doesn't, check for `host_permissions` conflict |
| `https://example.com/app/*` | `https://example.com/app` | Path without trailing slash doesn't match `/app/*` |

```js
// In Service Worker console — verify which tabs your patterns match right now
const tabs = await chrome.tabs.query({});
const pattern = 'https://example.com/*'; // replace with your pattern
console.log('Tabs matching your pattern:',
  tabs.filter(t => t.url?.match(
    new RegExp('^' + pattern.replace(/\*/g,'.*').replace(/\./g,'\\.') + '$')
  )).map(t => ({ id: t.id, url: t.url }))
);
```

### 4c — SPA navigation (page doesn't reload, URL changes)

`content_scripts` with `"run_at": "document_idle"` only fire on hard page loads. SPAs that use `history.pushState` do not trigger re-injection.

**Fix option A — programmatic injection on navigation detection:**
```js
// service-worker.js
chrome.tabs.onUpdated.addListener(async (tabId, changeInfo, tab) => {
  if (changeInfo.status === 'complete' && tab.url?.match(/your-pattern/)) {
    try {
      await chrome.scripting.executeScript({
        target: { tabId },
        files: ['content-script.js']
      });
    } catch (e) {
      // Tab may not accept scripting (chrome://, PDF, etc.) — safe to ignore
    }
  }
});
```

**Fix option B — listen for URL changes in the content script itself:**
```js
// content-script.js — detect SPA navigation
let lastUrl = location.href;
new MutationObserver(() => {
  if (location.href !== lastUrl) {
    lastUrl = location.href;
    // Re-run your initialization logic
    initializeForCurrentPage();
  }
}).observe(document.body, { subtree: true, childList: true });
```

### 4d — CSP blocking script execution

Some pages (Google apps, banking sites, strict security policies) have a Content-Security-Policy that blocks injected scripts. Check the DevTools console for messages like:
```
Refused to execute inline script because it violates the following Content Security Policy directive...
```

Content scripts injected via `manifest.json` `content_scripts` or `chrome.scripting.executeScript` with `files` are **not** subject to the page's CSP — but `executeScript` with `func` (injecting a function body) **is** subject to CSP in some contexts.

Fix: always inject via `files`, never via inline code in `func`.

### 4e — Shadow DOM hiding elements

If the page uses Shadow DOM (common in web components, some Google and Salesforce apps), your `document.querySelector` calls will fail even though the element exists.

```js
// Test in page DevTools console — check if target element is in shadow DOM
document.querySelectorAll('*').forEach(el => {
  if (el.shadowRoot) console.log('Shadow root found on:', el.tagName, el.className);
});
```

Fix: use `el.shadowRoot.querySelector(...)` to pierce shadow boundaries, or use `composed: true` event listeners.

### 4f — Force-inject for testing (bypass manifest matching)

While debugging, you can inject directly from DevTools without changing your manifest:
```js
// In Service Worker console — inject into the currently active tab
const [tab] = await chrome.tabs.query({ active: true, currentWindow: true });
await chrome.scripting.executeScript({
  target: { tabId: tab.id },
  files: ['content-script.js'] // path relative to your extension root
});
console.log('Injected into tab:', tab.id, tab.url);
```

---

## Category 5 — Access Gate / Subscription Check Failures

> "Access gate" = any logic that reads auth state from storage and decides whether to let the user proceed (subscription check, feature flag, role check, entitlement, etc.)

### Symptoms
- User authenticated successfully but extension still shows "no access" or "not authorized"
- Access works right after login but breaks after a few minutes without re-authenticating
- Different tabs show different access states
- Access check passes on first run but fails after service worker restarts

### 5a — Dump the full access gate state

```js
// Paste in Service Worker console
(async () => {
  const all = await chrome.storage.local.get(null);
  console.log('All storage:', all);
  
  // Find anything that looks like an access/entitlement flag
  const accessKeys = Object.entries(all).filter(([k]) =>
    /access|entitle|sub|plan|tier|role|premium|feature|flag/i.test(k)
  );
  console.log('Access-related keys:', Object.fromEntries(accessKeys));
})();
```

### 5b — Check the storage read order

A common bug: the access gate reads a key that was written **after** the gate check runs. Trace the sequence:

1. Service worker wakes (cold start)
2. Message arrives: "check access"
3. Gate reads `chrome.storage.local.get('hasAccess')` → **null** (not yet written)
4. Gate returns false → user sees "no access"
5. Auth token refresh completes, writes `hasAccess: true` → too late

Fix: ensure auth state is fully restored before processing any access check messages (see the `ensureInitialized` pattern in Category 1).

### 5c — In-memory cache staleness

Many access gates cache the result of a storage/API read in a module-level variable to avoid hitting storage on every call. This cache:
- Is wiped when the service worker terminates (Category 1)
- Does not update when another tab writes new values to storage
- May have a TTL that has not expired even though the underlying token changed

**Detect stale cache:**
```js
// Add this temporarily to your access gate function
async function checkAccess() {
  // Log both the cache state and the live storage state
  console.log('[access-gate] in-memory cache:', cachedAccessResult, 'cached at:', cacheTimestamp);
  const live = await chrome.storage.local.get('accessKey'); // adjust key name
  console.log('[access-gate] live storage:', live);
  // If these differ, your cache is stale
}
```

Fix options:
1. **Lower the TTL** — if you have a TTL of e.g. 60 seconds, reduce it or eliminate it for cases where the underlying token changes
2. **Invalidate on storage change** — listen for `chrome.storage.onChanged` and clear the in-memory cache when relevant keys change
3. **Pass-through on cold start** — if `cacheTimestamp` is 0 (fresh wake), always read from storage instead of using the cache

```js
// Storage change listener to invalidate access cache
chrome.storage.onChanged.addListener((changes, area) => {
  if (area === 'local' && Object.keys(changes).some(k => /token|auth|access/i.test(k))) {
    console.log('[access-gate] storage changed — invalidating cache');
    cachedAccessResult = null;
    cacheTimestamp = 0;
  }
});
```

### 5d — JWT claims not matching what the gate expects

If your access gate reads claims directly from the JWT (rather than a separate storage key), decode the token and verify every claim the gate checks:

```js
// Paste in Service Worker console
(async () => {
  // Replace 'idToken' with whatever key holds your token
  const { idToken } = await chrome.storage.local.get('idToken');
  if (!idToken) { console.log('No token found'); return; }
  
  const payload = JSON.parse(
    atob(idToken.split('.')[1].replace(/-/g,'+').replace(/_/g,'/'))
  );
  console.log('Full token payload:', payload);
  // Now look at every key — your gate likely checks specific custom claims
  // Common patterns: payload['https://your-domain/hasAccess'], payload.custom:plan, etc.
  console.log('All claim keys:', Object.keys(payload));
})();
```

---

## Category 6 — API Call Failures

### Symptoms
- Network requests from the extension fail with 401, 403, 429, or CORS errors
- Requests succeed in the browser but fail when made from the extension
- API calls work in the popup but not in the service worker (or vice versa)

### 6a — Watch the network traffic

The service worker and each extension context have their own network panels:
- **Service worker requests:** `chrome://extensions` → your extension → "Service worker" → Network tab
- **Popup requests:** Right-click icon → Inspect → Network tab
- **Content script requests:** These appear in the *page's* DevTools Network tab

```js
// In Service Worker console — verify the auth header is actually being set
const originalFetch = globalThis.fetch;
globalThis.fetch = async function(url, options = {}) {
  console.log('[fetch-debug]', url);
  console.log('[fetch-debug] headers:', options.headers);
  const response = await originalFetch(url, options);
  console.log('[fetch-debug] status:', response.status);
  return response;
};
// Now trigger the failing API call
```
(Remove this after debugging — it patches the global fetch)

### 6b — 401 — Token not attached or expired

**Checklist:**
1. Is the token present in storage? (Run the storage diagnostic)
2. Is the token expired? (Decode it — check `exp` claim vs `Date.now() / 1000`)
3. Is the correct header name being used? (`Authorization`, `X-Api-Key`, etc.)
4. Is the `Bearer ` prefix included? (`Authorization: Bearer <token>` — note the space)

```js
// Minimal auth header test
(async () => {
  const { accessToken } = await chrome.storage.local.get('accessToken'); // adjust key
  if (!accessToken) { console.log('No token'); return; }
  const res = await fetch('https://your-api-domain.com/api/me', {
    headers: { Authorization: `Bearer ${accessToken}` }
  });
  const body = await res.json().catch(() => res.text());
  console.log(res.status, body);
})();
```

### 6c — CORS errors from the service worker

In MV3, the service worker runs in a privileged context. CORS rules **do** apply, but they can be bypassed by declaring `host_permissions` in `manifest.json`:

```json
"host_permissions": [
  "https://your-api-domain.com/*"
]
```

With this declared, the service worker's fetch requests are treated as extension requests and bypass CORS. Without it, CORS applies normally and the server must return the right headers.

> Note: `host_permissions` applies to service worker and background context requests. Content script requests are still subject to normal CORS unless the request is proxied through the service worker.

**Check your current host_permissions:**
```js
// In Service Worker console
const manifest = chrome.runtime.getManifest();
console.log('host_permissions:', manifest.host_permissions);
console.log('permissions:', manifest.permissions);
```

### 6d — 401 retry pattern

When your token expires mid-session, API calls will start returning 401. The standard pattern is to attempt a refresh once and retry:

```js
async function fetchWithAuth(url, options = {}) {
  const { accessToken } = await chrome.storage.local.get('accessToken');
  const response = await fetch(url, {
    ...options,
    headers: { ...options.headers, Authorization: `Bearer ${accessToken}` }
  });
  
  if (response.status === 401) {
    console.log('[fetchWithAuth] 401 — attempting token refresh');
    const newToken = await refreshAccessToken(); // your refresh function
    if (!newToken) throw new Error('Could not refresh token — user must re-authenticate');
    
    // Retry once with the new token
    return fetch(url, {
      ...options,
      headers: { ...options.headers, Authorization: `Bearer ${newToken}` }
    });
  }
  
  return response;
}
```

### 6e — Rate limiting (429)

If your API returns 429, add exponential backoff:

```js
async function fetchWithBackoff(url, options = {}, maxRetries = 3) {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    const res = await fetch(url, options);
    if (res.status !== 429) return res;
    
    const retryAfter = res.headers.get('Retry-After');
    const waitMs = retryAfter ? parseInt(retryAfter) * 1000 : Math.pow(2, attempt) * 1000;
    console.log(`[fetchWithBackoff] 429 — waiting ${waitMs}ms before retry ${attempt + 1}`);
    await new Promise(r => setTimeout(r, waitMs));
  }
  throw new Error('Max retries exceeded');
}
```

---

## Starter Diagnostic — Paste This First

Run this in the **Service Worker** console before digging into any specific category. It gives you a full picture of storage state in one shot.

```js
// ============================================================
// Chrome Extension Starter Diagnostic
// Paste in: chrome://extensions → your extension → "Service worker"
// ============================================================
(async () => {
  console.group('=== Extension Diagnostic ===');
  
  // 1. Manifest info
  const manifest = chrome.runtime.getManifest();
  console.log('Extension:', manifest.name, 'v' + manifest.version);
  console.log('Manifest version:', manifest.manifest_version);
  
  // 2. All storage keys
  const all = await chrome.storage.local.get(null);
  console.log('All storage keys:', Object.keys(all));
  console.log('Storage entry count:', Object.keys(all).length);
  
  // 3. Auth-related keys (by heuristic — adjust regex to match your naming)
  const authEntries = Object.entries(all).filter(([k]) =>
    /token|auth|session|user|sub|access|refresh|pkce|verifier|state|plan|role/i.test(k)
  );
  if (authEntries.length > 0) {
    console.group('Auth-related storage:');
    authEntries.forEach(([k, v]) => {
      // Truncate long strings for readability
      const display = typeof v === 'string' && v.length > 80 ? v.slice(0, 80) + '…' : v;
      console.log(`  ${k}:`, display);
    });
    console.groupEnd();
  } else {
    console.log('No auth-related keys found (no token/auth/session/etc. in key names)');
  }
  
  // 4. Decode any JWT-looking values
  const jwts = authEntries.filter(([, v]) =>
    typeof v === 'string' && v.split('.').length === 3
  );
  if (jwts.length > 0) {
    console.group('Decoded JWTs:');
    jwts.forEach(([k, v]) => {
      try {
        const payload = JSON.parse(
          atob(v.split('.')[1].replace(/-/g, '+').replace(/_/g, '/'))
        );
        const nowSec = Date.now() / 1000;
        const status = payload.exp
          ? (payload.exp > nowSec ? 'VALID' : '*** EXPIRED ***')
          : 'no expiry';
        console.group(`${k} (${status})`);
        console.log('Payload:', payload);
        if (payload.exp) {
          console.log('Expires:', new Date(payload.exp * 1000).toISOString());
          console.log('Seconds until expiry:', Math.round(payload.exp - nowSec));
        }
        console.groupEnd();
      } catch (e) {
        console.log(`${k}: looks like a JWT but failed to decode`);
      }
    });
    console.groupEnd();
  }
  
  // 5. Service worker metadata
  console.log('SW script URL:', location.href);
  
  console.groupEnd();
  console.log('--- Diagnostic complete. Now check the relevant category above. ---');
})();
```

---

## Build / Reload Workflow

When you change source files, follow this sequence to ensure you are testing fresh code:

```bash
# 1. Rebuild (check your package.json for the correct command)
npm run build        # common for production
# or
npm run dev          # common for watch mode / development build
# or
npm run build:ext    # some projects use a custom target

# Check your package.json scripts section if unsure:
# cat package.json | grep -A 20 '"scripts"'
```

Then in Chrome:
1. `chrome://extensions`
2. Click the **reload icon** (circular arrow) next to your extension
3. Hard-reload any tab where your content script runs: `Cmd+Shift+R` / `Ctrl+Shift+R`
4. Close and reopen the popup (click the icon twice)
5. Re-open the Service Worker DevTools pane — it opened against the old worker, you need a fresh one

**Verify the new code loaded:**
```js
// In Service Worker console after reload
const manifest = chrome.runtime.getManifest();
console.log('Loaded version:', manifest.version);
// If you changed a version number in manifest.json, confirm it matches
```

---

## Quick Reference — Fix Table

| Symptom | Most likely category | First thing to check |
|---|---|---|
| Works for 30s then stops | Cat 1 — SW terminated | Is `chrome://extensions` showing "Service worker: stopped"? |
| "Receiving end does not exist" | Cat 1 or Cat 2 | Is the SW alive? Does it have an `onMessage` listener? |
| "Port closed before response" | Cat 2 | Is the listener returning `true`? |
| Web page can't message extension | Cat 2 | Is `externally_connectable` declared with the correct origin? |
| Auth popup opens, nothing happens | Cat 3 | Check redirect URI vs `chrome.identity.getRedirectURL()` |
| Token present, API returns 401 | Cat 3 or Cat 6 | Decode the token — is it expired? Is `aud`/`scope` correct? |
| Extension active, content script absent | Cat 4 | Check Sources → Content scripts panel in page DevTools |
| Content script works on load, breaks in SPA | Cat 4 | Add `history.pushState` / MutationObserver navigation detection |
| User logged in, still sees "no access" | Cat 5 | Check in-memory cache staleness; compare cache vs live storage |
| API calls fail from extension only | Cat 6 | Check `host_permissions`; watch Network tab in SW DevTools |
| API calls return CORS error | Cat 6 | Add API domain to `host_permissions` in manifest |
| Everything worked in dev, broken in prod | Cat 3 or Cat 4 | Check redirect URIs, CSP headers, URL pattern differences |

---

## Notes on MV3 Gotchas Worth Remembering

- **No persistent background page.** Every module-level variable is ephemeral. Treat the service worker as stateless and always read from `chrome.storage` at the start of each operation.
- **`chrome.storage.local` is NOT `localStorage`.** Content scripts and service workers share the same `chrome.storage.local` namespace. `window.localStorage` is per-origin and not accessible from the service worker at all.
- **`chrome.scripting.executeScript` requires `"scripting"` permission** in `manifest.json`. Forgetting it causes a silent failure.
- **Popup closes = popup context destroyed.** Never store critical state only in popup variables. Write to `chrome.storage` immediately.
- **`return true` in message listeners is load-bearing.** Omitting it causes the port to close before async `sendResponse` calls complete.
- **`chrome.identity.launchWebAuthFlow` opens a new tab.** The tab URL will be `chrome-extension://...` after the redirect. Make sure your OAuth provider allows this redirect URI exactly.
- **Content scripts run in an isolated world by default.** They cannot access the page's JavaScript variables. To communicate with page scripts, use `window.postMessage` or inject a script tag into the DOM.
