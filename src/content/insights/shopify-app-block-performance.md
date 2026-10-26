---
title: "Shopify App Block Performance: Benchmark Every Block Separately"
description: "Shopify app block performance varies wildly per block. Here is how to isolate and measure each theme app extension block so you fix the one that costs you."
primaryKeyword: "Shopify app block performance"
secondaryKeywords:
  - "theme app extension block speed"
  - "benchmark Shopify app blocks"
  - "Shopify app embed block performance"
  - "measure Shopify theme app extension"
  - "Shopify app block JavaScript weight"
category: performance
publishedDate: 2026-10-26
readingTime: 8
---

Two app blocks from the same vendor, sitting in the same theme, can differ in cost by a factor of ten, and nothing in the theme editor tells you which one is which.

## Key takeaways

- Theme app extensions are safer than script tags as a category, but individual blocks inside one extension vary enormously.
- App embed blocks are the expensive ones. They load on every page regardless of template, while app blocks load only where a merchant places them.
- Measure blocks one at a time by toggling a single block in the theme editor and re-recording. Nothing else gives you clean attribution.
- In our audit work a well-built block adds 5 to 20 KB and under 20 ms of main thread time. A badly built one adds 150 KB and blocks the main thread for 200 ms or more.
- The number that matters is main thread time on a throttled mobile profile, not transfer size.

## App blocks and app embeds are not the same thing

Both arrive through a theme app extension, and merchants routinely treat them as one feature. They behave differently in the only way that counts.

An **app block** is placed inside a section through the theme editor. It renders where the merchant puts it, on the templates the merchant chooses, and it does not exist on any other page. A product-page size chart is an app block. If a shopper never loads a product page, they never pay for it.

An **app embed block** is toggled on globally in the theme editor's App embeds panel. Shopify injects it into `theme.liquid`, which means every page: homepage, collection, cart, blog, 404. A consent banner, a chat widget, and an analytics wrapper are all app embeds. This is the category where costs accumulate quietly, because there is no page on the store that escapes them.

The distinction matters because the fix differs. A slow app block is usually a placement problem you can solve by moving it below the fold or removing it from templates that do not need it. A slow app embed is usually a vendor problem, and your only levers are configuration or replacement.

This is the layer of nuance missing from the broader comparison in our piece on [theme app extensions versus script tags](/insights/theme-app-extensions-vs-script-tags). Choosing an app that uses TAE is the right first filter. It is not the last one.

## Benchmarking one block at a time

The theme editor gives you the isolation mechanism for free. Use it.

### The isolation method

Duplicate your live theme first. Never do this on the published one.

1. In the duplicated theme, turn off every app embed and remove every app block. Record a baseline.
2. Enable exactly one block. Record again.
3. Turn it back off, enable the next one, record again.
4. Repeat until every block has its own number.

The delta between each recording and the baseline is that block's cost, measured on your theme, with your content, at your product counts. Vendor benchmarks are run on empty demo stores and are close to meaningless as a result.

For app embeds you can shortcut step 2. Most vendors expose a settings toggle that disables the embed without uninstalling the app, so you keep the configuration while removing the code.

### What to record

Load a representative page in Chrome DevTools with the Performance panel open, CPU throttling set to 4x, and network throttling set to Slow 4G. Reload and capture. From each recording, write down four numbers:

- **Total Blocking Time.** The headline number. It is the sum of the portion of every long task beyond 50 ms.
- **Script evaluation time attributed to the block's file.** Expand the bottom-up view and filter by the vendor's CDN domain.
- **Transfer size.** From the Network panel, filtered the same way.
- **Number of requests.** A block that makes six requests to fetch a config, a locale file, and a font is worse than its byte count suggests.

Transfer size is the one everybody quotes and the one that misleads most. A 40 KB JavaScript bundle that parses, builds a virtual DOM, and mounts a widget costs far more than a 90 KB image. Judge blocks by main thread time.

A quick way to see what a block pulls in before you record anything:

```bash
curl -s https://your-store.myshopify.com/products/any-product \
  | grep -oE 'https://cdn\.shopify\.com/extensions/[^"]+' \
  | sort -u
```

