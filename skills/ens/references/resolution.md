# Resolution: forward, multichain, CCIP-Read, ENSv2 readiness

How `name → address` actually works, plus the things that make it work across chains and offchain resolvers.

## Contents

- [How forward resolution works](#how-forward-resolution-works)
- [The Universal Resolver](#the-universal-resolver)
- [CCIP-Read](#ccip-read)
- [ENSv2 readiness](#ensv2-readiness)
- [Multichain coin types](#multichain-coin-types)
- [Batch resolution](#batch-resolution)
- [Footguns](#footguns)

---

## How forward resolution works

```
name "alice.eth"
        │
        ▼
  normalize (ENSIP-15)
        │
        ▼
  namehash → bytes32 node
        │
        ▼
  registry.resolver(node) → resolver address
        │
        ▼
  resolver.addr(node)            // Ethereum address
  resolver.addr(node, coinType)  // Multichain (ENSIP-9/-11)
  resolver.text(node, key)       // Text records (ENSIP-5)
  resolver.contenthash(node)     // Content hash (ENSIP-7)
        │
        ▼
  May revert with OffchainLookup → CCIP-Read flow
```

Resolution **always starts on Ethereum mainnet**, even if the dapp is on an L2. The registry, parent records, and most resolvers live on mainnet. Set `chainId: 1` for ENS calls regardless of your dapp's "active" chain.

L2 *primary names* (`address → name` on an L2) are a separate flow — see [profile.md](profile.md#l2-primary-names).

## The Universal Resolver

**What**: A single contract that wraps registry lookup + resolver call + CCIP-Read + multicall, exposing `resolve(name, data)` and `reverse(addr)` in one shot.

**Why prefer it**: Manual `registry.resolver(node) → resolver.addr(node)` doesn't transparently handle:
- ENSIP-10 wildcard resolution (offchain subnames like `jesse.base.eth`).
- CCIP-Read (Coinbase, Linea, Uniswap, ENS-managed offchain names).
- Reverse-then-forward verification.

Viem and ethers wire this up automatically. **You should not call the registry directly** unless you have a specific reason; use the library's high-level actions, which target the Universal Resolver.

**Address**: changes across deployments; check [docs.ens.domains/learn/deployments](https://docs.ens.domains/learn/deployments) for the current mainnet and per-L2 addresses. Don't hardcode.

## CCIP-Read

**Spec**: [EIP-3668](https://eips.ethereum.org/EIPS/eip-3668), [ENSIP-10](https://docs.ens.domains/ensip/10).

**Why it exists**: Onchain storage is expensive. Offchain resolvers let projects synthesize records dynamically (e.g., generate `<anything>.cb.id` on demand) and update records without sending a tx.

**The flow**:
1. Client calls a resolver function (e.g., `addr(node)`).
2. The resolver reverts with `OffchainLookup(sender, urls, callData, callbackFunction, extraData)`.
3. Client makes an HTTPS request to one of the `urls` with `callData`.
4. Gateway returns signed data.
5. Client calls `callbackFunction(response, extraData)` on the resolver, which verifies and returns the result.

**What you actually need to do as an integrator**: nothing, if you're using viem ≥ 2.35 or wagmi (CCIP-Read is on by default). If you've explicitly disabled it (`ccipRead: false` in viem) or are using a custom client, you'll silently fail to resolve `*.cb.id`, `*.linea.eth`, `*.uni.eth`, and similar.

**Common failures**:
- Disabling CCIP-Read globally because of a misunderstanding — re-enable it.
- Corporate firewall blocking outbound HTTPS to gateway URLs.
- Custom RPC provider that doesn't relay the gateway response.

## ENSv2 readiness

**Reference**: [docs.ens.domains/web/ensv2-readiness](https://docs.ens.domains/web/ensv2-readiness).

ENSv2 introduces L2-native names, namechain registries, and a unified resolution API via the Universal Resolver. For most apps, "being ENSv2-ready" means three things:

1. **Use a current library version**:
   - **viem ≥ 2.35** — ENSv2 support is automatic.
   - **ethers v6** — apply the ENS readiness patch (see ENS docs for the package).
   - **web3.js** — deprecated for ENS work; migrate.
2. **Don't gate on `.eth`**. ENSv2 expands what counts as an ENS name (DNS-imported, alternative TLDs). Detect ENS by attempting to resolve a normalized dotted string, not by suffix.
3. **Don't bypass the Universal Resolver** with manual registry calls.

ENSv2 doesn't break existing v1 names; it adds capability. The risk is your code accidentally taking a fast path that ignores v2-only features (e.g., querying a v1 resolver directly and missing the wildcard fallback).

## Multichain coin types

**Specs**: [ENSIP-9](https://docs.ens.domains/ensip/9), [ENSIP-11](https://docs.ens.domains/ensip/11).

A name can resolve to different addresses on different chains. The chain is selected by passing `coinType` to `addr(node, coinType)`:

| Chain | coinType source | Example |
|---|---|---|
| Bitcoin | SLIP-44 | `0` |
| Ethereum mainnet | SLIP-44 | `60` |
| Other L1s (Litecoin, Ripple, etc.) | SLIP-44 | per chain |
| **EVM L2s and sidechains** | ENSIP-11: `chainId | 0x80000000` | Base (8453) → `2147491701` |

**Encoding**: addresses are stored as raw bytes, not strings. To go from bytes → human-readable address (Bech32 for BTC, hex for EVM, base58 for XRP, etc.), use `@ensdomains/address-encoder`.

```ts
import { getEnsAddress } from 'viem/ens'
// Resolve alice.eth's address on Base (chainId 8453)
const baseAddr = await client.getEnsAddress({
  name: 'alice.eth',
  coinType: 8453 | 0x80000000,
})
```

If you ask for a coinType the user hasn't set, you get an empty result — not a fallback to mainnet.

## Batch resolution

For leaderboards / tables / N-row lists, **don't fire N independent `useEnsName` hooks**. Each is an RPC round-trip. Instead, multicall the Universal Resolver:

```ts
import { encodeFunctionData, parseAbi } from 'viem'

// Pseudo: build an array of resolve(name, data) calls and submit them via
// the Universal Resolver's `resolve` aggregate, or use multicall3 against
// resolver.addr(node) one per name.
```

Concrete approaches:
- **viem `multicall`** against the Universal Resolver's `resolve(name, data)` for each name.
- **wagmi `useReadContracts`** with the Universal Resolver ABI.
- **Subgraph** for very large batches that don't need real-time data — but note offchain (CCIP-Read) names won't appear there. See [contracts.md](contracts.md#subgraph).

## Footguns

1. **Setting `chainId` to the L2 chain for forward resolution.** Resolution lives on mainnet. Use `chainId: 1`.
2. **Hardcoding the Universal Resolver address.** It has multiple versions and changes per chain. Use the library default or fetch from deployments docs.
3. **Disabling CCIP-Read globally** (`ccipRead: false` in viem). Silently breaks Coinbase/Base/Linea/Uni names.
4. **Forgetting `coinType`** when sending on a non-mainnet chain. You'll send to the user's mainnet address, not their Base address.
5. **N+1 RPC calls** for leaderboards. Multicall.
6. **Skipping forward verification** in custom reverse-resolution code. Anyone can claim any name in their reverse record.
7. **`.eth` suffix gating.** Breaks DNS-imported, `.box`, and ENSv2 names.

## Sources

- [docs.ens.domains/web/resolution](https://docs.ens.domains/web/resolution)
- [docs.ens.domains/web/ensv2-readiness](https://docs.ens.domains/web/ensv2-readiness)
- [ENSIP-9 — Multichain Address Resolution](https://docs.ens.domains/ensip/9)
- [ENSIP-10 — Wildcard Resolution](https://docs.ens.domains/ensip/10)
- [ENSIP-11 — EVM-chain-specific addresses](https://docs.ens.domains/ensip/11)
- [EIP-3668 — CCIP-Read](https://eips.ethereum.org/EIPS/eip-3668)
- [docs.ens.domains/learn/deployments](https://docs.ens.domains/learn/deployments)
- [`@ensdomains/address-encoder`](https://www.npmjs.com/package/@ensdomains/address-encoder)
