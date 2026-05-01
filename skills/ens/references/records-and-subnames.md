# Records & Subnames (Scenario 2)

For apps that go beyond display/resolve — issuing subnames, editing custom records, importing DNS names, storing dweb sites or contract ABIs.

## Contents

- [Namehash mechanics](#namehash-mechanics)
- [Subnames: onchain vs offchain](#subnames-onchain-vs-offchain)
- [DNS import (ENSIP-6)](#dns-import-ensip-6)
- [Content hash (ENSIP-7)](#content-hash-ensip-7)
- [ABI text record (ENSIP-4)](#abi-text-record-ensip-4)
- [Chain-Registry Resolver (`<chain>.on.eth`)](#chain-registry-resolver-chainoneth)
- [Subname normalization](#subname-normalization)
- [Footguns](#footguns)

---

## Namehash mechanics

**Spec**: [ENSIP-1](https://docs.ens.domains/ensip/1).

Namehash is a recursive keccak256 over normalized labels:

```
namehash(""):       0x000…000
namehash("eth"):    keccak256(namehash("") || keccak256("eth"))
namehash("alice.eth"): keccak256(namehash("eth") || keccak256("alice"))
namehash("a.b.c"):  keccak256(namehash("b.c") || keccak256("a"))
```

Every label MUST be normalized (ENSIP-15) before hashing. `Vitalik.eth` and `vitalik.eth` are the same node only because normalization makes them identical pre-hash. If you compute the hash of an unnormalized label, you'll write to a node nobody resolves to.

For subnames, the hash chains: `cool.alice.eth` is `keccak(namehash("alice.eth") || keccak(normalize("cool")))`. The parent must exist; the parent owner controls subname registration in the registry.

## Subnames: onchain vs offchain

Two paths, very different operational properties.

### Onchain subnames

The parent owner sets the subname owner directly in the ENS Registry:

```solidity
// On the ENS Registry, the parent owner calls:
registry.setSubnodeOwner(parentNode, keccak256(label), childOwner)
// Then sets a resolver on the child node:
registry.setResolver(childNode, resolverAddress)
// Then the child owner sets records on that resolver.
```

- **Pros**: fully onchain; child owner has real control; works with every ENS-aware tool.
- **Cons**: every subname costs gas; not feasible at scale (1000s of users).
- **When to use**: small number of named entities (team members, official subdomains).

### Offchain subnames (CCIP-Read / wildcard)

The parent's resolver implements ENSIP-10 (`resolve(name, data)`) and synthesizes records on demand. When `unknown.app.eth` is queried, the resolver receives the full name and can return any record it wants — typically by calling out to a database via CCIP-Read.

- **Pros**: gasless creation/update; scales to millions of subnames; updates are instant.
- **Cons**: requires running (or trusting) a gateway; not visible in the ENS subgraph; child has no onchain control.
- **When to use**: user-generated names, large directories (`<user>.cb.id`, `<wallet>.uni.eth`), per-deployment subdomains.

This skill is vendor-neutral; building a CCIP-Read backend is a substantial engineering effort. The protocol pieces you need:

1. A resolver contract implementing ENSIP-10 + EIP-3668.
2. An HTTPS gateway that signs responses.
3. Set the resolver as the parent's resolver in the ENS Registry.

## DNS import (ENSIP-6)

**Spec**: [ENSIP-6](https://docs.ens.domains/ensip/6). **Guide**: [docs.ens.domains/learn/dns](https://docs.ens.domains/learn/dns).

Lets owners of DNSSEC-enabled DNS names (e.g., `example.com`) bring those names into ENS.

- Requires DNSSEC enabled at the registrar.
- Verification happens by publishing a TXT record proving control.
- After import, `example.com` resolves through ENS resolvers like any `.eth` name.

Practical note: ENSIP-6's record-by-record DNS sync is largely superseded by direct DNSSEC verification + a single resolver entry. For most app integrations, you don't need to *do* the import — you just need to handle non-`.eth` names in your resolution code (see [resolution.md](resolution.md), Rule 4).

## Content hash (ENSIP-7)

**Spec**: [ENSIP-7](https://docs.ens.domains/ensip/7).

The `contenthash` field stores a protocol-tagged pointer to distributed content. Wire format: `<protoCode uvarint><value bytes>`. Common codecs:

| Codec | Code (hex) | Use |
|---|---|---|
| IPFS | `0xe3` | CIDv0/v1 → fetch via IPFS gateway |
| IPNS | `0xe5` | mutable IPFS pointer |
| Swarm | `0xe4` | Swarm hash |
| Onion (Tor v3) | `0xbd` | onion address |
| Skynet | `0x90` | Sia Skynet |
| Arweave | varies | Arweave tx ID |

Why use it instead of a `url` text record:
- `url` is a centralized HTTP string. `contenthash` is content-addressed and protocol-routable.
- Browsers like Brave (and the Opera mobile browser) natively render `.eth` names with `contenthash` set to IPFS as decentralized websites.

To read/write content hash, use your library's helper (viem `getEnsContentHash`, ethers `resolver.getContentHash()`); the codec byte is parsed for you.

## ABI text record (ENSIP-4)

**Spec**: [ENSIP-4](https://docs.ens.domains/ensip/4).

Stores a contract ABI directly on a name's resolver. Encoding types are a bitfield:

| Type | Encoding |
|---|---|
| `1` | JSON (human-readable) |
| `2` | zlib-compressed JSON |
| `4` | CBOR |
| `8` | URI (external pointer) |

Resolver interface: `ABI(bytes32 node, uint256 contentType) → (uint256, bytes)`. The caller passes a bitfield of acceptable encodings; the resolver returns the first match.

Use case: dapps can fetch the ABI for a contract by ENS name without a separate registry like Etherscan. Often used together with [smart-contract naming](contracts.md#naming-a-smart-contract).

## Chain-Registry Resolver (`<chain>.on.eth`)

**Spec/docs**: [docs.ens.domains/resolvers/chain-registry-resolver](https://docs.ens.domains/resolvers/chain-registry-resolver).

A resolver where each label under `on.eth` represents a blockchain — `optimism.on.eth`, `base.on.eth`, `arbitrum.on.eth`, etc. Records on those names hold chain metadata: interoperable addresses (ERC-7930), text records, multichain addresses, contenthash, arbitrary data.

When to use:
- A dapp that needs canonical chain metadata (RPC endpoints, native token info, explorer URLs) without maintaining its own chain list.
- An agent that needs to map "the user said Base" to a chain ID + RPC.

The registry is enumerable; chain admins (designated by the registry owner) manage their chain's records.

## Subname normalization

ENSIP-15 normalizes the *whole* dotted name, not labels in isolation. Bidi rules and emoji ZWJ allowlists are evaluated across label boundaries. **Always call `ens_normalize` on the full name**:

```ts
// Wrong
const labels = input.split('.').map(label => normalize(label))
const fullName = labels.join('.')

// Right
const fullName = normalize(input)
```

For emoji subnames (e.g., `🚀.example.eth`):
- The emoji must be a fully-qualified sequence (with `FE0F` if part of a known sequence).
- Bare ZWJs are disallowed except inside an allowlisted emoji sequence.
- The presentation selector is stripped in the stored form (`ens_normalize`) but restored in the displayed form (`ens_beautify`).

See [normalization.md](normalization.md) for the full picture.

## Footguns

1. **Hashing an unnormalized label.** You'll write to a node nobody can resolve.
2. **Splitting a name to normalize each label.** Bidi/ZWJ rules are cross-label.
3. **Onchain subnames at scale.** Gas cost compounds; if you need 1000+, build CCIP-Read.
4. **Treating offchain subnames as queryable in the subgraph.** They aren't; they only exist at resolution time.
5. **Storing a URL in `contenthash`.** That's the wrong record. `contenthash` is binary-tagged; `url` is the text record for HTTP.
6. **ABI lookups assuming type=1 (JSON).** Real resolvers may only have CBOR or zlib-JSON. Pass a bitfield of all you can decode.
7. **Burning the parent resolver while subnames depend on it.** All onchain subname records become unreachable.

## Sources

- [ENSIP-1 — Namehash](https://docs.ens.domains/ensip/1)
- [ENSIP-4 — ABI Records](https://docs.ens.domains/ensip/4)
- [ENSIP-6 — DNS-in-ENS](https://docs.ens.domains/ensip/6)
- [ENSIP-7 — Content Hash](https://docs.ens.domains/ensip/7)
- [ENSIP-10 — Wildcard Resolution](https://docs.ens.domains/ensip/10)
- [docs.ens.domains/learn/dns](https://docs.ens.domains/learn/dns)
- [docs.ens.domains/resolvers/chain-registry-resolver](https://docs.ens.domains/resolvers/chain-registry-resolver)
- [docs.ens.domains/resolvers/ccip-read](https://docs.ens.domains/resolvers/ccip-read)
