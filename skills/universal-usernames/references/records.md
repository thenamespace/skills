# Records

How to fill `texts`, `addresses`, and `contenthash`. Source of truth: https://docs.namespace.ninja/developer-guide/universal-usernames/records

## Text records (`texts`)

Open keys, string values. Standard keys wallets read:

| Key | Holds |
| --- | --- |
| `name` | Display name |
| `description` | Short bio |
| `avatar` | `https://` or `ipfs://` image URL, or an ENSIP-12 NFT reference |
| `header` | Banner image, same formats as `avatar` |
| `url` | Website |
| `location` | Free text |
| `email` | Email address |
| `com.github` | GitHub username |
| `com.twitter` | X username |

- Custom keys: prefix with a domain the brand controls, such as `com.brand.role`.
- Keys are 1 to 256 characters (`TEXT_KEY_INVALID`). Values are strings (`TEXT_VALUE_INVALID` otherwise) of at most 4,096 characters (`TEXT_VALUE_TOO_LONG`). At most 128 keys (`TOO_MANY_TEXTS` fails the whole profile).
- Values are plain text. Strip markup before publishing.

## Address records (`addresses`)

Keys are canonical decimal ENS coin types (ENSIP-9 for non-EVM, ENSIP-11 `0x80000000 | chainId` for EVM, `60` for Ethereum mainnet). Values are addresses in the form users see.

| Chain | Coin type |
| --- | --- |
| Bitcoin | `0` |
| Litecoin | `2` |
| Dogecoin | `3` |
| Ethereum | `60` |
| Polkadot | `354` |
| Solana | `501` |
| Vara | `913` |
| Default EVM address | `2147483648` |
| OP Mainnet | `2147483658` |
| BNB Smart Chain | `2147483704` |
| Gnosis | `2147483748` |
| Polygon | `2147483785` |
| zkSync Era | `2147483972` |
| Base | `2147492101` |
| Arbitrum One | `2147525809` |
| Celo | `2147525868` |
| Avalanche C-Chain | `2147526762` |
| Linea | `2147542792` |
| Scroll | `2148018000` |

For any other EVM chain, compute `2147483648 + chainId`. Chain IDs are a different number space: `8453` has no coder, and `137` is Rootstock, not Polygon.

Formats:

- EVM: `0x` plus 40 hex characters, EIP-55 checksummed or all lowercase. A mixed-case address with a wrong checksum is `ADDRESS_INVALID`.
- Bitcoin: base58 or bech32 (`bc1q…`).
- Solana: base58 public key.

Errors: a key that isn't a canonical decimal in `0..4294967295` (`"ETH"`, `"0x3c"`, `"060"`) is `COIN_TYPE_INVALID`; a coin type without an `@ensdomains/address-encoder` 2.0.1 coder is `COIN_TYPE_UNSUPPORTED`; a value that doesn't decode for its coin, or is empty or longer than 1,024 characters, is `ADDRESS_INVALID`. At most 128 addresses (`TOO_MANY_ADDRESSES` fails the whole profile).

### Default EVM address (ENSIP-19)

`2147483648` is the default EVM address. Namespace applies ENSIP-19 exactly:

- An explicit chain record wins.
- An EVM chain without its own record returns the default. That includes Ethereum: ENSIP-19 defines `chainFromCoinType(60) = 1`, so a profile without `"60"` answers Ethereum lookups with the default.
- An invalid explicit record fails the lookup; it doesn't fall back.
- Non-EVM coins (Bitcoin, Solana, …) never use the default.

If a user has one address on every EVM chain, send only `"2147483648"` plus their non-EVM addresses.

## Contenthash (`contenthash`)

Optional URI, encoded to ENSIP-7 bytes with `content-hash` 3.1.0-rc.1:

| Protocol | Example |
| --- | --- |
| IPFS | `ipfs://QmYwAPJzv5CZsnAzt8auVZRnGiRAzQmmJYDh4T7SEBuwNA`, `ipfs://bafy…` |
| IPNS | `ipns://k51qzi5uqu5dlvj2baxnqndepeb86cbk3ng7n3i46uzyxzyqj2xjonzllnv0v8` |
| Swarm | `bzz://d1de9994b4d039f6548d191eb26786769f580809256b4685ef316805265ea162` |
| Arweave | `ar://ys32Pt8uC7TrVxHdOLByOspfPEq2LO63wREHQIM9SJQ` |

A missing or unknown protocol (a bare CID, `https://…`) is `CONTENTHASH_UNSUPPORTED`; a malformed value after a known protocol is `CONTENTHASH_INVALID`. Omit `contenthash` when the user has none.

## Missing records

Leave them out. The gateway returns the ENS empty value for anything the profile lacks. `null` is an invalid record, not an empty one.
