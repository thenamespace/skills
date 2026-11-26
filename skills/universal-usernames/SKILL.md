---
name: universal-usernames
description: Build the HTTPS profile endpoint for Namespace Universal Usernames, which turns an app's existing usernames into ENS subnames (happy in the app becomes happy.brand.eth). Covers username normalization, the profile v1 response, text records, coin-type addresses, contenthash, bearer-token auth, speed limits and security. Use when building, reviewing or debugging a Universal Usernames endpoint, mapping a user table to ENS records, or when a username fails to resolve.
---

# Universal Usernames endpoint

A brand owns an ENS name such as `brand.eth`. With Universal Usernames, every subname of it (`happy.brand.eth`) resolves through the Namespace gateway, which calls the brand's endpoint:

```
wallet → ENS → Namespace gateway → GET {base URL}/{username} → brand endpoint → user store
```

The gateway turns the JSON the endpoint returns into ENS records. A user exists in ENS if and only if the endpoint answers `200` for them.

## Before writing code, answer these

Read the codebase and answer each one. Don't guess: every answer changes the code.

1. **Where do usernames live?** The endpoint reads from the store the app already has. Never add a new database, table, cache or mirror for it: a second copy goes stale, and a stale copy serves the wrong user. The only schema change is the `ens_label` column (see Names), plus two columns for renames if the app allows them.
   ```
   App's own database                     → query the users table directly
   Auth provider, and the app already     → query that synced table
     syncs users into its own database
   Auth provider only                     → query the provider's user API by username, then
                                            check toEnsLabel(username) === requested label.
                                            Store ens_label in the provider's user metadata
                                            if it can search by metadata.
   Registry contract                      → read the contract, or the indexer the app
                                            already uses
   ```
2. **Can usernames change, be released or be reused?** If yes, build the rename tree under Names.
3. **Which profile fields are already public in the app?** Public means a signed-out visitor sees it on the user's profile page today. Only those fields become text records. A field that isn't on that page stays out, even if it seems harmless (a plan or tier, a join date).
4. **Does the app hold wallet addresses, and did users prove control of them?** Proven means the user signed a message with that wallet (SIWE, SIWS), or it's a non-custodial embedded wallet whose keys the user can export. A custodial wallet the app holds keys for counts only if the app already shows it to the user as their deposit address. A pasted address is never proven. Only proven addresses go in `addresses`.
5. **Are any accounts AI agents?** Only then do agent records apply (see Agent records).
6. **Does the user have a site on IPFS, IPNS, Arweave, Swarm, Tor or Sia?** Only then is `contenthash` set.

## Names

Each username becomes exactly one ENS label and must pass ENSIP-15 normalization. Wallets normalize before they look a name up, so the endpoint only ever receives normalized labels.

**Store a normalized copy** of every username in its own column (`ens_label`), computed when the username is created or changed, with a unique index. Look users up by that column only. Never normalize at request time against the raw username column. That forces a scan or a case-insensitive match that diverges from ENSIP-15, and it's slow.

```ts
import { normalize } from 'viem/ens';

/** The ENS label for a username, or null when it can't be one. */
export function toEnsLabel(username: string): string | null {
  try {
    const label = normalize(username);
    return label && !label.includes('.') ? label : null;
  } catch {
    return null;
  }
}
```

Walk this for every username:

```
toEnsLabel(username)
├── throws or returns null → store null. This user gets no ENS name. Don't rewrite their username.
└── returns a label
    ├── another user already has this label (Happy vs happy)
    │   └── the account whose username was set earliest keeps it (account creation
    │       time if that isn't recorded); store null for the other. A later signup or
    │       rename never takes a label from its holder. Never let the endpoint pick
    │       between two users at request time.
    └── unique → store it
```

Why null instead of fixing the username (`happy_doe` → `happy-doe`): a rewritten name can collide with a real user, and the user never chose it. Sending payments to a name the user didn't pick isn't acceptable.

After the backfill, report to the user how many usernames got null and give examples, so they know who has no ENS name. Typical nulls: `happy_doe` (an underscore is allowed only as the first character), `happy doe`, `@happy`, `happy!`, `ab--cd`, `happy.doe`. `Happy` isn't a null: it maps to `happy`.

For a new app, allow only `a-z`, `0-9` and `-` in usernames, so every username is valid. Existing usernames that break this rule keep their label if `toEnsLabel` accepts them (`_zed`).

