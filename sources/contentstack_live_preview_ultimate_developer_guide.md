# Contentstack live preview

## Proposal

This guide is written as a single, long-form, developer-first document that can be read top to bottom or used as a reference. It starts with first principles and architecture, then progressively dives into real implementations, edge cases, and framework-specific patterns.

The structure is intentionally layered:

- Part 1 builds a mental model of how Live Preview works internally
- Part 2 explains the protocols, APIs, and data flow in detail
- Part 3 walks through concrete implementations, starting simple and getting progressively more advanced
- Part 4 focuses on edit tags and Visual Builder, including why they exist and how to reason about them
- Part 5 covers real-world architectures, trade-offs, and failure modes

The goal is not just to show how to “turn on” Live Preview, but to help developers reason about it, debug it, and adapt it to non-standard architectures.

---

## Table of contents

1. What live preview actually is
2. Traditional preview vs live preview
3. Core architecture and moving parts
4. The live preview session lifecycle
5. Preview api vs delivery api
6. The live preview hash
7. Communication between cms and website
8. Rendering strategies and why they matter
9. Client-side rendering fundamentals
10. CSR with the live preview sdk
11. CSR without the sdk
12. Server-side rendering fundamentals
13. Live preview with server-side rendering
14. Static site generation and preview
15. Middleware and database-backed architectures
16. Edit tags explained
17. Applying edit tags in real apps
18. Visual builder architecture
19. Debugging, pitfalls, and best practices

---

## 1. What live preview actually is

Live Preview is a coordinated editing session between Contentstack and your website.

If you remember one thing: Live Preview is not “a different api base url”. It is a live session where Contentstack can signal your site to refetch draft content, and your site can respond in the right way for its rendering model.

### The mental model

There are three concurrent systems at play:

1. A content editing session (inside the Contentstack entry editor)
2. A preview rendering session (your site running in an iframe or a new tab)
3. A data access session (draft content served via preview services)

The Live Preview SDK sits between 1 and 2. The Preview API sits behind 3.

### What Live Preview is doing, mechanically

When an editor opens Live Preview:

- Contentstack generates a session identifier (the live preview hash)
- Contentstack loads your website with that hash in the url
- Your website initializes Live Preview
- A postMessage handshake happens between iframe and parent window
- Contentstack pushes “something changed” events
- Your website refetches draft content and rerenders

This design is intentional.

- The CMS never sends your content payload to the site. It sends events.
- The site never reads editor state directly. It refetches.
- The preview services never broadcast. They authorize and serve.

That separation keeps the system consistent across CSR, SSR, and SSG.

### What Live Preview is not

Live Preview is not:

- A websocket delivering updated json
- A magic iframe overlay that edits the dom
- A server feature that “streams html”
- A stable environment you can cache

If you treat it like any of those, your implementation will feel flaky.

### Terminology you will use a lot

- Preview token: a token that allows fetching draft content for a given environment
- Live preview hash: a short-lived session identifier that scopes draft responses
- Live Preview SDK: a client-side library that manages messaging and session state
- Edit tags: DOM attributes that map rendered elements back to entry fields
- Visual Builder: the on-page editing layer built on top of edit tags

---

## 2. Traditional preview vs live preview

Traditional preview is transactional. Live Preview is conversational.

### Traditional preview

Traditional preview usually looks like:

- Save entry
- Maybe publish to a preview environment
- Open a preview url
- Manually refresh as you iterate

This works, but it has obvious friction:

- Feedback loop is slow
- Editors constantly alt-tab
- Publishing becomes a proxy for “I want to see it”

### Live preview

Live Preview keeps the editor inside one continuous loop:

- Website is embedded next to the entry editor
- Content changes trigger events
- Website refetches and rerenders automatically

What changes for developers is that you must implement:

- Session-aware data fetching
- Event-driven rerendering
- A clean separation between published and draft data

If you do those well, editors get real-time confidence.

---

## 3. Core architecture and moving parts

At a high level, Live Preview consists of five pieces.

### Contentstack cms

- Hosts the entry editor
- Detects changes and emits events
- Creates and rotates the live preview hash
- Loads your site in an iframe or a new tab

### Your website

- Renders content
- Initializes the Live Preview SDK
- Listens for events and decides what to do
- Refetches draft content on demand

### Live preview sdk

