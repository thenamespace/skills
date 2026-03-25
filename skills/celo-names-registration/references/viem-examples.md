# Celo Names - Viem (TypeScript) Examples

All examples use [viem](https://viem.sh). Install with `npm install viem`.

## Setup

```typescript
import { createPublicClient, createWalletClient, http, namehash, type Address } from "viem";
import { privateKeyToAccount } from "viem/accounts";
import { celo } from "viem/chains";

const REGISTRAR = "0x9Eb22700eFa1558eb2e0E522eB1DECC8025C3127" as const;
const REGISTRY = "0x4d7912779679AFdC592CBd4674b32Fcb189395F7" as const;
const REVERSE_REGISTRAR = "0xa58E81fe9b61B5c3fE2AFD33CF304c454AbFc7Cb" as const;

const account = privateKeyToAccount(process.env.PRIVATE_KEY as `0x${string}`);

const publicClient = createPublicClient({
  chain: celo,
  transport: http("https://forno.celo.org"),
});

const walletClient = createWalletClient({
  account,
  chain: celo,
  transport: http("https://forno.celo.org"),
});
```

## ABIs

Minimal ABIs for each contract -- only the functions you need.

```typescript
const registrarAbi = [
  {
    name: "available",
    type: "function",
    stateMutability: "view",
    inputs: [{ name: "label", type: "string" }],
    outputs: [{ type: "bool" }],
  },
  {
    name: "rentPrice",
    type: "function",
    stateMutability: "view",
    inputs: [
      { name: "label", type: "string" },
      { name: "durationInYears", type: "uint64" },
    ],
    outputs: [{ type: "uint256" }],
  },
  {
    name: "rentPrice",
    type: "function",
    stateMutability: "view",
    inputs: [
      { name: "label", type: "string" },
      { name: "durationInYears", type: "uint64" },
      { name: "paymentToken", type: "address" },
    ],
    outputs: [{ type: "uint256" }],
  },
  {
    name: "register",
    type: "function",
    stateMutability: "payable",
    inputs: [
      { name: "label", type: "string" },
      { name: "durationInYears", type: "uint64" },
      { name: "owner", type: "address" },
      { name: "resolverData", type: "bytes[]" },
    ],
    outputs: [],
  },
  {
    name: "renew",
    type: "function",
    stateMutability: "payable",
    inputs: [
      { name: "label", type: "string" },
      { name: "durationInYears", type: "uint64" },
    ],
    outputs: [],
  },
] as const;

const registryAbi = [
  {
    name: "ownerOf",
    type: "function",
    stateMutability: "view",
    inputs: [{ name: "tokenId", type: "uint256" }],
    outputs: [{ type: "address" }],
  },
  {
    name: "expiries",
    type: "function",
    stateMutability: "view",
    inputs: [{ name: "node", type: "bytes32" }],
    outputs: [{ type: "uint256" }],
  },
  {
    name: "addr",
    type: "function",
    stateMutability: "view",
    inputs: [{ name: "node", type: "bytes32" }],
    outputs: [{ type: "address" }],
  },
  {
    name: "text",
    type: "function",
    stateMutability: "view",
    inputs: [
      { name: "node", type: "bytes32" },
      { name: "key", type: "string" },
    ],
    outputs: [{ type: "string" }],
  },
  {
    name: "setText",
    type: "function",
    stateMutability: "nonpayable",
    inputs: [
      { name: "node", type: "bytes32" },
      { name: "key", type: "string" },
      { name: "value", type: "string" },
    ],
    outputs: [],
  },
  {
    name: "setAddr",
    type: "function",
    stateMutability: "nonpayable",
    inputs: [
      { name: "node", type: "bytes32" },
      { name: "a", type: "address" },
    ],
    outputs: [],
  },
  {
    name: "safeTransferFrom",
    type: "function",
    stateMutability: "nonpayable",
    inputs: [
      { name: "from", type: "address" },
      { name: "to", type: "address" },
      { name: "tokenId", type: "uint256" },
    ],
    outputs: [],
  },
  {
    name: "multicall",
    type: "function",
    stateMutability: "nonpayable",
    inputs: [{ name: "data", type: "bytes[]" }],
    outputs: [{ name: "results", type: "bytes[]" }],
  },
  {
    name: "totalSupply",
    type: "function",
    stateMutability: "view",
    inputs: [],
    outputs: [{ type: "uint256" }],
  },
  {
    name: "balanceOf",
    type: "function",
    stateMutability: "view",
    inputs: [{ name: "owner", type: "address" }],
    outputs: [{ type: "uint256" }],
  },
] as const;

const reverseRegistrarAbi = [
  {
    name: "setName",
    type: "function",
    stateMutability: "nonpayable",
    inputs: [{ name: "name", type: "string" }],
    outputs: [{ type: "bytes32" }],
  },
] as const;
```

---

## Read Operations

### Check availability

```typescript
const isAvailable = await publicClient.readContract({
  address: REGISTRAR,
  abi: registrarAbi,
  functionName: "available",
  args: ["alice"],
});
console.log("Available:", isAvailable); // true or false
```

### Get price in native CELO

```typescript
import { formatEther } from "viem";

const price = await publicClient.readContract({
  address: REGISTRAR,
  abi: registrarAbi,
  functionName: "rentPrice",
  args: ["alice", 1n],
});
console.log("Price:", formatEther(price), "CELO");
```

### Get price in USDC

```typescript
import { formatUnits } from "viem";

const USDC = "0xcebA9300f2b948710d2653dD7B07f33A8B32118C" as Address;

const price = await publicClient.readContract({
  address: REGISTRAR,
  abi: registrarAbi,
  functionName: "rentPrice",
  args: ["alice", 1n, USDC],
});
console.log("Price:", formatUnits(price, 6), "USDC");
```

### Check name owner

```typescript
const node = namehash("alice.celo.eth");
const tokenId = BigInt(node);

const owner = await publicClient.readContract({
  address: REGISTRY,
  abi: registryAbi,
  functionName: "ownerOf",
  args: [tokenId],
});
console.log("Owner:", owner);
```

### Check expiry

```typescript
const node = namehash("alice.celo.eth");

const expiry = await publicClient.readContract({
  address: REGISTRY,
  abi: registryAbi,
  functionName: "expiries",
  args: [node],
});
console.log("Expires:", new Date(Number(expiry) * 1000));
```

### Read text record

```typescript
const node = namehash("alice.celo.eth");

const avatar = await publicClient.readContract({
  address: REGISTRY,
  abi: registryAbi,
  functionName: "text",
  args: [node, "avatar"],
});
console.log("Avatar:", avatar);
```

### Read address record

```typescript
const node = namehash("alice.celo.eth");

const addr = await publicClient.readContract({
  address: REGISTRY,
  abi: registryAbi,
  functionName: "addr",
  args: [node],
});
console.log("Address:", addr);
```

---

## Write Operations

### Register a name

```typescript
async function registerName(label: string, years: bigint) {
  const isAvailable = await publicClient.readContract({
    address: REGISTRAR,
    abi: registrarAbi,
    functionName: "available",
    args: [label],
  });

  if (!isAvailable) throw new Error(`${label}.celo.eth is not available`);

  const price = await publicClient.readContract({
    address: REGISTRAR,
    abi: registrarAbi,
    functionName: "rentPrice",
    args: [label, years],
  });

  const hash = await walletClient.writeContract({
    address: REGISTRAR,
    abi: registrarAbi,
    functionName: "register",
    args: [label, years, account.address, []],
    value: price,
  });

  const receipt = await publicClient.waitForTransactionReceipt({ hash });
  console.log("Registered!", receipt.transactionHash);
  return receipt;
}

await registerName("myname", 1n);
```

### Renew a name

```typescript
async function renewName(label: string, years: bigint) {
  const price = await publicClient.readContract({
    address: REGISTRAR,
    abi: registrarAbi,
    functionName: "rentPrice",
    args: [label, years],
  });

  const hash = await walletClient.writeContract({
    address: REGISTRAR,
    abi: registrarAbi,
    functionName: "renew",
    args: [label, years],
    value: price,
  });

  const receipt = await publicClient.waitForTransactionReceipt({ hash });
  console.log("Renewed!", receipt.transactionHash);
  return receipt;
}

await renewName("myname", 1n);
```

### Set primary name

```typescript
const hash = await walletClient.writeContract({
  address: REVERSE_REGISTRAR,
  abi: reverseRegistrarAbi,
  functionName: "setName",
  args: ["myname.celo.eth"],
});

await publicClient.waitForTransactionReceipt({ hash });
console.log("Primary name set!");
```

### Set text record

```typescript
const node = namehash("myname.celo.eth");

const hash = await walletClient.writeContract({
  address: REGISTRY,
  abi: registryAbi,
  functionName: "setText",
  args: [node, "avatar", "https://example.com/avatar.png"],
});

await publicClient.waitForTransactionReceipt({ hash });
```

### Set address record

```typescript
const node = namehash("myname.celo.eth");

const hash = await walletClient.writeContract({
  address: REGISTRY,
  abi: registryAbi,
  functionName: "setAddr",
  args: [node, "0xYourAddress"],
});

await publicClient.waitForTransactionReceipt({ hash });
```

### Transfer a name

```typescript
const node = namehash("myname.celo.eth");
const tokenId = BigInt(node);
const to = "0xRecipientAddress" as Address;

const hash = await walletClient.writeContract({
  address: REGISTRY,
  abi: registryAbi,
  functionName: "safeTransferFrom",
  args: [account.address, to, tokenId],
});

await publicClient.waitForTransactionReceipt({ hash });
console.log("Transferred!");
```

### Batch set multiple records with multicall

```typescript
import { encodeFunctionData } from "viem";

const node = namehash("myname.celo.eth");

const calls = [
  encodeFunctionData({
    abi: registryAbi,
    functionName: "setText",
    args: [node, "avatar", "https://example.com/avatar.png"],
  }),
  encodeFunctionData({
    abi: registryAbi,
    functionName: "setText",
    args: [node, "description", "My Celo name"],
  }),
  encodeFunctionData({
    abi: registryAbi,
    functionName: "setAddr",
    args: [node, account.address],
  }),
];

const hash = await walletClient.writeContract({
  address: REGISTRY,
  abi: registryAbi,
  functionName: "multicall",
  args: [calls],
});

await publicClient.waitForTransactionReceipt({ hash });
console.log("All records set!");
```

---

## Scripts

### Full registration with primary name

```typescript
import { createPublicClient, createWalletClient, http, formatEther, type Address } from "viem";
import { privateKeyToAccount } from "viem/accounts";
import { celo } from "viem/chains";

const REGISTRAR = "0x9Eb22700eFa1558eb2e0E522eB1DECC8025C3127" as const;
const REVERSE_REGISTRAR = "0xa58E81fe9b61B5c3fE2AFD33CF304c454AbFc7Cb" as const;

const registrarAbi = [
  {
    name: "available", type: "function", stateMutability: "view",
    inputs: [{ name: "label", type: "string" }],
    outputs: [{ type: "bool" }],
  },
  {
    name: "rentPrice", type: "function", stateMutability: "view",
    inputs: [{ name: "label", type: "string" }, { name: "durationInYears", type: "uint64" }],
    outputs: [{ type: "uint256" }],
  },
  {
    name: "register", type: "function", stateMutability: "payable",
    inputs: [
      { name: "label", type: "string" }, { name: "durationInYears", type: "uint64" },
      { name: "owner", type: "address" }, { name: "resolverData", type: "bytes[]" },
    ],
    outputs: [],
  },
] as const;

const reverseRegistrarAbi = [
  {
    name: "setName", type: "function", stateMutability: "nonpayable",
    inputs: [{ name: "name", type: "string" }],
    outputs: [{ type: "bytes32" }],
  },
] as const;

async function main() {
  const label = process.argv[2];
  const years = BigInt(process.argv[3] || "1");
  if (!label) {
    console.error("Usage: npx tsx register.ts <label> [years]");
    process.exit(1);
  }

  const account = privateKeyToAccount(process.env.PRIVATE_KEY as `0x${string}`);
  const publicClient = createPublicClient({ chain: celo, transport: http("https://forno.celo.org") });
  const walletClient = createWalletClient({ account, chain: celo, transport: http("https://forno.celo.org") });

  console.log(`Registering ${label}.celo.eth for ${years} year(s)...`);

  const available = await publicClient.readContract({
    address: REGISTRAR, abi: registrarAbi, functionName: "available", args: [label],
  });
  if (!available) { console.error(`${label}.celo.eth is not available`); process.exit(1); }

  const price = await publicClient.readContract({
    address: REGISTRAR, abi: registrarAbi, functionName: "rentPrice", args: [label, years],
  });
  console.log(`Price: ${formatEther(price)} CELO`);

  const regTx = await walletClient.writeContract({
    address: REGISTRAR, abi: registrarAbi, functionName: "register",
    args: [label, years, account.address, []], value: price,
  });
  await publicClient.waitForTransactionReceipt({ hash: regTx });
  console.log(`Registered! TX: ${regTx}`);

  const nameTx = await walletClient.writeContract({
    address: REVERSE_REGISTRAR, abi: reverseRegistrarAbi,
    functionName: "setName", args: [`${label}.celo.eth`],
  });
  await publicClient.waitForTransactionReceipt({ hash: nameTx });
  console.log(`Primary name set to ${label}.celo.eth`);
}

main().catch(console.error);
```

Run with:

```bash
PRIVATE_KEY=0x... npx tsx register.ts myname 1
```

### Batch check availability

```typescript
async function checkNames(labels: string[]) {
  const results = await Promise.all(
    labels.map(async (label) => {
      const available = await publicClient.readContract({
        address: REGISTRAR, abi: registrarAbi, functionName: "available", args: [label],
      });
      const price = await publicClient.readContract({
        address: REGISTRAR, abi: registrarAbi, functionName: "rentPrice", args: [label, 1n],
      });
      return { label, available, price: formatEther(price) };
    })
  );

  for (const r of results) {
    console.log(`${r.label}.celo.eth - available: ${r.available}, price: ${r.price} CELO/year`);
  }
}

await checkNames(["alice", "bob", "charlie", "dave"]);
```

### Look up name info

```typescript
import { namehash } from "viem";

async function lookupName(label: string) {
  const fullName = `${label}.celo.eth`;
  const node = namehash(fullName);
  const tokenId = BigInt(node);

  console.log(`=== ${fullName} ===`);
  console.log("Node:", node);

  try {
    const owner = await publicClient.readContract({
      address: REGISTRY, abi: registryAbi, functionName: "ownerOf", args: [tokenId],
    });
    console.log("Owner:", owner);
  } catch { console.log("Owner: not registered"); }

  try {
    const expiry = await publicClient.readContract({
      address: REGISTRY, abi: registryAbi, functionName: "expiries", args: [node],
    });
    console.log("Expires:", new Date(Number(expiry) * 1000));
  } catch {}

  try {
    const addr = await publicClient.readContract({
      address: REGISTRY, abi: registryAbi, functionName: "addr", args: [node],
    });
    console.log("Address:", addr);
  } catch { console.log("Address: not set"); }

  try {
    const avatar = await publicClient.readContract({
      address: REGISTRY, abi: registryAbi, functionName: "text", args: [node, "avatar"],
    });
    console.log("Avatar:", avatar || "not set");
  } catch {}
}

await lookupName("alice");
```