Renames and releases use two more columns on the same users row, `released_ens_label` and `released_at`. Walk this whenever a username changes or an account goes away:

```
User renames, or the account is deleted
└── In the same update: copy ens_label into released_ens_label, set released_at = now,
    then set ens_label to the new label (or null). The old name 404s at once.
    Soft-delete accounts: a hard delete loses the hold.
New username normalizes to a label that some row holds in released_ens_label
├── released under 90 days ago
│   ├── same row        → give it back
│   └── different row   → store null for now
└── 90 days or older    → clear that row's hold and assign normally
```

Why 90 days: wallets and contacts keep sending to `happy.brand.eth` after a rename. If the label passes to someone else straight away, those payments reach a stranger.

## Request

```http
GET {base URL}/{url-encoded normalized label}
Authorization: Bearer <token>
Accept: application/json
Accept-Encoding: identity
User-Agent: Namespace-UniversalUsernames/1
```

- **Base URL:** HTTPS with a valid certificate, on a public IP. A path is allowed (`https://api.brand.example/ens`). No query string, fragment or `user:pass@`.
- **Route:** one path segment after the base URL. Mount the endpoint on its own subdomain (`api.brand.example`) or a dedicated path (`/ens`), never at the root of an app that has its own routes. URL-decode the segment, then 404 unless `toEnsLabel(segment) === segment`. That guard costs nothing and skips a database query for junk.
- **Auth:** the brand saves a bearer token in the Namespace App next to the base URL. Always require it. Read it from an environment variable, compare in constant time, and reply `401` to anything else. A token for one origin isn't carried over when the base URL moves to a new origin, so save it again after changing hosts.
- **Methods:** only `GET`. The gateway never sends a body or query parameters.

## Response

Pick the status with this tree. The gateway treats every other outcome as a failed lookup, and a failure isn't the same as "not found".

```
OPTIONS from https://app.namespace.ninja                → 204 + CORS headers (App test only)
method isn't GET                                        → 405
token missing or wrong                                  → 401
segment isn't a normalized label                        → 404
no user with that ens_label                             → 404
user is private, suspended, banned, deleted or unverified → 404
user found and public                                   → 200 + profile
your database or code threw                             → 500, logged (lookup fails; never fake a 404)
```

Only `404` means "no such name". `3xx`, `401`, `403`, `429`, `5xx`, a timeout or a non-JSON body make the lookup fail, and wallets show an error. So never redirect, and never rate-limit requests that carry the valid token. The gateway already caches each answer and merges concurrent lookups for the same name.

Body for `200`, served as `Content-Type: application/json`, uncompressed:

```json
{
  "version": 1,
  "username": "happy",
  "texts": {
    "name": "Happy",
    "alias": "Happy",
    "avatar": "https://cdn.brand.example/avatars/happy.png",
    "description": "Building on brand",
    "com.twitter": "happy",
    "com.brand.tier": "gold"
  },
  "addresses": {
    "2147483648": "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045"
  },
  "contenthash": "ipfs://bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi"
}
```

| Rule | Breaks |
| --- | --- |
| `version` is the number `1` | whole profile |
| `username` equals the requested label byte for byte. Echo the path segment, never the stored display username | whole profile |
| `texts` and `addresses` are objects, `{}` when empty. Never `null` or missing | whole profile |
| No other top-level fields (no `id`, `createdAt`, `meta`) | whole profile |
| Body under 256 KiB, at most 128 texts and 128 addresses | whole profile |
| Text keys 1–256 characters, values are strings up to 4096 | that record |
| Address values are non-empty strings up to 1024, valid for their coin type | that record |
| `contenthash` is a supported URI up to 4096, or the field is left out | that record |

Build the object from an explicit allowlist of fields. Never spread a user row into the response. A spread leaks columns the day someone adds one, and an unknown top-level field fails the profile.

## Text records

`texts` holds everything that isn't an address. Map each public user field with this tree:

```
Is there a standard key for it? (table below)
├── Yes → use that key
└── No
    ├── Is it public and useful outside the app (tier, member-since, role)?
    │   ├── Yes → custom key: reverse domain of the brand + field, e.g. com.brand.tier
    │   └── No  → leave it out
    └── Is the user an AI agent? → also see Agent records
```

