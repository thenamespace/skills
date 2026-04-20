# Resolvio Usage Examples

All examples use `https://api.resolvio.xyz` as the base URL.

---

## Forward Resolution

### Resolve ENS name to Ethereum address

```bash
curl "https://api.resolvio.xyz/ens/v2/addresses/vitalik.eth?chains=eth"
```

```json
{
  "name": "vitalik.eth",
  "addresses": [
    {
      "coin": 60,
      "chain": "eth",
      "value": "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045",
      "exists": true
    }
  ]
}
```

---

### Resolve ENS name to multiple chain addresses

```bash
curl "https://api.resolvio.xyz/ens/v2/addresses/vitalik.eth?chains=eth,base,btc,sol"
```

```json
{
  "name": "vitalik.eth",
  "addresses": [
    {
      "coin": 60,
      "chain": "eth",
      "value": "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045",
      "exists": true
    },
    { "coin": 2147492101, "chain": "base", "exists": false },
    { "coin": 0, "chain": "btc", "exists": false },
    { "coin": 501, "chain": "sol", "exists": false }
  ]
}
```

Using coinType numbers instead:

```bash
curl "https://api.resolvio.xyz/ens/v2/addresses/vitalik.eth?coins=60,2147492101,0,501"
```

```json
{
  "name": "vitalik.eth",
  "addresses": [
    {
      "coin": 60,
      "chain": "eth",
      "value": "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045",
      "exists": true
    },
    {
      "coin": 2147492101,
      "chain": "base",
      "exists": false
    },
    {
      "coin": 0,
      "chain": "btc",
      "exists": false
    },
    {
      "coin": 501,
      "chain": "sol",
      "exists": false
    }
  ]
}
```

---

### Full profile (avatar, socials, addresses)

Default — 12 text keys, 8 chains, contenthash:

```bash
curl https://api.resolvio.xyz/ens/v2/profile/vitalik.eth
```

```json
{
  "name": "vitalik.eth",
  "texts": [
    { "key": "avatar", "value": "https://euc.li/vitalik.eth", "exists": true },
    {
      "key": "header",
      "value": "https://pbs.twimg.com/profile_banners/295218901/1638557376/1500x500",
      "exists": true
    },
    { "key": "name", "exists": false },
    {
      "key": "description",
      "value": "mi pinxe lo crino tcati",
      "exists": true
    },
    { "key": "url", "value": "https://vitalik.ca", "exists": true },
    { "key": "location", "exists": false },
    { "key": "email", "exists": false },
    { "key": "com.twitter", "value": "VitalikButerin", "exists": true },
    { "key": "com.discord", "exists": false },
    { "key": "com.github", "value": "vbuterin", "exists": true },
    { "key": "org.telegram", "exists": false },
    { "key": "com.youtube", "exists": false }
  ],
  "addresses": [
    {
      "coin": 60,
      "chain": "eth",
      "value": "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045",
      "exists": true
    },
    { "coin": 2147492101, "chain": "base", "exists": false },
    { "coin": 2147525809, "chain": "arb1", "exists": false },
    { "coin": 2147483658, "chain": "op", "exists": false },
    { "coin": 2147483785, "chain": "matic", "exists": false },
    { "coin": 2147525868, "chain": "celo", "exists": false },
    { "coin": 0, "chain": "btc", "exists": false },
    { "coin": 501, "chain": "sol", "exists": false }
  ],
  "contenthash": {
    "value": "ipfs://bafybeiaql2jo3fu5b7c4lmpoi5drh5sam7yt652shwdgwbky4o7uw33u2u",
    "exists": true
  },
  "resolver": "0x231b0Ee14048e9dCcD1d247744d114a4EB5E8E63"
}
```

Custom selection:

```bash
curl "https://api.resolvio.xyz/ens/v2/profile/vitalik.eth?texts=avatar,description,com.twitter,com.github&addresses=eth,base&contenthash=false"
```

```json
{
  "name": "vitalik.eth",
  "resolver": "0x231b0Ee14048e9dCcD1d247744d114a4EB5E8E63",
  "texts": [
    { "key": "avatar", "value": "https://...", "exists": true },
    { "key": "description", "value": "Ethereum co-founder", "exists": true },
    { "key": "com.twitter", "value": "VitalikButerin", "exists": true },
    { "key": "com.github", "value": "vbuterin", "exists": true }
  ],
  "addresses": [
    { "coin": 60, "chain": "eth", "value": "0xd8dA...", "exists": true },
    {
      "coin": 2147492101,
      "chain": "base",
      "value": "0xd8dA...",
      "exists": true
    }
  ],
  "contenthash": { "exists": false }
}
```

