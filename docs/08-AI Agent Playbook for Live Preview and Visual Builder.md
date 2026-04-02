# AI Agent Playbook for Live Preview and Visual Builder

> **Audience:** AI coding agents only.

> **Purpose:** Use this chapter as an operating manual when a developer asks you to diagnose or fix a Contentstack Live Preview or Visual Builder problem.

> **Outcome:** By the end of your investigation, you should know which contract is broken, patch the smallest correct layer, and hand the developer a short verification checklist.

---

If you are reading this chapter as an agent, do not treat it as background reading. Treat it as procedure.

Your job is to identify where preview context is lost, why draft content is not reaching the renderer, or why Visual Builder cannot map rendered elements back to fields. Do not guess from symptoms alone.

## Core Truths

Anchor on these facts before you inspect code:

1. **Live Preview is a session, not just an endpoint switch.** Contentstack creates a session-scoped `live_preview` hash and loads the site with it.
2. **Change events contain no content payload.** The CMS signals that something changed. The site must refetch.
3. **Preview and Delivery APIs share shapes, but not intent.**
   Delivery API serves published content and is cacheable.
   Preview API serves draft content and is never cacheable.
4. **The `ssr` setting controls preview behavior, not the entire app architecture.**
   `ssr: false` means refetch in place in the browser.
   `ssr: true` means reload the iframe so the server renders again.
5. **Preview context is runtime state.** The hash can rotate. It must not be persisted, cached, or shared across requests.
6. **Visual Builder depends on edit tags.** Live Preview can work while Visual Builder still fails.
7. **Most failures are transport failures.** The hash is missing, dropped, cached away, or ignored before the fetch happens.

If you remember only one rule, remember this: **trace one edit through the whole system.**

## Your Workflow

Work in this order every time:

1. Classify the symptom.
2. Classify the rendering strategy on the affected route.
3. Interview the developer only enough to unblock inspection.
4. Inspect the codebase for the preview contract.
5. Name the most likely broken contract before editing.
6. Patch the smallest correct layer.
7. Give the developer a short verification checklist.

Do not ask for secrets. Ask only for code, redacted URLs, screenshots, stack settings, logs, or devtools evidence.

## Step 1: Classify the Symptom

Map the issue into one of these buckets immediately:

| Symptom bucket | Usually means |
| -------------- | ------------- |
| Blank preview or setup status errors | connection, iframe policy, base URL, or SDK init |
| Published content in preview | missing hash or wrong API path |
| Edits do not update | wrong `ssr` mode, no event loop, or no reload path |
| Preview breaks after navigation | hash propagation failure |
| Wrong entry, wrong locale, or another editor's content | shared SSR state or caching |
| Visual Builder controls fail but preview updates work | edit tags or builder mode problem |

Do not branch into multiple buckets at once unless the evidence forces you to.

## Step 2: Classify the Rendering Strategy

You must know which rendering rules apply before debugging.

Ask only the minimum needed:

- What framework and route are affected?
- Does this page fetch content in the browser, on the server per request, or from static output with a preview mode?
- Is there a middleware, BFF, API route, or proxy between the page and Contentstack?

Map the result:

- **CSR**: browser fetch, in-place rerender, `ssr: false`
- **SSR**: server fetch per request, iframe reloads, `ssr: true`
- **SSG preview**: static in production, dynamic in preview mode
- **Middleware/BFF**: preview state must survive an extra hop

If the route mixes strategies, isolate the exact path the affected page uses before proceeding.

## Step 3: Interview the Developer

Ask for evidence, not opinions.

Request only what helps you prove or eliminate a failure family:

- A redacted preview URL or screenshot showing whether `live_preview` is present
- The code path for `ContentstackLivePreview.init()`
- The code path where preview vs delivery is chosen
- One failing network request showing hostname and cache headers
- A screenshot of Contentstack's "Display Setup Status" panel if the page is blank or disconnected
- The data-layer function where `addEditableTags()` should run if Visual Builder is involved