| Key | Value |
| --- | --- |
| `name` and `alias` | Display name. Set both to the same value: ENSIP-18 renamed `name` to `alias`, and many apps still read only `name`. |
| `avatar` | `https://` or `ipfs://` image URL, or an NFT as `eip155:1/erc721:0x…/1` (ENSIP-12). Use a stable URL. If the app only has signed or expiring URLs, serve a route on your own domain that redirects to a fresh one, or leave `avatar` out. Never sign a URL during the lookup. |
| `header` | Banner image, same formats as `avatar` |
| `description` | Short bio |
| `url` | Website. Send only `https://` URLs and drop anything else: `javascript:` or `http:` in a public record is a phishing vector |
| `email` | Only if the user switched on showing it on their public profile. Indexers keep it after you remove it, so never send it on an implied or default setting |
| `location`, `timezone` | `Berlin, Germany`, `Europe/Berlin` (IANA) |
| `com.twitter`, `com.github`, `org.telegram`, `com.discord` | Handle only: no `@`, no URL |

Rules for every text record:

- **Omit a key instead of sending `""`.** An empty string still resolves, and apps render a blank field.
- **Never send `null`, numbers or booleans.** Convert to a string or omit. Dates are `YYYY-MM-DD` in UTC, booleans are `"true"` or `"false"`. A non-string value fails that record.
- **Custom keys:** lowercase, the brand's website domain reversed (`kitebase.xyz` → `xyz.kitebase.plan`), never the ENS name, hyphens between words (`com.brand.member-since`). Never invent a custom key for something a standard key covers (no `com.brand.twitter`).
- **Linked accounts:** only send a social handle the user connected through OAuth, never free text from a profile field. Apps render these records as the user's own account, so a typed-in handle lets anyone claim someone else's. When the user disconnects the account, clear the stored handle in the same step.

### Agent records (only for agent accounts)

Add these if and only if the account is an AI agent. A human user never gets them. An agent also gets the standard keys above. These follow draft specs (ENSIP-26, ENSIP-25).

| Key | When | Value |
| --- | --- | --- |
| `agent-context` | Always, for an agent | What the agent does and how to talk to it, in plain text. Reusing `description` is fine |
| `agent-endpoint[<protocol>]` | For each protocol the agent serves: `mcp`, `a2a`, `oasf`, `web` | The endpoint URL, `https://` only, e.g. `agent-endpoint[mcp]` = `https://agents.brand.example/happy/mcp`. Send only endpoints the app runs or has verified, for the same reason as linked accounts |
| `agent-registration[<registry>][<agentId>]` | Only when the app registered the agent in an ERC-8004 registry and knows that registry's chain and address | `"1"`. `<registry>` is the registry's ERC-7930 interoperable address in lowercase hex, `<agentId>` is decimal: `agent-registration[0x000100000101148004a169fb4a3325136eb29fa0ceb6d2e539a432][42]`. Missing the registry or the ID → leave the key out |

## Addresses

`addresses` maps a coin type to an address. The key is the coin type as a decimal string, and the value is the plain address string the user would paste into a wallet. The gateway encodes it to bytes.

### Working out the coin type

Pick the rule by chain family. Never guess a number: a wrong coin type puts the address on the wrong chain.

```
Is the chain EVM (Ethereum, Base, Optimism, Arbitrum, Polygon, …)?
├── Yes
│   ├── Same address on every EVM chain (most users) → 2147483648 only
│   │     This is the default EVM address (ENSIP-19, chainId 0). It also answers
│   │     for Ethereum mainnet, so never add 60 with the same address.
│   └── A different address on one chain → 0x80000000 | chainId (ENSIP-11)
│         In code: (0x80000000 | chainId) >>> 0, or viem's toCoinType(chainId).
│         Base 8453 → 2147492101, Optimism 10 → 2147483658, Arbitrum 42161 → 2147525809
└── No → the chain's SLIP-44 number
          Bitcoin 0, Litecoin 2, Dogecoin 3, Solana 501.
          For any other chain, look the number up in the SLIP-44 registry:
          https://github.com/satoshilabs/slips/blob/master/slip-0044.md
```

| Chain | Key | Value format |
| --- | --- | --- |
| Every EVM chain (default) | `2147483648` | `0x…`, checksummed with viem's `getAddress` |
| One EVM chain | `0x80000000 \| chainId` in decimal | `0x…`, checksummed |
| Bitcoin | `0` | Mainnet only: `bc1…`, `1…` or `3…`. Never a testnet `tb1…` |
| Solana | `501` | Base58 public key, 32–44 characters |

