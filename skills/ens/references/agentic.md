# Agentic ENS (Scenario 3)

Using ENS as identity and discovery for AI agents — what's standardized, what tools exist, what to install.

This is the newest area in the ENS ecosystem; many pieces are draft-stage. Treat the specifics here as a snapshot — verify status against the source URLs before committing to a design.

## Contents

- [Why ENS for agents](#why-ens-for-agents)
- [ENSIP-25 — agent name verification](#ensip-25--agent-name-verification)
- [ERC-8004 — trustless agents](#erc-8004--trustless-agents)
- [ENS MCP servers](#ens-mcp-servers)
- [Patterns for "an agent that uses ENS"](#patterns-for-an-agent-that-uses-ens)
- [Footguns](#footguns)

---

## Why ENS for agents

ENS gives an agent:
- A **portable, human-readable identity** (`my-agent.eth`) that can move across hosts and frameworks.
- A **canonical address** (the resolved owner / signer).
- **Profile records** (description, capabilities, endpoints) via text records (ENSIP-5/18).
- **Pluggable verification** — anyone can check the agent's claimed identity onchain.
- **Optional payment routing** via the resolved address.

Compared to ad-hoc agent registries (a JSON file on a website, a GitHub repo of agent cards), ENS gives you the same trust properties as Ethereum addresses — control of the name maps to control of a key.

## ENSIP-25 — agent name verification

**Status**: Draft. **Reference**: search "ENSIP-25" on [docs.ens.domains/ensips](https://docs.ens.domains/ensips).

ENSIP-25 proposes a standard for AI agents to claim and prove an ENS name as their identity. The intent is:
- Agents register or claim a name (or subname under a host's parent).
- A standardized text record set declares what the agent is, what it does, and how to reach it.
- Verifiers (other agents, dapps, marketplaces) can confirm "this server is actually `my-agent.eth`" by checking onchain ownership against the agent's request signatures.

Because the spec is draft, treat ENSIP-25 as the *direction* the ENS community is heading rather than a stable contract. If you build now, structure your agent metadata as text records under a name you control — that pattern survives most likely changes.

## ERC-8004 — trustless agents

**Status**: Draft (Ethereum standards track). Search [eips.ethereum.org](https://eips.ethereum.org) and the [ERCs repo](https://github.com/ethereum/ERCs).

ERC-8004 proposes a comprehensive framework with three registries:

| Registry | Role |
|---|---|
| **Identity Registry** | ERC-721-based agent registration with portable identifiers and metadata (endpoints for MCP, A2A, ENS names, DIDs, email). |
| **Reputation Registry** | Aggregates feedback / scores. |
| **Validation Registry** | Independent attestation hooks (TEE attestation, cryptographic proofs, etc.). |

Trust models are pluggable (reputation, crypto-economic, attestation) and tiered by value-at-risk. Critically for ENS: ERC-8004 explicitly supports ENS as an agent endpoint, so an agent's NFT card can carry an `ens` field that resolves to its address and profile.

For developers: if you want your agent discoverable via ERC-8004 *and* ENS, set up both: register an ENS name, set the ERC-8004 NFT to reference that name, and put your agent metadata in the name's text records.

## ENS MCP servers

MCP (Model Context Protocol) servers let an agent (Claude, etc.) call ENS as a tool. Notable options in the ecosystem (status as of recent research; check the repos for currency):

| Server | Scope | When to use |
|---|---|---|
| **ens-mcp** (community) | Purpose-built for ENS — name lookups, profile data, availability, pricing. ~20 tools. | Building an agent whose primary surface is ENS work. |
| **mcp-etherscan-server** (community) | Etherscan tools including ENS resolution alongside balances, txs, ABIs. | Broader chain-data needs where ENS is one of many. |
| **ethid-mcp** (community) | Multi-protocol identity — ENS + EFP + EIK + SIWE. | Building identity-layer agents. |

There is no official ENS Labs MCP server at the time of writing. Verify the repos before installing — anyone can publish under "ENS" branding; check the maintainers and recent activity.

For Claude Code / Claude Desktop, install via the standard MCP config (`mcp.json` or settings UI). The agent then has tools like `ens_resolve_name`, `ens_get_text`, etc., callable in the conversation.

## Patterns for "an agent that uses ENS"

### 1. Agent identity

```
1. Register a name (or subname under a parent you control).
2. Set the address record to the agent's signer key.
3. Set text records:
   - description: what the agent does
   - url: where to reach it
   - avatar: a recognizable icon
   - (custom) e.g., 'agent.endpoint', 'agent.a2a', 'agent.mcp'
4. Optional: register the agent's NFT card per ERC-8004 referencing the name.
```

### 2. Agent calling ENS (via MCP)

Install an ENS MCP server. The agent now has tool access to `resolve_name`, `get_text`, `reverse_lookup`. It can:
- Translate "send to vitalik.eth" into an address.
- Read the recipient's profile to confirm identity.
- Verify reverse-then-forward for any address it shows the user.

### 3. Agent verifying another agent

```
1. Receive a message claiming to be from `other-agent.eth`.
2. Resolve `other-agent.eth` → expected address.
3. Verify the message was signed by that address.
4. Read profile records for capabilities / endpoints.
```

This works today with stable ENS primitives — no ENSIP-25 needed. ENSIP-25 will eventually standardize *what* records to set, but you can build the verification flow against current resolvers.

## Footguns

1. **Treating draft specs as stable.** ENSIP-25 and ERC-8004 are not finalized. Wrap your usage in adapters so you can swap as the standards land.
2. **Trusting an agent's "claimed" name without resolving.** Same problem as user reverse resolution — verify forward.
3. **Storing agent secrets in text records.** Anyone can read text records. They're public.
4. **Skipping normalization on agent names.** Agents can be victims of homoglyph spoofing too. Normalize before trusting.
5. **Installing an unaudited MCP server.** ENS MCPs run with whatever permissions you grant. Read the source and pin a version.

## Sources

- [docs.ens.domains/ensips](https://docs.ens.domains/ensips) (search ENSIP-25)
- [eips.ethereum.org](https://eips.ethereum.org) (search ERC-8004)
- [Model Context Protocol spec](https://modelcontextprotocol.io)
- [github.com/ensdomains/ensips](https://github.com/ensdomains/ensips)
- ENS-related MCP servers — search github.com for `ens mcp`