Extension assets are served from that path, so this gives you a list of every extension asset on the page, which you can then map back to vendors.

## The block patterns that cost the most

After enough audits the same three shapes keep appearing.

### Blocks that render nothing until JavaScript runs

The whole point of a theme app extension is server-rendered Liquid. A block that outputs an empty `<div id="app-root">` and then mounts a client-side widget into it has thrown that advantage away and reinvented the script tag with extra steps.

You can spot these in the page source. If the block's markup in the raw HTML response is a single empty container, the content is client-rendered. That means it cannot be the LCP element, it will almost certainly cause layout shift, and it adds parse and mount work to the critical path. Review widgets and recommendation carousels are the usual offenders, and the mitigation is the same as for [review widgets specifically](/insights/lazy-load-shopify-review-widgets): reserve the space and defer the mount.

### Blocks that fetch on every page

An app embed that calls its own API on load, on every template, is paying a full network round trip for data that rarely changes. Loyalty widgets fetching point balances, currency converters fetching rates, and geolocation banners fetching a country lookup all do this.

Look for the pattern in the Network panel: a request to the vendor's domain that fires before any user interaction, on a page where the widget is not even visible. A well-built version of the same feature caches the response, defers the call until the widget enters the viewport, or renders server side from a Liquid metafield.

### Blocks that ship their own framework

Two apps each bundling their own copy of a UI framework is 90 KB of duplicated runtime that neither vendor knows about. Nobody is doing anything wrong individually, and the store still pays twice.

You cannot fix this from the theme. What you can do is count it during vendor selection, which is why per-app attribution matters more than a single overall score. If you would rather not bisect blocks by hand, you can [find the slow apps by name](/find-slow-shopify-apps) and start from the ranked list instead.

## Setting a per-block budget

Numbers only change behaviour when there is a threshold attached to them.

A workable budget for most stores, from what we consistently observe:

| Block type | JavaScript | Main thread | Requests |
|---|---|---|---|
| App embed (every page) | under 25 KB | under 30 ms | 1 |
| Above-fold app block | under 40 KB | under 50 ms | 2 |
| Below-fold app block | under 60 KB, deferred | under 50 ms | 2 |

Anything over those figures needs a justification tied to revenue. A subscription widget on the product page earning real money can spend more than a badge that nobody clicks.

Apply the budget at install time, not at audit time. The team at [Defyn Digital](https://defyn.com.au) benchmarks a candidate app on a duplicated theme before it ever reaches production, which takes about twenty minutes and avoids the far more awkward conversation about removing an app the merchandising team has already built a campaign around.

## Common questions

### Can I lazy load an app block myself?

Sometimes. If the block renders its own markup and only hydrates behaviour, you can wrap it in a container and defer the vendor's script until it scrolls into view. If the block is fully client-rendered, deferring it just moves the layout shift further down the page. Check the raw HTML first, and never edit files inside the extension itself, because they are vendor-owned and overwritten on every update.

### Why does one block slow down pages where it is not placed?

That is an app embed, not an app block, and it is on every page by definition. Check the App embeds panel in the theme editor. This is the single most common misdiagnosis we see, and it usually resolves in one click once the merchant knows the panel exists.

### Does removing a block remove its code entirely?

For app blocks and app embeds, yes. That is one of the genuine advantages over the legacy ScriptTag API. Removing the block removes its assets from the page immediately. Uninstalling the app removes the extension. What it does not remove is any data the vendor already holds, which is a separate question.

## Wrap-up

Treat every block as its own line item. Duplicate the theme, disable everything, enable one block at a time, and record Total Blocking Time on a 4x throttled mobile profile. Then sort by main thread cost rather than file size, fix or replace anything above the budget, and pay closest attention to app embeds because they charge you on every page view. If the blocks turn out to be clean, the bottleneck is somewhere else and the [seven mobile speed fixes](/insights/shopify-mobile-speed-7-fixes) are the better starting point. To skip the manual bisection entirely, [see how the audit works](/how-it-works).
