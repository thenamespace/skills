# Subnames (Scenario 3)

A subname is a name issued under a parent ENS name (`alice.app.eth` under `app.eth`). The parent's owner controls who/what gets a subname and how. Subnames behave like any other ENS name — they resolve to addresses, hold text records, hold contenthash — and apps consume them through the same resolution surface as `app.eth` itself.

For namehash mechanics and per-record interfaces, see [records.md](records.md). For normalization, see [normalization.md](normalization.md).

## Contents

- [Why issue subnames — use cases](#why-issue-subnames--use-cases)
- [Technical paths](#technical-paths)
  - [Path A — Onchain subnames on mainnet](#path-a--onchain-subnames-on-mainnet)
  - [Path B — Onchain subnames on an L2 (custom resolver)](#path-b--onchain-subnames-on-an-l2-custom-resolver)
  - [Path C — Offchain subnames (CCIP-Read / wildcard)](#path-c--offchain-subnames-ccip-read--wildcard)
- [Choosing a path](#choosing-a-path)
- [DNS import (ENSIP-6)](#dns-import-ensip-6)
- [Subname normalization gotchas](#subname-normalization-gotchas)
- [Footguns](#footguns)

---

## Why issue subnames — use cases

Subnames let one ENS name become a namespace. Three audiences, with patterns each one tends to use.

### Individuals

- **Label your own wallets.** `metamask.alice.eth`, `ledger.alice.eth`, `embedded-dapp1.alice.eth` — readable names instead of address fragments. _Caveat_: mapping all your wallets to one public name links them on-chain; not advised if you want pseudonymity across wallets.
- **Decentralized blog / personal site.** Set `contenthash` on `alice.eth` (or a subname like `blog.alice.eth`) to an IPFS site — `vitalik.eth`-style. Brave and similar browsers render it natively.
- **Project subdomains.** A creator can issue subnames for their team members: `happy.company.eth`, `art.company.eth`, each with its own records.

### Communities

- **Member identity / membership NFTs.** `happy.shefi.eth`, `<member>.pizzadao.eth`. The subname acts as both an identity badge and (when issued onchain as ERC-721 via the Name Wrapper or a custom registrar) a transferable membership token.
- **Flaunt-on-socials reach.** Members put `@happy.shefi.eth` in their bios; every appearance is community marketing. The "name" itself does the distribution work.
- **Monetization.** Sell subnames at fixed or tiered pricing; enforce policy in a custom registrar contract (renewals, length-based pricing, allowlists).
- **DAO sub-orgs.** `treasury.dao.eth`, `core-team.dao.eth` — named addresses for working groups, with their own records and reverse names.

### Products

- **Wallets.** Issue a subname per user under your namespace (e.g. `<user>.uni.eth`, `<user>.cb.id`). Users get a portable identity that travels across dapps without you running a centralized username service.
- **Wallet-as-a-Service / embedded wallets.** Provide subname issuance as platform infra so every dapp using your WaaS can give its users a real ENS handle without doing the protocol work themselves.
- **L2 naming services.** A chain stands up a parent (`base.eth` → Basenames, `linea.eth` → Linea Names, `celo.eth` → Celo Names) and issues subnames as the identity layer for that chain. Users get one canonical name per L2.
- **Social / games / DIDs.** Replace siloed in-app usernames with an ENS subname so the same handle works across every dapp the user touches. The user's reputation, profile, and contacts become portable.

The pattern in all three: subnames let you give your users / members a real, portable ENS identity _under your brand_, without each user needing to register their own `.eth` second-level name.

---

## Technical paths

Three ways to issue subnames, in increasing order of programmability and decreasing cost-per-subname.

### Path A — Onchain subnames on mainnet

Use this when subnames are few, infrequent, and don't need any custom logic. Parent owner sets each child directly in the ENS Registry on Ethereum mainnet:

```solidity
// Parent owner calls on the ENS Registry:
registry.setSubnodeOwner(parentNode, keccak256(label), childOwner);
// Then sets a resolver on the child node:
registry.setResolver(childNode, resolverAddress);
// Child owner sets records on that resolver.
```

- Fully onchain on mainnet; child has real ownership and can transfer.
- Visible in the [subgraph](subgraph.md) and to every ENS-aware tool.
- Every action — issuance, transfer, record update — costs mainnet gas.

**Fits**: a handful to a few hundred named entities (team members, DAO sub-orgs, official subdomains) where per-subname gas is acceptable.

**Doesn't fit**: anything programmatic at scale, paid issuance with custom rules, or end-user-facing flows where users won't pay gas.

### Path B — Onchain subnames on an L2 (custom resolver)

Use this when you want **programmable issuance with onchain ownership**, but mainnet gas is too expensive — e.g. paid subnames, allowlists, length-based pricing, renewals, or community minting at scale.

Pattern:

1. Deploy a custom registrar/resolver contract on an L2 (Base, Optimism, Arbitrum, etc.) that holds subname state and exposes whatever issuance policy you want (`mint(label, to, …)`, pricing, gating).
2. On mainnet, set the parent's resolver to a thin contract that bridges resolution requests to the L2 — typically using ENSIP-10 wildcard + CCIP-Read so the L2 state is read transparently from any client.
3. Clients query `something.your-name.eth` against any standard ENS resolver path; the L2 state is fetched and returned without the client knowing it lived on an L2.

- Onchain ownership: child still has a real, transferable record (just on an L2).
- Programmable: you write the registrar logic — pricing, expiry, fuses, allowlists.
- L2-priced gas, so paid issuance and renewals stay economical.
- Some tools that read mainnet directly may not see L2 state without going through the bridging resolver — pick libraries that route through the Universal Resolver (see [resolution.md](resolution.md)).

**Fits**: communities and products selling/issuing subnames where ownership semantics matter (NFTs, transfers, paid renewals), but mainnet gas would kill the unit economics.

### Path C — Offchain subnames (CCIP-Read / wildcard)

Use this when subnames must be **gasless to issue and update**, often free, and you control the gateway. The parent's resolver implements ENSIP-10 (`resolve(name, data)`) and synthesizes records on demand — typically by calling out to a database via [CCIP-Read](resolution.md#ccip-read).

- Gasless creation and updates; scales to millions of subnames; updates are instant.
- Requires running (or trusting) an HTTPS gateway that signs responses; the resolver's callback verifies the signature.
- **No NFT, no onchain record per subname.** From the chain's perspective the parent has a wildcard resolver; individual children don't exist as registry entries.
- **Behaves like an ENS name otherwise.** It resolves to an address, holds text records, holds contenthash — every consumer that uses the Universal Resolver path treats it identically to an onchain name.
- **Not visible in the subgraph** (see [subgraph.md → offchain limitation](subgraph.md#critical-limitation-offchain-names-are-invisible)).
- Child has no onchain control; the parent's gateway is the source of truth.

The protocol pieces, vendor-neutral:

1. A resolver contract implementing ENSIP-10 + EIP-3668 (its functions revert with `OffchainLookup`).
2. An HTTPS gateway that signs responses; the resolver's callback verifies the signature.
3. Set that resolver as the parent's resolver in the ENS Registry.

For the client-side flow (automatic in viem ≥ 2.35 / wagmi), see [resolution.md → CCIP-Read](resolution.md#ccip-read).

**Fits**: end-user-facing identity at scale where the goal is UX (free, instant) rather than onchain provenance — wallet usernames, social handles, gamer tags, dapp-side aliases.

**Doesn't fit**: anything that needs onchain ownership, transferability, or subgraph visibility.

---

## Choosing a path

| Need                                     | Path A (mainnet onchain) | Path B (L2 onchain)          | Path C (offchain)               |
| ---------------------------------------- | ------------------------ | ---------------------------- | ------------------------------- |
| Few named entities, no custom logic      | ✓                        | overkill                     | overkill                        |
| Paid / programmatic issuance at scale    | gas-prohibitive          | ✓                            | ✓ (if you don't need ownership) |
| Onchain ownership + transferability      | ✓                        | ✓                            | ✗                               |
| NFT / membership token semantics         | ✓                        | ✓                            | ✗                               |
| Free for end users, gasless updates      | ✗                        | partial (L2 fees)            | ✓                               |
| Visible in subgraph                      | ✓                        | partial (depends on indexer) | ✗                               |
| Resolves like a normal ENS name in dapps | ✓                        | ✓ (via Universal Resolver)   | ✓ (via Universal Resolver)      |

You can mix paths under one parent: keep "official" subnames (`team.app.eth`) onchain on mainnet, sell "premium" subnames onchain on an L2, and issue free user handles offchain — provided your resolver layering decides precedence per label and you document it.

---

## DNS import (ENSIP-6)

**Spec**: [ENSIP-6](https://docs.ens.domains/ensip/6). **Guide**: [docs.ens.domains/learn/dns](https://docs.ens.domains/learn/dns).

Owners of DNSSEC-enabled DNS names (`example.com`) can bring those names — and their subnames — into ENS:

- Requires DNSSEC enabled at the registrar.
- Verification by publishing a TXT record proving control.
- After import, `example.com` resolves through ENS resolvers like any `.eth` name; its subnames behave like onchain subnames of an ENS parent.

For most app integrations, you don't need to _do_ the import — you just need to handle non-`.eth` names in your resolution code (see [Rule 5 in SKILL.md](../SKILL.md#the-five-rules-that-apply-to-every-ens-integration)).

## Subname normalization gotchas

ENSIP-15 normalizes the _whole_ dotted name, not labels in isolation. Bidi rules and emoji ZWJ allowlists are evaluated across label boundaries — see [normalization.md](normalization.md). The recurring trap:

```ts
// Wrong — splits a name and breaks cross-label rules
const fullName = input
  .split(".")
  .map((label) => normalize(label))
  .join(".");

// Right — normalize the full name once
const fullName = normalize(input);
```

Emoji subnames (`🚀.app.eth`): the emoji must be a fully-qualified sequence; bare ZWJs are disallowed except inside an allowlisted sequence; presentation selectors are stripped in the stored form (`ens_normalize`) but restored for display (`ens_beautify`).

## Footguns

1. **Onchain subnames on mainnet at scale.** Gas compounds; if you need 1000+, move to Path B or Path C.
2. **Treating offchain subnames as queryable in the subgraph.** They aren't; they only exist at resolution time. Decide upfront whether a feature needs to enumerate them.
3. **Splitting a subname to normalize each label.** Bidi/ZWJ rules are cross-label.
4. **Forgetting to set a resolver on the child node.** `setSubnodeOwner` doesn't set a resolver — records won't read until you do.
5. **Burning the parent resolver while subnames depend on it.** All onchain subname records become unreachable; offchain spaces stop resolving.
6. **Mixing onchain children under an offchain wildcard parent without precedence rules.** Decide which wins for a given label and document it; otherwise debugging is miserable.
7. **Promising users "their NFT" for an offchain subname.** There is no NFT in Path C. Be explicit in product copy.
8. **Issuing subnames on an L2 without a mainnet resolver bridge.** Generic ENS clients won't find them. Route through the Universal Resolver path.

## Sources

- [ENSIP-1 — Namehash](https://docs.ens.domains/ensip/1)
- [ENSIP-6 — DNS-in-ENS](https://docs.ens.domains/ensip/6)
- [ENSIP-10 — Wildcard Resolution](https://docs.ens.domains/ensip/10)
- [EIP-3668 — CCIP-Read](https://eips.ethereum.org/EIPS/eip-3668)
- [docs.ens.domains/learn/dns](https://docs.ens.domains/learn/dns)
- [docs.ens.domains/resolvers/ccip-read](https://docs.ens.domains/resolvers/ccip-read)
