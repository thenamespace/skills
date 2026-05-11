# ENS for AI Agents (Scenario 7)

How an AI agent uses ENS — both as a *consumer* (resolving names the user mentions) and as an *identity* (the agent itself has an ENS name, discoverable and verifiable).

For resolution mechanics, see [resolution.md](resolution.md). For setting records, see [records.md](records.md). For normalization, see [normalization.md](normalization.md). This file only covers what's specific to agents.

## Two angles

**Agent as consumer.** The user says "send 0.1 ETH to alice.eth on Base." Treat ENS names like any other untrusted input: normalize, resolve fresh on the right `coinType`, confirm the resolved address before signing. Same rules as a wallet UI. Don't introduce agent-specific shortcuts.

**Agent as identity.** The agent owns an ENS name (`my-agent.eth`). That name is how other agents and humans reach it, learn about it, and verify it's the agent it claims to be.

The rest of this file is about the identity angle.

## How an agent uses ENS as identity

Three building blocks, all draft specs — wrap usage in adapters.

| Spec | What it does | Where it lives |
|---|---|---|
| **ENSIP-26** | Two text record keys (`agent-context`, `agent-endpoint[<protocol>]`) for protocol-agnostic discovery. | ENS resolver `text()` |
| **ENSIP-25** | One parameterized text record key cryptographically linking an ENS name to an entry in an onchain agent registry. | ENS resolver `text()` |
| **ERC-8004** | The onchain registry that ENSIP-25 binds to (Identity + Reputation + Validation). | Ethereum smart contracts |

You can use them à la carte: ENSIP-26 alone for plain discovery, ENSIP-25 + ERC-8004 for verified identity, all three for full identity + discovery + reputation.

## ENSIP-26 — discovery records

**Spec**: [PR #65](https://github.com/ensdomains/ensips/pull/65) on `ensdomains/ensips` (draft, check current spec before relying on key format).

| Key | Purpose | Value |
|---|---|---|
| `agent-context` | Free-form discovery entry point. | Plain text / Markdown / YAML / JSON. No enforced schema. |
| `agent-endpoint[<protocol>]` | Protocol-specific connection URL. | URI (`https://`, `ipfs://`). |

Defined `<protocol>` values: `mcp`, `a2a`, `oasf`, `web`. Others can be added.

Example records on `my-agent.eth`:

```
agent-context        = "https://my-agent.example/about"
agent-endpoint[mcp]  = "https://my-agent.example/mcp/v1"
agent-endpoint[web]  = "https://my-agent.example"
```

These are flexible discovery hints, not cryptographic proofs. A consumer reads them and connects.

## ENSIP-25 — verification record

**Spec**: [docs.ens.domains/ensip/25](https://docs.ens.domains/ensip/25) (draft).

A single parameterized text key that proves "this ENS name is bound to this onchain agent ID":

```
key:   agent-registration[<registry>][<agentId>]
value: "1"  (any non-empty UTF-8)
```

- `<registry>`: ERC-7930 interoperable address (chain + address packed) of the agent registry.
- `<agentId>`: identifier issued by that registry.

No new resolver interface — it's a standard ENSIP-5 `text()` lookup. Non-empty value = the ENS name owner has acknowledged the binding. Without this record, any onchain entry claiming "I am `my-agent.eth`" is self-asserted and spoofable.

## ERC-8004 in brief

**Spec**: [eips.ethereum.org/EIPS/eip-8004](https://eips.ethereum.org/EIPS/eip-8004) (draft).

Three independent registries:

- **Identity Registry (ERC-721)** — `register(agentURI, metadata) → agentId`. Pointer to off-chain JSON describing the agent (services, endpoints, payment wallet).
- **Reputation Registry** — `giveFeedback(agentId, …)`; aggregated ratings + pointers to detailed reports.
- **Validation Registry** — agents request third-party validators; validators respond onchain with scored results.

For agent integration, you mostly need to know:
- Identity issues the `agentId` you'll use in ENSIP-25.
- The three registries are independent — registration in one isn't presence in others.
- The agent's ENS name is declared in the Identity Registry's `agentURI` JSON, but **that claim is self-asserted** until backed by ENSIP-25.

Full registry interfaces, AgentURI schema, and `agentWallet` / EIP-712 details are in the [ERC-8004 spec](https://eips.ethereum.org/EIPS/eip-8004) — read it before implementing.

## Setting up an agent's ENS identity

1. Pick / register `my-agent.eth` (or a subname under your org).
2. Set ENSIP-26 discovery records via your library's `setText` — context + endpoint(s).
3. *(Optional, if joining ERC-8004)* Register with the Identity Registry → get `agentId`. Set the ENSIP-25 `agent-registration[<registry>][<agentId>] = "1"` record on the same ENS name.
4. *(Optional)* Engage with Reputation / Validation registries as the use case requires.

Setting records uses the same mechanics as any other ENS write — see [records.md](records.md).

## Verification flow (security-critical)

When you receive a connection or message claiming to come from `their-agent.eth` with `agentId N`:

```
1. Normalize their-agent.eth, forward-resolve to its address.
2. Verify any signed payload was signed by that address (or by getAgentWallet(N) if using ERC-8004).
3. Construct the ENSIP-25 key: agent-registration[<registry-erc7930>][<N>].
4. Read text(namehash("their-agent.eth"), key).
5. Non-empty → the binding is acknowledged. Empty → DO NOT trust the agentId/ENS pairing.
6. (Optional) Walk the Identity Registry's agentURI; check Reputation / Validation for trust signals.
```

Step 4 is the cryptographic proof. Steps 1–2 are the normal ENS forward-resolution rules — see [resolution.md](resolution.md) and [records.md → reverse](records.md#reverse-resolution).

## Footguns

1. **Trusting an ERC-8004 agent's `services: ENS` entry without ENSIP-25.** Self-asserted; spoofable. Always read the verification record.
2. **Wrong ERC-7930 encoding in the ENSIP-25 key.** A raw hex address, missing leading zeros, or wrong byte order produces a key the resolver doesn't have a record for — verification silently fails. Use a library that produces canonical ERC-7930 encodings.
3. **`agent-context` parsed without defenses.** No schema is enforced. Handle plain text, YAML, JSON, Markdown, or garbage defensively.
4. **Assuming one `agentId` works across all three ERC-8004 registries.** They're independent. Verify the registry contract address returned by Reputation/Validation matches the Identity Registry you trust.
5. **`agentWallet` updates via generic `setMetadata()`.** Won't change the wallet — `agentWallet` is protected by EIP-712 and requires `setAgentWallet(newWallet, deadline, signature)`.
6. **Treating draft specs as stable.** ENSIP-25, ENSIP-26, and ERC-8004 are all draft. Wrap usage in adapters; re-check before shipping.
7. **Skipping normalization on agent ENS names.** Agents are exposed to homoglyph spoofing the same way users are.

## Sources

- [ENSIP-25 — registry verification text record](https://docs.ens.domains/ensip/25)
- [ENSIP-26 — discovery records (draft PR #65)](https://github.com/ensdomains/ensips/pull/65)
- [ERC-8004 — three-registry framework](https://eips.ethereum.org/EIPS/eip-8004)
- [ERC-7930 — Interoperable Addresses](https://eips.ethereum.org/EIPS/eip-7930)
