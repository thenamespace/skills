---
name: mint-manager-sdk
description: Mint ENS subnames on L1 and L2 with the @thenamespace/mint-manager SDK. Use when checking whether a subname can be minted, quoting a mint price, building or submitting a mint transaction, setting ENS records at mint time, or handling a refusal such as an allowlist, a reservation, or an expired listing.
---

# Namespace Mint Manager SDK

`@thenamespace/mint-manager` prepares ENS subname mint transactions for names listed on Namespace. It builds the contract call (`abi`, `args`, `value`) and the caller submits it with their own wallet library. It holds no private key and sends no transaction.

A parent name is listed either on Ethereum Mainnet (**L1**) or on Base / Optimism (**L2**). The SDK resolves the listing itself and returns the contract call for the right path, so pass whole names and let it work out the chain.

## Installation

```sh
npm install @thenamespace/mint-manager viem
```

This skill covers **v2**, which adds `checkName`, `prepareMint`, typed `MintManagerError` codes, ENSIP-15 normalization, and the `maxValue` price cap.

`viem` is a peer dependency. Install it in the consuming project so a single copy stays in the tree.

## The mint path

Five steps, in this order. Each one is cheap to get wrong out of order: a transaction built from a stale check is the failure this sequence exists to prevent.

### 1. Create the client

```typescript
import { createMintClient } from "@thenamespace/mint-manager";

const client = createMintClient({
  isTestnet: false,
  // Strongly recommended: the default is a shared public endpoint and is rate limited.
  customRpcUrls: {
    1: process.env.MAINNET_RPC_URL!,
    8453: process.env.BASE_RPC_URL!,
  },
});
```

`isTestnet: true` switches the listing source (Sepolia) and the API environment (staging) **together**, so a quote can never come from a different network than the transaction that follows it. Mainnet listings are invisible to a testnet client, and vice versa.

Done when the client is constructed with an RPC URL for every chain the app touches.

### 2. Check the name

`checkName` answers "can this address mint this name, and if not, why" in one call. Prefer it over the deprecated availability methods, which each answer half the question.

```typescript
const check = await client.checkName("alice.oppunk.eth", {
  minterAddress: "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045",
  expiryInYears: 1,
});
```

The result is a discriminated union on `status`. Narrow before reading anything else — price fields exist only on `"available"`.

| `status` | Meaning | Next move |
| --- | --- | --- |
| `available` | Free, and this minter may mint it now | Go to step 3 |
| `taken` | Registered already | Offer a different label |
| `blocked` | Free, but this minter is gated out | Read `reasons`, act per the table below |

Every result also carries `name`, `label`, `parentName`, `listingType` (`"L1"` / `"L2"`) and `chainId`, all derived from the listing.

Done when all three branches of `status` are handled.

### 3. Handle a refusal on the right axis

`reasons` entries are not interchangeable. Two axes decide the response: whether the obstacle is the **name** or the **minter**, and whether **another label helps**. Getting this wrong sends a user back to the name field when their wallet was the problem.

| Reason | Obstacle | Another label helps? | Tell the user |
| --- | --- | --- | --- |
| `SUBNAME_TAKEN` | name | yes | Registered and gone for everyone. Offer alternatives. |
| `SUBNAME_RESERVED` | name | yes | The parent owner holds this label back. It is *not* registered, so the registry reports it free. Offer alternatives. |
| `MINTER_NOT_WHITELISTED` | minter | no | This address is not on the parent's allowlist. Switch wallets or request access. |
| `MINTER_NOT_TOKEN_OWNER` | minter | no | The parent is token-gated. The user can usually go and acquire a qualifying token. |
| `VERIFIED_MINTER_ADDRESS_REQUIRED` | minter | no | The wallet is recognised but uncleared. Send them through the parent's verification flow. |
| `LISTING_EXPIRED` | listing | no | The minting window closed. No label under this parent is mintable until the owner relists. |

Validation stops at the first failure, so `reasons` is a partial list: clearing one can reveal another. Re-run `checkName` after the user acts rather than assuming what remains.

`SUBNAME_RESERVED` is the entry worth reading twice. A reserved name is held back, not minted, so the registry reports it free. That is why `checkName` consults the registry before deciding between `taken` and `blocked`.

### 4. Build the transaction

Pass the check result. The listing is already cached, so nothing is refetched.

```typescript
import { parseEther } from "viem";

const tx = await client.prepareMint(check, {
  minterAddress,
  owner,                  // defaults to minterAddress
  expiryInYears: 1,
  maxValue: parseEther("0.01"),
});
```