- Establishes a postMessage handshake
- Tracks session and hash updates
- Exposes a small api surface for you to subscribe to changes
- Coordinates reload behavior for SSR vs CSR

### Preview services

This is the part most people hand-wave. For implementation accuracy, treat preview services as:

- An alternate content access path that can serve draft content
- A mechanism that uses preview token plus live preview hash to scope what you are allowed to see

### Delivery services

- The normal published content channel
- Cacheable and safe for production

### A practical diagram

You can keep this picture in your head:

```
Editor types in CMS
        |
        | postMessage event (entry changed)
        v
Website receives event
        |
        | refetch content using preview token + live preview hash
        v
Preview services return latest draft json
        |
        v
Website rerenders
```

Notice what is not present: the CMS never pushes the draft json to your site.

---

## 4. The live preview session lifecycle

A Live Preview session begins when an editor opens an entry with Live Preview enabled.

### Phase 1: session creation

Contentstack creates a session and generates a live preview hash.

This hash is not a “stack setting”. It is runtime state.

### Phase 2: site load

Contentstack loads your website with query params such as:

- content type uid
- entry uid
- locale
- live preview hash

This allows your site to know what it should render and how it should fetch.

### Phase 3: handshake

Your site initializes the Live Preview SDK, which sends an init message to the parent window.

The CMS responds with an acknowledgment.

At this point, the channel is open.

### Phase 4: steady state

While the editor types:

- CMS emits events
- SDK receives them
- Your code is notified via callbacks

Your responsibility is to refetch and rerender.

### Phase 5: session end

The session ends when the editor closes the entry or navigates away.

The hash will eventually expire and should be treated as invalid after the session.

### Why the hash can rotate

If your implementation assumes the hash never changes, it will break in subtle ways.

Always treat the current hash as:

- dynamic
- owned by the SDK
- provided by the CMS events

---

## 5. Preview api vs delivery api

One of the most important concepts to internalize is that Live Preview does not introduce a new content model. It introduces a new access path.

Both the Preview API and Delivery API:

- share the same routes
- accept the same query structure
- return the same content shape

The difference is authentication and intent.

### Delivery api

- published content only
- cacheable
- safe for production

### Preview api

- draft and unpublished content
- never cacheable
- requires preview token and live preview hash

### The easiest way to avoid bugs

Decide early whether you want your site to fetch content:

- directly from Contentstack in the browser, or
- through your own backend proxy

Both work. Mixing them often creates inconsistencies.

---

## 6. The live preview hash

The live preview hash is a short-lived session identifier.

It serves three purposes:

- authenticates preview requests
- scopes preview content to a user session
- prevents preview data from leaking across sessions

### Practical implications

- do not store it in a database
- do not put it in a long-lived cookie
- do not cache responses keyed only by url path

Treat it like you would treat a signed, expiring token.

### CSR vs SSR

- In CSR, the SDK manages hash updates and your fetch calls simply need to use the latest context.
- In SSR, the server must receive the hash on every request. That means you must propagate it during navigation.

---

## 7. Communication between cms and website

Live Preview communication is built on the browser postMessage api.

The SDK abstracts it, but you should still understand what is happening.

### The event types you care about

You will commonly encounter a flow like:

- init: sent by your site
- init-ack: sent by the CMS
- client-data-send: sent by the CMS when content changes

### What your code should do

You should not try to interpret editor deltas.

When you get “something changed”, refetch the content you need.

That keeps your app deterministic.

### Why this matters for debugging

When Live Preview feels broken, you can debug in layers:

1. Is the preview iframe url receiving the expected query params?
2. Is the SDK initializing?
3. Are postMessage events arriving?
4. Are you refetching with preview context?
5. Are you rendering the new data?

If you can answer those, you can fix almost any Live Preview issue.

---

## 8. Rendering strategies and why they matter

Before touching any SDK code, you need to decide which rendering model you are operating in. Live Preview is extremely sensitive to this choice.

Most Live Preview bugs are not SDK bugs. They are architectural mismatches between how a site renders and how preview events are handled.

### The three rendering strategies

Live Preview officially supports three high-level rendering strategies:

- client-side rendering (CSR)
- server-side rendering (SSR)
- static site generation (SSG)

They all use the same preview services, the same hash, and the same editor events. What changes is *where* data is fetched and *when* rerendering happens.

