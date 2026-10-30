# Endpoint contract

The HTTP contract between the Namespace gateway and the brand's endpoint. Source of truth: https://docs.namespace.ninja/developer-guide/universal-usernames/endpoint-reference

## Request

```http
GET https://api.brand.example/ens/users/happy
Accept: application/json
User-Agent: Namespace-UniversalUsernames/1
Authorization: Bearer <credential>
```

- URL: the connected endpoint URL with any trailing slash removed, then `/`, then `encodeURIComponent(username)`.
- `Authorization` is sent only when the owner configured bearer authentication.
- The username is one ENSIP-15 normalized label: no dots, already lowercase. Decode the path segment once (`caf%C3%A9` is `café`) and look it up as-is.

### Endpoint URL rules

- `https://` only.
- No `user:pass@`, no query string, no fragment, at most 2,048 characters. A path such as `/ens/users` is fine.
- Must resolve to a public IPv4 or IPv6 address. Private, loopback, link-local, CGNAT, multicast, and cloud metadata addresses are blocked.
- Redirects are never followed. Connect the final URL.

## Response: profile schema v1

`200`, `Content-Type: application/json` (a `; charset=utf-8` parameter is fine), body:

```json
{
  "version": 1,
  "username": "happy",
  "texts": { "name": "Happy", "avatar": "https://cdn.brand.example/happy.png", "com.github": "happy", "com.brand.role": "member" },
  "addresses": {
    "60": "0x1111111111111111111111111111111111111111",
    "2147492101": "0x2222222222222222222222222222222222222222",
    "0": "bc1qar0srrr7xfkvy5l643lydnw9re59gtzzwf5mdq"
  },
  "contenthash": "ipfs://QmYwAPJzv5CZsnAzt8auVZRnGiRAzQmmJYDh4T7SEBuwNA"
}
```

The frozen JSON schema (also exported as `USERNAME_PROFILE_SCHEMA_V1` from `@thenamespace/universal-usernames`):

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "Universal Usernames profile v1",
  "type": "object",
  "additionalProperties": false,
  "required": ["version", "username", "texts", "addresses"],
  "properties": {
    "version": { "const": 1 },
    "username": { "type": "string", "minLength": 1 },
    "texts": {
      "type": "object",
      "maxProperties": 128,
      "propertyNames": { "minLength": 1, "maxLength": 256 },
      "additionalProperties": { "type": "string", "maxLength": 4096 }
    },
    "addresses": {
      "type": "object",
      "maxProperties": 128,
      "propertyNames": { "pattern": "^(0|[1-9][0-9]*)$" },
      "additionalProperties": { "type": "string", "minLength": 1, "maxLength": 1024 }
    },
    "contenthash": { "type": "string", "minLength": 1, "maxLength": 4096 }
  }
}
```

Field rules:

- `version` is the number `1`.
- `username` equals the requested, decoded username exactly.
- `texts` and `addresses` are required objects and may be empty.
- Missing records are omitted, never `null`.
- `addresses` keys are canonical decimal coin types from `0` to `4294967295`.
- `contenthash` is optional: `ipfs://`, `ipns://`, `bzz://`, or `ar://`.
- No other top-level fields. No duplicate keys at any depth: the validator reads the raw text, so a repeated key fails even when `JSON.parse` would keep the last one.
- Text values are plain text and are never rendered as HTML by well-behaved clients.

## Status codes

| Endpoint answers | Namespace reads it as | Wallet gets |
| --- | --- | --- |
| `200` | The profile | The requested record, or an empty value if the profile lacks it |
| `404` | Username absent | A signed empty value |
| `401`, `403` | `UNAUTHORIZED` | An error |
| `429` | `RATE_LIMITED` | An error |
| `3xx` | `HTTP_REDIRECT` (not followed) | An error |
| anything else | `HTTP_ERROR_STATUS` | An error |

## What a lookup returns

| Situation | Result |
| --- | --- |
| Profile has the record | Signed record |
| Profile valid, record missing | Signed empty: zero address for `addr(node)`, `0x` for `addr(node, coinType)`, `""` for text, empty contenthash |
| Endpoint `404` | Signed empty |
| The requested record is malformed | Error for that lookup only; other records still resolve |
| Envelope invalid (bad JSON, duplicate key, wrong version, unknown top-level field, username mismatch, too many records) | Error for every lookup of that username |
| Timeout, `401`/`403`/`429`/`5xx`, redirect, wrong content type, too large | Error, never treated as absent |
| Connection paused or suspended | Error, no signature |
| Deeper name like `team.happy.brand.eth` | Signed empty, endpoint not called |
| Name with an onchain owner | Answered onchain by HybridResolverV2, endpoint not called |

A connected parent never falls back to Namespace's stored offchain subnames.

## Limits

| Limit | Value |
| --- | --- |
| Total time per request | 5 s, including connect |
| Body size | 262,144 bytes (256 KiB) after decompression |
| Text records / address records | 128 each |
| Text key | 1 to 256 characters |
| Text value | up to 4,096 characters |
| Address value | 1 to 1,024 characters |
| Contenthash | up to 4,096 characters |
| Request rate from Namespace | 10/s per parent, bursts of 20, at most 10 concurrent; lookups beyond that fail |

## Caching and proof expiry

- Each `200` and `404` is cached for exactly 60 s from fetch, per username and connection revision. The window doesn't slide.
- Failures aren't cached; nothing stale is served after an outage.
- Concurrent lookups for one username share one request.
- Signed proofs expire no later than the cached profile, so at most 60 s. With under 10 s left, the gateway refetches first.
- Wallets, apps, and indexers may cache resolved values longer; that's outside Namespace's control.

## Bearer authentication

- Optional. Namespace sends `Authorization: Bearer <credential>` on every request when configured.
- Compare in constant time and answer `401` on mismatch.
- The credential is stored encrypted and never shown again after saving.
- During rotation Namespace validates the new token before switching, so accept both tokens until the owner finishes rotating.
- The token protects the endpoint, not the data: resolvable records are public.
