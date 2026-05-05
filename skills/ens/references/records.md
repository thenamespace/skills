# Records — profile, reverse, custom records (Scenario 2)

Reading and writing data on an ENS name. Covers everything stored against a name: profile fields and avatars, reverse resolution (the trap that bites everyone), L2 primary names, content hash, ABI, arbitrary bytes, and the chain-registry resolver.

For normalization rules that apply to every record read/write, see [normalization.md](normalization.md). For how a resolution call gets to a resolver in the first place, see [resolution.md](resolution.md).

## Contents

- [Namehash mechanics](#namehash-mechanics)
- [Text records](#text-records)
- [Avatars](#avatars)
- [Reverse resolution](#reverse-resolution)
- [L2 primary names](#l2-primary-names)
- [Content hash (ENSIP-7)](#content-hash-ensip-7)
- [ABI text record (ENSIP-4)](#abi-text-record-ensip-4)
- [Arbitrary bytes records (ENSIP-24)](#arbitrary-bytes-records-ensip-24)
- [Chain-Registry Resolver (`<chain>.on.eth`)](#chain-registry-resolver-chainoneth)
- [Footguns](#footguns)

---

## Namehash mechanics

**Spec**: [ENSIP-1](https://docs.ens.domains/ensip/1).

Namehash is a recursive keccak256 over normalized labels:

```
namehash(""):           0x000…000
namehash("eth"):        keccak256(namehash("") || keccak256("eth"))
namehash("alice.eth"):  keccak256(namehash("eth") || keccak256("alice"))
namehash("a.b.c"):      keccak256(namehash("b.c") || keccak256("a"))
```

Every label MUST be normalized (ENSIP-15) before hashing — see [normalization.md](normalization.md). Compute the hash of an unnormalized label and you write to a node nobody resolves to.

### `namehash` vs `labelhash` — don't conflate them

- **`namehash(name)`** — recursive 32-byte node identifier for a *full name*. Every registry / resolver call takes this as `node`.
- **`labelhash(label)`** — just `keccak256(label)` for a *single label*. Used inside namehash, and taken directly by some registrar functions (e.g. `register(labelhash, owner, ...)`).

Symptoms of mixing them:

- `resolver.addr(labelhash("vitalik"))` instead of `resolver.addr(namehash("vitalik.eth"))` → returns zero. "Name not found." Silent.
- `registrar.register(namehash("alice.eth"), …)` instead of `registrar.register(labelhash("alice"), …)` → reverts or registers the wrong thing.

Always normalize the *full* name once with `ens_normalize`, then derive whichever hash the function expects. Use library helpers (`viem.namehash`, `viem.labelhash`, `ethers.namehash`) — they don't normalize for you.

---

## Text records

**Spec**: [ENSIP-5](https://docs.ens.domains/ensip/5) (global + service keys). **Extended**: [ENSIP-18](https://docs.ens.domains/ensip/18).

Generic key-value strings on a name. Resolver interface:

```solidity
function text(bytes32 node, string key) returns (string);
function setText(bytes32 node, string key, string value);
```

Keys are convention, not enforced. Common standardized keys:

**Global (ENSIP-5)**: `avatar`, `description`, `display`, `email`, `keywords`, `mail`, `notice`, `location`, `phone`, `url`

**Service keys** (reverse-DNS notation, ENSIP-5): `com.github`, `com.twitter`, `com.linkedin`, `com.discord`, `io.keybase`, `org.telegram`

**Extended profile (ENSIP-18)**: `alias`, `theme`, `header`, `timezone`, `language`, `primary-contact`

```ts
import { useEnsText } from 'wagmi'

const { data: twitter } = useEnsText({ name: 'alice.eth', key: 'com.twitter', chainId: 1 })
```

For multiple records on one name, multicall via the Universal Resolver instead of N hooks (see [resolution.md → batch](resolution.md#batch-resolution)).

---

## Avatars

**Spec**: [ENSIP-12](https://docs.ens.domains/ensip/12). **Guide**: [docs.ens.domains/web/avatars](https://docs.ens.domains/web/avatars).

The `avatar` text record is a URI. Allowed schemes:

| Scheme | Resolution |
|---|---|
| `https://...` | Direct image URL. Reject HTML responses. |
| `ipfs://<cid>` | Resolve through an IPFS gateway (e.g., dweb.link). Optionally rewrite to `https://`. |
| `data:image/...` | RFC 2397 inline data URI. Validate MIME type. |
| `eip155:<chain>/erc721:<contract>/<tokenId>` | NFT avatar. Resolve via the NFT contract. **Verify ownership.** |
| `eip155:<chain>/erc1155:<contract>/<tokenId>` | Same as above but ERC-1155. |

### NFT avatars: verify ownership

Per ENSIP-12: the NFT must be owned by the address the ENS name resolves to. If you skip this check, anyone can set their `avatar` to someone else's NFT and you'll display it as theirs.

```
1. Parse the eip155 URI → chainId, contract, tokenId, standard (erc721/erc1155).
2. Forward-resolve the ENS name → expected owner address.
3. Call ownerOf(tokenId) on the contract (or balanceOf for ERC-1155 > 0).
4. If owner !== expected owner → drop the avatar. Render fallback.
```

`useEnsAvatar` / `getEnsAvatar` in viem do this for you. If you write a custom avatar resolver, do not skip step 4.

### Naive `<img src={avatar}>` is wrong

A raw text-record value can be an `eip155:` URI, an `ipfs://` URI, or a data URI — none of which `<img src>` handles. Always pipe the value through your library's `getEnsAvatar` (which returns a resolvable HTTPS URL or null), or implement scheme handling yourself.

### Display tips

- Use `useEnsAvatar` with a normalized name.
- Show a fallback (gradient blockie or default icon) on `null`.
- Validate image dimensions / size on the client to avoid layout jumps.
- Cache aggressively; avatars rarely change.

---

## Reverse resolution

**The flow** (`address → name`):

1. Compute `<addr-lowercased-without-0x>.addr.reverse` — e.g., `1234…abcd.addr.reverse`.
2. Resolve that name like any forward name → returns the claimed primary name.
3. **Forward-resolve the claimed name** → must equal the original address.
4. Only if it matches, render the name.

### Why forward verification is non-negotiable

Anyone can call `setName('vitalik.eth')` on the Reverse Registrar from their address. There is no permission check that the caller actually owns `vitalik.eth`. The only proof is forward resolution: does `vitalik.eth` actually resolve to your address? If yes, the claim is real. If no, you're looking at a spoof.

> "You **must** verify it by performing a forward resolution on that name to confirm it still resolves to the original address." — [docs.ens.domains/web/reverse](https://docs.ens.domains/web/reverse)

**Library behavior — verify yours**:

| Library | Default behavior |
|---|---|
| **viem** `getEnsName` (and `useEnsName` in wagmi) | Verifies by default via the Universal Resolver's `reverse()`. |
| **ethers v6** `provider.lookupAddress()` | Historically did *not* verify automatically — check your version's docs and add the forward check yourself if it doesn't. |
| **Custom code** walking `addr.reverse` manually | Does not verify unless you write the second resolution. Don't write this. |

**If your library doesn't verify, do it yourself:**

```ts
import { createPublicClient, http } from 'viem'
import { mainnet } from 'viem/chains'
import { normalize } from 'viem/ens'

const client = createPublicClient({ chain: mainnet, transport: http() })

async function safeGetEnsName(address: `0x${string}`) {
  const name = await client.getEnsName({ address })
  if (!name) return null

  // Forward-verify even if the library claims to — defense in depth.
  const forward = await client.getEnsAddress({ name: normalize(name) })

  // Address comparison is case-insensitive — checksum casing is informational.
  if (forward?.toLowerCase() !== address.toLowerCase()) {
    return null   // claim does not verify; treat as no name
  }
  return name
}
```

**The case-sensitivity rule**: always `.toLowerCase()` both sides when comparing addresses. EIP-55 checksum casing is a *display* convention; equality must be done lowercased. A non-checksummed address like `0xabc…` is the same address as `0xAbC…`.

**Best advice**: don't write your own reverse code. Use the library function. If you must, follow the pattern above.

---

## L2 primary names

**Spec**: ENSIP-19 (L2 reverse resolution). **Guide**: [docs.ens.domains/web/reverse](https://docs.ens.domains/web/reverse).

### What changed

Historically, primary names lived only on Ethereum mainnet (`addr.reverse`). With L2 adoption, ENS deployed per-chain Reverse Registrars on Base, Optimism, Arbitrum, Linea, Scroll, and other supported L2s. A user can now have *different* primary names on each chain.

The L2 reverse namespace is `<coinType-hex>.reverse`. For Optimism (chainId 10), the coinType is `0x80000000 | 10 = 0x8000000A`, so the namespace is `8000000a.reverse`.

### How dapps should resolve a primary name

The default of "always query mainnet `addr.reverse`" is **wrong** in a multi-L2 world. The correct order:

```
1. Determine the chain the user is *active on* (the chain they're connected to, or the
   chain you're sending on). That is the chain whose reverse record represents the
   user's chosen identity in this context.
2. Query the Reverse Registrar on THAT chain for the address.
3. Forward-verify the returned name on the same chain.
4. If no name is set on that L2, fall back to mainnet (the user's default identity).
```

In viem, pass `chainId` to `getEnsName` to target a specific L2's reverse registrar. wagmi inherits this — pass `chainId` to `useEnsName`.

### UX implications

- A user's "name" on Base may differ from their name on mainnet. **Don't assume one stable identity across chains.**
- For a multichain leaderboard, decide whether you display the chain-specific primary or the mainnet default — and apply that choice consistently.
- For a single-chain dapp, prefer the chain-specific primary; fall back to mainnet only if no chain-specific record exists.
- Showing the wrong chain's name is not just a cosmetic miss — it can confuse users about which identity they're transacting under.

---

## Content hash (ENSIP-7)

**Spec**: [ENSIP-7](https://docs.ens.domains/ensip/7).

Protocol-tagged pointer to distributed content. Wire format: `<protoCode uvarint><value bytes>`.

| Codec | Code (hex) | Use |
|---|---|---|
| IPFS | `0xe3` | CIDv0/v1 → fetch via IPFS gateway |
| IPNS | `0xe5` | mutable IPFS pointer |
| Swarm | `0xe4` | Swarm hash |
| Onion (Tor v3) | `0xbd` | onion address |
| Skynet | `0x90` | Sia Skynet |
| Arweave | varies | Arweave tx ID |

Use it instead of a `url` text record when:
- The content is content-addressed and protocol-routable rather than a centralized HTTP URL.
- You want browsers like Brave to render `.eth` names with `contenthash` set to IPFS as decentralized websites.

Use library helpers (viem `getEnsContentHash`, ethers `resolver.getContentHash()`) — the codec byte is parsed for you.

---

## ABI text record (ENSIP-4)

**Spec**: [ENSIP-4](https://docs.ens.domains/ensip/4).

Stores a contract ABI on a name's resolver. Encoding types are a bitfield:

| Type | Encoding |
|---|---|
| `1` | JSON (human-readable) |
| `2` | zlib-compressed JSON |
| `4` | CBOR |
| `8` | URI (external pointer) |

Resolver: `ABI(bytes32 node, uint256 contentType) → (uint256, bytes)`. Pass a bitfield of acceptable encodings; the resolver returns the first match.

Use case: dapps fetch the ABI for a contract by ENS name without a separate registry. Pairs naturally with [contract naming](smart-contracts.md#naming-a-smart-contract).

---

## Arbitrary bytes records (ENSIP-24)

**Spec**: [ENSIP-24](https://docs.ens.domains/ensip/24).

Generic `data(bytes32 node, bytes32 key) → bytes` — the binary analog of text records.

Use cases:
- Hashed commitments (proof-of-presence, voting receipts).
- Interoperable address formats not covered by ENSIP-9/-11.
- App-specific binary metadata (signatures, encrypted blobs).
- Storage that text records can't hold (raw bytes, zero-byte values, non-UTF-8 data).

When **NOT** to use: if your data is human-readable, a text record is simpler, indexable in the subgraph, and broadly supported.

Optional companion `ISupportedDataKeys(bytes32 node) → bytes32[]` lets a resolver enumerate supported keys.

---

## Chain-Registry Resolver (`<chain>.on.eth`)

**Docs**: [docs.ens.domains/resolvers/chain-registry-resolver](https://docs.ens.domains/resolvers/chain-registry-resolver).

A resolver where each label under `on.eth` represents a chain — `optimism.on.eth`, `base.on.eth`, `arbitrum.on.eth`. Records hold chain metadata: ERC-7930 interoperable addresses, text records, multichain addresses, contenthash, arbitrary data.

When to use:
- A dapp that needs canonical chain metadata (RPC endpoints, native token info, explorer URLs) without maintaining its own chain list.
- An agent mapping "the user said Base" → chain ID + RPC.

The registry is enumerable; chain admins (designated by the registry owner) manage their chain's records.

---

## Footguns

1. **Hashing an unnormalized label.** Writes to a node nobody can resolve.
2. **Confusing `namehash` and `labelhash`.** See callout above.
3. **Skipping forward verification** in custom reverse-resolution code. Spoof city.
4. **`<img src={textRecord}>`** for avatars. Doesn't handle IPFS/data/NFT URIs. Use the library's `getEnsAvatar`.
5. **Not verifying NFT ownership** before rendering an `eip155:` avatar.
6. **Querying mainnet `addr.reverse` on an L2 dapp** when the user has set a chain-specific primary. You'll show the wrong name (or none).
7. **Assuming all profile keys exist** — most names have only a couple of records set. Render gracefully on null.
8. **N+1 hooks** for showing many users' profiles. Multicall via the Universal Resolver.
9. **Storing a URL in `contenthash`.** That's the wrong record. `contenthash` is binary-tagged; `url` is the text record for HTTP.
10. **ABI lookups assuming type=1 (JSON).** Real resolvers may only have CBOR or zlib-JSON. Pass a bitfield of all you can decode.
11. **Using `data(node, key)` for things text records cover.** Less interop, less indexing.
12. **Setting records before setting a resolver on the node.** No-op. The registry must point at a resolver first.

## Sources

- [docs.ens.domains/web/reverse](https://docs.ens.domains/web/reverse)
- [docs.ens.domains/web/avatars](https://docs.ens.domains/web/avatars)
- [ENSIP-1 — Namehash](https://docs.ens.domains/ensip/1)
- [ENSIP-4 — ABI Records](https://docs.ens.domains/ensip/4)
- [ENSIP-5 — Text Records](https://docs.ens.domains/ensip/5)
- [ENSIP-7 — Content Hash](https://docs.ens.domains/ensip/7)
- [ENSIP-12 — Avatar Text Records](https://docs.ens.domains/ensip/12)
- [ENSIP-18 — Extended Profile Records](https://docs.ens.domains/ensip/18)
- [ENSIP-19 — L2 Reverse Resolution](https://docs.ens.domains/ensip/19)
- [ENSIP-24 — Arbitrary Data Resolver](https://docs.ens.domains/ensip/24)
- [docs.ens.domains/resolvers/chain-registry-resolver](https://docs.ens.domains/resolvers/chain-registry-resolver)
