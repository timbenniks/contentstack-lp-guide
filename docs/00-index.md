# The Ultimate Guide to Contentstack Live Preview

**Everything you need to build bulletproof real-time preview experiences.**

Live Preview creates a continuous feedback loop where editors see their changes materialize instantly. This guide gives you the mental models and practical knowledge to implement it correctly, debug it when things break, and adapt it to any rendering strategy.

## What You'll Build

This guide uses the [Contentstack Next.js Kickstart](https://github.com/contentstack/kickstart-next) as a running example. For a variant that routes all content through a middleware API layer (keeping tokens server-side), see the [Next.js Middleware Kickstart](https://github.com/contentstack/kickstart-next-middleware). By the end, you'll understand every line across these four files.

`**lib/contentstack.ts`** configures the SDK, initializes Live Preview, and fetches content:

```typescript
import contentstack, { QueryOperation } from "@contentstack/delivery-sdk";
import ContentstackLivePreview, { IStackSdk } from "@contentstack/live-preview-utils";

// Delivery SDK instance: when the live preview hash is present, requests go to the Preview API (drafts).
export const stack = contentstack.stack({
  apiKey: process.env.NEXT_PUBLIC_CONTENTSTACK_API_KEY as string,
  deliveryToken: process.env.NEXT_PUBLIC_CONTENTSTACK_DELIVERY_TOKEN as string,
  environment: process.env.NEXT_PUBLIC_CONTENTSTACK_ENVIRONMENT as string,
  live_preview: {
    enable: true,
    preview_token: process.env.NEXT_PUBLIC_CONTENTSTACK_PREVIEW_TOKEN,
    host: "rest-preview.contentstack.com",
  },
});

// Run in the browser only. Wires the iframe session to your stack and enables Visual Builder chrome.
export function initLivePreview() {
  ContentstackLivePreview.init({
    ssr: false, // CSR: refetch in place instead of full reloads
    mode: "builder",
    stackSdk: stack.config as IStackSdk, // So the SDK can inject hash + content type context
    stackDetails: { apiKey: "...", environment: "..." },
    editButton: { enable: true },
  });
}

export async function getPage(url: string) {
  const result = await stack
    .contentType("page").entry().query()
    .where("url", QueryOperation.EQUALS, url)
    .find();
  const entry = result.entries?.[0];
  // Populates `entry.$` with data-cslp props for click-to-edit in preview
  if (entry) contentstack.Utils.addEditableTags(entry, "page", true);
  return entry;
}
```

`**components/Preview.tsx**` runs the Live Preview loop, refetching content on every editor change:

```typescript
"use client";
import { useState, useEffect, useCallback } from "react";
import ContentstackLivePreview from "@contentstack/live-preview-utils";
import { getPage, initLivePreview } from "@/lib/contentstack";
import Page from "./Page";

export default function Preview({ path }: { path: string }) {
  const [page, setPage] = useState();
  const getContent = useCallback(async () => {
    setPage(await getPage(path)); // Preview API returns latest draft for current hash
  }, [path]);

  useEffect(() => {
    initLivePreview();
    // Fires on every editor change in the stack UI → refetch → re-render
    ContentstackLivePreview.onEntryChange(getContent);
  }, [path]);

  if (!page) return <p>Loading...</p>;
  return <Page page={page} />;
}
```

`**components/Page.tsx**` renders content with edit tags so editors can click any element to jump to its field:

```tsx
import { VB_EmptyBlockParentClass } from "@contentstack/live-preview-utils";

export default function ContentDisplay({ page }: { page: Page | undefined }) {
  return (
    <main>
      {/* Spreads add data-cslp (and related attrs) so Visual Builder can target each field */}
      <h1 {...page.$?.title}>{page?.title}</h1>

      {/* VB_EmptyBlockParentClass marks the modular-blocks container when empty so VB can add blocks */}
      <div className={`blocks ${VB_EmptyBlockParentClass}`} {...page.$?.blocks}>
        {page?.blocks?.map((item, index) => (
          <section key={index} {...page.$?.[`blocks__${index}`]}>
            <h2 {...item.block.$?.title}>{item.block.title}</h2>
          </section>
        ))}
      </div>
    </main>
  );
}
```

`**app/page.tsx**` routes between preview and production:

```typescript
import { getPage, isPreview } from "@/lib/contentstack";
import Page from "@/components/Page";
import Preview from "@/components/Preview";

export default async function Home() {
  // Preview session: client wrapper subscribes to Live Preview; production: one server fetch
  if (isPreview) return <Preview path="/" />;
  const page = await getPage("/");
  return <Page page={page} />;
}
```

The first file connects to Contentstack and sets up preview credentials. The second subscribes to editor changes and refetches on every keystroke. The third renders content with `data-cslp` edit tags and `VB_EmptyBlockParentClass` for empty modular blocks. The fourth decides which rendering path to use.

## Who This Guide Is For

- **Frontend developers** implementing Live Preview across CSR, SSR, or SSG architectures
- **Full-stack engineers** working with middleware, BFFs, and database-backed content systems
- **Technical architects** evaluating Live Preview for enterprise deployments
- **Anyone debugging** a Live Preview issue who wants to isolate problems systematically

## Which Rendering Strategy Are You Using?

Your rendering strategy determines how Live Preview operates. Pick your chapter:


| Strategy | How it handles "show me the new state"                 | SDK config   | Chapter                                                                       |
| -------- | ------------------------------------------------------ | ------------ | ----------------------------------------------------------------------------- |
| **CSR**  | Refetch data in the browser, re-render in place        | `ssr: false` | [Client-Side Rendering](./02-Client-Side%20Rendering.md)                         |
| **SSR**  | Reload the iframe, re-render on the server             | `ssr: true`  | [Server-Side Rendering](./03-Live%20Preview%20with%20Server-Side%20Rendering.md) |
| **SSG**  | Bypass static output via preview mode, behave like SSR | `ssr: true`  | [Static Site Generation](./04-Static%20Site%20Generation%20and%20Preview.md)     |


**Not sure which you're using?**

- **CSR**: Initial load shows a spinner while content fetches. Navigation doesn't trigger full page reloads. Content is fetched in `useEffect` or `mounted()`.
- **SSR**: View source shows real content. Your framework uses `getServerSideProps`, `asyncData`, or loaders. Content is fetched before HTML is sent.
- **SSG**: Pages are built during `npm run build`. Content updates require a rebuild and redeploy.

**Hybrid architectures** (e.g., Next.js mixing strategies): each page follows one strategy. Configure Live Preview based on what that specific page uses.

## Guide Structure

### Foundations

- [1. How Live Preview Works](./01-How%20Live%20Preview%20Works.md) - The mental model, architecture, session lifecycle, and APIs

### Rendering Strategies

- [2. Client-Side Rendering](./02-Client-Side%20Rendering.md)  - SDK setup, subscriptions, and refetch patterns for SPAs
- [3. Server-Side Rendering](./03-Live%20Preview%20with%20Server-Side%20Rendering.md)  - Reload-based preview, request-scoped clients, hash propagation
- [4. Static Site Generation](./04-Static%20Site%20Generation%20and%20Preview.md)  - Preview mode, framework escape hatches
- [5. Middleware and Complex Architectures](./05-Middleware%20and%20Database-Backed%20Architectures.md)  - BFF, edge middleware, database caching

### Advanced Features

- [6. Edit Tags and Visual Builder](./06-Edit%20Tags%20and%20Visual%20Builder.md)  - Click-to-edit, field paths, Visual Builder integration

### Operations

- [7. Debugging and Best Practices](./07-Debugging%2C%20Pitfalls%2C%20and%20Best%20Practices.md)  - Systematic debugging, common pitfalls, checklists

## Prerequisites

Before diving in, you should have:

- A working Contentstack stack with content types and entries
- A frontend application (any framework) that fetches and renders Contentstack content
- Basic familiarity with your framework's data fetching and rendering
- Access to your stack's Settings to enable Live Preview and generate preview tokens

### Enabling Live Preview in Your Stack

Enable Live Preview in your stack settings, set a default preview environment, and add base URLs for each locale in your environment configuration. For step-by-step instructions, see the [official setup guide](https://www.contentstack.com/docs/developers/set-up-live-preview/set-up-live-preview-for-your-website).

Optionally, enable the "Display Setup Status" toggle for real-time configuration feedback during setup. Enable "Always Open in New Tab" if you run SDK v4.0.0+ to preview outside the iframe.

## What's Next

You now have the prerequisites in place and basic configuration done. Next, lets take a quick deep dive into how Live Preview works and how it works with different rendering strategies. Proceed to [How Live Preview Works](./01-How%20Live%20Preview%20Works.md) to get started.