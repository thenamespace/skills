# Mint Manager examples

Complete programs using `@thenamespace/mint-manager`. The mint path they follow is explained in [SKILL.md](SKILL.md); types and error codes are in [reference.md](reference.md).

Every example reads credentials and endpoints from the environment. A committed private key is compromised the moment it is pushed, and a committed provider URL leaks the API quota to anyone who clones the repo.

## 1. Check before you offer

A name-availability endpoint for a mint UI. Returns the three outcomes a form needs, each with advice that matches the obstacle.

```typescript
import { MintManagerError, createMintClient } from "@thenamespace/mint-manager";

const client = createMintClient({
  customRpcUrls: {
    1: process.env.MAINNET_RPC_URL!,
    8453: process.env.BASE_RPC_URL!,
  },
});

type Availability =
  | { state: "available"; priceEth: number; feeEth: number; chainId: number }
  | { state: "unavailable"; retryLabel: boolean; message: string }
  | { state: "unknown"; message: string };

export async function checkAvailability(
  name: string,
  minterAddress: string
): Promise<Availability> {
  try {
    const check = await client.checkName(name, { minterAddress });

    if (check.status === "available") {
      return {
        state: "available",
        priceEth: check.estimatedPriceEth,
        feeEth: check.estimatedFeeEth,
        chainId: check.chainId,
      };
    }

    // The obstacle decides the advice. Sending someone back to the name field
    // when their wallet is the problem wastes their time on every retry.
    const minterGated = check.reasons.some((r) =>
      [
        "MINTER_NOT_WHITELISTED",
        "MINTER_NOT_TOKEN_OWNER",
        "VERIFIED_MINTER_ADDRESS_REQUIRED",
      ].includes(r)
    );

    if (check.reasons.includes("LISTING_EXPIRED")) {
      return {
        state: "unavailable",
        retryLabel: false,
        message: "This collection has closed. No name under it can be minted right now.",
      };
    }

    if (minterGated) {
      return {
        state: "unavailable",
        retryLabel: false,
        message: "This wallet cannot mint here. Switch to one that has access.",
      };
    }

    return {
      state: "unavailable",
      retryLabel: true,
      message: "That name is not available. Try another.",
    };
  } catch (err) {
    if (err instanceof MintManagerError) {
      // An unreachable registry means the status is unknown, not taken.
      // Rendering it as "taken" tells users a free name is gone.
      if (err.code === "RPC_ERROR") {
        return { state: "unknown", message: "Could not reach the registry. Try again." };
      }
      if (err.code === "INVALID_NAME" || err.code === "INVALID_LABEL") {
        return { state: "unavailable", retryLabel: true, message: "That is not a valid ENS name." };
      }
      if (err.code === "LISTING_NOT_FOUND") {
        return { state: "unknown", message: "That parent name is not listed on Namespace." };
      }
    }
    throw err;
  }
}
```

## 2. End-to-end mint on Base Sepolia

Quote, build, simulate, broadcast, confirm. This is the order a production integration should use.

