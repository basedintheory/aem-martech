# AEM Martech Plugin — Agent Instructions

## 1. Overview

The AEM Martech plugin optimizes Adobe marketing technology for AEM Edge Delivery Services. It decomposes the monolithic Adobe Launch approach into three phased lifecycle components, each powered by the Adobe Experience Platform Web SDK (Alloy) and the Adobe Client Data Layer (ACDL):

- **Eager** — Personalization. Runs before first paint to prevent flash of original content (FOOC).
- **Lazy** — Analytics. Runs after the page is interactive to avoid impacting Largest Contentful Paint (LCP).
- **Delayed** — Third-party tags. Deferred via `setTimeout` to load non-critical Launch scripts after the page is fully interactive.

The plugin source lives at `plugins/martech/src/` after installation.

**Your job:** Instrument the project's existing `head.html` and `scripts/scripts.js` to import and call the plugin at the correct lifecycle points. Do **not** install the plugin files — they are already committed.

---

## 2. Plugin File Structure

| File | Purpose |
| :--- | :--- |
| `src/index.js` | Entry point. Exports all public APIs (see below). |
| `src/alloy.min.js` | Bundled Adobe Experience Platform Web SDK (Alloy) v2.28.0. |
| `src/acdl.min.js` | Adobe Client Data Layer v2.0.2. |

### Exported functions from `src/index.js`

| Export | Role |
| :--- | :--- |
| `initMartech` | Initialize the library (call once in `loadEager`). |
| `updateUserConsent` | Set user consent (IAB TCF 2.0). |
| `martechEager` | Eager-phase logic (personalization). |
| `martechLazy` | Lazy-phase logic (analytics). |
| `martechDelayed` | Delayed-phase logic (third-party tags). |
| `pushToDataLayer` | Push generic payload to ACDL. |
| `pushEventToDataLayer` | Push a standardized event to ACDL. |
| `sendEvent` | Proxy for `alloy('sendEvent', …)`. |
| `sendAnalyticsEvent` | Helper for direct analytics events. |
| `initRumTracking` | Initialize Real User Monitoring. |
| `isPersonalizationEnabled` | Returns `true` if personalization is active. |
| `getPersonalizationForView` | Retrieve propositions for a named view (SPA). |
| `applyPersonalization` | Apply propositions to the DOM. |

---

## 3. Integration Points

### 3a. `head.html` — Add preload and preconnect hints

Append the following lines at the end of the project's `head.html`:

```html
<link rel="preload" as="script" crossorigin="anonymous" href="/plugins/martech/src/index.js"/>
<link rel="preload" as="script" crossorigin="anonymous" href="/plugins/martech/src/alloy.min.js"/>
<link rel="preconnect" href="https://edge.adobedc.net"/>
<!-- Change to adobedc.demdex.net if third-party cookies are enabled -->
```

**Purpose:** Early browser fetching of plugin scripts without blocking rendering, plus early connection to the Adobe Edge Network.

**Path format:** Absolute paths starting with `/plugins/martech/src/`.

---

### 3b. `scripts/scripts.js` — Import statement

Add at the top of `scripts/scripts.js`, alongside other imports:

```js
import {
  initMartech,
  updateUserConsent,
  martechEager,
  martechLazy,
  martechDelayed,
} from '../plugins/martech/src/index.js';
```

**Path format:** Relative from `scripts/` — use `../plugins/martech/src/index.js`.

---

### 3c. `scripts/scripts.js` — Configuration and `loadEager()` modifications

Inside the existing `loadEager` function, call `initMartech` **before** any section loading. It takes two arguments: the Web SDK configuration and the library-specific configuration.

