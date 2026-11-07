# Twitter API & X Data in Convex with Xquik

Use this Convex Twitter component by Xquik to read X data & receive webhooks.
It adds typed actions, signed webhook delivery & live event queries to your app.

Xquik is the world's fastest and cheapest X Twitter scraper service with the most complete X data.
Most other Apify Actors charge before filtering or deduplication.
Xquik charges only for delivered, unique, filter-matching results and never pads rows.
Xquik is independent and not affiliated with X Corp.

<!-- trust:win:start -->

The Xquik REST API read 45 tweets per second at $0.15 per 1,000 on 2026-10-15. The fastest of 4 rival Actors read 12.5 per second. The cheapest cost $0.173 per 1,000. [Compare each run](https://xquik.com/benchmarks#benchmark-x-tweet-scraper).

<!-- trust:win:end -->

<!-- START: Include on https://convex.dev/components -->

## Install the Convex Twitter component

[![Convex Component](https://www.convex.dev/components/badge/convex-xquik)](https://www.convex.dev/components/convex-xquik)

```bash
npm install xquik-convex
```

Add the component in `convex/convex.config.ts`.

```ts
import { defineApp } from "convex/server";
import { v } from "convex/values";
import xquik from "xquik-convex/convex.config.js";

const app = defineApp({ env: { XQUIK_API_KEY: v.optional(v.string()) } });
app.use(xquik, { httpPrefix: "/xquik/", env: { XQUIK_API_KEY: app.env.XQUIK_API_KEY } });
export default app;
```

Set your key from the [Xquik dashboard](https://xquik.com/dashboard).
Every component function answers `missing_api_key` until you set it.

```bash
npx convex env set XQUIK_API_KEY xq_your_key
```

## Call the Twitter API from TypeScript in Convex

The component has 190 typed actions, 1 per public operation in the [API reference](https://docs.xquik.com/api-reference/overview).
Each action takes the operation's path, query, header & body fields side by side.
Convex checks the arguments & the result of each call.

```ts
import { v } from "convex/values";
import { components } from "./_generated/api";
import { internalAction } from "./_generated/server";

export const latest = internalAction({
  args: { username: v.string() },
  handler: async (ctx, { username }) =>
    await ctx.runAction(components.xquik.operations.searchTweets, { q: `from:${username}`, limit: 20 }),
});
```

Next, read the [Xquik Convex guide](https://docs.xquik.com/guides/convex), the [webhook steps](#receive-twitter-webhook-events) or the [live event query](#show-events-live).

<!-- trust:start -->

## Benchmarks & usage

[![Twitter API p99 server read time](https://img.shields.io/endpoint?url=https%3A%2F%2Fxquik.com%2Fbadges%2Frest-p99.json)](https://xquik.com/benchmarks#speed)
[![Twitter API package downloads](https://img.shields.io/endpoint?url=https%3A%2F%2Fxquik.com%2Fbadges%2Fdownloads.json)](https://xquik.com/benchmarks#downloads)

- 6.4 s p99 REST API server read time over 7 days, failed reads included ([source](https://xquik.com/benchmarks#speed)) <!-- trust:rest-p99 6317 -->
- 63,000+ downloads of Xquik's Twitter API packages on npm, PyPI, RubyGems, NuGet, Packagist & Terraform ([source](https://xquik.com/benchmarks#downloads)) <!-- trust:downloads 63178 -->

Checked on 2026-11-06. See each number with its date & source on [xquik.com/benchmarks](https://xquik.com/benchmarks) or as [JSON](https://xquik.com/trust.json).

<!-- trust:end -->

## Read results

Call `components.xquik.operations.getUser` with `{ id: "elonmusk" }` for a profile.
A failed call throws a `ConvexError`.
Its data has `status`, `error`, `message` & `retryAfter` in seconds for a 429.

Each result has the type & validator of its API reference schema.
The `name` of a `getUser` result is a `string`.
The result holds only the fields that schema lists.
A JSON-LD key such as `@id` arrives as `id`.
Import `OperationResult<"getUser">` from `xquik-convex` to name a result type.

## Receive Twitter webhook events

```ts
import { Xquik, eventFields } from "xquik-convex";
import { v } from "convex/values";
import { components, internal } from "./_generated/api";
import { internalAction, internalMutation } from "./_generated/server";

const xquik = new Xquik(components.xquik);

export const watch = internalAction({
  args: {},
  handler: async (ctx) =>
    await xquik.createWebhook(ctx, { eventTypes: ["tweet.new", "tweet.reply"], onEvent: internal.x.onEvent }),
});

export const onEvent = internalMutation({
  args: { event: eventFields },
  handler: async (ctx, { event }) => {
    console.info(event.eventType, event.streamEventId);
  },
});
```

The component creates the Xquik webhook at `https://<deployment>.convex.site/xquik/webhooks/<token>`.
It keeps the signing secret in its own table.
Each delivery must carry a valid `X-Xquik-Signature` & a timestamp within 5 minutes.
A replayed nonce gets a 401.
The component stores a retry with the same `streamEventId` once.
Your handler runs once per endpoint.
The `webhook.test` alert gets a 200 & sets `testedAt` on the endpoint.

To add a webhook made elsewhere, call `xquik.addWebhook(ctx, { secret })` from a mutation.
Paste the URL it returns.

## Show events live

```ts
import { paginationOptsValidator } from "convex/server";
import { components } from "./_generated/api";
import { query } from "./_generated/server";

export const events = query({
  args: { paginationOpts: paginationOptsValidator },
  handler: async (ctx, { paginationOpts }) => {
    if ((await ctx.auth.getUserIdentity()) === null) throw new Error("Sign in first.");
    return await ctx.runQuery(components.xquik.events.list, { paginationOpts });
  },
});
```

Use it with `usePaginatedQuery(api.x.events, {}, { initialNumItems: 20 })`.
Pass `filter: { eventType }` or `filter: { monitorId }` to narrow the list.

## Schedule calls

```ts
await xquik.schedule(ctx, "listEvents", { limit: 100 }, { key: "events", everyMs: 5 * 60 * 1000 });
```

A call with the `key` of a live job returns that job's ID.
Reads make up to 5 attempts on a 429, 5xx or network error.
So do writes that take an `Idempotency-Key`, and the component fills that key.
A result that breaks its schema fails the job without a retry.
Scheduled `listEvents` jobs store their events with the webhook events.
Read a job with `components.xquik.jobs.get`.
Stop it with `components.xquik.jobs.cancel`.

## Keep credits safe

Component functions run only from your backend.
Wrap them in internal functions.
A public function that spends credits must check the caller's identity & rate limit.
The [example app](https://github.com/Xquik-dev/xquik-convex/blob/main/example/README.md) keeps every credit-spending function internal.

## Test your app

```ts
import { convexTest } from "convex-test";
import xquikTest from "xquik-convex/test";

const t = convexTest(schema, modules);
xquikTest.register(t);
```

<!-- END: Include on https://convex.dev/components -->

## Questions

### Do I need an X developer account?

No. Public reads need only an Xquik API key.
Private reads & X account actions need a connected X account.

### Is this the official X API?

No. Xquik runs its own REST API.
This component calls that API, not the official X API.

## Develop

```bash
npm ci
npm run generate
bin/check-local
```

`example/` holds a runnable Convex app that uses the component.
`npm run generate` rebuilds the operation registry from `openapi.json`.
The tests prove that every public operation in `openapi.json` is an action.
They also prove that every event type gets a receiver test.
See [CONTRIBUTING.md](CONTRIBUTING.md) for each check.

## Documentation

- [Xquik Convex guide](https://docs.xquik.com/guides/convex)
- [Xquik API reference](https://docs.xquik.com/api-reference/overview)
- [Webhook signature verification](https://docs.xquik.com/webhooks/verification)
- [Organization support policy](https://github.com/Xquik-dev/.github/blob/main/SUPPORT.md)
- [Security policy](SECURITY.md)
- [Contributing](CONTRIBUTING.md)

## More Xquik tools

- [X (Twitter) scraper API & Twitter MCP server](https://github.com/Xquik-dev/x-twitter-scraper) with REST, MCP & Skill setup.
- Twitter API SDKs for [TypeScript](https://github.com/Xquik-dev/x-twitter-scraper-typescript), [Python](https://github.com/Xquik-dev/x-twitter-scraper-python), [Go](https://github.com/Xquik-dev/x-twitter-scraper-go), [Ruby](https://github.com/Xquik-dev/x-twitter-scraper-ruby), [Java](https://github.com/Xquik-dev/x-twitter-scraper-java), [Kotlin](https://github.com/Xquik-dev/x-twitter-scraper-kotlin), [C#](https://github.com/Xquik-dev/x-twitter-scraper-csharp) & [PHP](https://github.com/Xquik-dev/x-twitter-scraper-php).
- [Twitter CLI & X CLI](https://github.com/Xquik-dev/x-twitter-scraper-cli) for terminal workflows.
- The [X API alternative skill](https://github.com/Xquik-dev/x-api) & [Twitter advanced search operators](https://github.com/Xquik-dev/x-twitter-advanced-search).
- Twitter agent plugins for [OpenClaw](https://github.com/Xquik-dev/tweetclaw) & [Hermes Agent](https://github.com/Xquik-dev/hermes-tweet), and the [Twitter API documentation source](https://github.com/Xquik-dev/xquik-docs).
- Twitter integrations for [n8n](https://github.com/Xquik-dev/n8n-nodes-xquik), [Prefect](https://github.com/Xquik-dev/prefect-xquik), [Haystack](https://github.com/Xquik-dev/xquik-haystack), [Paperclip](https://github.com/Xquik-dev/paperclip-plugin-xquik), [Terraform](https://github.com/Xquik-dev/terraform-provider-x-twitter-scraper), [Supabase](https://github.com/Xquik-dev/xquik-supabase) & [Strapi](https://github.com/Xquik-dev/strapi-plugin-xquik).
- Twitter MCP for [AI agents](https://github.com/Xquik-dev/xquik-mcp), [VS Code](https://github.com/Xquik-dev/xquik-vscode) & [Raycast](https://github.com/Xquik-dev/xquik-raycast).
- [Xquik X API & Twitter scraper](https://xquik.com) and the [Twitter API documentation](https://docs.xquik.com/api-reference/overview).

Xquik is an independent third-party service. Not affiliated with X Corp. "Twitter" and "X" are trademarks of X Corp.