### Why rendering strategy affects Live Preview

Live Preview answers one core question repeatedly:

“Something changed. How do I show the new state?”

The answer depends entirely on your rendering model.

- CSR answers by refetching data in the browser
- SSR answers by re-running the request on the server
- SSG answers by bypassing static output and temporarily behaving like SSR

If you try to mix these answers, Live Preview will feel unreliable.

### A rule of thumb

- CSR: refetch, no reload
- SSR: reload, refetch on server
- SSG: preview-only dynamic fallback

Everything else in this guide builds on that distinction.

---

## 9. Client-side rendering fundamentals

Client-side rendering is the simplest and most forgiving Live Preview model.

In CSR:

- the html shell is static
- content is fetched at runtime in the browser
- the application instance stays alive

This maps perfectly to Live Preview’s event-driven nature.

### CSR data flow without Live Preview

Normally, a CSR app works like this:

1. browser loads html and js
2. app boots
3. app fetches content
4. app renders

### CSR data flow with Live Preview

With Live Preview enabled:

1. browser loads html and js inside the CMS iframe
2. app boots and initializes the Live Preview SDK
3. app fetches preview content
4. CMS emits change events
5. app refetches content
6. app rerenders

Crucially, steps 2–6 all happen without a page reload.

### Why CSR feels instant

Because the app instance never dies:

- state can be replaced
- components can rerender
- editors see changes immediately

This is why CSR is often recommended for marketing pages and editorial experiences.

### The main CSR responsibility

In CSR, *you* own rerendering.

The SDK will tell you when something changed. It will not refetch content for you. That separation keeps the system framework-agnostic.

Your job is to:

- listen for entry change events
- refetch the correct data
- update application state

### Common CSR mistakes

- assuming the event contains updated content
- refetching without preview context
- forgetting to unsubscribe listeners on unmount
- triggering multiple refetches per change

CSR Live Preview works best when your data fetching is centralized and deterministic.

---

## 10. CSR with the live preview sdk

This is the recommended implementation path for client-rendered applications.

The Live Preview SDK handles communication and session state. You remain responsible for data fetching and rendering.

### SDK initialization

Initialization must happen once, early in the app lifecycle, and only in preview contexts.

```ts
import ContentstackLivePreview from '@contentstack/live-preview-utils';

ContentstackLivePreview.init({
  enable: true,
  mode: 'preview',
  ssr: false,
  stackDetails: {
    apiKey: process.env.NEXT_PUBLIC_CS_API_KEY,
    environment: process.env.NEXT_PUBLIC_CS_ENV,
  },
});
```

Key points:

- ssr must be false
- this code must run in the browser
- initialization should not depend on route-level logic

### Subscribing to entry changes

The SDK exposes a simple subscription model:

```ts
ContentstackLivePreview.onEntryChange(() => {
  refetchContent();
});
```

This callback fires when:

- an entry is edited
- content is saved
- content is published

The SDK intentionally does not differentiate. Your app should always refetch.

### What refetchContent should do

A correct refetch strategy:

- uses the same query as the initial fetch
- includes preview context automatically
- replaces existing state atomically

Avoid partial merges or diff-based updates. Determinism beats cleverness.

### Handling multiple entries

If a page renders multiple entries:

- refetch all entries used on the page, or
- build a dependency map from page → entries

Premature optimization here often leads to stale renders.

### Cleanup and lifecycle

In component-based frameworks:

- register listeners on mount
- unregister on unmount

This prevents duplicated refetches and memory leaks during navigation.

---

## 11. CSR without the sdk

Implementing Live Preview without the SDK means implementing the communication protocol yourself.

This is not recommended for production, but understanding it clarifies what the SDK actually does.

### What the SDK normally abstracts

Without the SDK, you must handle:

- postMessage handshake
- message origin validation
- hash updates
- reload vs refetch decisions
- navigation synchronization

### Minimal manual setup

A simplified example:

```ts
window.addEventListener('message', (event) => {
  if (event.data?.data?.type === 'client-data-send') {
    window.livePreviewHash = event.data.data.hash;
    refetchContent();
  }
});

window.parent.postMessage(
  {
    from: 'live-preview',
    type: 'init',
    data: {
      config: {
        shouldReload: false,
        isSSR: false,
        href: window.location.href,
      },
    },
  },
  '*'
);
```