### Which addresses to send

```
For each wallet on the user's record:
├── Not proven (pasted, imported, unverified) → leave it out
├── Fails the format check for its chain      → leave it out, log it
└── Proven and valid                          → add it under its coin type
```

- **Proven** means the user signed a message with that wallet (SIWE, SIWS), or it's a non-custodial embedded wallet whose keys the user can export. This record decides where money goes, so a pasted address would let anyone redirect payments meant for that user.
- **Validate before sending and drop only the bad record.** A thrown error becomes a `5xx` and fails the whole lookup. EVM: `isAddress`, then `getAddress`. Other chains: a format check is enough, because the gateway decodes the address fully and a value it rejects fails only that record.
- An invalid value under a chain-specific key doesn't fall back to the default address, so that chain resolves to nothing.

## Contenthash

Optional. Set it only when the user has a decentralized site. Send the URI, and the gateway encodes it.

| Network | Format |
| --- | --- |
| IPFS | `ipfs://<CID>`, v0 (`Qm…`) or v1 (`bafy…`) |
| IPNS | `ipns://<key or name>` |
| Arweave | `ar://<43-character transaction ID>` |
| Swarm | `bzz://<64 hex characters>` |
| Tor | `onion3://<56-character address>` |
| Sia | `sia://<46-character skylink>` |

If the app stores a bare CID, prefix it with `ipfs://`. Never send an `https://` gateway URL or `null`. If there's no site, leave the field out. The gateway validates the hash, and an invalid one fails only that record.

## Speed

The endpoint answers in under 300 ms. The gateway's hard limit is 5 seconds, but that covers DNS, TLS and the whole round trip, and a wallet's send screen sits waiting on every lookup. Over the limit, the lookup fails.

- One indexed read by `ens_label`. Join or denormalize the profile fields so it's a single query.
- The user lookup is the only outbound call a request makes. Never fetch the avatar to check it, call an RPC to verify a wallet, or refresh OAuth during a lookup. Do that work when the user saves their profile, and store the result on the user row.
- Deploy in the same region as the app's database. On serverless, use an always-warm runtime or keep one instance warm: a 1–2 s cold start eats most of the budget.
- Don't add a cache. The gateway already caches each answer for up to 60 s, so profile edits show up within a minute. A cache on your side delays edits and keeps serving accounts that went private.

## Build steps

1. Answer the questions under Before writing code.
2. Add the `ens_label` column with a unique index, backfill it with `toEnsLabel` and the collision rule, and set it wherever usernames are created or changed.
3. Write the handler. Follow `references/example-endpoint.md` in this skill: one Cloudflare Worker file, with the user lookup left for you to point at your existing API or database.
4. Deploy, then check from a shell:
   ```sh
   curl -i -H "Authorization: Bearer $TOKEN" https://api.brand.example/happy
   curl -i https://api.brand.example/happy           # expect 401
   curl -i -H "Authorization: Bearer $TOKEN" https://api.brand.example/nobody   # expect 404
   ```
5. On the Universal Usernames page of the Namespace App, save the base URL and token, then use **Test username**. The test runs in the browser, so it needs CORS for `https://app.namespace.ninja` with the `Authorization` header allowed. The example handler already does this. Real lookups don't come from a browser, so CORS doesn't affect resolution.
6. Set the resolver and enable Universal Usernames in the App. Existing offchain or L2 subnames of the name stop resolving when you do.

## Good practices

- **Everything returned is public.** Anyone can guess a username and read the profile, and indexers copy what they read, so removing a field later doesn't take it back. Return only what the user already shows publicly in the app.
- **Return `404` for private, suspended, banned and deleted accounts**, the same response as for an unknown user, so the endpoint doesn't reveal which accounts exist.
- **Never reassign a released label inside 90 days.** Follow the rename tree under Names.
- **Treat the endpoint like a payments API.** Change it through code review, log every `5xx`, and alert on error rates. To rotate the token, set the new one as the current token and the old one as the previous token, save the new one in the App, then delete the previous one.
- **The wallet that owns `brand.eth` controls every username.** Keep it on a hardware wallet or a multisig.
- **Send every public field that passes the Text records tree, and nothing else.** Up to 20 records needs no review. Past 20, review the custom keys and keep only the ones another app would show or act on. Apps don't render the rest, and every extra record adds bytes, response time and exposure. Never drop a standard key or a proven address to get under 20.
