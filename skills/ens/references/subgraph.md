# ENS Subgraph

Querying ENS data at scale via The Graph — leaderboards, search, analytics, historical data — and the critical limitation every integrator hits.

**Guide**: [docs.ens.domains/web/subgraph](https://docs.ens.domains/web/subgraph).

## What it indexes

The official ENS subgraph indexes onchain events for second-level `.eth` domains and DNS-imported names. You can query:

- **Domains** — name, label, owner, resolver, parent, expiry, wrapped/unwrapped status.
- **Registrations** — when registered, by whom, expiry timestamps.
- **Resolvers** — addresses, text records, contenthashes set onchain.
- **Accounts** — domains owned by an address (onchain only).
- **WrappedDomains** — names held in the Name Wrapper, with their fuses.

GraphQL endpoint: hosted on The Graph; check the [ENS docs](https://docs.ens.domains/web/subgraph) for the current URL. Production usage requires a Graph API key.

## Use cases

- **Leaderboards** — "top 100 holders by domain count".
- **Search** — "all domains containing `vault`" (substring search via The Graph).
- **Analytics** — registrations over time, expiration cliffs, average holding period.
- **Bulk lookup** — "all names owned by `0xabcd…`" without N RPC calls.
- **Historical state** — "who owned `vitalik.eth` last year?".

## Critical limitation: offchain names are invisible

The subgraph indexes onchain events. **Offchain names (CCIP-Read synthesized) do not appear in the subgraph.** Examples that won't show up:

- `*.cb.id` (Coinbase)
- `*.uni.eth` (Uniswap)
- `*.linea.eth` (Linea)
- `*.base.eth` for L2 names like `jesse.base.eth` if registered offchain
- Any custom CCIP-Read namespace

Implications:

- A subgraph query for "names owned by X" misses every offchain name they've claimed.
- An "all names containing 'foo'" search misses synthesized names entirely.
- A leaderboard built solely from the subgraph undercounts users active in offchain spaces.
- For features that must include offchain names: you need to query resolvers directly per-name. There is no protocol-level enumeration of offchain spaces — that's by design (CCIP-Read names are dynamic and may be infinite).

## Real-time vs eventually-consistent

The subgraph is near-real-time but not transactional:

- New onchain events appear after a few blocks of indexing lag.
- During reorgs, the subgraph re-indexes — short-lived inconsistencies possible.
- **Don't use the subgraph for value-bearing lookups** (signing, sending). Resolve fresh from L1 via your library; see [Rule 3 in SKILL.md](../SKILL.md#the-five-rules-that-apply-to-every-ens-integration).

## Practical query pattern

```graphql
query NamesByOwner($owner: ID!) {
  account(id: $owner) {
    domains {
      name
      labelName
      createdAt
      expiryDate
      resolver { addr { id } }
    }
  }
}
```

Variables: `{ owner: "0xabc...".toLowerCase() }` (the subgraph stores addresses lowercased).

For text records:

```graphql
query Profile($node: ID!) {
  domain(id: $node) {
    name
    resolver {
      texts
      addr { id }
      contentHash
    }
  }
}
```

The `texts` field returns an array of *keys* that have been set; the values are not directly indexed (you query them onchain or via a per-key resolver call). This is intentional — text records can be arbitrary, and indexing every value would balloon the subgraph.

## Indexing alternatives

If the official subgraph doesn't fit your needs (different schema, custom aggregation, offchain name coverage):

- **Run a custom subgraph** — fork the [ENS subgraph repo](https://github.com/ensdomains/ens-subgraph) and modify.
- **Index resolver events yourself** — for offchain spaces, you'd need a custom indexer that observes the gateway, not the chain.
- **Hybrid: subgraph + onchain** — use the subgraph for the long tail of onchain names and supplement with per-name resolver calls for known offchain spaces.

## Footguns

1. **Treating subgraph results as authoritative for sends.** Use L1 resolution at signing time.
2. **Assuming offchain names appear.** They don't. Decide upfront whether your feature needs to cover them.
3. **Querying with a checksummed address.** The subgraph stores lowercased; lowercase your input.
4. **Forgetting indexing lag.** New registrations may take a minute or two to appear.
5. **Skipping a Graph API key in production.** Free tier rate-limits will bite under load.
6. **Treating the `texts` array as values.** It's just the keys; values still come from a resolver call (or the subgraph's `texts(key)` filter, depending on your exact query).

## Sources

- [docs.ens.domains/web/subgraph](https://docs.ens.domains/web/subgraph)
- [github.com/ensdomains/ens-subgraph](https://github.com/ensdomains/ens-subgraph)
- [The Graph](https://thegraph.com)