This barely scratches the surface.

### Why this approach breaks easily

Manual implementations often fail because:

- message formats change subtly
- hash rotation is missed
- cleanup is forgotten
- security boundaries are ignored

The SDK exists to centralize these concerns and keep your app stable across updates.

### When manual implementation makes sense

Very rare cases:

- extreme bundle-size constraints
- non-browser JS environments
- educational tooling

For everything else, use the SDK.

---

## 12. Server-side rendering fundamentals

Server-side rendering fundamentally changes how Live Preview behaves.

In CSR, your app is already running when content changes arrive. In SSR, your app only exists for the duration of a request. That single fact explains almost every SSR Live Preview pitfall.

### The SSR mental model

In an SSR setup:

- The browser requests a url
- The server fetches content
- The server renders html
- The response is sent
- The server-side context is destroyed

When Live Preview is enabled, this still holds true.

There is no long-lived server process “listening” for preview updates. Every update is handled by reloading the iframe and issuing a new request.

That means Live Preview for SSR is:

- request-driven
- reload-based
- stateless between requests

Once you internalize that, SSR Live Preview becomes predictable.

### What changes compared to CSR

In CSR:

- Content updates arrive via events
- You refetch data in the browser
- You rerender without navigation

In SSR:

- Content updates arrive via events
- The SDK decides a reload is required
- The iframe reloads the page
- The server handles a new request

Your server never receives postMessage events directly.

### The non-negotiable SSR requirements

Every SSR Live Preview implementation must do the following, without exception:

1. Extract the live preview hash from the incoming request
2. Decide whether this request is a preview request
3. Switch content fetching to preview services when needed
4. Disable caching for preview responses
5. Render html using draft content

If any one of these steps is missing, Live Preview will appear broken or inconsistent.

### Extracting the live preview hash

When Contentstack loads your site in Live Preview mode, it appends query parameters such as:

- live_preview
- content_type_uid
- entry_uid
- locale

The exact parameter names are stable and should be treated as part of the contract.

In SSR, you must extract the hash from the request itself, not from global state.

Conceptually:

```ts
const hash = request.query.live_preview
```

If there is no hash, the request is not a Live Preview request.

### Switching api behavior per request

This is where many implementations go wrong.

The live preview hash is request-scoped. That means your content client must also be request-scoped.

Incorrect pattern:

- Create a single Contentstack SDK instance
- Mutate it when a preview request comes in
- Reuse it for other requests

This will leak preview state across users.

Correct pattern:

- Create a new SDK instance per request
- Configure it based on the presence of the hash
- Dispose of it after rendering

Think of preview configuration as part of request state, not app configuration.

### Reload behavior

In SSR, Live Preview updates always require a reload.

The SDK communicates this to the CMS by sending an init message with:

- shouldReload: true
- isSSR: true

This tells Contentstack that any content change should trigger an iframe reload rather than a client-side refetch.

### Why caching must be disabled

Preview responses must never be cached because:

- They include draft content
- They are scoped to a single editor session
- The same url path can produce different results for different editors

If you cache preview responses at the CDN, server, or application layer, editors will see stale or incorrect content.

A safe rule:

If live_preview is present, bypass all caches.

### Navigation and hash propagation

One subtle SSR issue is navigation.

When an editor clicks a link inside the preview iframe:

- The browser navigates to a new url
- The live preview hash must be preserved

If you drop the hash, the next request will hit the Delivery API instead of Preview API.

The Live Preview SDK helps with this by:

- Intercepting navigation
- Appending the preview parameters to links

If you build custom routing logic, you must ensure this behavior is preserved.

### Summary

SSR Live Preview is not complex, but it is unforgiving.

If you treat preview state as request state, create isolated clients, disable caching, and allow reloads, it works reliably.

The next section applies these principles concretely in Next.js.

---

## 13. Live preview with server-side rendering

This section deliberately avoids framework specifics.

Whether you are using Next.js, Nuxt, Remix, Astro SSR, Express with templates, or a custom Node server, Live Preview with SSR follows the same rules.

Frameworks differ in APIs. The constraints do not.

### The universal SSR Live Preview contract

If your site uses SSR, Live Preview expects the following contract to be true:

- The browser reloads the page on content change
- The server receives the live preview hash on every request
- The server decides preview vs production per request
- The server fetches content accordingly
- No preview response is cached

