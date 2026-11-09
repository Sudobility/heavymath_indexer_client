# @sudobility/heavymath_indexer_client

React and React Native compatible client library for the Heavymath prediction market indexer API. Provides a four-layer architecture (hooks, stores, business, network) with React Query integration, Zustand stores for optimistic updates, and SSE real-time subscriptions.

The library has **zero runtime dependencies** — everything it needs (React, React Query, Zustand, the shared type packages) is a peer dependency the consumer provides. HTTP transport is injected via a `NetworkClient` (from `@sudobility/types`), so the library runs unchanged in the browser, Node, or React Native.

## Installation

Packages in the `@sudobility` scope are published to **GitHub Packages**, not the public npm registry. The repo's `.npmrc` already points the scope there:

```
@sudobility:registry=https://npm.pkg.github.com
```

Installing (here or in a consumer) requires a GitHub token with `read:packages` in your environment / global `.npmrc`. Then:

```bash
bun add @sudobility/heavymath_indexer_client
```

### Peer Dependencies

```bash
bun add react @tanstack/react-query zustand @sudobility/types @sudobility/heavymath_types
```

## Usage

```typescript
import { IndexerClient, useActiveMarkets, useFavorites } from '@sudobility/heavymath_indexer_client';

// Create the client with an injected NetworkClient (e.g. webNetworkClient from @sudobility/di).
// Third argument `testMode` (default false) stamps client-generated timestamps with a fixed time.
const client = new IndexerClient('http://localhost:42069', networkClient);

// Use hooks in React components (requires a QueryClientProvider in the tree)
function Dashboard({ wallet }) {
  const { data: markets } = useActiveMarkets(client, 20);
  const { favorites, addFavorite, removeFavorite } = useFavorites(client, wallet);
  // ...
}
```

## Available Hooks

### Markets
`useMarkets`, `useActiveMarkets`, `useMarket`, `useMarketPredictions`, `useMarketHistory`, `useMarketDetails`

### Predictions
`usePredictions`, `useUserPredictions`, `useActiveBets`, `usePastBets`, `usePrediction`, `useUserBettingHistory`

### Dealers
`useDealers`, `useIsDealer`, `useDealerNFTs`, `useDealer`, `useDealerPermissions`, `useDealerMarkets`, `useDealerDashboard`

### Withdrawals / Oracle
`useWithdrawals`, `useDealerWithdrawals`, `useSystemWithdrawals`, `useMarketWithdrawals`, `useOracleRequests`, `useOracleRequest`, `useMarketOracle`

### Favorites (with Zustand optimistic updates)
`useFavorites`, `useCategoryFavorites`, `useIsFavorite`, `useFavoriteCounts`

### Auth (SIWE flow)
`useAuthNonce`, `useAuthVerify`, `useAuthSession`

### Discussions / Comments
`useDiscussion`, `useDiscussionComments`, `usePostComment`, `useDeleteComment`

### Sports (proxied through the indexer, per-sport + cross-sport search)
`useSportsProxy`, `useSportsSearch`, plus per-sport hooks for football, basketball, hockey, nfl, baseball, rugby, f1, mma, handball, volleyball (see `src/hooks/sports/`)

### Stadiums / Leaderboard
`useStadiums`, `useStadium`, `useLeaderboard`

### SSE (real-time)
`useSSE`, `useMarketUpdates`, `useAllMarketUpdates`, `useUserPredictionUpdates`

### Stats
`useMarketStats`, `useHealth`

SSE hooks take an endpoint URL as their first argument; all other hooks take an `IndexerClient`.

## Architecture

```
Hooks (React Query)  -->  Zustand Stores  -->  Business Layer (caching)  -->  Network Layer (HTTP)
```

- **Network layer** (`IndexerClient`): one method per REST endpoint, uses the injected `NetworkClient`.
- **Business layer** (`IndexerService`): high-level methods with in-memory TTL caching, for non-React callers.
- **Zustand stores** (`useFavoritesStore`, `useAuthStore`): local persistence + optimistic updates, used by the favorites/auth hooks.
- **Hooks**: React Query wrappers consumed by components.

## Development

```bash
bun install          # Install deps (needs a GitHub Packages read token; see Installation)
bun run build        # Compile TypeScript to dist/, then fix ESM import specifiers
bun run typecheck    # Type validation (tsc --noEmit)
bun run test:run     # Run unit tests once (Vitest)
bun run test:watch   # Run unit tests in watch mode
bun run test:coverage # Unit tests with v8 coverage
bun run lint         # ESLint check
bun run format       # Prettier write
bun run check-all    # lint + typecheck + unit tests
```

Unit tests live in `src/__tests__/`; integration tests (run with `bun run test:integration`) live in `src/__integration__/`.

## Related Packages

- `@sudobility/types` -- base types and the `NetworkClient` interface
- `@sudobility/heavymath_types` -- shared domain type definitions
- `@sudobility/di` -- `NetworkClient` implementations and dependency injection
- `heavymath_indexer` -- the backend indexer this client connects to

## License

BUSL-1.1
