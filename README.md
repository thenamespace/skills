# Namespace Skills

Skills for integrating [Namespace](https://namespace.ninja) or ENS in your applications or agents.

## Skills

### `skills/offchain-ens-subname-sdk`

Create and manage gasless ENS subnames off-chain using the `@thenamespace/offchain-manager` SDK. Supports `.eth` domains, imported web2 domains, and alternative TLDs.

- **Auth**: API key from [dev.namespace.ninja](https://dev.namespace.ninja)
- **Setup**: `npm install @thenamespace/offchain-manager`
- **Entry point**: [`skills/offchain-ens-subname-sdk/SKILL.md`](skills/offchain-ens-subname-sdk/SKILL.md)

### `skills/celonames`

Register, renew, and manage Celo Names (`*.celo.eth`) on-chain via the L2Registrar contract on Celo mainnet. Names are ERC721 NFTs with cross-chain resolution via CCIP-Read.

- **Auth**: Wallet with CELO or stablecoin balance
- **Setup**: Celo RPC access (e.g., `https://forno.celo.org`)
- **Entry point**: [`skills/celonames/SKILL.md`](skills/celonames/SKILL.md)

### `skills/resolvio`

Resolve ENS names and addresses via the Resolvio API, including full profile lookups (texts, addresses, contenthash) and batch reverse resolution for leaderboards/tables.

- **Auth**: None required
- **Setup**: HTTP client (examples use `curl`)
- **Entry point**: [`skills/resolvio/SKILL.md`](skills/resolvio/SKILL.md)

## Which skill should I use?

| Scenario                                                  | Skill                      |
| --------------------------------------------------------- | -------------------------- |
| I want gasless, off-chain subnames for my app's users     | `offchain-ens-subname-sdk` |
| Your agent needs to register a `*.celo.eth` name on-chain | `celonames`                |
| I need ENS profile/reverse resolution over HTTP           | `resolvio`                 |

## Installation

```bash
npx skills add thenamespace/skills --yes
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