If any one of these breaks, preview breaks.

### Request lifecycle in SSR preview

A single preview update looks like this:

1. Editor changes content
2. CMS emits a change event
3. SDK signals that a reload is required
4. The iframe reloads the current url
5. The server receives a new request with live_preview params
6. The server fetches draft content
7. The server renders html
8. The browser displays the updated page

There is no partial update path in SSR.

### Initial request handling

On the very first request inside Live Preview:

- The request already contains the live preview hash
- The server must immediately treat it as a preview request

This means:

- do not fetch published content first
- do not rely on client-side hydration to fix things later

If the first html is wrong, editors will see flicker or stale content.

### Per-request content clients

This cannot be overstated.

Never share a Contentstack SDK instance across requests in SSR when Live Preview is involved.

Correct mental model:

- each request creates its own content client
- preview configuration lives only for that request
- the client is discarded after render

This avoids:

- preview state leaking between users
- race conditions under load
- extremely hard-to-debug inconsistencies

### Preview detection

Preview detection is intentionally simple:

```ts
const isPreview = Boolean(request.query.live_preview)
```

If isPreview is true:

- use preview api
- include preview token
- include live preview hash
- disable caching

If false:

- use delivery api
- allow caching

Do not overthink this logic.

### Navigation and link handling

In SSR preview mode, navigation must preserve context.

That means:

- internal links must carry the live preview params
- server-side redirects must forward them
- middleware must not strip them

If a single navigation drops the hash, the next request silently switches to published content.

This is one of the most common SSR preview bugs.

### Reload configuration

During SDK initialization on the client, SSR apps must signal reload behavior:

```ts
ContentstackLivePreview.init({
  enable: true,
  ssr: true,
  mode: 'preview',
});
```

This tells Contentstack:

- do not expect client-side refetching
- reload the iframe when content changes

### Why SSR preview feels slower

Compared to CSR, SSR preview:

- performs a full roundtrip per change
- re-renders the entire page
- invalidates more layers

This is not a flaw. It is a consequence of SSR’s guarantees.

The trade-off is correctness and SEO parity.

### When SSR preview is the right choice

SSR preview makes sense when:

- seo is critical
- html must reflect content immediately
- personalization happens on the server
- content-driven routing is complex

If editorial speed is the top priority, CSR often feels better.

### Summary

SSR Live Preview is strict, predictable, and framework-agnostic.

If you treat preview as request-scoped state, reload aggressively, and never cache draft content, it works across all SSR frameworks.

---



## 15. Static site generation and preview

Static site generation sits in an awkward middle ground for Live Preview.

By definition, SSG produces html ahead of time. Live Preview requires fresh, draft-aware rendering on demand. Those two goals conflict unless you introduce an escape hatch.

### The SSG mental model

In SSG:

- pages are rendered at build time
- content is baked into static files
- requests are normally served without hitting your server logic

That model is excellent for performance, but incompatible with live editing.

### How Live Preview works with SSG

Live Preview does **not** update static files.

Instead, every serious SSG framework implements a preview mode that temporarily switches behavior:

- Next.js preview mode
- Nuxt hybrid rendering
- Astro SSR fallback

In preview mode:

- static generation is bypassed
- requests are handled dynamically
- draft content can be fetched

Conceptually, SSG preview is just SSR with guardrails.

### The SSG preview contract

When Live Preview is active:

- static output must be ignored
- the live preview hash must be respected
- content must be fetched per request
- caching must be disabled

When Live Preview is inactive:

- static output is safe
- delivery api can be cached aggressively

### The most common SSG mistake

Trying to “patch” static content with client-side refetching.

This usually results in:

- hydration mismatches
- layout flicker
- inconsistent editor experience

If you use SSG, accept that preview means temporarily leaving SSG.

### Summary

SSG and Live Preview can coexist, but only if your framework supports a true preview mode.

If it does not, treat that framework as CSR or SSR for preview purposes.

---

## 16. Middleware and database-backed architectures

Many real-world architectures do not fetch Contentstack data directly in the frontend.

Common patterns include:

- backend-for-frontend (BFF)
- edge middleware
- server-side data aggregation
- database-backed content caching

Live Preview still works in these setups, but only if preview context flows end to end.

### The core rule

Preview context must travel with the request.

That means:

