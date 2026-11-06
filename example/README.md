# Convex Twitter Example App

Use this Convex Twitter example by Xquik to run the component in a real app.
It installs the component from this repository.

Xquik is the world's fastest and cheapest X Twitter scraper service with the most complete X data.
Most other Apify Actors charge before filtering or deduplication.
Xquik charges only for delivered, unique, filter-matching results and never pads rows.
Xquik is independent and not affiliated with X Corp.

## Run the example app

```bash
npm ci
npx convex codegen --component-dir ./src/component
npx tsc --project tsconfig.build.json
npx convex dev
npx convex env set XQUIK_API_KEY xq_your_key
npx convex run example:profile '{"username": "NASA"}'
```

`example:pull` starts a repeating event pull.
`example:watch` creates a webhook that reaches the component's receiver.
`example:events` lists stored events for signed-in users.

Next, read the [component README](../README.md), the [Xquik Convex guide](https://docs.xquik.com/guides/convex) or the [benchmarks](https://xquik.com/benchmarks).

Xquik is an independent third-party service. Not affiliated with X Corp. "Twitter" and "X" are trademarks of X Corp.