```typescript
import {
  createPublicClient, createWalletClient, formatEther, http, parseEther,
} from "viem";
import { privateKeyToAccount } from "viem/accounts";
import { baseSepolia } from "viem/chains";
import { ChainName, MintManagerError, createMintClient } from "@thenamespace/mint-manager";

const RPC_URL = process.env.BASE_SEPOLIA_RPC_URL ?? "https://sepolia.base.org";
const account = privateKeyToAccount(process.env.PRIVATE_KEY as `0x${string}`);
const subname = `${process.env.LABEL}.${process.env.PARENT_NAME}`;

// One RPC URL shared by the SDK's reads and the wallet clients, so a check and
// the transaction that follows it cannot disagree because they hit different
// providers.
const mintClient = createMintClient({
  isTestnet: true,
  customRpcUrls: { [baseSepolia.id]: RPC_URL },
});
const publicClient = createPublicClient({ chain: baseSepolia, transport: http(RPC_URL) });
const walletClient = createWalletClient({ account, chain: baseSepolia, transport: http(RPC_URL) });

async function main() {
  // Step 1: one call covers whether the name is free AND whether this address
  // may mint it.
  const check = await mintClient.checkName(subname, {
    minterAddress: account.address,
    expiryInYears: 1,
  });

  if (check.status !== "available") {
    console.error(`Cannot mint ${subname}: ${check.status} (${check.reasons.join(", ")})`);
    process.exit(1);
  }

  console.log(
    `Available on chain ${check.chainId}: ` +
      `${check.estimatedPriceEth} ETH + ${check.estimatedFeeEth} ETH fee`
  );

  // Step 2: build. maxValue is the number shown to the user. Without it, a
  // price that moves between the quote and the signature is simply charged,
  // and the wallet confirmation is the first they hear of it.
  const quotedWei = parseEther(
    (check.estimatedPriceEth + check.estimatedFeeEth).toFixed(18)
  );

  const tx = await mintClient.prepareMint(check, {
    minterAddress: account.address,
    owner: account.address,
    expiryInYears: 1,
    maxValue: quotedWei,
    // Applied atomically with the mint, so the name is never live without them.
    records: {
      texts: [{ key: "description", value: "Minted with the Namespace SDK" }],
      addresses: [
        { chain: ChainName.Ethereum, value: account.address },
        { chain: ChainName.Base, value: account.address },
      ],
    },
  });

  console.log(`Transaction built, value ${formatEther(tx.value)} ETH`);

  // Step 3: simulate. Runs the call against current state and surfaces a
  // revert reason without spending gas.
  const { request } = await publicClient.simulateContract({
    address: tx.contractAddress,
    abi: tx.abi,
    functionName: tx.functionName,
    args: tx.args,
    value: tx.value,
    account,
  });

  // Step 4: broadcast exactly what was simulated, then wait.
  const hash = await walletClient.writeContract(request);
  const receipt = await publicClient.waitForTransactionReceipt({ hash });

  if (receipt.status !== "success") {
    console.error(`Reverted on chain: ${hash}`);
    process.exit(1);
  }

  console.log(`Minted ${subname} in block ${receipt.blockNumber}`);
}

main().catch((err) => {
  if (err instanceof MintManagerError) {
    // Codes are stable; match on them rather than on message text.
    const advice: Record<string, string> = {
      RPC_ERROR: "The RPC endpoint failed. Not an answer about the name — retry.",
      PRICE_EXCEEDS_MAX: "The signed quote exceeded the price quoted. Re-quote and confirm.",
      SIGNATURE_EXPIRED: "The authorization expired. Call prepareMint again.",
    };
    console.error(err.code, advice[err.code] ?? err.message, err.details);
    process.exit(1);
  }
  throw err;
});
```

## 3. Suggesting alternatives

When a label is taken, propose variants — but only when the obstacle is the name. A minter gate fails identically on every label, so generating suggestions there produces a list of names the user still cannot mint.

```typescript
import { createMintClient, normalizeLabel } from "@thenamespace/mint-manager";

const client = createMintClient();

export async function suggest(
  parentName: string,
  label: string,
  minterAddress: string,
  count = 3
): Promise<string[]> {
  const base = normalizeLabel(label);
  const first = await client.checkName(`${base}.${parentName}`, { minterAddress });

  if (first.status === "available") return [first.name];

  // Gated on the minter: every variant fails the same way. Return nothing
  // rather than a list of dead ends.
  const nameIsTheObstacle =
    first.status === "taken" || first.reasons.includes("SUBNAME_RESERVED");
  if (!nameIsTheObstacle) return [];

  const candidates = [`${base}1`, `${base}-eth`, `the${base}`, `${base}x`, `${base}2`];
  const found: string[] = [];

  for (const candidate of candidates) {
    if (found.length >= count) break;
    const check = await client.checkName(`${candidate}.${parentName}`, { minterAddress });
    if (check.status === "available") found.push(check.name);
  }

  return found;
}
```

## 4. Batch availability across a listing

Sweep a set of labels with one client. The listing is cached for 15 minutes, so only the first call pays for resolving it.

```typescript
import { MintManagerError, createMintClient } from "@thenamespace/mint-manager";

const client = createMintClient({
  customRpcUrls: { 8453: process.env.BASE_RPC_URL! },
  // A sweep re-reads the same listing repeatedly; a shorter TTL keeps the
  // answers fresh without re-resolving it on every label.
  listingCacheMilliseconds: 60_000,
});

export async function sweep(
  labels: string[],
  parentName: string,
  minterAddress: string
) {
  const results = [];

  for (const label of labels) {
    try {
      const check = await client.checkName(`${label}.${parentName}`, {
        minterAddress,
        rpc: "always",
      });
      results.push({
        label,
        status: check.status,
        priceEth: check.status === "available" ? check.estimatedPriceEth : undefined,
      });
    } catch (err) {
      // One bad label should not end the sweep. Record it as unknown and
      // keep going — an RPC blip is not a verdict on the name.
      const code = err instanceof MintManagerError ? err.code : "UNKNOWN";
      results.push({ label, status: "error" as const, code });
    }
  }

  return results;
}
```

## Runnable scripts

The package ships working scripts in [`packages/mint-manager/examples/`](https://github.com/thenamespace/namespacesdk/tree/release/packages/mint-manager/examples): an availability sweep, a testnet quote, and an end-to-end mint that signs and broadcasts.

Fund the signer from the [Base Sepolia faucet](https://www.alchemy.com/faucets/base-sepolia) before running the mint script.
