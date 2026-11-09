# CLAUDE.md — Heavymath Indexer Client

> **Git policy — never auto-commit or auto-push.** Leave work in the working tree.
> Run `git commit`/`git push`/`gh pr create` **only when the user asks in that turn**.
> This repo does **not** use feature branches — stay on the current branch.

## What this is

`@sudobility/heavymath_indexer_client` — React / React Native client library for the Heavymath
prediction market indexer REST API. Four layers: **hooks → stores → business → network**.
Published as an ESM package (`dist/`), **zero runtime dependencies** (all are peer deps).

- **Stack**: TypeScript 6, React Query 5, Zustand 5, Vitest 4, happy-dom.
- **Package manager**: **Bun** (never npm/yarn/pnpm).
- Source of truth for versions is `package.json`; do not hardcode them elsewhere.

## Commands (verified against package.json)

```bash
bun install            # needs a GitHub Packages read token (see .npmrc note below)
bun run build          # tsc -p tsconfig.build.json + node scripts/fix-esm-imports.mjs
bun run typecheck      # tsc --noEmit
bun run test:run       # unit tests once (alias: test:unit)
bun run test:watch     # unit tests, watch
bun run test:coverage  # v8 coverage
bun run test:integration   # integration tests (config + src/__integration__/ dir)
bun run lint           # eslint src
bun run lint:fix
bun run format         # prettier --write
bun run check-all      # lint + typecheck + test:unit  ← run before handing off
```

- **Do NOT run `bun test`** — that invokes Bun's native test runner, not Vitest. Use
  `bun run test:run` / `bun run test:watch`.
- Deps are not installed by default here; `node_modules` is often absent until `bun install`.

## Architecture

```
React Hooks (@tanstack/react-query)   src/hooks/      take IndexerClient as 1st arg
        │                                              (SSE hooks take an endpoint URL instead)
        ▼
Zustand Stores  src/stores/           favorites-store, auth-store — persist + optimistic updates
        ▼
Business Layer  src/business/         IndexerService — high-level methods with in-memory TTL cache
        ▼
Network Layer   src/network/          IndexerClient — one method per REST endpoint
```

- **Dependency injection**: `IndexerClient(endpointUrl, networkClient, testMode?)`. The library
  **never constructs a `NetworkClient`** — the consumer injects one (`@sudobility/di` on web, a
  custom RN impl, or a `vi.fn()` mock in tests). `testMode` (default false) only affects the
  fixed timestamp used for client-generated responses.
- Every layer has a barrel `index.ts`; `src/index.ts` re-exports all four plus `src/types.ts`.
  **New export → add it to the layer barrel** (and `src/index.ts` only for a new directory).

### Endpoint path gotchas (routes are NOT all REST-clean)

Methods map ~1:1 to endpoints, but several paths are non-obvious — check `IndexerClient.ts`
rather than guessing:
- `getMarkets` → `GET /api/markets/list` ; `getDealers` → `GET /api/dealers/list`
- `getLeaderboard` → `GET /api/wallets/leaderboard`
- Discussions: `getDiscussion` is query-based (`/api/discussions?subject_type=&sport=&subject_id=`);
  `postComment` → `POST /api/discussions/comments`; `deleteComment` → `DELETE /api/discussions/comments/:id`
  (comment write routes are flat, not nested under a discussion id).
- `checkMarketResolution` and `triggerResolve` **do not throw on 4xx** — they return a
  discriminated `{ ok: true | false }` result so the UI can show the reason. Everything else
  throws `handleApiError` on `!response.ok`.
- `triggerLock` / `triggerResolve` use a 120s timeout (on-chain calls via the resolver wallet).

## Types

- Domain types come from `@sudobility/heavymath_types`; response/network types from
  `@sudobility/types`. **Re-exported** through `src/types.ts` — import from the library, and
  **do not define a type locally if it exists in a shared package**.
- Locally-defined types (not in shared packages): the `*Filters` interfaces, `DiscussionQuery`,
  `MarketResolutionCheck`, `Leaderboard*`, and all **SSE types** (`SubscriptionChannel`,
  `SSEEventType`, `SSEMessage` union, event-data shapes).
- Deprecated aliases exist for backward compat (`Market`→`MarketData`, etc.) — don't use in new code.

## Conventions

- **Query keys**: hierarchical, `'heavymath'`-prefixed — `['heavymath', 'markets', filters]`.
- **staleTime** guideline: sports 30s · predictions/oracle/health 1m · markets/withdrawals 2m ·
  dealers/favorites/stats/leaderboard 5m · stadiums 24h.
- Hooks that take an optional id/address use `enabled: !!id` so they no-op while undefined.
- **URL building**: always `buildUrl(this.baseUrl, path)` + `encodeURIComponent` on path segments;
  build query strings with `URLSearchParams`.
- No `any` (prefer `unknown` + guards); strict mode is on. Add JSDoc to exported functions.
- Don't import `zustand` outside `src/stores/` — use the store's exported hook.

## Gotchas

- **Build needs the post-step.** `tsc` emits extensionless relative imports (`moduleResolution:
  bundler`); `scripts/fix-esm-imports.mjs` rewrites them to `./x.js` / `./dir/index.js` so Node's
  ESM loader can resolve them. `bun run build` runs it automatically — don't ship a raw `tsc` output.
- **Optimistic favorites use negative ids** (`id = -Date.now()`). Code that checks `id > 0` will
  drop pending items; test `id < 0` for "pending".
- Favorites store persists to `localStorage` key `heavymath-favorites` (no-op outside a browser;
  RN needs a custom storage adapter). Auth store key is `heavymath-auth`.
- **SSE uses the browser `EventSource`** — polyfill it in React Native / SSR.
- `IndexerService.getInstance(config)` is a singleton: only the first config wins; use
  `new IndexerService(config)` for a distinct instance. Call `clearCache()` for fresh data.
- All React Query hooks require a `QueryClientProvider` ancestor.

## CI/CD

`.github/workflows/ci-cd.yml` calls the reusable `johnqh/workflows/.../unified-cicd.yml@main` on
push/PR to `main`/`develop`. **Publishes to GitHub Packages** (`publish-target: github`; `.npmrc`
points the `@sudobility` scope at `npm.pkg.github.com`), secrets passed through. Installs/publishes
need `GH_PACKAGES_READ_TOKEN` / the relevant token in the environment.

## Ecosystem

```
heavymath_indexer (REST API) → heavymath_indexer_client (HERE) → heavymath_lib → heavymath_app
```
- `heavymath_lib` wraps the sports proxy + favorites hooks; `heavymath_app` consumes IndexerClient
  (via its IndexerContext), market/dealer/SSE/favorites hooks, and the type exports.
- When the indexer adds/changes an endpoint, update the matching `IndexerClient` method **and** hook here.
