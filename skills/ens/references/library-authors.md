# For Library Authors (Scenario 5)

If you're implementing ENS in a client library (the next viem/ethers, a Go/Rust/Swift/Python library, a CLI), the surface is bigger than what app developers see. This page covers the protocol corners library authors must get right.

## Contents

- [What "correct" means at the library level](#what-correct-means-at-the-library-level)
- [Normalization](#normalization)
- [CCIP-Read implementation](#ccip-read-implementation)
- [Reverse resolution + forward verification](#reverse-resolution--forward-verification)
- [Multichain addresses](#multichain-addresses)
- [Content hash codecs](#content-hash-codecs)
- [Universal Resolver path](#universal-resolver-path)
- [Wildcard / ENSIP-10](#wildcard--ensip-10)
- [EIP-165 interface detection](#eip-165-interface-detection)
- [Conformance test suites](#conformance-test-suites)
- [Footguns](#footguns)

---

## What "correct" means at the library level

Three things separate a real ENS library from a toy:

1. **Normalization at every input boundary** — if your `getEnsAddress(name)` doesn't normalize, your library is broken.
2. **CCIP-Read works out of the box, on by default** — if you require devs to flip a flag, you'll silently break Coinbase / Linea / Uniswap names.
3. **Reverse resolution is forward-verified** — if `getEnsName(addr)` returns a string without verification, you've shipped a spoof vector.

Everything else (multichain, content hash, wildcard, batching) is value-add but secondary to those three.

## Normalization

Use [`@adraffy/ens-normalize`](https://github.com/adraffy/ens-normalize) — the canonical, JS-language reference implementation of [ENSIP-15](https://docs.ens.domains/ensip/15). For non-JS libraries:

- Port or wrap the spec's data tables (`spec.json`).
- Implement: NFC, IDNA mapping, label tokenization, allowed/mapped/disallowed character classes, emoji ZWJ allowlist, fenced characters, bidi rules.
- Throw on invalid input — never silently coerce.

Expose:
- `normalize(name) -> string` (canonical, for hashing)
- `beautify(name) -> string` (for display, restores presentation selectors)
- `split(name) -> string[]` (normalized labels)
- `tokenize(label) -> token[]` (debugging surface)

Conform to the [ENSIP-15 normalization tests](https://github.com/adraffy/ens-normalize/tree/main/derive) — any deviation is a bug.

## CCIP-Read implementation

**Spec**: [EIP-3668](https://eips.ethereum.org/EIPS/eip-3668).

In your `eth_call` wrapper:

1. Catch reverts with selector `OffchainLookup(address sender, string[] urls, bytes callData, bytes4 callbackFunction, bytes extraData)`.
2. Verify `sender == address-of-call` (security: a contract can only request offchain lookups for itself).
3. Try each `url` in order. Substitute `{sender}` and `{data}` placeholders per spec.
4. POST `{ data: callData, sender }` (or GET with the placeholders).
5. Receive `{ data: <bytes> }`.
6. Re-call the original contract with `callbackFunction(extraData_arg, response_arg)` ABI-encoded.
7. Return the result to the caller.
8. Honor a max recursion depth (e.g., 4) to prevent infinite redirect loops.

**Defaults**: enable CCIP-Read by default. If you ship `enableCcipRead: false` as the default, devs will silently fail to resolve a third of mainstream ENS names.

**Trust model**: CCIP-Read responses are typically signed and verified inside the resolver's callback function — your library doesn't validate signatures, the resolver does. Your job is the transport.

## Reverse resolution + forward verification

The Reverse Registrar maps `<addrLower>.addr.reverse` → primary name. **Anyone** can `setName(...)` to claim *any* string. The only proof of legitimacy is a forward resolution that points back to the same address.

Your `getEnsName(addr)` MUST:

```
1. Compute the reverse node: namehash(`${addr.slice(2).toLowerCase()}.addr.reverse`)
2. Resolve the name via the Reverse Registrar / Universal Resolver.
3. Forward-resolve the returned name → addr2.
4. If addr2 != addr, return null. Otherwise, return the name.
```

Don't expose a `skipForwardVerification` flag. There's no legitimate reason to skip it; offering the option is a footgun.

For L2 primary names ([ENSIP-19](https://docs.ens.domains/ensip/19)), the reverse namespace differs per chain: `<chainCoinTypeHex>.reverse`. The Universal Resolver on each L2 handles this; your library should accept a `chainId` parameter and route to the right reverse registrar.

## Multichain addresses

Implement `addr(node, coinType)` per [ENSIP-9](https://docs.ens.domains/ensip/9):

- coinType `0` (BTC), `60` (ETH), etc. via SLIP-44.
- For EVM L2s ([ENSIP-11](https://docs.ens.domains/ensip/11)): `coinType = chainId | 0x80000000`.
- Decode raw bytes to chain-appropriate string formats — Bech32 for BTC, Base58Check for XRP, EIP-55 hex for EVM, etc. Use [`@ensdomains/address-encoder`](https://www.npmjs.com/package/@ensdomains/address-encoder) or port its codec table.
- Default `coinType=60` if not specified, but warn on this default if the user is on a non-Ethereum chain.

Round-trip test: encode a known address, decode it, verify equality. Do this for every supported chain.

## Content hash codecs

Implement [ENSIP-7](https://docs.ens.domains/ensip/7) by parsing the multicodec varint prefix:

| Codec | Code | Notes |
|---|---|---|
| IPFS | `0xe3` | Followed by CID bytes (v0 or v1). |
| IPNS | `0xe5` | Mutable IPFS pointer. |
| Swarm | `0xe4` | Swarm hash. |
| Onion v2 | `0xbc` | Largely deprecated. |
| Onion v3 | `0xbd` | Tor v3. |
| Skynet | `0x90` | Sia. |
| Arweave | varies | Tx ID. |

Expose:
- `decodeContentHash(bytes) -> { protocol: string, value: string, raw: bytes }`.
- `encodeContentHash(protocol, value) -> bytes`.

Round-trip test these too. Read [multicodec.io](https://multicodec.io) for the canonical codec list; ENS uses a subset.

## Universal Resolver path

Prefer routing all reads through the Universal Resolver:

- Single `resolve(bytes name, bytes data) → bytes` call replaces registry lookup + resolver call.
- Handles ENSIP-10 wildcard fallback.
- Handles CCIP-Read transparently (still emits the `OffchainLookup` revert that your wrapper catches).
- For batch reads on a single name, use the `resolve` overload accepting an array of `data` bytes.

Address: per chain; check [docs.ens.domains/learn/deployments](https://docs.ens.domains/learn/deployments). Multiple versions exist during the ENSv2 transition — pick the latest stable per chain and make it overridable via config.

## Wildcard / ENSIP-10

[ENSIP-10](https://docs.ens.domains/ensip/10) defines `resolve(bytes name, bytes data)` on resolvers. When resolving `unknown.app.eth`:

1. Walk the labels: try `unknown.app.eth` → `app.eth` → `eth`, looking for a resolver at each step.
2. If a parent's resolver implements ENSIP-10, call its `resolve(name, data)` with the *full* DNS-encoded name.
3. The resolver synthesizes the answer (often via CCIP-Read).

The Universal Resolver handles this walk for you. If you implement resolution manually, replicate it. Don't stop at the first registered name — that misses every wildcard offchain space.

## EIP-165 interface detection

Resolvers can implement different combinations of interfaces — `addr(node)`, multicoin `addr(node, coinType)`, `text(node, key)`, `contenthash(node)`, `ABI(node, types)`, ENSIP-10 `resolve(name, data)`, ENSIP-24 `data(node, key)`, etc. Not every resolver implements every interface.

**Your library must check `supportsInterface(bytes4) → bool`** before calling optional methods on arbitrary resolvers, and expose a graceful API to the caller (return null/empty rather than letting an unsupported call revert).

Common interface IDs (for reference; canonical values come from each ENSIP):

| Interface | Selector |
|---|---|
| `addr(bytes32)` | `0x3b3b57de` |
| `addr(bytes32, uint256)` | `0xf1cb7e06` |
| `text(bytes32, string)` | `0x59d1d43c` |
| `contenthash(bytes32)` | `0xbc1c58d1` |
| `ABI(bytes32, uint256)` | `0x2203ab56` |
| `resolve(bytes, bytes)` (ENSIP-10) | `0x9061b923` |

**Caching**: resolver shape doesn't change often. Cache `(resolver, interfaceId) → bool` for the lifetime of the client. One `supportsInterface` call per resolver per session is cheap; thousands across a session is wasteful.

**Universal Resolver bypass**: most consumer reads should go through the Universal Resolver, which abstracts the interface-check question for you. Direct resolver calls (typically library-internal) need the explicit check.

## Conformance test suites

Test against:

- **ENS test names** — the ENS team publishes test names with known records (`*.ens.eth` and similar). Resolve them and assert.
- **`@adraffy/ens-normalize` tests** — normalization correctness.
- **Known offchain names** — `test.offchaindemo.eth`, a name from `*.cb.id`, a name from `*.linea.eth`. If your library resolves these without configuration, CCIP-Read works.
- **Reverse spoof name** — set up a test where address A's reverse claims `vitalik.eth`. Your library must return `null`, not `vitalik.eth`.
- **Multichain** — a name with addresses set on BTC, ETH, Base, OP. Round-trip each.

## Footguns

1. **Shipping CCIP-Read off by default.** Silently breaks the most popular offchain spaces.
2. **No forward verification on reverse.** You've shipped a spoof primitive.
3. **Custom normalization** that "approximates" ENSIP-15. There is no approximation — exact match required.
4. **Hardcoded Universal Resolver address.** Multiple versions; per-chain. Make it config.
5. **Splitting subname labels for normalization.** ENSIP-15 is whole-name; piecewise breaks bidi/ZWJ rules.
6. **Defaulting all multichain `addr` calls to coinType 60.** Devs send to mainnet ETH addresses on Base. Require explicit coinType for non-mainnet contexts, or expose a clear default-warning.
7. **Missing CCIP-Read recursion limit.** A malicious gateway can loop you.
8. **`OffchainLookup` without `sender == call.target` check.** A contract could request lookups *as if* from another contract.

## Sources

- [ENSIP-1, -5, -7, -9, -10, -11, -12, -15, -19, -24](https://docs.ens.domains/ensips)
- [EIP-3668 — CCIP-Read](https://eips.ethereum.org/EIPS/eip-3668)
- [`@adraffy/ens-normalize`](https://github.com/adraffy/ens-normalize)
- [`@ensdomains/address-encoder`](https://www.npmjs.com/package/@ensdomains/address-encoder)
- [docs.ens.domains/resolvers/universal](https://docs.ens.domains/resolvers/universal)
- [docs.ens.domains/learn/deployments](https://docs.ens.domains/learn/deployments)
- [github.com/ensdomains/ens-contracts](https://github.com/ensdomains/ens-contracts)
- [github.com/wevm/viem/tree/main/src/actions/ens](https://github.com/wevm/viem/tree/main/src/actions/ens) — reference JS implementation
