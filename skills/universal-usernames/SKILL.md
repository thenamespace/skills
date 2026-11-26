---
name: universal-usernames
description: Build the HTTPS profile endpoint for Namespace Universal Usernames, which turns an app's existing usernames into ENS subnames (happy in the app becomes happy.brand.eth). Covers username normalization, the profile v1 response, text records, coin-type addresses, contenthash, bearer-token auth, speed limits, tests and security. Use when building, reviewing or debugging a Universal Usernames endpoint, mapping a user table to ENS records, or when a username fails to resolve.
---

# Universal Usernames endpoint

A brand owns an ENS name such as `brand.eth`. With Universal Usernames, every subname of it (`happy.brand.eth`) resolves through the Namespace gateway, which calls the brand's endpoint:

```
wallet → ENS → Namespace gateway → GET {base URL}/{username} → brand endpoint → user store
```

The gateway turns the JSON the endpoint returns into ENS records. Nobody mints anything. A user exists in ENS if and only if the endpoint answers `200` for them.

## Before writing code, answer these

Read the codebase and answer each one. Don't guess: every answer changes the code.

1. **Where do usernames live?** A database table, an auth provider, an API or a registry contract. The endpoint reads from that source directly.
2. **Can usernames change, be released or be reused?** That decides the release rule under Good practices.
3. **Which profile fields are already public in the app?** Only those become text records.
4. **Does the app hold wallet addresses, and did users prove control of them** (a signed message, SIWE, an embedded wallet the app created)? Only proven addresses go in `addresses`.
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
    │   └── the account that held the label first keeps it; store null for the other.
    │       Never let the endpoint pick between two users at request time.
    └── unique → store it
```

Why null instead of fixing the username (`happy_doe` → `happy-doe`): a rewritten name can collide with a real user, and the user never chose it. Sending payments to a name the user didn't pick isn't acceptable.

Quick reference: `happy`, `happy-doe`, `123`, `héllo`, `😀`, `_happy` work. `Happy` works as `happy`. `happy_doe` (underscore after the first character), `happy doe`, `@happy`, `happy!`, `happy+1`, `ab--cd` are invalid. `happy.doe` never reaches the endpoint, because the dot makes it two labels.

For a new app, allow only `a-z`, `0-9` and `-` in usernames so every one of them is valid.

## Request

```http
GET {base URL}/{url-encoded normalized label}
Authorization: Bearer <token>
Accept: application/json
Accept-Encoding: identity
User-Agent: Namespace-UniversalUsernames/1
```

- **Base URL:** HTTPS with a valid certificate, on a public IP. A path is allowed (`https://api.brand.example/ens`). No query string, fragment or `user:pass@`. The gateway strips trailing slashes.
- **Route:** one path segment after the base URL. URL-decode it, then 404 unless `toEnsLabel(segment) === segment`. That guard costs nothing and skips a database query for junk.
- **Auth:** the brand saves a bearer token in the Namespace App next to the base URL. Always require it. Read it from an environment variable, compare in constant time, and reply `401` to anything else. A token for one origin isn't carried over when the base URL moves to a new origin, so save it again after changing hosts.
- **Methods:** only `GET`. The gateway never sends a body or query parameters.

## Response

Pick the status with this tree. The gateway treats every other outcome as a failed lookup, and a failure isn't the same as "not found".

```
token missing or wrong                                  → 401
segment isn't a normalized label                        → 404
no user with that ens_label                             → 404
user is private, suspended, banned, deleted or unverified → 404
user found and public                                   → 200 + profile
your own dependency failed                              → 5xx (lookup fails; never fake a 404)
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

Strict rules. Breaking one in the first group fails the whole profile, while a bad record only fails lookups of that record.

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
| `avatar` | `https://` or `ipfs://` image URL, or an NFT as `eip155:1/erc721:0x…/1` (ENSIP-12). Use a stable URL: a signed or expiring URL breaks once it expires. |
| `header` | Banner image, same formats as `avatar` |
| `description` | Short bio |
| `url` | Website, with `https://` |
| `email` | Only if the user made it public in the app |
| `location`, `timezone` | `Berlin, Germany`, `Europe/Berlin` (IANA) |
| `com.twitter`, `com.github`, `org.telegram`, `com.discord` | Handle only: no `@`, no URL |

Rules for every text record:

- **Omit a key instead of sending `""`.** An empty string still resolves, and apps render a blank field.
- **Never send `null`, numbers or booleans.** Convert to a string (`"2024-05-01"`, `"true"`) or omit. A non-string value fails that record.
- **Custom keys:** lowercase, reverse-domain of a domain the brand owns, hyphens between words (`com.brand.member-since`). Never invent a custom key for something a standard key covers (no `com.brand.twitter`).
- **Linked accounts:** only send a social handle the user connected through OAuth or verified, not free text from a bio field. Apps show these as verified links.

