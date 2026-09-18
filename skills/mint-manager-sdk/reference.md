# Mint Manager API reference

Type definitions, configuration, error codes and chain tables for `@thenamespace/mint-manager`. The mint path itself is in [SKILL.md](SKILL.md).

## Client

```typescript
function createMintClient(config?: MintClientConfig): MintClient;
```

### MintClientConfig

| Option | Type | Default | Notes |
| --- | --- | --- | --- |
| `isTestnet` | `boolean` | `false` | Sepolia listings and staging APIs, switched together. |
| `customRpcUrls` | `Record<number, string>` | `{}` | Keyed by numeric chain id. Strongly recommended. |
| `mintSource` | `string` | `"namespace-sdk"` | Attribution tag sent with mints. |
| `listingCacheMilliseconds` | `number` | `900000` | Listing metadata TTL (15 min). Cache is bounded at 500 entries. |
| `timeoutMilliseconds` | `number` | `30000` | HTTP timeout for both APIs. |
| `logger` | `Logger` | silent | Supply one to see diagnostics. The SDK logs nothing on its own. |
| `listManagerUri` / `mintManagerUri` | `string` | per environment | Advanced overrides. Must be `https` (loopback may use `http`). |
| `environment` | `"staging" \| "production"` | — | **Deprecated.** Derived from `isTestnet`; ignored. |
| `cursomRpcUrls` | `Record<number, string>` | — | **Deprecated.** Misspelling of `customRpcUrls`, still honoured. |

`Logger` is `{ debug, info, warn, error }`, each taking a single `string`.

### MintClient methods

```typescript
checkName(name: string, options: CheckNameOptions): Promise<NameCheck>;
prepareMint(target: string | NameCheck, options: PrepareMintOptions): Promise<MintTransactionResponse>;
getMintDetails(request: MintDetailsRequest): Promise<MintDetailsResponse>;
getMintTransactionParameters(request: MintTransactionRequest): Promise<MintTransactionResponse>;

/** @deprecated Use checkName. */
isL1SubnameAvailable(subname: string): Promise<boolean>;
/** @deprecated Use checkName. */
isL2SubnameAvailable(subname: string, chainId: number): Promise<boolean>;
```

`checkName` + `prepareMint` are the pair to reach for. `getMintDetails` and `getMintTransactionParameters` are the lower-level equivalents that take a pre-split `label` + `parentName`; `prepareMint` calls the latter internally.

## Types

### NameCheck

```typescript
interface NameCheckBase {
  name: string;          // ENSIP-15 normalized full name
  label: string;         // leftmost label
  parentName: string;    // everything to the right
  listingType: "L1" | "L2";
  chainId: number;       // derived from the listing, never supplied
}

type NameCheck =
  | (NameCheckBase & {
      status: "available";
      estimatedPriceEth: number;
      estimatedFeeEth: number;
      isStandardFee: boolean;   // fee is fixed rather than taken out of the price
    })
  | (NameCheckBase & {
      status: "taken";
      reasons: MintingValidationErrorType[];
    })
  | (NameCheckBase & {
      status: "blocked";
      reasons: MintingValidationErrorType[];
      /** False only when rpc: "never" left availability unresolved. */
      nameAvailabilityConfirmed: boolean;
    });
```

### CheckNameOptions

```typescript
interface CheckNameOptions {
  minterAddress: string;          // gates are evaluated against this address
  expiryInYears?: number;
  rpc?: "auto" | "always" | "never";
}
```

The `rpc` policy decides when the registry is consulted directly:

| Policy | Registry lookup | Use when |
| --- | --- | --- |
| `"auto"` (default) | Only when the API's answer leaves availability unresolved | Almost always |
| `"always"` | Every call | The API's view may be stale and correctness beats latency |
| `"never"` | Skipped | Staying off-chain matters; accept `nameAvailabilityConfirmed: false` |

Request cost by situation:

| Situation | Requests | Result |
| --- | --- | --- |
| Name mintable | listing + API | `available` |
| Name taken, open listing | listing + API | `taken` |
| Gated listing | listing + API + registry | `blocked` or `taken` |

