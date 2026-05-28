# Namespace Skills

Skills for integrating [Namespace](https://namespace.ninja) or ENS in your applications or agents.

## Skills

### `skills/ens`

Vendor-neutral guide to the ENS protocol for app developers, auditors, and AI-agent builders. Covers normalization (ENSIP-15), forward + reverse resolution, CCIP-Read, multichain coin types, L2 primary names (ENSIP-19), records (text, avatar, contenthash, ABI, arbitrary bytes), subnames across three paths (mainnet onchain / L2 onchain custom resolver / offchain CCIP-Read) with use cases, smart-contract naming, subgraph queries, ENSv2 readiness, and AI-agent identity (ENSIP-25 / ENSIP-26 / ERC-8004).

- **Auth**: None required
- **Setup**: Library of choice (viem ≥ 2.35 + wagmi recommended; ethers v6 supported)
- **Entry point**: [`skills/ens/SKILL.md`](skills/ens/SKILL.md)
- **Install**: `npx skills add thenamespace/skills -s ens`

### `skills/offchain-ens-subname-sdk`

Create and manage gasless ENS subnames off-chain using the `@thenamespace/offchain-manager` SDK. Supports `.eth` domains, imported web2 domains, and alternative TLDs.

- **Auth**: API key from [dev.namespace.ninja](https://dev.namespace.ninja)
- **Setup**: `npm install @thenamespace/offchain-manager`
- **Entry point**: [`skills/offchain-ens-subname-sdk/SKILL.md`](skills/offchain-ens-subname-sdk/SKILL.md)
- **Install**: `npx skills add thenamespace/skills -s offchain-ens-subname-sdk`

### `skills/celonames`

Register, renew, and manage Celo Names (`*.celo.eth`) on-chain via the L2Registrar contract on Celo mainnet. Names are ERC721 NFTs with cross-chain resolution via CCIP-Read.

- **Auth**: Wallet with CELO or stablecoin balance
- **Setup**: Celo RPC access (e.g., `https://forno.celo.org`)
- **Entry point**: [`skills/celonames/SKILL.md`](skills/celonames/SKILL.md)
- **Install**: `npx skills add thenamespace/skills -s celonames`

### `skills/resolvio`

Resolve ENS names and addresses via the Resolvio API, including full profile lookups (texts, addresses, contenthash) and batch reverse resolution for leaderboards/tables.

- **Auth**: None required
- **Setup**: HTTP client (examples use `curl`)
- **Entry point**: [`skills/resolvio/SKILL.md`](skills/resolvio/SKILL.md)
- **Install**: `npx skills add thenamespace/skills -s resolvio`

### `skills/ens-components`

React UI kit for ENS and Namespace flows. Covers ENS name registration, record editing, onchain subname minting, offchain subnames, avatar/header upload, theming, and callback-driven customization. Built on wagmi.

- **Auth**: None required (wallet connection handled by wagmi)
- **Setup**: `npm install @thenamespace/ens-components`
- **Entry point**: [`skills/ens-components/SKILL.md`](skills/ens-components/SKILL.md)
- **Install**: `npx skills add thenamespace/skills -s ens-components`

## Which skill should I use?

| Scenario                                                                                      | Skill                      |
| --------------------------------------------------------------------------------------------- | -------------------------- |
| I'm integrating, debugging, or auditing ENS in my app (vendor-neutral protocol guide)         | `ens`                      |
| I want gasless, off-chain subnames for my app's users                                         | `offchain-ens-subname-sdk` |
| Your agent needs to register a `*.celo.eth` name on-chain                                     | `celonames`                |
| I need ENS profile/reverse resolution over HTTP                                               | `resolvio`                 |
| I want React components for ENS registration, records, or subnames                            | `ens-components`           |

## Installation

Install all skills at once:

```bash
npx skills add thenamespace/skills --yes
```

Or install a single skill with the `-s` flag:

```bash
npx skills add thenamespace/skills -s offchain-ens-subname-sdk
```

## Specification

These skills follow the [Agent Skills specification](https://agentskills.io/specification). See [spec/agent-skills-spec.md](spec/agent-skills-spec.md) for details.

## Official Links

- [Namespace Platform](https://namespace.ninja)
- [Developer Dashboard](https://dev.namespace.ninja)
- [Offchain Manager SDK](https://www.npmjs.com/package/@thenamespace/offchain-manager)
- [Documentation](https://docs.namespace.ninja)

## License

MIT