- the live preview hash must reach the layer that fetches content
- preview vs delivery must be decided centrally

### BFF pattern

In a BFF architecture:

- frontend calls your backend
- backend calls Contentstack

In preview mode:

- frontend forwards live preview params
- backend detects preview context
- backend uses preview api
- backend disables caching

Never let the frontend “guess” preview state if the backend owns content fetching.

### Middleware considerations

If you use middleware:

- do not strip preview query params
- do not normalize urls in preview mode
- do not apply redirects that drop the hash

Middleware bugs are one of the most common causes of preview mysteriously breaking on navigation.

### Database-backed caching

If you store content in a database:

- preview content must never be persisted
- preview responses must bypass storage

Treat preview data as radioactive.

Persisting it almost always leads to leaks or stale editor views.

### Summary

Live Preview is compatible with complex architectures, but only if preview context is treated as first-class request state and never cached or persisted.

---

## 17. Edit tags explained

Edit tags are the bridge between rendered html and CMS fields.

They exist so Contentstack can answer a simple question:

“When the editor clicks this thing on the page, which field should open?”

### What edit tags are

Edit tags are data attributes injected into your rendered markup.

They encode:

- entry uid
- content type uid
- locale
- field path

This metadata allows Contentstack to map DOM elements back to entry fields.

### What edit tags are not

Edit tags are not:

- visual overlays
- Live Preview itself
- required for preview rendering

You can have Live Preview without edit tags. You cannot have Visual Builder without them.

### Why edit tags must be explicit

The CMS has no knowledge of your frontend component structure.

Without edit tags:

- a heading is just a heading
- a button is just a button

Edit tags give semantic meaning to rendered output.

### Field paths matter

Edit tags must reference the correct field path.

For nested fields, modular blocks, and references, this path can be non-trivial.

Incorrect paths result in:

- clicks opening the wrong field
- clicks doing nothing

Accuracy matters more than coverage.

---

## 18. Applying edit tags in real apps

In real applications, content is rarely flat.

You will encounter:

- modular blocks
- nested groups
- references
- repeated components

### A practical strategy

Treat edit tags as part of your rendering contract.

Good patterns:

- central helper functions for generating edit tags
- consistent mapping from content model → component props
- explicit handling of arrays and indexes

Bad patterns:

- scattering string literals
- guessing field paths
- applying tags conditionally without reasoning

### Granularity trade-offs

You do not need edit tags everywhere.

Focus on:

- major content blocks
- text editors care about
- elements that should be clickable

Too many tags can be as confusing as too few.

### Testing edit tags

The easiest way to test edit tags:

- open Live Preview
- enable Visual Builder
- click around aggressively

If clicks feel intuitive, your tagging is good.

---

## 19. Visual builder architecture

Visual Builder is built entirely on top of Live Preview and edit tags.

It does not introduce a new preview system.

### How Visual Builder works

At a high level:

- the page renders with edit tags
- the CMS scans the DOM
- clickable regions are detected
- interactions are mapped back to fields

All data fetching still happens through Live Preview.

### Why Visual Builder breaks

Most Visual Builder issues come from:

- missing or incorrect edit tags
- dynamic rendering that changes DOM structure
- conditional components that appear only after client-side effects

If the CMS cannot find a stable element, it cannot bind editing.

### Designing for Visual Builder

If Visual Builder matters:

- prefer predictable DOM output
- avoid rendering critical content only after async effects
- keep field-to-component mapping obvious

Visual Builder rewards boring, explicit rendering.

---

## 20. Debugging, pitfalls, and best practices

Most Live Preview problems fall into a small number of categories.

### Common failure modes

- preview hash missing after navigation
- cached preview responses
- shared SDK instances in SSR
- middleware stripping query params
- refetching published content in preview

### A systematic debugging approach

Ask these questions in order:

1. Is the live preview hash present in the url?
2. Is the SDK initializing successfully?
3. Are change events firing?
4. Is the correct api being used?
5. Is caching disabled?
6. Is the rendered output actually using new data?

Answering these removes guesswork.

### Best practices

- treat preview as runtime state
- isolate preview logic
- prefer determinism over optimization
- document your preview architecture

### Final advice

Live Preview works best when it is boring.

If your implementation feels clever, it is probably fragile.

A predictable preview experience beats a fast one that lies.

---