```js
async function loadEager(doc) {
  // Optional: hook in consent check for personalization gating
  const isConsentGiven = true; /* your consent logic here */

  const martechLoadedPromise = initMartech(
    // 1. WebSDK Configuration
    // Docs: https://experienceleague.adobe.com/en/docs/experience-platform/web-sdk/commands/configure/overview
    {
      datastreamId: 'DATASTREAM_ID_HERE',    // REQUIRED — user must replace
      orgId: 'IMS_ORG_ID_HERE',             // REQUIRED — user must replace
      // debugEnabled is automatically set to true on localhost and .page URLs.
      // defaultConsent is automatically set to "pending".
      onBeforeEventSend: (payload) => {
        // Optional callback to modify XDM payload before send.
        // Return false to prevent the event from being sent.
      },
      edgeConfigOverrides: {
        // Optional datastream overrides for different environments.
      },
    },
    // 2. Library Configuration
    {
      personalization: !!getMetadata('target') && isConsentGiven,
      launchUrls: [/* user's Launch script URLs here */],
      // See Configuration Reference Table below for all options.
    },
  );

  // ... rest of loadEager (see 3d for section loading)
}
```

---

### 3d. `scripts/scripts.js` — Wait for personalization in `loadEager()`

After calling `initMartech`, adjust the section-loading block to wait for the martech eager phase. This prevents content flicker from personalized content:

```js
// ... inside loadEager, after initMartech call
if (main) {
  decorateMain(main);
  document.body.classList.add('appear');
  await Promise.all([
    martechLoadedPromise.then(martechEager),
    loadSection(main.querySelector('.section'), waitForFirstImage),
  ]);
}
```

**Key behavior:**
- `martechLoadedPromise.then(martechEager)` chains the eager phase after initialization completes.
- `Promise.all` runs personalization and first-section loading in parallel.
- If personalization is disabled in config, `martechEager` resolves immediately — no performance penalty.
- If personalization is enabled, `martechEager()` **must** resolve before the first section renders to prevent FOOC.

---

### 3e. `scripts/scripts.js` — `loadLazy()` modifications

Add `await martechLazy()` after other lazy-loading logic (e.g. after `loadFooter`). This triggers analytics beacon setup:

```js
async function loadLazy(doc) {
  // ...
  loadFooter(doc.querySelector('footer'));
  await martechLazy();
  // ...
}
```

**Important:** Await `martechLazy()` independently — do **not** wrap it in a `Promise.all` with other operations.

---

### 3f. `scripts/scripts.js` — `loadDelayed()` modifications

Call `martechDelayed()` within the existing `setTimeout`. This loads third-party scripts from `launchUrls`:

```js
function loadDelayed() {
  window.setTimeout(() => {
    martechDelayed();
    import('./delayed.js');
  }, 3000);
}
```

---

## 4. Configuration Reference Table

### Web SDK Configuration (`webSDKConfig`)

| Parameter | Type | Default | User Input Required | Description |
| :--- | :--- | :--- | :--- | :--- |
| `datastreamId` | `string` | — | **Yes** | Adobe Experience Platform datastream ID. |
| `orgId` | `string` | — | **Yes** | Adobe IMS Organization ID (e.g. `XXXXX@AdobeOrg`). |
| `edgeDomain` | `string` | `'edge.adobedc.net'` | No | Adobe Edge Network domain. Use a CNAME for first-party data collection. |
| `defaultConsent` | `string` | `'pending'` | No | Initial consent state: `'in'`, `'out'`, or `'pending'`. |
| `idMigrationEnabled` | `boolean` | `false` | No | Set `true` to migrate legacy Analytics visitor IDs. |
| `targetMigrationEnabled` | `boolean` | `false` | No | Set `true` to migrate from at.js to Web SDK. |
| `onBeforeEventSend` | `function` | `undefined` | No | Callback to modify XDM payload before send. Return `false` to cancel. |
| `edgeConfigOverrides` | `object` | `undefined` | No | Datastream overrides for different environments. |

### Library Configuration (`martechConfig`)

| Parameter | Type | Default | User Input Required | Description |
| :--- | :--- | :--- | :--- | :--- |
| `analytics` | `boolean` | `true` | No | Enable Adobe Analytics tracking. |
| `alloyInstanceName` | `string` | `'alloy'` | No | Global name for the Alloy instance. |
| `dataLayer` | `boolean` | `true` | No | Enable Adobe Client Data Layer (ACDL). |
| `dataLayerInstanceName` | `string` | `'adobeDataLayer'` | No | Global name for the ACDL instance. |
| `includeDataLayerState` | `boolean` | `true` | No | Include full data layer state on every event. |
| `launchUrls` | `string[]` | `[]` | **Yes** (if applicable) | Adobe Launch container URLs to load in delayed phase. |
| `personalization` | `boolean` | `true` | No | Enable Adobe Target / Journey Optimizer personalization. |
| `performanceOptimized` | `boolean` | `true` | No | Use aggressive performance optimizations. |
| `personalizationTimeout` | `number` | `1000` | No | Max milliseconds to wait for personalization before rendering. |
| `shouldProcessEvent` | `function` | `() => true` | No | Filter function — return `false` to prevent an event from being sent. |

