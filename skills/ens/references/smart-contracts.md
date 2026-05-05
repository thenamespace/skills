# Smart Contracts (Scenario 4 — naming, addresses, ENS as infrastructure)

Naming a deployed contract, the key contract addresses you'll touch, EIP-165 interface checks, and using ENS as protocol infrastructure (service discovery, upgradeability pointers, registry of official extensions).

For querying ENS data at scale (subgraph, indexers), see [subgraph.md](subgraph.md). For the Universal Resolver in depth, see [resolution.md → Universal Resolver](resolution.md#the-universal-resolver). For ENSIP-24 arbitrary data records, see [records.md](records.md#arbitrary-bytes-records-ensip-24).

## Contents

- [Naming a smart contract](#naming-a-smart-contract)
- [Key contract addresses](#key-contract-addresses)
- [Verify resolver interface support (EIP-165)](#verify-resolver-interface-support-eip-165)
- [ENS as infrastructure: service discovery, upgrades, registries](#ens-as-infrastructure-service-discovery-upgrades-registries)
- [Footguns](#footguns)

---

## Naming a smart contract

**Guide**: [docs.ens.domains/web/naming-contracts](https://docs.ens.domains/web/naming-contracts).

When a contract has a primary ENS name, Etherscan and explorers display the name instead of the raw address, dapps can identify it, and your users see a meaningful label.

The mechanism is the same as for EOAs: set a reverse record on the Reverse Registrar from the contract's address. Three patterns:

### 1. Ownable pattern (recommended for upgradeable / owned contracts)

```solidity
import "@openzeppelin/contracts/access/Ownable.sol";

interface IReverseRegistrar {
    function setNameForAddr(address addr, address owner, address resolver, string calldata name)
        external returns (bytes32);
}

contract MyContract is Ownable {
    function setEnsName(address reverseRegistrar, address resolver, string calldata name) external onlyOwner {
        IReverseRegistrar(reverseRegistrar).setNameForAddr(address(this), owner(), resolver, name);
    }
}
```

Lets the owner update the name later (e.g., after a rebrand).

### 2. Constructor pattern (immutable contracts)

```solidity
constructor(address reverseRegistrar, string memory name) {
    IReverseRegistrar(reverseRegistrar).setName(name);
}
```

The contract claims its name at deployment. Cannot be changed after.

### 3. ReverseClaimer module

ENS provides `ReverseClaimer` (mainnet only) — inherit it and pass the owner who'll control the reverse node:

```solidity
import "@ensdomains/ens-contracts/contracts/reverseRegistrar/ReverseClaimer.sol";

contract MyContract is ReverseClaimer {
    constructor(ENS ens, address claimant) ReverseClaimer(ens, claimant) {}
}
```

After deployment, `claimant` can call the Reverse Registrar to set the name.

### Verify after deployment

Reverse-resolve the contract's address and confirm the name forwards back to it. Same forward-verification rule as for users (see [records.md → Reverse resolution](records.md#reverse-resolution)). For multichain contracts, use the L2's Reverse Registrar — the mainnet one won't set an L2 primary name.

### Why bother

- Etherscan shows the name in transaction lists and contract pages.
- dapps can identify the contract programmatically (their reverse lookup returns a meaningful name).
- Users confirming a transaction can see they're interacting with `vault.protocol.eth`, not `0x4a2…f9b`.
- Phishing imitations are easier to spot — the official name resolves; a fake address doesn't.

## Key contract addresses

These addresses change across deployments and per-chain. **Always look them up at integration time** rather than hardcoding:

- **Live deployments**: [docs.ens.domains/learn/deployments](https://docs.ens.domains/learn/deployments)
- **Source contracts**: [github.com/ensdomains/ens-contracts](https://github.com/ensdomains/ens-contracts)

Roles you'll encounter:

| Contract | Role |
|---|---|
| **ENS Registry** | The root contract. Maps namehash → owner, resolver, TTL. |
| **Public Resolver** | The default resolver most names point at; supports addr, multicoin addr, text, contenthash, ABI. |
| **Reverse Registrar** | Per chain. Maps address → primary name. Each L2 has its own deployment. |
| **Universal Resolver** | Aggregates registry + resolver + CCIP-Read into a single call. Multiple versions deployed during the ENSv2 transition. See [resolution.md](resolution.md#the-universal-resolver). |
| **DNS Registrar** | Handles ENSIP-6 DNS imports (DNSSEC verification). |
| **`.eth` Registrar Controller** | The contract for registering/renewing `.eth` second-level names. |

If you write contracts that integrate ENS, take addresses from a config file (or use the `ens-contracts` package's deployment exports), so you can swap them per network and chain without code changes. See [ensv2-readiness.md](ensv2-readiness.md) for the version-handling angle.

## Verify resolver interface support (EIP-165)

ENS resolvers can implement different combinations of interfaces — `addr`, multicoin `addr`, `text`, `contenthash`, `ABI`, ENSIP-10 `resolve`, ENSIP-24 `data`, etc. **Not every resolver implements every interface.** Calling a method on a resolver that doesn't support it can revert your transaction or return garbage.

The standard solution is [EIP-165](https://eips.ethereum.org/EIPS/eip-165): every resolver exposes `supportsInterface(bytes4 interfaceId) → bool`. Check before calling.

```solidity
bytes4 constant ADDR_INTERFACE_ID = 0x3b3b57de;          // addr(bytes32)
bytes4 constant TEXT_INTERFACE_ID = 0x59d1d43c;          // text(bytes32, string)
bytes4 constant CONTENTHASH_INTERFACE_ID = 0xbc1c58d1;   // contenthash(bytes32)

if (resolver.supportsInterface(TEXT_INTERFACE_ID)) {
    // Safe to call resolver.text(node, "url")
}
```

Practical patterns:

- **Cache the interface bitmap per resolver address.** Resolvers don't change shape often; one `supportsInterface` call per (resolver, interface) per session is cheap.
- **Implement graceful fallback.** If the resolver doesn't support an interface, return null/empty rather than reverting the user's whole flow.
- **For library authors**: see [library-authors.md](library-authors.md) for additional conformance guidance.

If you write a custom resolver, **implement EIP-165 properly**. Signal every interface you support via `supportsInterface()` and document them. Custom resolvers that omit EIP-165 are unreliable for any consumer that's checking — and the careful consumers are the ones you want using your resolver.

## ENS as infrastructure: service discovery, upgrades, registries

Beyond "human-readable address," ENS is useful as a programmable identity / pointer layer for protocol architecture. Three common patterns:

### Service discovery between contracts

Use ENS records to resolve service addresses at runtime instead of hardcoding them:

```solidity
// Resolve "oracle.protocol.eth" → current oracle address
address oracle = resolver.addr(namehash("oracle.protocol.eth"));
```

When you upgrade or rotate the oracle, update the ENS record once — every contract that resolves that name picks up the new address on next call. No coordinated re-deploy.

Caveats: an ENS lookup costs gas (an external call + storage read). Cache aggressively; use it for cold-path reads (admin, governance, periodic rotation) rather than hot paths (per-trade, per-block).

### Upgradeability pointers

Similar pattern for proxy implementations: the proxy reads `implementation.protocol.eth` to find its current implementation address. The owner updates the ENS record to ship a new version.

This is heavier than a typical EIP-1967 storage slot and only worth it if you want the implementation pointer to be **public, human-readable, and externally auditable** — auditors and integrations can verify which implementation a proxy is pointing at by resolving a name.

### Registry of official extensions / integrations

For a protocol with many official extensions (modules, plugins, integrations), maintain them as subnames:

```
core.protocol.eth     → main contract
oracle.protocol.eth   → oracle module
strategy-aave.protocol.eth → Aave strategy
strategy-comp.protocol.eth → Compound strategy
```

Users and tools can enumerate official extensions by walking the subname space (or via the [subgraph](subgraph.md)). A user who sees `strategy-foo.protocol.eth` resolve to a contract knows that contract was sanctioned by the protocol owner — it's signed by the namespace itself.

### Storing protocol metadata

Use [text records (ENSIP-5)](records.md#text-records--overview) on a protocol's name to publish:
- Human-readable docs URL (`url`).
- Contact / support handles (`com.twitter`, `org.telegram`).
- ABI for the main contract (ENSIP-4 — see [records.md](records.md#abi-text-record-ensip-4)).
- Distributed-web docs site (`contenthash` → IPFS).

This makes the protocol self-describing without a separate registry website.

## Footguns

1. **Hardcoded addresses.** Multiple Universal Resolver versions exist; per-chain addresses differ. Always lookup; see [ensv2-readiness.md](ensv2-readiness.md).
2. **Naming a contract once, never verifying.** Forward-resolve afterward to confirm it works.
3. **Reverse Registrar mismatch across chains.** Each L2 has its own; using the mainnet one from an L2 contract won't set the L2 primary name.
4. **Skipping `supportsInterface()` checks** when integrating with arbitrary resolvers — calls to unsupported methods can revert.
5. **Custom resolvers that don't implement EIP-165.** Other contracts can't reliably integrate with you.
6. **Calling the registry/resolver directly in a dapp.** Use the library helper that wraps the Universal Resolver — it's not just convenience, it's correctness.
7. **Hot-path ENS lookups for service discovery.** Each call is gas-heavy. Cache and refresh.

## Sources

- [docs.ens.domains/web/naming-contracts](https://docs.ens.domains/web/naming-contracts)
- [docs.ens.domains/learn/deployments](https://docs.ens.domains/learn/deployments)
- [docs.ens.domains/resolvers/universal](https://docs.ens.domains/resolvers/universal)
- [docs.ens.domains/resolvers/interfaces](https://docs.ens.domains/resolvers/interfaces)
- [docs.ens.domains/resolvers/public](https://docs.ens.domains/resolvers/public)
- [EIP-165 — Standard Interface Detection](https://eips.ethereum.org/EIPS/eip-165)
- [github.com/ensdomains/ens-contracts](https://github.com/ensdomains/ens-contracts)
