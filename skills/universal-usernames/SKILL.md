---
name: universal-usernames
description: Build the HTTPS endpoint that makes a brand's existing usernames resolve as ENS names (username.brand.eth) through Namespace Universal Usernames. Use when implementing or fixing a universal usernames endpoint or its profile JSON (schema v1), mapping users to ENS text records, coin types, or contenthash, or fixing an issue code reported by the universal-usernames validator.
---

# Universal usernames endpoint

You are building the brand's side of Namespace Universal Usernames: one HTTPS endpoint that answers `GET {endpointUrl}/{username}` with that user's profile. When a wallet resolves `happy.brand.eth`, the Namespace gateway calls the endpoint for `happy`, validates the answer, signs the record the wallet asked for, and ENS verifies the signature onchain. The brand's system stays the authority: what the endpoint returns is what every wallet shows.

You build, test, and validate the endpoint. The owner of `brand.eth` then connects it in the Namespace App with their wallet, which you can't do for them.

## Steps

1. **Find the user store.** Locate where usernames live and the fields to publish: display name, avatar, wallet addresses. Done when you can name the exact query that finds a user by username, and you know whether stored usernames are [ENSIP-15](https://docs.ens.domains/ensip/15) normalized. Namespace only sends normalized labels (`happy`, never `Happy`). If stored usernames aren't normalized, look them up by a normalized column or normalize on write with `@adraffy/ens-normalize`.
2. **Add the route** with the adapter for the repo's framework from [references/adapters.md](references/adapters.md): Hono, Express 5, or a Next.js route handler. For another framework, port the same handler. Done when `GET {base}/{username}` reaches your lookup with the decoded username.
3. **Map the user to a profile** using [references/records.md](references/records.md). Done when every published field has a valid text key, decimal coin type, or contenthash URI, and missing values are left out.
4. **Wire the status codes.** `200` with the profile, `404` when the user is absent, and an error status (`500`) when the store fails. Done when a store failure can't produce a `404`.
5. **Settle authentication.** Ask the user whether the endpoint should be public or require `Authorization: Bearer <token>`. For a token, read it from `ENDPOINT_TOKEN`, compare in constant time, and answer `401` on mismatch.
6. **Test it.** Cover: a `200` body that validates against schema v1 and echoes the username, `404` for a missing user, `500` when the lookup throws, and `401` for a missing or wrong token when one is configured. Done when those tests pass.
7. **Validate the deployed URL:**

   ```bash
   npx @thenamespace/universal-usernames validate https://api.brand.example/ens/users --username happy --token "$ENDPOINT_TOKEN"
   ```

   Use a username that exists. Done when the CLI exits `0`. Fix each reported code with [references/validation-errors.md](references/validation-errors.md). If the package isn't installable, request the URL with `curl -i` and check the body against the schema in [references/endpoint-contract.md](references/endpoint-contract.md).
8. **Hand off to the owner.** Give them the endpoint URL (the base, without a username), the auth mode and token, and a sample username. Tell them to open https://app.namespace.ninja/offchain, go to the **Usernames** tab with the wallet that owns `brand.eth`, then validate, save (one signature), point `brand.eth` at HybridResolverV2 (one or two transactions; the App copies the parent's own records first), and activate (one signature). The full walkthrough is https://docs.namespace.ninja/developer-guide/guide/connect-universal-usernames.

## Rules that decide correctness

- **404 means absent.** Only a `404` tells Namespace the user doesn't exist, and wallets then get an empty result. Timeouts, `401`, `403`, `429`, `5xx`, redirects, and non-JSON responses are errors, never "no such user". Let store failures surface as `500`.
- **Echo the username.** `username` in the body is the decoded path segment you received, exactly. A display name or a differently cased value fails the whole profile.
- **Coin types, not chain IDs.** `addresses` keys are decimal ENS coin types: `"60"` Ethereum, `"0"` Bitcoin, `"501"` Solana, `"2147492101"` Base, `"2147483648"` the default EVM address. `"8453"` is not Base.
- **Leave missing records out.** `texts` and `addresses` are always objects, possibly `{}`. Never send `null`, extra top-level fields, or duplicate keys.
- **Everything resolvable is public.** Anyone who resolves `happy.brand.eth` reads every record the endpoint returns, even when it requires a bearer token. Publish only what belongs on a public profile.
- **60-second freshness.** Namespace caches each `200` and `404` for 60 seconds and signs proofs that expire within that window. Wallets can cache longer. A change in the brand's system shows up within about a minute, so no cache-busting is needed.
- **One label.** Usernames are single labels under the parent. `team.happy.brand.eth` never reaches the endpoint.
- **Reachable over public HTTPS.** `https://` only, no redirects, a public address, `Content-Type: application/json`, and a complete answer within 5 seconds.

## References

- [references/endpoint-contract.md](references/endpoint-contract.md): request format, URL rules, schema v1, status codes, what each failure returns, limits, caching, bearer authentication.
- [references/records.md](references/records.md): text keys, coin type table, address formats, the ENSIP-19 default EVM address, contenthash protocols.
- [references/adapters.md](references/adapters.md): complete Hono, Express 5, and Next.js handlers, plus the `createUsernameHandler` shortcut.
- [references/validation-errors.md](references/validation-errors.md): every validator issue code with its cause and fix.
