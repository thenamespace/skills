# Profile: text records, avatars, reverse, L2 primary names

How to display a user — name, avatar, socials — and how reverse resolution actually works (including the trap that bites everyone).

## Contents

- [Standardized text record keys](#standardized-text-record-keys)
- [Reverse resolution](#reverse-resolution)
- [Avatars](#avatars)
- [L2 primary names](#l2-primary-names)
- [Footguns](#footguns)

---

## Standardized text record keys

Defined by [ENSIP-5](https://docs.ens.domains/ensip/5) (global keys + service keys) and [ENSIP-18](https://docs.ens.domains/ensip/18) (extended profile fields).

**Global keys (ENSIP-5):**
- `avatar`, `description`, `display`, `email`, `keywords`, `mail`, `notice`, `location`, `phone`, `url`

**Service keys (reverse-DNS notation, ENSIP-5):**
- `com.github`, `com.twitter`, `com.linkedin`, `com.discord`, `com.peepeth`, `io.keybase`, `org.telegram`

**Extended profile keys (ENSIP-18):**
- `alias`, `theme`, `header`, `timezone`, `language`, `primary-contact`

The keys are not enforced by the resolver — they're a convention. Anything goes; agree on common keys for interop.

```ts
import { useEnsText } from 'wagmi'

const { data: twitter } = useEnsText({ name: 'alice.eth', key: 'com.twitter', chainId: 1 })
```

For multiple records on one name, multicall via the Universal Resolver instead of N hooks (same pattern as [batch resolution](resolution.md#batch-resolution)).

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

## Footguns

1. **Skipping forward verification** in custom reverse-resolution code. Spoof city.
2. **`<img src={textRecord}>`** for avatars. Doesn't handle IPFS/data/NFT URIs. Use the library's `getEnsAvatar`.
3. **Not verifying NFT ownership** before rendering an `eip155:` avatar.
4. **Querying mainnet `addr.reverse` on an L2 dapp** when the user has set a chain-specific primary. You'll show the wrong name (or none).
5. **Assuming all profile keys exist** — most names have only a couple of records set. Render gracefully on null.
6. **N+1 hooks** for showing many users' profiles. Multicall.

## Sources

- [docs.ens.domains/web/reverse](https://docs.ens.domains/web/reverse)
- [docs.ens.domains/web/avatars](https://docs.ens.domains/web/avatars)
- [ENSIP-5 — Text Records](https://docs.ens.domains/ensip/5)
- [ENSIP-12 — Avatar Text Records](https://docs.ens.domains/ensip/12)
- [ENSIP-18 — Extended Profile Records](https://docs.ens.domains/ensip/18)
- [ENSIP-19 — L2 Reverse Resolution](https://docs.ens.domains/ensip/19)