### Agent records (only for agent accounts)

Add these if and only if the account is an AI agent. A human user never gets them. They follow draft specs (ENSIP-26, ENSIP-25), so check the current spec before relying on exact key formats.

| Key | When | Value |
| --- | --- | --- |
| `agent-context` | Always, for an agent | Short description of the agent, or a URL to it |
| `agent-endpoint[<protocol>]` | For each protocol the agent serves: `mcp`, `a2a`, `oasf`, `web` | The endpoint URL, e.g. `agent-endpoint[mcp]` = `https://agents.brand.example/happy/mcp` |
| `agent-registration[<registry>][<agentId>]` | Only when the app registered the agent in an ERC-8004 registry | `"1"`. `<registry>` is the registry's ERC-7930 interoperable address |

## Addresses

Keys are ENS coin types as decimal strings. Values are plain, human-readable address strings: the gateway encodes them to bytes. Walk this per user:

```
Has the user proven control of an EVM address?
├── No  → no EVM key
└── Yes → "2147483648": getAddress(addr)
          This default answers for Ethereum and every EVM chain. Don't also send "60" with the same address.
          Does the user want a different address on one specific chain?
          └── Yes → add toCoinType(chainId) from viem, e.g. Base 8453 → "2147492101"
Has the user proven a Bitcoin address?  → "0": mainnet form (bc1…, 1…, 3…)
Has the user proven a Solana address?   → "501": base58 address
Other chains → the chain's SLIP-44 number
```

- **Checksum EVM addresses** with viem's `getAddress`. Wrap it in try/catch and drop only that record if a stored address is malformed. A thrown error that becomes a `5xx` fails the whole lookup.
- **Never send an address the user hasn't proven they control.** This record decides where money goes. A pasted, unverified address lets anyone redirect payments meant for that user.
- An invalid value under a specific chain key doesn't fall back to the default address, so validate before you send it.

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

Don't send a bare CID, an `https://` gateway URL or `null`. If there's no site, leave the field out.

## Speed

The gateway allows 5 seconds for the entire request, DNS and TLS included. A wallet is waiting behind every lookup, so budget for under 300 ms at the endpoint.

- One indexed read by `ens_label`. Join or denormalize the profile fields so it's a single query.
- No calls to third-party APIs while handling a request: no fetching the avatar to check it, no RPC call to verify a wallet, no OAuth refresh. Do that work when the user saves their profile, and store the result.
- Deploy close to the database, and keep the endpoint off cold-start paths where you can.
- Don't add your own cache longer than 60 seconds. The gateway caches each answer for up to 60 s, so profile edits show up within a minute. A long cache on your side delays edits and keeps serving accounts that went private.

## Build steps

1. Answer the questions under Before writing code.
2. Add the `ens_label` column with a unique index, backfill it with `toEnsLabel` and the collision rule, and set it wherever usernames are created or changed.
3. Write the handler. Follow `references/example-endpoint.md` in this skill: it's framework-neutral and tested.
4. Write tests for every case listed there.
5. Deploy, then check from a shell:
   ```sh
   curl -i -H "Authorization: Bearer $TOKEN" https://api.brand.example/happy
   curl -i https://api.brand.example/happy           # expect 401
   curl -i -H "Authorization: Bearer $TOKEN" https://api.brand.example/nobody   # expect 404
   ```
6. On the Universal Usernames page of the Namespace App, save the base URL and token, then use **Test username**. The test runs in the browser, so it needs CORS for `https://app.namespace.ninja` with the `Authorization` header allowed. Real lookups don't come from a browser, so CORS doesn't affect resolution.
7. Set the resolver and enable Universal Usernames in the App. Existing offchain or L2 subnames of the name stop resolving when you do.

## Good practices

- **Everything returned is public and permanent in practice.** Anyone can guess a username and read the profile. Return only what the user already shows publicly in the app.
- **Return `404` for private, suspended, banned and deleted accounts**, the same response as for an unknown user, so the endpoint doesn't reveal which accounts exist.
- **Don't reassign a released username right away.** Hold it for at least 90 days. Wallets and contacts keep sending to `happy.brand.eth`, and after a reassignment the money goes to the new owner.
- **Treat the endpoint like a payments API.** Change it through code review, log every `5xx`, and alert on error rates. To rotate the token, make the endpoint accept both the old and the new token, save the new one in the App, then drop the old one.
- **The wallet that owns `brand.eth` controls every username.** Keep it on a hardware wallet or a multisig.
- **Keep the response minimal.** Twenty correct records beat a hundred noisy ones. Every extra field costs bytes and response time, and adds to what's exposed.
