# Resolution: forward, multichain, CCIP-Read, ENSv2 readiness

How `name → address` actually works, plus the things that make it work across chains and offchain resolvers.

## Contents

- [How forward resolution works](#how-forward-resolution-works)
- [The Universal Resolver](#the-universal-resolver)
- [CCIP-Read](#ccip-read)
- [Multichain coin types](#multichain-coin-types)
- [Batch resolution](#batch-resolution)
- [Fresh L1 data for value-bearing transactions](#fresh-l1-data-for-value-bearing-transactions)
- [Footguns](#footguns)

For ENSv2 readiness specifics (which library versions, what changed, the migration matrix), see [ensv2-readiness.md](ensv2-readiness.md).

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

L2 *primary names* (`address → name` on an L2) are a separate flow — see [records.md](records.md#l2-primary-names).

## The Universal Resolver

A single contract that aggregates everything resolution touches into one call. It wraps:

- Registry lookup (find the resolver for a namehash).
- Resolver call (the actual `addr` / `text` / `contenthash` query).
- ENSIP-10 wildcard fallback (walk up the tree until a wildcard resolver is found).
- CCIP-Read (handle the gateway round-trip transparently).
- Reverse-then-forward verification (for `reverse(addr)`).
- Multicall (`resolve(name, data[])` for multiple records on one name in one tx).

**Why prefer it**: Manual `registry.resolver(node) → resolver.addr(node)` skips wildcard, CCIP-Read, and verification. So that "simpler" path is **broken** for any name that uses any of those features — which is most of mainstream ENS today (Coinbase, Base, Linea, Uniswap, every offchain space).

**Surface**:
- `resolve(bytes name, bytes data) → bytes` — DNS-encoded name + ABI-encoded resolver call. Returns the result.
- `resolve(bytes name, bytes[] data) → bytes[]` — batch records on a single name in one shot.
- `reverse(bytes addrLower) → (string name, address resolver, address reverseResolver)` — reverse with bidirectional verification baked in.

viem and ethers route through this automatically. **Use the library's high-level action; don't call the registry directly.**

**Address**: multiple deployed versions exist during the ENSv2 transition, plus per-L2 deployments. Always look up [docs.ens.domains/learn/deployments](https://docs.ens.domains/learn/deployments) at integration time; never hardcode. If you must call from a smart contract, take the address from a config file and grab the ABI from [`@ensdomains/ens-contracts`](https://github.com/ensdomains/ens-contracts).

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

**UX considerations**:
- CCIP-Read adds an HTTPS round-trip on top of the chain call. **Budget 1–3 seconds** for resolution in user-facing flows; longer for slow networks.
- **Always show a clear "name unresolved" state** when resolution times out — don't leave the user staring at a spinner. A timeout is not the same as "no name set" (which returns instantly with `null`); distinguish them in the UI.
- For send flows, render the resolved address *before* the user signs. If resolution is still pending, the sign button stays disabled.

**Common failures**:
- Disabling CCIP-Read globally because of a misunderstanding — re-enable it.
- Corporate firewall blocking outbound HTTPS to gateway URLs.
- Custom RPC provider that doesn't relay the gateway response.

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
- **Subgraph** for very large batches that don't need real-time data — but note offchain (CCIP-Read) names won't appear there. See [subgraph.md](subgraph.md).

## Fresh L1 data for value-bearing transactions

Cached resolutions, indexer data, the subgraph, and third-party ENS APIs are all *eventually* consistent. An updated `addr` record may take minutes to propagate. For displays and discovery this is fine; **for value-bearing transactions it is not**.

The user is signing for the *address*. If the address you display was cached and the on-chain record changed since, the user signs for the wrong recipient. There is no recovery.

Rules:

- For sends, swaps, approvals, and any signed action: resolve fresh from an L1 RPC at the moment the user is about to sign.
- Show the resolved address (full 0x or unambiguous truncation) next to the name in the confirmation UI, and re-resolve if the user delays the signature.
- Do not use the subgraph or a cached value as the source of truth for the address being signed.
- For high-value flows (treasury, custody, payroll), run your own Ethereum node or use an audited L1 RPC provider — and verify the integrity of the software in the resolution path.

This is a generalization of [Rule 3 in SKILL.md](../SKILL.md#the-five-rules-that-apply-to-every-ens-integration). It applies regardless of library or chain.

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
