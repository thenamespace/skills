# Agentic ENS — ENSIP-25, ENSIP-26, ERC-8004

How AI agents use ENS as identity and discovery, in concrete terms. All three specs are **draft / not final** — interfaces may shift. Wrap your usage in adapters and treat addresses / keys as configuration, not constants.

## Contents

- [The three specs at a glance](#the-three-specs-at-a-glance)
- [ENSIP-25 — registry verification text record](#ensip-25--registry-verification-text-record)
- [ENSIP-26 — protocol-agnostic discovery records](#ensip-26--protocol-agnostic-discovery-records)
- [ERC-8004 — the three registries](#erc-8004--the-three-registries)
- [End-to-end: registering and being discovered](#end-to-end-registering-and-being-discovered)
- [Verification flow (the security-critical part)](#verification-flow-the-security-critical-part)
- [Footguns](#footguns)

---

## The three specs at a glance

| Spec | What it does | Where it lives | Status |
|---|---|---|---|
| **ENSIP-25** | One text record key that cryptographically links an ENS name to an ERC-8004 agent ID. | ENS resolver `text()` | Draft |
| **ENSIP-26** | Two text record keys for protocol-agnostic agent discovery (context + endpoints). | ENS resolver `text()` | Draft (PR #65 in `ensdomains/ensips`) |
| **ERC-8004** | Three onchain registries — Identity (ERC-721), Reputation, Validation — for agent registration, feedback, and third-party verification. | Ethereum smart contracts | Draft / proposed |

**They compose**:
- ERC-8004 gives an agent an onchain ID and a place for reputation/validation.
- ENSIP-25 lets the agent prove "this ENS name is mine" by setting a verification text record.
- ENSIP-26 lets the agent (or anyone holding an ENS name) publish discovery info — how to reach the agent and where its registry record lives — without depending on ERC-8004.

You can use ENSIP-26 alone (just discovery), or ENSIP-25 + ERC-8004 together (verified identity), or all three (full identity + discovery + reputation).

---

## ENSIP-25 — registry verification text record

**Spec**: [docs.ens.domains/ensip/25](https://docs.ens.domains/ensip/25). **Status**: Draft.

ENSIP-25 standardizes a single parameterized text record key that links an ENS name to an entry in a registry contract (ERC-8004 being the primary intended target).

**Key format**:

```
agent-registration[<registry>][<agentId>]
```

- `<registry>`: an ERC-7930 *interoperable address* identifying the registry contract (chain + address packed). E.g., `0x000100000101148004a169fb4a3325136eb29fa0ceb6d2e539a432`.
- `<agentId>`: the agent identifier issued by that registry. E.g., `167`.

**No new resolver interface** — uses standard ENSIP-5 `text(bytes32 node, string key)`. Returns a UTF-8 string; non-empty means "the name owner has acknowledged this registry binding," typically `"1"`.

**Verification protocol** (what a verifier does):

```
1. Get the agent's claimed ENS name, agentId, and registry address from
   the ERC-8004 registry (e.g., from the agent's services list).
2. Construct the key: `agent-registration[<registry-erc7930>][<agentId>]`.
3. Call text(namehash(claimedName), key) on the resolver of the claimed name.
4. If non-empty value → the binding is acknowledged. The ENS name is verified
   to that agent.
```

This is the **cryptographic proof** that the ENS name actually belongs to the agent ID — without it, anyone could claim any name in their ERC-8004 services list.

---

## ENSIP-26 — protocol-agnostic discovery records

**Spec**: draft in [PR #65](https://github.com/ensdomains/ensips/pull/65) on `ensdomains/ensips`. **Status**: Draft, early-stage; check current spec before relying on key format.

Two text record keys for "how do I reach this agent / where do I learn more about it":

| Key | Purpose | Value |
|---|---|---|
| `agent-context` | Free-form discovery entry point. Pointer to registries, descriptions, configuration. | Plain text, Markdown, YAML, or JSON. No enforced schema. |
| `agent-endpoint[<protocol>]` | Protocol-specific connection URL. | URI (`https://`, `http://`, `ipfs://`). |

Defined `<protocol>` values: `mcp`, `a2a`, `oasf`, `web`. Others can be added.

**Example records on `my-agent.eth`**:

```
agent-context           = "https://registry.example.com/agents/my-agent"
agent-endpoint[mcp]     = "https://my-agent.eth/mcp/v1"
agent-endpoint[web]     = "https://my-agent.eth"
```

**No verification step** — these are flexible discovery hints, not cryptographic proofs. A consumer reads them and connects.

**Difference from ENSIP-25**: ENSIP-25 *proves* a registry binding. ENSIP-26 *describes* how to reach the agent. Pair them when you want both.

---

## ERC-8004 — the three registries

**Spec**: [eips.ethereum.org/EIPS/eip-8004](https://eips.ethereum.org/EIPS/eip-8004) (also in [`ethereum/ERCs`](https://github.com/ethereum/ERCs/blob/master/ERCS/erc-8004.md)). **Status**: Draft.

Three independent registries:

### Identity Registry (ERC-721)

Issues a unique `agentId` per registered agent and stores a pointer to off-chain metadata.

```solidity
function register(string agentURI, MetadataEntry[] metadata) returns (uint256 agentId);
function getMetadata(uint256 agentId, string metadataKey) returns (bytes);
function setAgentWallet(uint256 agentId, address newWallet, uint256 deadline, bytes signature);
function getAgentWallet(uint256 agentId) returns (address);
```

`agentURI` points to a JSON file (typically IPFS or HTTPS) describing the agent:

```json
{
  "type": "https://eips.ethereum.org/EIPS/eip-8004#registration-v1",
  "name": "MyAgent",
  "description": "...",
  "image": "ipfs://...",
  "services": [
    { "name": "ENS", "endpoint": "my-agent.eth", "version": "v1" },
    { "name": "MCP", "endpoint": "https://mcp.my-agent.eth/", "version": "2025-06-18" }
  ],
  "active": true,
  "registrations": [{ "agentId": 167, "agentRegistry": "0x..." }],
  "x402Support": false
}
```

The `services` array is where the agent declares its ENS name (and other endpoints). **This claim is self-asserted and not verified by ERC-8004 itself** — pair with ENSIP-25 to get cryptographic proof.

`agentWallet` (a reserved metadata key) is the agent's payment address — protected by EIP-712 signature, only changeable via `setAgentWallet()`.

### Reputation Registry

Onchain feedback for agents; aggregates ratings/tags with pointers to off-chain detailed reports.

```solidity
function giveFeedback(
  uint256 agentId,
  int128 value, uint8 valueDecimals,    // e.g., 4500 with 2 decimals = 45.00
  string tag1, string tag2,
  string endpoint,
  string feedbackURI, bytes32 feedbackHash
);

function getSummary(uint256 agentId, address[] clients, string tag1, string tag2)
  returns (uint64 count, int128 summaryValue, uint8 summaryValueDecimals);

function readFeedback(uint256 agentId, address client, uint64 idx)
  returns (int128 value, uint8 decimals, string tag1, string tag2, bool isRevoked);
```

### Validation Registry

Agents request third-party validators to verify claims/work; validators respond onchain with scored results.

```solidity
function validationRequest(
  address validatorAddress, uint256 agentId,
  string requestURI, bytes32 requestHash
) returns (bytes32 requestHash);

function validationResponse(
  bytes32 requestHash,
  uint8 response,            // 0–100 score, or binary
  string responseURI, bytes32 responseHash,
  string tag
);

function getValidationStatus(bytes32 requestHash)
  returns (address validator, uint256 agentId, uint8 response, ..., uint256 lastUpdate);
```

The three registries are **independent contracts**. An agent registered in Identity isn't automatically present in Reputation or Validation — each is queried separately. AgentId schemes may differ across deployments; verify the registry contract address returned by Reputation/Validation against your Identity Registry.

---

## End-to-end: registering and being discovered

Concrete walk-through for `my-agent.eth` registering with an ERC-8004 deployment and becoming discoverable.

**Step 1 — Register with the Identity Registry**:

```solidity
// Agent owner pins a JSON file describing the agent (with ENS in services[]):
// ipfs://Qm.../agent.json

uint256 agentId = identityRegistry.register("ipfs://Qm.../agent.json", []);
// returns e.g. agentId = 167
```

**Step 2 — Set the ENSIP-25 verification record on `my-agent.eth`**:

```
key:   agent-registration[0x000100000101148004a169fb4a3325136eb29fa0ceb6d2e539a432][167]
value: "1"
```

(Set via standard ENS resolver `setText(node, key, value)`.)

This step is what makes the agent's ENS claim *verifiable*. Skip it and the `services: ENS` entry in the agent URI is a self-claim with no proof.

**Step 3 — Optionally set ENSIP-26 discovery records on `my-agent.eth`**:

```
agent-context        = "https://registry.example.com/agents/my-agent"
agent-endpoint[mcp]  = "https://mcp.my-agent.eth/v1"
agent-endpoint[web]  = "https://my-agent.eth"
```

**Step 4 — Optionally accept reputation / validation**: clients call `reputationRegistry.giveFeedback(167, ...)` after interacting; validators respond to `validationRequest`s the agent files.

The agent is now discoverable two ways: directly via ENS text records (ENSIP-26), or via ERC-8004 registry walks (with ENSIP-25 verification).

---

## Verification flow (the security-critical part)

When you receive a connection / message claiming to come from `their-agent.eth` + `agentId N`:

```
1. Resolve their-agent.eth to its address (forward resolution).
2. Verify any signed payload was signed by that address (or by getAgentWallet(N)).
3. Construct the ENSIP-25 key:
     agent-registration[<registry-erc7930>][<N>]
4. Call text(namehash("their-agent.eth"), key).
5. If non-empty → the agent owner has acknowledged the registry binding.
   If empty → the claim does NOT verify. Treat as untrusted.
6. (Optional) Walk to the Identity Registry, fetch the agentURI, parse the
   services array, optionally check Reputation / Validation registries for
   trust signals.
```

The ENSIP-25 step is the cryptographic proof; the ERC-8004 services list alone is self-asserted and spoofable.

---

## Footguns

1. **Trusting the `services: ENS` entry from an ERC-8004 agent without checking ENSIP-25.** The services array is self-asserted. Without the ENSIP-25 verification record, any agent can claim any ENS name. Always verify the `agent-registration[…][…]` text record.

2. **Wrong ERC-7930 encoding in the ENSIP-25 key.** The registry address inside the key uses a packed interoperable-address format. A raw hex address, missing leading zeros, or wrong byte order produces a key the resolver doesn't have a record for — verification silently fails. Use a library that produces canonical ERC-7930 encodings.

3. **Unpinned agent URIs.** If you `register()` with an IPFS URI that later becomes unavailable, verifiers can't fetch your services list. Pin (and ideally treat the URI as immutable for the lifetime of the agent registration). If you must update, use `setAgentURI()` and keep the old URI resolvable for a transition window.

4. **`agent-context` parsed without defenses.** No schema is enforced. Tools reading `agent-context` must handle plain text, YAML, JSON, Markdown, or garbage — defensively.

5. **Assuming one `agentId` works across all three ERC-8004 registries.** They're independent. Verify the registry contract address returned by Reputation/Validation matches the Identity Registry you trust.

6. **`agentWallet` updates via generic `setMetadata()`.** That call may succeed but won't actually change the wallet — `agentWallet` is protected by EIP-712 signature and requires `setAgentWallet(newWallet, deadline, signature)`.

7. **Treating draft specs as stable.** ENSIP-25, ENSIP-26, and ERC-8004 are all draft. Wrap usage in adapters, version-tag your records, and re-check the specs before shipping production agent infrastructure.

8. **Skipping normalization on agent ENS names.** Agents are exposed to homoglyph spoofing the same way users are. Normalize claimed names before any resolution or comparison. See [normalization.md](normalization.md).

## Sources

- [ENSIP-25 (docs.ens.domains)](https://docs.ens.domains/ensip/25) — verification text record
- [ENSIP-26 PR #65 (ensdomains/ensips)](https://github.com/ensdomains/ensips/pull/65) — discovery records (draft)
- [ERC-8004 (eips.ethereum.org)](https://eips.ethereum.org/EIPS/eip-8004) — three-registry framework
- [ERC-8004 in the ERCs repo](https://github.com/ethereum/ERCs/blob/master/ERCS/erc-8004.md) — latest draft
- [ERC-7930 — Interoperable Addresses](https://eips.ethereum.org/EIPS/eip-7930)
- [ENSIP-5 — Text Records](https://docs.ens.domains/ensip/5) — the underlying record interface
