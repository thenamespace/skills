---
name: celo-names-registration
description: Use for any task involving celo.eth name [celonames] registration, price checks, renewals, transfers, primary name setup, or reading name records. Use this skill even if the user just says "register a name on Celo" or "check if a .celo.eth name is available".
---

# Celo Names Registration

Celo Names is an ENS-based naming system on Celo L2. Names are subnames of `celo.eth`, formatted as `{label}.celo.eth` (e.g., `alice.celo.eth`). Each name is an ERC-721 NFT on Celo. Cross-chain resolution from Ethereum L1 works via CCIP-Read.

## Network

|       |                          |
| ----- | ------------------------ |
| Chain | Celo Mainnet (42220)     |
| RPC   | `https://forno.celo.org` |

## Contracts

| Contract                        | Address                                      |
| ------------------------------- | -------------------------------------------- |
| L2Registrar (paid registration) | `0x9Eb22700eFa1558eb2e0E522eB1DECC8025C3127` |
| L2Registry (NFT + records)      | `0x4d7912779679AFdC592CBd4674b32Fcb189395F7` |
| ReverseRegistrar (primary name) | `0xa58E81fe9b61B5c3fE2AFD33CF304c454AbFc7Cb` |
| L2SelfRegistrar (free claim)    | `0x063E9F0bA0061F6C3c6169674c81f43BE21fe8cc` |
| RegistrarStorage                | `0xaAF67A46b99bE9a183580Cd86236cd0c6f2a85cb` |
| L1Resolver (Ethereum)           | `0xC6FC912C5DACb6BF0a24Bad113493c900fEA254a` |

## Payment Tokens

| Token         | Address                                      | Decimals |
| ------------- | -------------------------------------------- | -------- |
| CELO (native) | `0x0000000000000000000000000000000000000000` | 18       |
| USDC          | `0xcebA9300f2b948710d2653dD7B07f33A8B32118C` | 6        |
| USDT          | `0x48065fbbe25f71c9282ddf5e1cd6d6a887483d5e` | 6        |
| cUSD          | `0x765DE816845861e75A25fCA122bb6898B8B1282a` | 18       |

## Pricing

| Length   | USD/Year |
| -------- | -------- |
| 3 chars  | $400     |
| 4 chars  | $160     |
| 5 chars  | $50      |
| 6+ chars | $5       |

Duration: 1-10,000 years. 10% goes to ENS treasury.

## ABI - L2Registrar

The registrar handles paid name registration, renewal, and price checks.

```solidity
// Check if a label is available for registration
function available(string label) view returns (bool)

// Get price in native CELO (pass zero address as token)
function rentPrice(string label, uint64 durationInYears) view returns (uint256)

// Get price in a specific ERC-20 token
function rentPrice(string label, uint64 durationInYears, address paymentToken) view returns (uint256)

// Register with native CELO - send price as msg.value
function register(string label, uint64 durationInYears, address owner, bytes[] resolverData) payable

// Register with ERC-20 token via permit
function registerERC20(string label, uint64 durationInYears, address owner, bytes[] resolverData, address paymentToken, (uint256 value, uint256 deadline, uint8 v, bytes32 r, bytes32 s) permit)

// Renew with native CELO - send price as msg.value
function renew(string label, uint64 durationInYears) payable

// Renew with ERC-20 token via permit
function renewERC20(string label, uint64 durationInYears, address paymentToken, (uint256 value, uint256 deadline, uint8 v, bytes32 r, bytes32 s) permit)
```

## ABI - L2Registry

The registry is the NFT contract that holds names and resolver records. Token ID = `uint256(namehash("label.celo.eth"))`.

```solidity
// ERC-721
function ownerOf(uint256 tokenId) view returns (address)
function balanceOf(address owner) view returns (uint256)
function safeTransferFrom(address from, address to, uint256 tokenId)
function totalSupply() view returns (uint256)

// Name expiry (returns unix timestamp)
function expiries(bytes32 node) view returns (uint256)

// Resolver records
function addr(bytes32 node) view returns (address)
function addr(bytes32 node, uint256 coinType) view returns (bytes)
function text(bytes32 node, string key) view returns (string)
function contenthash(bytes32 node) view returns (bytes)

// Set resolver records (only name owner)
function setAddr(bytes32 node, address a)
function setAddr(bytes32 node, uint256 coinType, bytes a)
function setText(bytes32 node, string key, string value)
function setContenthash(bytes32 node, bytes hash)

// Batch multiple record updates
function multicall(bytes[] data) returns (bytes[])
function multicallWithNodeCheck(bytes32 nodehash, bytes[] data) returns (bytes[])
```

## ABI - ReverseRegistrar

Sets which name displays as your "primary name" when someone looks up your address.

```solidity
// Set your primary name (caller's address)
function setName(string name) returns (bytes32)

// Set primary name for another address (requires authorization)
function setNameForAddr(address addr, address owner, address resolver, string name) returns (bytes32)

// Get the reverse node for an address
function node(address addr) pure returns (bytes32)
```

## Registration Flow

```
1. available("alice")          → true/false
2. rentPrice("alice", 1)       → price in wei (native CELO)
3. register("alice", 1, owner, [])  with value = price
4. (optional) setName("alice.celo.eth")  → set as primary name
```

Pass only the label to the registrar -- `alice`, not `alice.celo.eth`.

## Computing namehash

The node (namehash) for `alice.celo.eth` is computed recursively:

```
namehash("") = 0x0000000000000000000000000000000000000000000000000000000000000000
namehash("eth") = keccak256(namehash("") + keccak256("eth"))
namehash("celo.eth") = keccak256(namehash("eth") + keccak256("celo"))
namehash("alice.celo.eth") = keccak256(namehash("celo.eth") + keccak256("alice"))
```

## Common Mistakes

| Mistake                            | Fix                                                 |
| ---------------------------------- | --------------------------------------------------- |
| Forgetting `value` in `register()` | Always pass `value: price` for native CELO payments |
| Using full name as label           | Pass only the label: `alice`, not `alice.celo.eth`  |
| Wrong `durationInYears` type       | It's `uint64`, not `uint256` (in V2 registrar)      |
| Expired name still shows owner     | Check `expiries(node)` against `block.timestamp`    |
| Not setting primary name           | Registration alone doesn't set reverse resolution   |

## Examples

For implementation examples with working code, read the appropriate reference file:

- **Cast (Foundry CLI)**: Read `references/cast-examples.md` for command-line examples using `cast call` and `cast send`
- **Viem (TypeScript)**: Read `references/viem-examples.md` for TypeScript examples using viem

## Environment Variables

```bash
PRIVATE_KEY=0x...              # Wallet private key (never commit)
CELO_RPC=https://forno.celo.org
```