### Why checkName exists

The two availability methods each answer half the question. `isL2SubnameAvailable` reports registry ownership and knows nothing about allowlists, reservations or expired listings. `getMintDetails` knows about those, but its validation stops at the first failure — on a gated listing it never evaluates the name, so a free name and a taken name produce byte-identical responses:

```text
gated listing, free name   ->  canMint:false, ["MINTER_NOT_WHITELISTED"]
gated listing, taken name  ->  canMint:false, ["MINTER_NOT_WHITELISTED"]
```

`checkName` reconciles the two sources and consults the registry exactly when that ambiguity appears.

### PrepareMintOptions

```typescript
interface PrepareMintOptions {
  minterAddress: string;
  owner?: string;          // receives the subname; defaults to minterAddress
  expiryInYears?: number;
  records?: EnsRecords;
  maxValue?: bigint;       // hard cap on price + fee, in wei
}
```

### MintTransactionResponse

```typescript
interface MintTransactionResponse {
  contractAddress: Address;  // L1 or L2 mint controller
  abi: any;
  functionName: string;      // "mint"
  args: any[];               // [signed content, signature, resolverData, sourceTag]
  account: string;           // the minter
  value: bigint;             // price + fee, in wei
}
```

### EnsRecords

```typescript
interface EnsRecords {
  texts?: { key: string; value: string }[];
  addresses?: { chain: ChainName | number; value: string }[];  // number = coin type
  contenthash?: { type: ContenthashType; value: string };
}
```

`ContenthashType` members: `Ipfs` (`ipfs-ns`), `Ipns`, `Onion`, `Swarm`, `Arweave`, `Skynet`. Reference the members; the multicodec strings behind them are an implementation detail.

`ChainName` members include `Ethereum`, `Base`, `Optimism`, `Arbitrum`, `Polygon`, `Bsc`, `Avalanche`, `Gnosis`, `Zksync`, `Linea`, `Scroll`, `Bitcoin`, `Starknet`, `Solana`, `Cosmos`, `Near`, `Default`.

### MintDetailsRequest / MintDetailsResponse

```typescript
interface MintDetailsRequest {
  parentName: string;
  label: string;
  minterAddress: string;
  expiryInYears?: number;
  /** @deprecated Ignored — the network comes from the client's own isTestnet. */
  isTestnet?: boolean;
}

interface MintDetailsResponse {
  canMint: boolean;
  estimatedPriceEth: number;
  estimatedFeeEth: number;
  isStandardFee: boolean;
  validationErrors: MintingValidationErrorType[];
}
```

### MintTransactionRequest

```typescript
interface MintTransactionRequest {
  parentName: string;
  label: string;
  minterAddress: Address;
  owner?: string;
  expiryInYears?: number;
  records?: EnsRecords;
  maxValue?: bigint;
}
```

## Error codes

Every failure is a `MintManagerError` carrying `code`, `message`, `details` (the offending values) and sometimes `docsUrl` and `cause`. Codes are stable across minor versions.

| Code | Cause | Handling |
| --- | --- | --- |
| `INVALID_NAME` | A full name failed ENSIP-15 normalization, or was empty / not a string | Reject the input at the form, show the normalizer's reason |
| `INVALID_LABEL` | A label was empty, contained `"."`, was an encoded labelhash, or failed ENSIP-15 | Usually a full name passed where a label was expected |
| `INVALID_ADDRESS` | A value that must be a 20-byte EVM address was not one | Resolve ENS names to addresses first; checksums must be correct |
| `UNSUPPORTED_CHAIN` | No Namespace deployment on the requested chain in this environment | Check `isTestnet` — mainnet and testnet chains are separate sets |
| `UNSUPPORTED_LISTING` | The listing type is not `L1` or `L2` | Off-chain (gasless) names mint with `@thenamespace/offchain-manager` |
| `NAME_NOT_AVAILABLE` | A non-`available` check result reached `prepareMint` | Handle `taken` and `blocked` before preparing |
| `LISTING_NOT_FOUND` | No listing for the parent name on the selected network | List it at app.namespace.ninja, or check `isTestnet` and the spelling |
| `RPC_ERROR` | A contract read or JSON-RPC call failed | **Not** an answer about the name. Retryable; supply `customRpcUrls` |
| `API_ERROR` | The Namespace HTTP API returned non-2xx | 404 usually means unlisted on this network; 5xx deserves backoff |
| `MINT_PARAMS_MISMATCH` | The API signed a different label, parent or owner than requested | Normalize inputs first, then request parameters with the normalized values |
| `PRICE_EXCEEDS_MAX` | The quoted total exceeded `maxValue` | Nothing was submitted. Re-quote, then raise the cap or show the new price |
| `SIGNATURE_EXPIRED` | The mint authorization is past its expiry | Call `prepareMint` again and submit promptly |
| `CONFIG_ERROR` | `createMintClient` got unusable options (e.g. a plain-http URI override) | Fix the options and construct the client again |