`tx` carries `abi`, `args`, `functionName`, `contractAddress`, `account` and `value` (wei, already a `bigint` — pass it through untouched).

**Set `maxValue` whenever a price was shown to a user.** The mint API returns a signed price the contract will honour. The SDK verifies that signature covers the requested name, parent and owner; `maxValue` closes the remaining gap by capping the charge, so a price that moves between quote and signature throws `PRICE_EXCEEDS_MAX` instead of being charged silently.

`prepareMint` also accepts a bare name (`prepareMint("alice.oppunk.eth", { minterAddress })`) when mintability is already known. That path re-resolves the listing.

Done when the built `tx.value` is bounded by a `maxValue` the user agreed to.

### 5. Simulate, then send

```typescript
const { request } = await publicClient.simulateContract({
  address: tx.contractAddress,
  abi: tx.abi,
  functionName: tx.functionName,
  args: tx.args,
  value: tx.value,
  account,
});

const hash = await walletClient.writeContract(request);
const receipt = await publicClient.waitForTransactionReceipt({ hash });
```

Simulating first surfaces a revert reason for free. Skipping it is how users pay gas for a mint that a stale signature or a moved price had already doomed.

Signed parameters are short-lived. Submit promptly instead of caching `tx`; a stale one throws `SIGNATURE_EXPIRED`.

## Records at mint time

Records are encoded into resolver calldata and applied in the same transaction, so the name arrives configured rather than empty.

```typescript
import { ChainName, ContenthashType } from "@thenamespace/mint-manager";

const tx = await client.prepareMint(check, {
  minterAddress,
  records: {
    texts: [{ key: "avatar", value: "https://example.com/avatar.png" }],
    addresses: [
      { chain: ChainName.Ethereum, value: evmAddress },
      { chain: ChainName.Base, value: evmAddress },
    ],
    contenthash: {
      type: ContenthashType.Ipfs,
      value: "bafybeicnesqbuvzjxhkylkzwaqxi5jvbvzf7z4rjnkvjnvbsrqxlgnzpqu",
    },
  },
});
```

`texts`, `addresses` and `contenthash` are each optional. An unknown chain name or a malformed address throws rather than being skipped, so a paid mint cannot quietly land with no records on it.

Use the `ChainName` and `ContenthashType` enum members, not their raw string values — the members are the stable API.

## Names are normalized for you

Every name and label is normalized per [ENSIP-15](https://docs.ens.domains/ensip/15) before it is hashed, sent to the API, or compared. `viem`'s `namehash()` normalizes nothing on its own, so without this `"Alice.eth"` and `"alice.eth"` hash to different nodes and an availability check reports a registered name as free.

```typescript
import { normalizeName, normalizeLabel } from "@thenamespace/mint-manager";

normalizeName("Alice.ETH");   // "alice.eth"
normalizeLabel("Alice");      // "alice"
normalizeLabel("alice.eth");  // throws INVALID_LABEL — a label has no dots
```

Use these same helpers anywhere the app stores or compares names, so its keys agree with the SDK's.

## Errors

Every failure is a `MintManagerError` with a stable `code`. Branch on `code`; `message` is written for a human reading a log and is free to change.

```typescript
import { MintManagerError } from "@thenamespace/mint-manager";

try {
  const check = await client.checkName(name, { minterAddress });
} catch (err) {
  if (!(err instanceof MintManagerError)) throw err;
  switch (err.code) {
    case "RPC_ERROR":
      // Infrastructure, not the name. Offer a retry.
      break;
    case "PRICE_EXCEEDS_MAX":
      // Re-quote and confirm the new price with the user.
      break;
    default:
      console.error(err.code, err.message, err.details);
  }
}
```

**`RPC_ERROR` is not an answer about the name.** A failed registry lookup leaves the status *unknown*. Surface it as a temporary lookup failure with a retry; rendering it as "name taken" tells users a free name is gone and they walk away. The usual cause is rate limiting on the shared public endpoint — supply `customRpcUrls`, keyed by the numeric chain id that `err.details.chainId` reports.

Full code list, `details` payloads and every remedy: [reference.md](reference.md).

## Reference

Types, configuration options, chain tables, chain-conversion helpers, the deprecated availability methods, and upgrading from 1.1.x: [reference.md](reference.md).

Complete runnable programs — availability sweep, testnet quote, end-to-end mint with records: [examples.md](examples.md).