---

## 5. Consent Management

- `updateUserConsent(consent)` accepts an object with IAB TCF 2.0 consent categories: `collect`, `marketing`, `personalize`, `share`.
- Call it when the user interacts with a cookie consent banner.
- **Do NOT wire this up automatically** — it depends on the project's specific consent management solution.
- Mark as a **user-configured integration point** and leave a placeholder comment.

### Example: AEM Consent Banner Block

```js
import { updateUserConsent } from '../plugins/martech/src/index.js';

function consentEventHandler(ev) {
  const collect = ev.detail.categories.includes('CC_ANALYTICS');
  const marketing = ev.detail.categories.includes('CC_MARKETING');
  const personalize = ev.detail.categories.includes('CC_TARGETING');
  const share = ev.detail.categories.includes('CC_SHARING');
  updateUserConsent({ collect, marketing, personalize, share });
}
window.addEventListener('consent', consentEventHandler);
window.addEventListener('consent-updated', consentEventHandler);
```

### Example: OneTrust

```js
import { updateUserConsent } from '../plugins/martech/src/index.js';

function consentEventHandler(ev) {
  const groups = ev.detail;
  const collect = groups.includes('C0002');
  const personalize = groups.includes('C0003');
  const share = groups.includes('C0008');
  updateUserConsent({ collect, personalize, share });
}
window.addEventListener('consent.onetrust', consentEventHandler);
```

---

## 6. Advanced APIs (Do NOT instrument during initial setup)

These functions are exported from the plugin but should **not** be wired up during initial instrumentation. They are for project-specific use cases:

| Function | Purpose |
| :--- | :--- |
| `pushToDataLayer(payload)` | Push arbitrary data to Adobe Client Data Layer. |
| `pushEventToDataLayer(event, xdm, data, configOverrides)` | Push a standardized event to the data layer. |
| `sendEvent(payload)` | Send a raw event via `alloy('sendEvent', …)`. |
| `sendAnalyticsEvent(xdmData, dataMapping, configOverrides)` | Send a direct analytics event. |
| `initRumTracking(sampleRUM, options)` | Initialize Real User Monitoring tracking. |
| `isPersonalizationEnabled()` | Check if personalization is active (only when personalization enabled). |
| `getPersonalizationForView(viewName)` | Fetch propositions for a named view / SPA route. |
| `applyPersonalization(viewName)` | Apply propositions to the DOM for a view. |

---

## 7. Common Gotchas

- **Call order matters:** `initMartech` **must** be called before `martechEager`, `martechLazy`, or `martechDelayed`.
- **Personalization and FOOC:** If personalization is enabled, `martechEager()` **must** resolve before the first section renders. Use the `Promise.all` pattern shown in section 3d.
- **`launchUrls` are delayed:** Scripts listed in `launchUrls` load during the delayed phase — never put critical scripts here.
- **Bundled Alloy:** The plugin bundles its own Web SDK (`alloy.min.js`). Do **not** add a separate Web SDK `<script>` tag.
- **Import paths:** From `scripts/scripts.js`, use `../plugins/martech/src/index.js` (relative).
- **Preload paths:** In `head.html`, use absolute paths starting with `/plugins/martech/src/`.
- **`defaultConsent`:** The library defaults consent to `'pending'`. Setting it to `'in'` grants consent without user agreement — ensure this complies with your legal requirements.
- **Launch container conflicts:** Do **not** include the `Adobe Experience Platform Web SDK`, `Adobe Analytics`, or `Adobe Target` extensions in your Launch container — the plugin handles these directly.
- **`martechLazy()` sequencing:** Await `martechLazy()` independently. Do not wrap it in `Promise.all` with other operations.