`MINT_PARAMS_MISMATCH`, `PRICE_EXCEEDS_MAX` and `SIGNATURE_EXPIRED` are the safety net around the signed quote: the SDK verifies the signature describes the requested mint before handing it to a wallet, so a wrong or tampered-with response cannot reach the user as a valid-looking transaction.

## Chains

| Network | Chain id | Path |
| --- | --- | --- |
| Ethereum Mainnet | 1 | L1 |
| Sepolia | 11155111 | L1 testnet |
| Base | 8453 | L2 |
| Base Sepolia | 84532 | L2 testnet |
| Optimism | 10 | L2 |

### Chain helpers

```typescript
import {
  ListingChain, getChainId, getChainName,
  SUPPORTED_NETWORK_IDS, networkIdToChainName, chainNameToNetworkId,
  getCoinTypeForChain, isSupportedL2Network, isSupportedNetworkId,
  getSupportedNetworkIds, getSupportedChainNames, getChainDisplayName, isValidChainName,
} from "@thenamespace/mint-manager";

getChainId(ListingChain.Base);              // 8453
getChainName(8453);                         // ListingChain.Base
networkIdToChainName(8453);                 // ChainName.Base   (for ENS records)
chainNameToNetworkId(ChainName.Base);       // 8453
getCoinTypeForChain(ChainName.Base);        // 2147485001       (ENSIP-11 coin type)
```

Three representations coexist, and mixing them is the common bug: numeric **chain id** for blockchain operations, `ChainName` for ENS multi-coin address records, and `ListingChain` for the SDK's internal listing lookups.

## Registry-only availability

Deprecated in favour of `checkName`, still working. Reach for them only to read raw registry ownership with no minter in hand.

```typescript
const free       = await client.isL1SubnameAvailable("alice.namespace.eth");
const freeOnBase = await client.isL2SubnameAvailable("alice.namespace.eth", 8453);
```

Both throw `RPC_ERROR` when the registry is unreachable. A failed lookup is never reported as "taken".

## Upgrading from 1.1.x

The public API is unchanged and existing code keeps working. Four behaviours changed, each a case where the old result was wrong:

- **Availability checks throw instead of returning `false` on RPC failure.** An outage used to be indistinguishable from a taken name. Add a `try`/`catch` if these are called bare.
- **Names are normalized before hashing.** A name that could not be normalized used to produce an arbitrary namehash; it now throws `INVALID_NAME`.
- **`ContenthashType.Ipfs` writes the `ipfs-ns` codec.** It previously wrote `p2p` (`0xa503`), which ENS clients do not read as IPFS. `Swarm`, `Arweave` and `Skynet` previously threw `multicodec not recognized` and now work.
- **Unknown chain names in `records.addresses` throw** rather than being skipped silently.

### ENS v2

[ENS v2](https://docs.ens.domains/ensv2/migration) moves name ownership into per-name subregistries, so a v1 registry `owner()` lookup reports a migrated name as unowned. Nothing is on mainnet yet, and this is why `checkName` treats the Namespace API as the primary source of truth with the registry as corroboration: the API can follow the migration without an SDK release. Normalization is unaffected — ENSIP-15 does not change in v2.