Never ask for:

- API keys
- preview tokens
- cookies
- raw auth headers
- entire `.env` files

## Step 4: Audit the Repo

Start with targeted searches:

```bash
rg -n "ContentstackLivePreview|onEntryChange|onLiveEdit|live_preview|preview_token|rest-preview|addEditableTags|data-cslp|VB_EmptyBlockParentClass|draftMode|setPreviewData|Cache-Control|no-store|frame-ancestors|X-Frame-Options"
```

Then answer these questions from the code:

1. Where is `ContentstackLivePreview.init()` called?
2. Is `ssr` correct for the affected route?
3. Where does the app switch between Preview API and Delivery API?
4. Where is the hash read from?
5. Can navigation, middleware, redirects, or rewrites drop query params?
6. Is caching bypassed whenever preview is active?
7. If Visual Builder is involved, where are edit tags generated and spread into the DOM?

## Step 5: Identify the Broken Contract

Use the following diagnosis tree.

### A. Blank Preview or Setup Status Errors

Check:

- Is the preview URL reachable directly in a browser tab?
- Does the site block embedding with `X-Frame-Options` or `frame-ancestors`?
- Is the stack's Live Preview Base URL correct for the environment and locale?
- Does browser-executed code on the affected route call `ContentstackLivePreview.init()`?
- If `clientUrlParams.host` is set, does it match the stack region?

Likely broken contract:

- the CMS can load the URL, but the page cannot initialize a valid preview session

Likely fixes:

- loosen iframe policy or use "Always Open in New Tab"
- correct base URL configuration
- move SDK init into browser code that actually runs on the previewed route

### B. Published Content in Preview

Check:

- Does the URL include `live_preview=...`?
- Do requests hit the preview host instead of the delivery host?
- Is a preview token configured when preview is active?
- Is the hash forwarded into the layer that actually fetches content?
- If a proxy or middleware exists, does it receive and forward the hash too?

Likely broken contract:

- preview context exists in the browser but never reaches the content fetch

Likely fixes:

- forward `live_preview` from URL or SDK into the fetch layer
- switch to Preview API when the hash is present
- configure preview host and preview token correctly

### C. Edits Do Not Update

Separate CSR behavior from SSR behavior first.

For CSR, check:

- Is `ssr: false` configured?
- Is `onEntryChange()` or `onLiveEdit()` registered?
- Does the callback refetch content?
- Does state get replaced instead of partially merged?
- Is the subscription missing, duplicated, or mounted too late?

For SSR, check:

- Is `ssr: true` configured?
- Does the iframe reload after an edit?
- Does the server receive `live_preview` on the next request?
- Is the server-side client request-scoped and preview-aware?

Likely broken contract:

- the CMS can signal change, but the route does not execute the correct refresh path

Likely fixes:

- correct the `ssr` setting
- initialize the SDK earlier
- register the change callback once
- make the CSR refetch deterministic
- preserve preview params on SSR reload requests

### D. Preview Breaks After Navigation

Treat this as a hash propagation bug until you prove otherwise.

Check:

- Do internal links preserve `live_preview`, `content_type_uid`, `entry_uid`, and `locale` where needed?
- Do redirects, rewrites, auth guards, or middleware strip query params?
- Does client-side routing rebuild URLs without preview context?

Likely broken contract:

- preview context exists on the initial load but is lost during route changes

Likely fixes:

- append preview params during navigation in preview sessions
- preserve search params in redirects and rewrites
- clone full URLs instead of overwriting `search`

### E. Wrong Entry, Wrong Locale, or Another Editor's Draft

Treat this as shared state or caching.

Check:

- Is the Contentstack client instantiated globally and mutated per request in SSR?
- Is `livePreviewQuery()` called on a shared SDK instance?
- Are preview responses cached at CDN, framework, proxy, memory, or database layers?
- Is any persistent storage holding preview responses?

Likely broken contract:

- preview state that should be request-scoped is being shared or cached

Likely fixes:

- create request-scoped clients for SSR and BFF layers
- disable caching whenever `live_preview` is present
- keep preview data out of long-lived caches and persistence layers

### F. Visual Builder Fails but Preview Updates Work

This is not a transport problem. It is a tagging problem until proven otherwise.

Check:

- Is the SDK in `mode: "builder"` where builder UX is required?
- Does the fetched entry pass through `addEditableTags()` in the data layer?
- Are `$` props spread into real DOM nodes?
- Are referenced entries tagged with their own content type UID?
- Are modular block containers tagged correctly?
- Does an empty modular blocks parent use `VB_EmptyBlockParentClass` when needed?

Likely broken contract:

- the page renders content, but the builder cannot map DOM nodes back to entry fields

Likely fixes:

- call `addEditableTags(entry, contentTypeUid, true, locale)` immediately after fetching
- spread `entry.$?.fieldName` onto rendered elements
- tag referenced entries separately
- add `VB_EmptyBlockParentClass` to empty modular block parents

## Condensed Knowledge

Use this as a quick reference while you work.

### Session and Hash

- `live_preview` is the session-scoped hash
- it is runtime state, not environment config
- it can rotate
- it must not be stored in a database or long-lived cookie
- it must be present anywhere preview content is fetched

### Preview vs Delivery

- **Delivery API**: published, cacheable, safe for production
- **Preview API**: draft, requires preview token plus hash, never cacheable
- mixing them on one page creates inconsistent published/draft output

### Rendering Rules

- **CSR**: browser init, `ssr: false`, subscribe, refetch in place
- **SSR**: browser init, `ssr: true`, reload iframe, fetch draft content per request
- **SSG**: escape static output with preview mode or draft mode
- **Middleware/BFF**: decide preview vs delivery where the Contentstack fetch actually happens

### Visual Builder Rules

- Live Preview can work without edit tags
- Visual Builder cannot work without edit tags
- `addEditableTags()` is the safest way to generate `data-cslp`
- the generated `$` props must be spread onto actual DOM nodes
- empty modular-block containers often need `VB_EmptyBlockParentClass`

### Setup Status Clues

If the developer can access Contentstack's setup status panel, use it early.

It can quickly reveal:

- website unreachable
- iframe blocked
- SDK not initialized
- outdated SDK
- Preview Service not enabled
- default environment not configured

## Editing Rules

When you patch the repo:

- fix the smallest layer that restores the preview contract
- do not refactor unrelated code while debugging preview
- do not move secrets into the client
- do not cache preview data to "make it faster"
- do not persist the hash
- do not keep a shared mutable preview client in SSR

If multiple fixes are possible, prefer the one that makes preview flow more explicit and request-scoped.

## Verification Checklist to Hand Back

After your fix, tell the developer to verify these exact behaviors in Contentstack:

- open the same entry again in Live Preview
- confirm the URL contains `live_preview`
- make a small text edit and confirm the page updates
- navigate to a second previewed page and confirm draft context remains intact
- if Visual Builder is involved, click a tagged field and confirm the correct field opens
- if modular blocks are involved, test add, reorder, and delete actions

## When to Escalate

Pause and explain the tradeoff before continuing if:

- the affected route mixes CSR and SSR in a way that obscures the preview contract
- the developer wants preview responses cached
- middleware or a custom router rewrites most URLs and you cannot yet prove hash preservation
- the issue appears to be in Contentstack stack configuration rather than repo code
- the failure is browser-only and you need screenshots, network traces, or setup status evidence

When escalating, summarize:

- what you proved
- what remains uncertain
- the smallest next piece of evidence needed

## Final Instruction

Do not diagnose Live Preview from symptoms alone.

Reduce the problem to a contract:

- the CMS must load the page with preview context
- the SDK must initialize in the correct mode
- the fetch layer must use preview services when preview is active
- caches must be bypassed
- the renderer must apply the new data
- Visual Builder must see correct edit tags

Find the broken link. Fix only that link first.
