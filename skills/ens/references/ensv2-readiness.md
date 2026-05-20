# ENSv2 Readiness

What changed in ENSv2, what your app needs to do (or stop doing) to be compatible.

**Authoritative source**: [docs.ens.domains/web/ensv2-readiness](https://docs.ens.domains/web/ensv2-readiness). Always cross-check against the live page — v2 is an active rollout.

## What v2 shifts

**v1 names still work.** The risk is v1-era fast paths that silently miss v2-era names (offchain, L2-primary, non-`.eth`). Update your library first — most apps need nothing else.

## Library matrix

| Library | What "v2-ready" means | How to verify |
|---|---|---|
| **viem** | Version **≥ 2.35.0** — UR routing and CCIP-Read on by default. | `npm ls viem`, then run the [test vectors](#verify). |
| **wagmi** | Latest v2.x — inherits viem's readiness. | `pnpm why viem` to confirm viem ≥ 2.35. |
| **ethers** | Both v5 and v6 need the ENS patch — install [`@ensdomains/ethers-patch-v5`](https://github.com/ensdomains/ethers-patch) or `@ensdomains/ethers-patch-v6` and `import` it at app entry. Native v6 support lands in **v6.16.0**. | Run the [test vectors](#verify). |
| **web3.js** | Deprecated for ENS — migrate. | n/a |
| **Other languages** | Library must target UR and implement EIP-3668 CCIP-Read. Otherwise route through an RPC that does. | Run the [test vectors](#verify). |

## Verify

Two canonical pass/fail checks. Wire both into an integration test so a future dependency bump can't silently regress them.

| What it proves | Resolve | Expect | Failure signal |
|---|---|---|---|
| **Universal Resolver** is v2 | `ur.integration-tests.eth` | `0x2222222222222222222222222222222222222222` | `0x1111…1111` → library/UR is pre-v2 |
| **CCIP-Read** path works | `test.offchaindemo.eth` | `0x779981590E7Ccc0CFAe8040Ce7151324747cDb97` | `null` / revert → CCIP-Read disabled or unsupported |

**Multichain example.** `test.ses.eth` resolves to different addresses per chain — Ethereum Mainnet `0x2B0F09F23193de2Fb66258a10886B9f06903276c`, Base `0x7d3a48269416507E6d207a9449E7800971823Ffa`. Omitting `coinType` returns the Mainnet address by default, which isn't guaranteed to work on L2s.

## v1-era patterns to fix


| ❌ Pattern | ✅ Replacement | Why it matters |
|---|---|---|
| `name.endsWith('.eth')` to detect "is this an ENS name?" | Normalize, check for a `.`, try to resolve. See [SKILL.md Rule 5](../SKILL.md#the-five-rules-that-apply-to-every-ens-integration). | Excludes DNS-imported names (`.com`, `.xyz`, `.box`), alt TLDs, emoji names — a growing share of mainstream ENS. |
| Manual `registry.resolver(node) → resolver.addr(node)` | Library high-level actions (`getEnsAddress`, `getEnsName`, etc.) routed through the Universal Resolver. | Skips wildcard resolution and CCIP-Read. Returns `null` for offchain and wildcard names. |
| `ccipRead: false` (or omitting EIP-3668 handling in custom code) | Default-on (viem ≥ 2.35); explicit in custom clients. | Silently breaks Coinbase / Linea / Uniswap / Base offchain names. |
| Mainnet `addr.reverse` only for primary names | Per-chain reverse registrars (ENSIP-19). See [records.md → L2 primary names](records.md#l2-primary-names). | Misses users whose primary lives on an L2. |
| Hardcoded Universal Resolver address | Read from [learn/deployments](https://docs.ens.domains/learn/deployments) at integration time. | Multiple UR versions during the transition; addresses differ per chain. |
| Omitting `coinType` on non-Mainnet sends | Pass `chainId \| 0x80000000` (ENSIP-9/-11). See [resolution.md → Multichain coin types](resolution.md#multichain-coin-types). | Defaults to Mainnet address, not guaranteed valid on L2. |
| Raw `<img src={avatarTextRecord}>` | Library avatar action (handles `eip155:` NFT URIs with ownership verification). | Naive rendering misses NFT avatars and skips ownership checks. |

## Sources

- [docs.ens.domains/web/ensv2-readiness](https://docs.ens.domains/web/ensv2-readiness) — authoritative
- [docs.ens.domains/learn/deployments](https://docs.ens.domains/learn/deployments) — live contract addresses
- [EIP-3668 — CCIP-Read](https://eips.ethereum.org/EIPS/eip-3668)
- [ENSIP-19 — L2 Reverse Resolution](https://docs.ens.domains/ensip/19)
- [ensdomains/ethers-patch](https://github.com/ensdomains/ethers-patch)