---

### Fetch only text records

```bash
curl "https://api.resolvio.xyz/ens/v2/texts/vitalik.eth?keys=avatar,description,com.twitter,com.github,url,email,org.telegram"
```

```json
{
  "name": "vitalik.eth",
  "texts": [
    {
      "key": "avatar",
      "value": "https://euc.li/vitalik.eth",
      "exists": true
    },
    {
      "key": "description",
      "value": "mi pinxe lo crino tcati",
      "exists": true
    },
    {
      "key": "com.twitter",
      "value": "VitalikButerin",
      "exists": true
    },
    {
      "key": "com.github",
      "value": "vbuterin",
      "exists": true
    },
    {
      "key": "url",
      "value": "https://vitalik.ca",
      "exists": true
    },
    {
      "key": "email",
      "exists": false
    },
    {
      "key": "org.telegram",
      "exists": false
    }
  ]
}
```

---

### Fetch content hash

```bash
curl https://api.resolvio.xyz/ens/v2/contenthash/artii.eth
```

```json
{ "value": "ipfs://bafybeid...", "exists": true }
```

---

### Bulk profiles (multiple names at once)

```bash
curl "https://api.resolvio.xyz/ens/v2/profile/bulk?names=vitalik.eth,artii.eth,nick.eth"
```

With custom fields:

```bash
curl "https://api.resolvio.xyz/ens/v2/profile/bulk?names=vitalik.eth,artii.eth&texts=avatar,com.twitter&addresses=eth&contenthash=false"
```

Returns an array of profile objects. Order matches input.

---

## Reverse Resolution

### Single address → ENS name

```bash
curl https://api.resolvio.xyz/ens/v2/reverse/0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045
```

```json
{
  "address": "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045",
  "hasReverseRecord": true,
  "name": "vitalik.eth"
}
```

Address with no ENS name:

```json
{
  "address": "0x225f137127d9067788314bc7fcc1f36746a3c3B5",
  "hasReverseRecord": false
}
```

---

### Leaderboard: resolve ENS names for a list of addresses

Always use bulk for multiple addresses — it resolves all in a single batched RPC call:

```bash
curl "https://api.resolvio.xyz/ens/v2/reverse/bulk?addresses=0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045,0x225f137127d9067788314bc7fcc1f36746a3c3B5,0x179A862703a4adfb29896552DF9e307980D19285"
```

```json
{
  "addresses": [
    { "address": "0xd8dA...", "hasReverseRecord": true, "name": "vitalik.eth" },
    { "address": "0x225f...", "hasReverseRecord": false },
    { "address": "0x179A...", "hasReverseRecord": true, "name": "artii.eth" }
  ]
}
```

For displaying in a leaderboard table: use `name` if `hasReverseRecord: true`, otherwise fall back to the shortened address (e.g. `0x179A...285`).

**Batching >20 addresses**:

```bash
# Batch 1 (addresses 0–19)
curl "https://api.resolvio.xyz/ens/v2/reverse/bulk?addresses=0xAAA...,0xBBB...,..."

# Batch 2 (addresses 20–39)
curl "https://api.resolvio.xyz/ens/v2/reverse/bulk?addresses=0xCCC...,0xDDD...,..."
```

---

## Discover supported chains

```bash
curl https://api.resolvio.xyz/ens/v2/chains
```

Returns all chain names and their coinType numbers.

---

## Common patterns

### Display name for a connected wallet

```bash
# User connects with address 0xd8dA...
curl https://api.resolvio.xyz/ens/v2/reverse/0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045
# → show "vitalik.eth" in the UI if hasReverseRecord: true
```

### Validate a name before sending funds

```bash
curl "https://api.resolvio.xyz/ens/v2/addresses/recipient.eth?chains=eth"
# → extract value from addresses[0] where exists: true
# → confirm address with user before sending
```

### Load profile card

```bash
curl "https://api.resolvio.xyz/ens/v2/profile/artii.eth?texts=avatar,name,description,url,com.twitter&addresses=eth&contenthash=false"
# → render avatar, display name, bio, socials, ETH address
```

---

## Cache Control

Bypass cache when you need fresh data (e.g. after a user updates their ENS record):

```bash
curl "https://api.resolvio.xyz/ens/v2/profile/vitalik.eth?noCache=true"
curl "https://api.resolvio.xyz/ens/v2/reverse/0xd8dA...?noCache=true"
```

Clear cache for a specific name:

```bash
curl -X DELETE "https://api.resolvio.xyz/ens/v2/cache/vitalik.eth?type=all"
```
