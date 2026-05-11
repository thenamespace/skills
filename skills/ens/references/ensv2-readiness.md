# ENSv2 Readiness

What changed in ENSv2, what your app needs to do (or stop doing) to be compatible, and a checklist.

**Authoritative source**: [docs.ens.domains/web/ensv2-readiness](https://docs.ens.domains/web/ensv2-readiness). Always cross-check against the live page — v2 is an active rollout.

## Contents

- [What v2 changes](#what-v2-changes)
- [The library version matrix](#the-library-version-matrix)
- [What to stop doing](#what-to-stop-doing)
- [What to start using](#what-to-start-using)
- [Readiness checklist](#readiness-checklist)
- [Migration notes](#migration-notes)

---

## What v2 changes

ENSv2 is the protocol's evolution toward L2-native names, gasless registrations, and unified cross-chain resolution. The changes that matter to integrators:

- **Universal Resolver as the canonical entry point** — instead of querying the registry directly to find a resolver and then calling it, all reads route through the Universal Resolver, which transparently handles wildcard fallback, CCIP-Read, and reverse-then-forward verification. See [resolution.md → Universal Resolver](resolution.md#the-universal-resolver).
- **Per-chain registries and reverse registrars** — L2s have their own deployments. Names can live primarily on an L2; a user can have different primary names per chain. See [records.md → L2 primary names](records.md#l2-primary-names).
- **Wildcard / synthesized subnames** are first-class — large offchain spaces (`*.cb.id`, `*.uni.eth`, `*.linea.eth`) resolve through the same surface as onchain names.
- **DNS-imported names and alternative TLDs** are mainstream — `.eth` is no longer the only suffix worth handling.

ENSv2 doesn't break v1 names; it adds capability and shifts the canonical access pattern. The risk to your app isn't that v1 names stop working — it's that your code takes a v1-era fast path that silently misses v2-era names (offchain, L2-primary, non-`.eth`).

## The library version matrix

| Library | What "v2-ready" means | How to verify |
|---|---|---|
| **viem** | Version **≥ 2.35.0**. ENSv2 routing through the Universal Resolver and CCIP-Read are on by default. | `cat package.json \| grep viem` and check `viem.sh` for the changelog of your version. |
| **wagmi** | Latest v2.x — built on viem, inherits its v2 readiness. Make sure your wagmi pulls a viem ≥ 2.35. | `pnpm why viem` (or `npm ls viem`) to see the resolved version. |
| **ethers** | **v6 with the ENS readiness patch applied.** v5 is not v2-ready. | Check ENS docs for the current ethers ENS package; install and verify resolution against a known offchain name. |
| **web3.js** | Deprecated for ENS. Migrate to viem or ethers v6. | n/a |
| **Other languages** (Go, Rust, Python, Swift, etc.) | Use a library that targets the Universal Resolver and implements EIP-3668 CCIP-Read. If yours doesn't, route through an RPC that does, or implement against the Universal Resolver directly. | Try resolving a known CCIP-Read name like `1.offchainexample.eth`; if it returns null, your stack isn't v2-ready. |

## What to stop doing

Each of these is a v1-era pattern that v2 silently breaks:

- **Manually calling `registry.resolver(node) → resolver.addr(node)`.** Skips wildcard resolution and CCIP-Read. Returns null for any offchain or wildcard name.
- **Gating on `.eth` suffix** to detect "is this an ENS name?" Use a normalized-string-with-a-dot check and try to resolve it. See [Rule 5 in SKILL.md](../SKILL.md#the-five-rules-that-apply-to-every-ens-integration).
- **Disabling CCIP-Read** (`ccipRead: false` in viem, omitting EIP-3668 handling in custom code). Silently breaks Coinbase / Linea / Uniswap / Base offchain names — a sizable chunk of mainstream ENS.
- **Querying mainnet `addr.reverse` only** for primary names. L2 primary names live on per-chain reverse registrars. See [records.md → L2 primary names](records.md#l2-primary-names).
- **Hardcoding the Universal Resolver address**. Multiple versions exist during the transition; per-chain addresses differ. Use [docs.ens.domains/learn/deployments](https://docs.ens.domains/learn/deployments) at integration time.

## What to start using

- **Universal Resolver** for every read — let your library route there.
- **Per-chain `chainId`** when calling `getEnsName` / `getEnsAddress` / `getEnsAvatar` / `getEnsText` for L2-active users.
- **Multichain `coinType`** when sending on a chain other than Ethereum mainnet (ENSIP-9/-11). See [resolution.md → Multichain coin types](resolution.md#multichain-coin-types).
- **`@adraffy/ens-normalize`** (or your library's wrapper) — normalize *every* user-input name. See [normalization.md](normalization.md).
- **EIP-165 `supportsInterface()`** before calling optional resolver methods on arbitrary resolvers. See [smart-contracts.md → Verify resolver interface support](smart-contracts.md#verify-resolver-interface-support-eip-165).

## Readiness checklist

Run through this on any existing app to confirm v2 readiness:

- [ ] Library is at the v2-ready version (viem ≥ 2.35, ethers v6 with patch, etc.).
- [ ] CCIP-Read is enabled (default in viem ≥ 2.35; explicit in older or custom clients).
- [ ] All reads route through the library's high-level actions (`getEnsName`, `getEnsAddress`, etc.) — no manual `registry.resolver()` calls.
- [ ] No hardcoded ENS contract addresses anywhere in the codebase.
- [ ] Detection of "is this a name?" doesn't string-match `.eth` — supports DNS-imported names and alternative TLDs.
- [ ] Reverse resolution targets the chain the user is active on (not just mainnet), with mainnet as fallback.
- [ ] Multichain sends pass the right `coinType` (ENSIP-9/-11 — `chainId | 0x80000000` for EVM L2s).
- [ ] Avatars are resolved through the library (handles `eip155:` NFT URIs with ownership verification, not raw `<img src>`).
- [ ] The integration resolves a known offchain name correctly — e.g., a `*.cb.id` name. Add an integration test for this.

If any item misses, see [SKILL.md → Audit mode](../SKILL.md#audit-mode--is-my-existing-ens-integration-correct) for the broader audit walk-through.

## Migration notes

If you're upgrading from a v1-era integration:

- **Bumping viem to ≥ 2.35 is the single biggest win** — most v2 capability comes for free.
- **Test against offchain names** before declaring done. `*.cb.id`, `*.uni.eth`, and `*.linea.eth` are good integration-test targets — if any of them return null when they should resolve, your CCIP-Read path is broken.
- **Decide your L2 primary policy** explicitly: are you showing the chain-specific primary on L2 dapps, or always falling back to mainnet? Document and apply consistently.
- **Audit your `.eth` string-matches** with a global search. Most apps have at least one (`if (name.endsWith('.eth'))`); they all need to go.
- **Don't try to be exhaustive about offchain spaces.** New ones launch regularly; the right pattern is "let the Universal Resolver handle it" rather than "maintain a list of supported wildcard parents."

## Sources

- [docs.ens.domains/web/ensv2-readiness](https://docs.ens.domains/web/ensv2-readiness) — authoritative
- [docs.ens.domains/learn/deployments](https://docs.ens.domains/learn/deployments) — live contract addresses
- [docs.ens.domains/resolvers/universal](https://docs.ens.domains/resolvers/universal)
- [EIP-3668 — CCIP-Read](https://eips.ethereum.org/EIPS/eip-3668)
- [ENSIP-10 — Wildcard Resolution](https://docs.ens.domains/ensip/10)
- [ENSIP-19 — L2 Reverse Resolution](https://docs.ens.domains/ensip/19)
