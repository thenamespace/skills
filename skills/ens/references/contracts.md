# Smart Contracts & Subgraph (Scenario 4)

Naming a deployed contract, querying ENS data at scale, key contract addresses, and ENSIP-24 arbitrary data.

## Contents

- [Naming a smart contract](#naming-a-smart-contract)
- [Key contract addresses](#key-contract-addresses)
- [ENS Subgraph](#ens-subgraph)
- [ENSIP-24: arbitrary bytes records](#ensip-24-arbitrary-bytes-records)
- [Universal Resolver, in depth](#universal-resolver-in-depth)
- [Footguns](#footguns)

---

## Naming a smart contract

**Guide**: [docs.ens.domains/web/naming-contracts](https://docs.ens.domains/web/naming-contracts).

When a contract has a primary ENS name, Etherscan and explorers display the name instead of the raw address, dapps can identify it, and your users see a meaningful label.

The mechanism is the same as for EOAs: set a reverse record on the Reverse Registrar from the contract's address. Three patterns:

### 1. Ownable pattern (recommended for upgradeable / owned contracts)

```solidity
import "@openzeppelin/contracts/access/Ownable.sol";

interface IReverseRegistrar {
    function setNameForAddr(address addr, address owner, address resolver, string calldata name)
        external returns (bytes32);
}

contract MyContract is Ownable {
    function setEnsName(address reverseRegistrar, address resolver, string calldata name) external onlyOwner {
        IReverseRegistrar(reverseRegistrar).setNameForAddr(address(this), owner(), resolver, name);
    }
}
```

Lets the owner update the name later (e.g., after a rebrand).

### 2. Constructor pattern (immutable contracts)

```solidity
constructor(address reverseRegistrar, string memory name) {
    IReverseRegistrar(reverseRegistrar).setName(name);
}
```

The contract claims its name at deployment. Cannot be changed after.

### 3. ReverseClaimer module

ENS provides `ReverseClaimer` (mainnet only) — inherit it and pass the owner who'll control the reverse node:

```solidity
import "@ensdomains/ens-contracts/contracts/reverseRegistrar/ReverseClaimer.sol";

contract MyContract is ReverseClaimer {
    constructor(ENS ens, address claimant) ReverseClaimer(ens, claimant) {}
}
```

After deployment, `claimant` can call the Reverse Registrar to set the name.

### Verify after deployment

Reverse-resolve the contract's address and confirm the name forwards back to it. Same forward-verification rule as for users (see [profile.md](profile.md#reverse-resolution)).

## Key contract addresses

These addresses change across deployments and per-chain. **Always look them up at integration time** rather than hardcoding:

- **Live deployments**: [docs.ens.domains/learn/deployments](https://docs.ens.domains/learn/deployments)
- **Source contracts**: [github.com/ensdomains/ens-contracts](https://github.com/ensdomains/ens-contracts)

Roles you'll encounter:

| Contract | Role |
|---|---|
| **ENS Registry** | The root contract. Maps namehash → owner, resolver, TTL. |
| **Public Resolver** | The default resolver most names point at; supports addr, multicoin addr, text, contenthash, ABI. |
| **Reverse Registrar** | Per chain. Maps address → primary name. Each L2 has its own deployment. |
| **Universal Resolver** | Aggregates registry + resolver + CCIP-Read into a single call. Multiple versions deployed during the ENSv2 transition. |
| **DNS Registrar** | Handles ENSIP-6 DNS imports (DNSSEC verification). |
| **`.eth` Registrar Controller** | The contract for registering/renewing `.eth` second-level names. |

If you write contracts that integrate ENS, take addresses from a config file (or use the `ens-contracts` package's deployment exports), so you can swap them per network and chain without code changes.

## ENS Subgraph

**Guide**: [docs.ens.domains/web/subgraph](https://docs.ens.domains/web/subgraph).

The official ENS subgraph (powered by The Graph) indexes onchain ENS events and exposes a GraphQL endpoint. Useful for:

- Leaderboards and search ("top 100 holders by domain count")
- Historical analytics (registrations over time, expirations)
- Bulk lookup ("all names owned by 0xabcd…")

Schema highlights: `Domain`, `Registration`, `Resolver`, `WrappedDomain`, `Account`, plus per-record entities.

### Critical limitation: offchain names

The subgraph indexes onchain events. Offchain names (CCIP-Read synthesized, e.g., `*.cb.id`, `*.uni.eth`, `*.linea.eth`) **do not appear in the subgraph** — they only exist at resolution time. Implications:

- A subgraph query for "names owned by X" will miss every offchain name they've claimed.
- An "all names containing 'foo'" search misses synthesized names entirely.
- For features that must include offchain names, query resolvers directly (per-name) — there is no protocol-level enumeration of offchain spaces.

### Real-time vs eventually-consistent

The subgraph is near-real-time but not transactional. Don't use it for settlement-critical lookups (signing, sending). For those, query the chain directly via your library.

### Practical pattern

```ts
const NAMES_BY_OWNER = `
  query NamesByOwner($owner: ID!) {
    account(id: $owner) {
      domains { name labelName createdAt expiryDate }
    }
  }
`
// Submit to the official ENS subgraph endpoint (see ENS docs for the current URL — requires The Graph API key for production).
```

## ENSIP-24: arbitrary bytes records

**Spec**: [ENSIP-24](https://docs.ens.domains/ensip/24).

Adds a generic `data(bytes32 node, bytes32 key) → bytes` field to resolvers. It's the binary analog of text records.

Use cases:
- Hashed commitments (proof-of-presence, voting receipts).
- Interoperable address formats not covered by ENSIP-9/-11.
- App-specific binary metadata (signatures, encrypted blobs).

When NOT to use: if your data is human-readable, a text record (ENSIP-5) is simpler and indexable in the subgraph.

## Universal Resolver, in depth

The Universal Resolver wraps:
- Registry lookup (find the resolver for a namehash).
- Resolver call (the actual `addr` / `text` / `contenthash` query).
- ENSIP-10 wildcard fallback (walk up the tree until a wildcard resolver is found).
- CCIP-Read (handle the gateway round-trip transparently).
- Reverse-then-forward verification (for `reverse(addr)`).
- Multicall (`resolve(name, data[])` for multiple records on one name in one tx).

This is why **manual `registry.resolver(node) → resolver.addr(node)` is wrong** in modern ENS code — it skips wildcard, CCIP-Read, and verification. Use the library helper that wraps the Universal Resolver.

If you must call it directly (e.g., from a smart contract): grab the address from the deployments page, ABI from `ens-contracts`, and use `resolve(bytes name, bytes data)` with a DNS-encoded name.

## Footguns

1. **Hardcoded addresses.** Multiple Universal Resolver versions exist; per-chain addresses differ. Always lookup.
2. **Treating subgraph results as authoritative.** Misses offchain names and is eventually-consistent.
3. **Naming a contract once, never verifying.** Forward-resolve afterward to confirm it works.
4. **Reverse Registrar mismatch across chains.** Each L2 has its own; using the mainnet one from an L2 contract won't set the L2 primary name.
5. **Using `data(node, key)` for things text records cover.** Less interop, less indexing.
6. **Calling the registry/resolver directly in a dapp.** Use the library helper that wraps the Universal Resolver — it's not just convenience, it's correctness.

## Sources

- [docs.ens.domains/web/naming-contracts](https://docs.ens.domains/web/naming-contracts)
- [docs.ens.domains/web/subgraph](https://docs.ens.domains/web/subgraph)
- [docs.ens.domains/learn/deployments](https://docs.ens.domains/learn/deployments)
- [docs.ens.domains/resolvers/universal](https://docs.ens.domains/resolvers/universal)
- [ENSIP-24 — Arbitrary Data Resolver](https://docs.ens.domains/ensip/24)
- [github.com/ensdomains/ens-contracts](https://github.com/ensdomains/ens-contracts)
