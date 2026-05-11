---
name: ens
description: Use whenever a developer is integrating, debugging, or auditing ENS (Ethereum Name Service) in an app — displaying ENS names instead of 0x addresses, resolving names↔addresses, showing profiles/avatars, accepting ENS in send/payment flows, building subnames, smart-contract naming, or asking "is my ENS code correct". Triggers on phrases like "add ENS support", "show ENS name", "resolve ENS", "ENS avatar", "useEnsName", "ENS reverse lookup", "ENS subname", "ENS profile", "CCIP-Read", "ENSIP-15", "ENS normalization", "L2 primary name", "ENSv2 ready", or any general "ENS" question. Use even when the user just says "ENS" without specifying a task — there are critical correctness rules (normalization, forward-verification of reverse lookups, multichain coinTypes, CCIP-Read) that apply across every ENS integration and this skill enforces them. Vendor-neutral; covers the ENS protocol itself, not any single product.
---

# ENS for Developers

This skill guides ENS integration work. It covers two modes:

- **Greenfield** — adding ENS to an app for the first time.
- **Audit** — checking an existing integration is correct against current ENS best practices (ENSv2-ready, normalization, multichain, L2 primary, CCIP-Read).

Most apps (~99%) only need [Scenario 1](#scenario-1--the-99-case-display-resolve-profile). Bigger surface areas — custom records, subnames, contract naming, querying at scale, library implementation, AI-agent identity — are organized as scenarios below and detailed in `references/`.

## Scope and source of truth

- **Source of truth**: [docs.ens.domains](https://docs.ens.domains). When in doubt or addresses/specs may have moved, check there.
- **Stack**: principles are stack-agnostic; concrete code defaults to **viem ≥ 2.35** + **wagmi**, with notes for ethers v6.
- **Out of scope**: any specific naming product or registrar SDK. Pure protocol + canonical libraries only.

## The five rules that apply to every ENS integration

These show up in every scenario. If a single one is missing, the integration is wrong even if it appears to work.

1. **Normalize at every input boundary** with `@adraffy/ens-normalize` (or your library's wrapper). Never `toLowerCase()` an ENS name. Normalization is what prevents homoglyph attacks (`аpple.eth` with a Cyrillic `а`) and ensures `Vitalik.eth` and `vitalik.eth` hash to the same node. Apply it to: search inputs, address-book entries, anything before hashing/comparing/sending. See [references/normalization.md](references/normalization.md).
2. **Forward-verify after every reverse resolution.** Anyone can set their reverse record to claim *any* name. The only thing that proves a name actually belongs to an address is forward-resolving the claimed name and checking it points back. viem/wagmi do this by default; ethers v6 historically did not — verify your library's behavior or add the second resolution yourself. See [references/records.md](references/records.md).
3. **For value-bearing transactions, resolve fresh against an L1 RPC** — not via a cached value, an indexer, the subgraph, or a third-party API. Cached and indexed data is eventually consistent; an updated `addr` record may not propagate for minutes. The user is signing for the *address*, so the address must be the live one at signing time. Run your own node for high-value flows, or use an audited L1 RPC provider. See [references/resolution.md](references/resolution.md).
4. **Don't hardcode contract addresses.** Registry, Universal Resolver, and Reverse Registrar addresses change across upgrades and per-chain deployments. Use the addresses your library ships with, or look up [docs.ens.domains/learn/deployments](https://docs.ens.domains/learn/deployments) at integration time. Universal Resolver in particular has multiple deployed versions during the ENSv2 transition. See [references/ensv2-readiness.md](references/ensv2-readiness.md).
5. **Don't gate on `.eth` suffix.** ENS supports DNS-imported names (`.com`, `.xyz`, `.box`) and other TLDs. If you need to detect "is this an ENS name?", check for a `.` in a normalized string and try to resolve it — don't string-match TLDs.

---

## Scenario 1 — The 99% case (display, resolve, profile)

What app owners typically want:
- Show an ENS name wherever an address would otherwise appear (connect-wallet header, leaderboards, tx history).
- Accept an ENS name in send/payment inputs and resolve it to an address before submitting a transaction.
- Show a profile card (name, avatar, optional socials) for a user.

### A. Pick the right starting point

| You're using... | Do this |
|---|---|
| **wagmi** (React) | Use `useEnsName`, `useEnsAddress`, `useEnsAvatar`, `useEnsText`. Already covers the 99%. |
| **viem** without React | Use `getEnsName`, `getEnsAddress`, `getEnsAvatar`, `getEnsText`. Same surface as wagmi. |
| **ethers v6** | Use `provider.lookupAddress(addr)` and `provider.resolveName(name)`. For avatars/text, instantiate the resolver. |
| **RainbowKit / ConnectKit / Reown AppKit** | ENS name + avatar in the connect button is on by default (built on wagmi). Confirm it's not overridden in your config. |
| **No JS** (backend, mobile, other lang) | Call ENS via your chain RPC and the Universal Resolver. See [references/resolution.md](references/resolution.md). |

### B. Show ENS instead of addresses (display)

The pattern: try to reverse-resolve the address. If a name comes back, render it. Otherwise, render a truncated address. The library handles forward-verification.

```ts
// wagmi
import { useEnsName, useEnsAvatar } from 'wagmi'
import { normalize } from 'viem/ens'

function AddressDisplay({ address }: { address: `0x${string}` }) {
  const { data: name } = useEnsName({ address, chainId: 1 })  // resolution always starts on mainnet
  const { data: avatar } = useEnsAvatar({ name: name ? normalize(name) : undefined, chainId: 1 })
  return (
    <span>
      {avatar && <img src={avatar} alt="" />}
      {name ?? `${address.slice(0, 6)}…${address.slice(-4)}`}
    </span>
  )
}
```

Notes:
- `chainId: 1` is correct *even on L2 dapps* — ENS records live on mainnet (with L2 primary names being a separate flow; see [references/records.md](references/records.md)).
- `normalize()` before `useEnsAvatar` is the boundary requirement. wagmi v2 will throw if you pass an unnormalized name.
- For batch leaderboards, don't call N hooks — query the Universal Resolver in a multicall. See [references/resolution.md](references/resolution.md#batch-resolution).

### C. Accept ENS in send/payment inputs

```ts
import { normalize } from 'viem/ens'
import { publicClient } from './client'

async function resolveRecipient(input: string): Promise<`0x${string}`> {
  // 1. Normalize. This will throw on invalid input — catch and tell the user.
  const name = normalize(input.trim())

  // 2. Resolve. viem ≥ 2.35 handles CCIP-Read and the Universal Resolver automatically.
  const address = await publicClient.getEnsAddress({ name })
  if (!address) throw new Error(`No address set for ${name}`)

  return address
}
```

Confirmation step (recommended): show the user the resolved address before they sign. ENS resolution can change; the user is signing for the *address*, not the name.

For multichain sends ("send to alice.eth on Base"), pass `coinType` — see [references/resolution.md](references/resolution.md#multichain-coin-types).

### D. Show a profile

The standard text records are defined by ENSIP-5 and ENSIP-18. Common ones: `avatar`, `description`, `url`, `com.twitter`, `com.github`, `org.telegram`, `email`, `location`, `header`. Full list and "how to render avatars correctly" in [references/records.md](references/records.md).

---

## Audit mode — is my existing ENS integration correct?

Run through this checklist. Each item maps to a section of the references. Flag any miss.

**Resolution & display**
- [ ] All ENS name input is normalized with `@adraffy/ens-normalize` (or library wrapper) before hashing/storing/comparing. No `toLowerCase()`. ([normalization](references/normalization.md))
- [ ] `namehash` is computed from a normalized name, never raw input; `namehash` is not confused with `labelhash`. ([records](references/records.md#namehash-mechanics))
- [ ] Reverse resolution (`address → name`) is forward-verified before being shown. ([profile](references/records.md#reverse-resolution))
- [ ] Address comparisons are case-insensitive (`.toLowerCase()` both sides) — checksum casing is informational. ([profile](references/records.md#reverse-resolution))
- [ ] App never compares pre-normalized names — `name1 === name2` only after normalize. ([normalization](references/normalization.md))
- [ ] Value-bearing sends resolve fresh from an L1 RPC, not from cache, indexer, or subgraph. ([resolution](references/resolution.md))
- [ ] Reverse records are queried on the chain where the user is *active* — not just mainnet. L2 primary names are honored. ([profile](references/records.md#l2-primary-names))
- [ ] When sending to an ENS name on a non-Ethereum chain, the resolution requests the right `coinType` (ENSIP-9/-11). ([resolution](references/resolution.md#multichain-coin-types))

**Avatars & profile**
- [ ] Avatar rendering handles `https://`, `ipfs://`, `data:` and `eip155:` (NFT) URIs. Naive `<img src={textRecord}>` is a bug. ([profile](references/records.md#avatars))
- [ ] NFT avatars verify ownership before rendering (otherwise spoof risk). ([profile](references/records.md#avatars))

**Library & infra**
- [ ] viem is **≥ 2.35** for ENSv2 readiness; ethers users have applied the ENS patch. ([ensv2-readiness](references/ensv2-readiness.md))
- [ ] CCIP-Read is enabled (default in viem ≥ 2.35; explicit in older or custom clients). Disabling it breaks Coinbase/Base/Linea/Uni subnames. ([resolution](references/resolution.md#ccip-read))
- [ ] CCIP-Read flows have a sane timeout and a clear "name unresolved" UI state. ([resolution](references/resolution.md#ccip-read))
- [ ] No hardcoded ENS contract addresses. ([deployments](https://docs.ens.domains/learn/deployments))
- [ ] No `.eth`-suffix gating. ([Rule 5](#the-five-rules-that-apply-to-every-ens-integration))
- [ ] Custom resolver code calls `supportsInterface()` (EIP-165) before invoking optional resolver methods. ([library-authors](references/library-authors.md))

**Batch / leaderboard cases**
- [ ] Bulk address→name lookups use a single multicall via the Universal Resolver, not N RPC round-trips. ([resolution](references/resolution.md#batch-resolution))

If any item misses, link the user to the matching reference and propose a concrete fix.

---

## Other scenarios

Beyond Scenario 1, each of these is a distinct integration shape. Read the linked file when the user's task lands there; don't pre-load them.

| # | Scenario | Read |
|---|---|---|
| 2 | **Custom records** — write/read text, contenthash, ABI, arbitrary bytes; namehash mechanics; chain-registry resolver (`<chain>.on.eth`). | [references/records.md](references/records.md) |
| 3 | **Subnames** — issue names under a parent, onchain vs offchain (CCIP-Read wildcard), DNS-imported parents (ENSIP-6). | [references/subnames.md](references/subnames.md) |
| 4 | **Smart contracts** — naming a deployed contract, key contract addresses, EIP-165 interface checks, ENS as service-discovery infrastructure. | [references/smart-contracts.md](references/smart-contracts.md) |
| 5 | **Querying at scale** — subgraph for leaderboards / search / historical data, and the offchain-name limitation. | [references/subgraph.md](references/subgraph.md) |
| 6 | **ENSv2 readiness** — what changed, library version matrix, migration checklist. | [references/ensv2-readiness.md](references/ensv2-readiness.md) |
| 7 | **AI agents** — agent owns an ENS identity; ENSIP-26 discovery, ENSIP-25 verification, brief on ERC-8004. | [references/agentic.md](references/agentic.md) |
| 8 | **Library authors** — implementing ENS in a client library (the next viem/ethers, a Go/Rust/Swift port). | [references/library-authors.md](references/library-authors.md) |

Foundational references — read whenever the relevant correctness rule applies, regardless of scenario:

| Topic | Read |
|---|---|
| Normalization (ENSIP-15) | [references/normalization.md](references/normalization.md) |
| Resolution (Universal Resolver, CCIP-Read, multichain, batch) | [references/resolution.md](references/resolution.md) |
| Profile display (text keys, avatars, reverse, L2 primary) | [references/records.md](references/records.md) |

---

## When this skill should defer

- **Specific naming-product questions** (registrar SDKs, subname-as-a-service, hosted resolvers): the user likely wants a vendor's docs or another skill — answer the protocol-level part here, but flag that the product-specific integration is outside this skill.
- **Wallet UX details unrelated to ENS** (signing flows, gas estimation): answer only the ENS slice.

## Working style for this skill

When helping a user:

1. **Identify mode**: are they building from scratch or auditing existing code? If unclear, ask one question.
2. **Stay in Scenario 1** unless the user's task clearly needs more. Don't dump the audit checklist on someone who just wants `useEnsName`.
3. **Cite ENSIPs and docs.ens.domains URLs** when explaining *why* a rule exists. Devs will read them; trust > assertion.
4. **For audits**, walk the checklist out loud, marking pass/fail with a short reason. End with a prioritized fix list.
5. **For code**, prefer the user's existing library (don't migrate them off ethers to viem unsolicited).
6. **Read the relevant reference file before writing code in an unfamiliar area** — the references contain the spec details that prevent subtle bugs.

## Stay current

ENS evolves through ENSIPs. The protocol that's true today (multichain primary names, CCIP-Read, ENSv2 Universal Resolver) wasn't true two years ago, and v3-shaped things will land. When something in this skill seems stale or contradicts a doc you find, **trust [docs.ens.domains](https://docs.ens.domains)** and flag the drift so this skill can be updated. Track new ENSIPs at [docs.ens.domains/ensips](https://docs.ens.domains/ensips).
