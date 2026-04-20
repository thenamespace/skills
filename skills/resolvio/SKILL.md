---
name: resolvio
description: Use when resolving ENS names to addresses, looking up ENS profiles or social/text records, reverse-resolving addresses to ENS names, or building leaderboards/tables that need ENS names for multiple addresses via HTTP curl request.
---

# Resolvio ENS Resolution API

**Base URL**: `https://api.resolvio.xyz`  
**OpenAPI JSON**: `https://api.resolvio.xyz/api-docs.json`

All endpoints are **GET** requests. No authentication required.

---

## ENS name → address

```bash
curl "https://api.resolvio.xyz/ens/v2/addresses/vitalik.eth?chains=eth"
```

Response: `{ "name": "vitalik.eth", "addresses": [{ "coin": 60, "chain": "eth", "value": "0xd8dA...", "exists": true }] }`

Extract `addresses[0].value` when `exists: true`. If `exists: false`, no address is set for that chain.

---

## Full profile (texts + addresses + contenthash)

One RPC call. Default includes 12 text keys, 8 chains, and contenthash.

```bash
curl https://api.resolvio.xyz/ens/v2/profile/vitalik.eth
```

Narrow to what you need:

```bash
curl "https://api.resolvio.xyz/ens/v2/profile/vitalik.eth?texts=avatar,description,com.twitter&addresses=eth,base&contenthash=false"
```

**Always prefer `/profile` over separate calls** — it fetches everything in a single RPC call.

---

## Address → ENS name (primary name / reverse resolution)

```bash
curl https://api.resolvio.xyz/ens/v2/reverse/0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045
```

Response: `{ "address": "0xd8dA...", "hasReverseRecord": true, "name": "vitalik.eth" }`  
No name set: `{ "address": "0x...", "hasReverseRecord": false }`

---

## Bulk reverse resolution (leaderboards, tables)

Resolves up to 20 addresses in a **single batched RPC call**. Always use this when displaying ENS names for multiple addresses — never loop single calls.

```bash
curl "https://api.resolvio.xyz/ens/v2/reverse/bulk?addresses=0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045,0x225f137127d9067788314bc7fcc1f36746a3c3B5"
```

Response:

```json
{
  "addresses": [
    { "address": "0xd8dA...", "hasReverseRecord": true, "name": "vitalik.eth" },
    { "address": "0x225f...", "hasReverseRecord": false }
  ]
}
```

For >20 addresses, split into batches of 20.

---

## Text records (avatar, socials, bio)

```bash
curl "https://api.resolvio.xyz/ens/v2/texts/vitalik.eth?keys=avatar,description,com.twitter,com.github,url"
```

Common keys: `avatar`, `name`, `description`, `url`, `email`, `location`, `com.twitter`, `com.github`, `com.discord`, `org.telegram`, `com.youtube`

---

## Key behaviours

- **All requested records always returned.** Missing record → `{ "exists": false }`, never silently dropped.
- **`exists: false` means no `value` field.** Always check `exists` before reading `value`.
- **Cache TTL**: ~5 min forward, ~30 min reverse. Bypass with `?noCache=true`.
- **Chain names and coinTypes are interchangeable**: `chains=eth` = `coins=60`. Full list: `GET /ens/v2/chains`.

---

## Reference & examples

- Full API reference (all params, response shapes, error codes): [references/api.md](references/api.md)
- Detailed usage examples (profiles, leaderboards, multi-chain): [references/examples.md](references/examples.md)
