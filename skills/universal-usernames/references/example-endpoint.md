# Example endpoint

A template to hook up to what the app already has. It's one Cloudflare Worker file that runs as is on sample users, so it deploys and passes the App's **Test username** before you change anything. Then point `getUser` at the app's user API or database, and leave the rest of the file alone. It writes no database code: reading the app's data is the app's job.

The file has seven parts, in the order a request uses them:

1. **Your user data:** the `User` shape, sample users and `getUser`. This is the only part you change.
2. **Usernames:** `toEnsLabel`, the ENSIP-15 check.
3. **Coin types:** SLIP-44 numbers for non-EVM chains, `evmCoinType(chainId)` for EVM chains.
4. **Records:** each value is checked on its own with zod, and a bad one is dropped, never changed. Also `nftAvatar()` for NFT avatars.
5. **The profile:** builds the v1 JSON.
6. **Auth and CORS:** the bearer token (two during rotation) and the App's test button.
7. **The route:** one path segment, status codes, and error handling that answers 500, never a fake 404.

## Setup

```sh
npm create cloudflare@latest ens-profile-endpoint -- --type=hello-world --lang=ts
cd ens-profile-endpoint
npm install viem zod
```

```jsonc wrangler.jsonc
{
  "name": "ens-profile-endpoint",
  "main": "src/index.ts",
  "compatibility_date": "2026-11-01",
  // node:crypto, for the constant-time token check
  "compatibility_flags": ["nodejs_compat"],
  // The base URL you save in the Namespace App
  "routes": [{ "pattern": "api.brand.example", "custom_domain": true }]
}
```

The token is a secret, never a value in `wrangler.jsonc`. To rotate it, put the old value in `ENS_ADAPTER_TOKEN_PREVIOUS`, put the new one in `ENS_ADAPTER_TOKEN`, save the new one in the App, then delete the previous secret.

```sh
npx wrangler secret put ENS_ADAPTER_TOKEN
npx wrangler deploy
```

## src/index.ts

```ts src/index.ts
/**
 * Universal Usernames profile endpoint, as a Cloudflare Worker.
 *
 * Namespace calls GET {base URL}/{username} and turns the JSON this returns
 * into ENS records, so `happy` in your app resolves as `happy.brand.eth`.
 *
 * Runs as is on sample data. To go live, change only getUser (section 1).
 */
import { createHash, timingSafeEqual } from "node:crypto";
import { getAddress, isAddress } from "viem";
import { normalize } from "viem/ens";
import { z } from "zod";

interface Env {
  ENS_ADAPTER_TOKEN?: string;
  /** Set only while rotating the token, then delete it. */
  ENS_ADAPTER_TOKEN_PREVIOUS?: string;
}

// The path the Worker is mounted under. "" when it has its own subdomain
// (https://api.brand.example/happy), "/ens" for https://brand.example/ens/happy.
const BASE_PATH = "";

// ---------------------------------------------------------------------------
// 1. Your user data
// ---------------------------------------------------------------------------

/** What this endpoint needs to know about a user. Map your own fields to it. */
type User = {
  isPublic: boolean; // false for private, suspended, banned or deleted accounts
  displayName?: string;
  bio?: string;
  avatar?: string; // an image URL, or nftAvatar(...) for an NFT
  website?: string;
  twitter?: string; // only handles connected through OAuth
  github?: string;
  wallets?: { // not every user has one
    evm?: string; // used on every EVM chain
    base?: string; // only if the user set a different address for Base
    bitcoin?: string;
    solana?: string;
  };
  contenthash?: string; // e.g. "ipfs://bafy..."
};

// Sample users, so the Worker runs as is. Delete them once getUser reads your data.
const SAMPLE_USERS = new Map<string, User>([
  [
    "happy", // everything
    {
      isPublic: true,
      displayName: "Happy",
      bio: "Building on brand",
      avatar: "https://cdn.brand.example/avatars/happy.png",
      twitter: "happy",
      wallets: {
        evm: "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045",
        bitcoin: "bc1qar0srrr7xfkvy5l643lydnw9re59gtzzwf5mdq",
        solana: "9WzDXwBbmkg8ZTbNMqUxvQRAyrZzDsGYdLVL9zYtAWWM",
      },
    },
  ],
  ["writer", { isPublic: true, displayName: "Writer", github: "writer" }], // texts only
  [
    "site", // a website only
    { isPublic: true, contenthash: "ipfs://bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi" },
  ],
  ["hidden", { isPublic: false, displayName: "Hidden" }], // private: answers 404
]);

/**
 * Finds a user by their normalized username. Returns null when there's no such
 * user. Throw when the lookup itself fails, so the endpoint answers 500.
 *
 * Read from what the app already has. Don't add a database for this.
 * Keep it to one call, and well under the gateway's 5 s limit.
 */
async function getUser(label: string): Promise<User | null> {
  // Option A, the common case: your internal user API (most databases sit
  // behind one).
  //
  // const res = await fetch(
  //   `https://internal.brand.example/users/by-username/${encodeURIComponent(label)}`,
  //   { signal: AbortSignal.timeout(2000) },
  // );
  // if (res.status === 404) return null;
  // if (!res.ok) throw new Error(`User API answered ${res.status}`);
  // const row = await res.json();
  // return { isPublic: !row.private, displayName: row.display_name, ... };

  // Option B: if the Worker can reach your database, read it directly. One
  // query, matching the normalized username exactly, mapped to User like above.

  return SAMPLE_USERS.get(label) ?? null;
}

// ---------------------------------------------------------------------------
// 2. Usernames
// ---------------------------------------------------------------------------

/** The ENS label for a username, or null when it can't be one. */
export function toEnsLabel(username: string): string | null {
  try {
    const label = normalize(username);
    return label && !label.includes(".") ? label : null;
  } catch {
    return null; // e.g. "happy_doe", "@happy", "happy doe"
  }
}

// ---------------------------------------------------------------------------
// 3. Coin types: which chain each address is for
// ---------------------------------------------------------------------------

// Non-EVM chains use their SLIP-44 number. Any other chain's number:
// https://github.com/satoshilabs/slips/blob/master/slip-0044.md
// Don't use 60 (Ethereum) for EVM wallets: DEFAULT_EVM below covers Ethereum too.
const SLIP44 = {
  bitcoin: 0,
  dogecoin: 3,
  solana: 501,
} as const;

// EVM chains use 0x80000000 | chainId (ENSIP-11). Same as viem's toCoinType.
function evmCoinType(chainId: number): number {
  return (0x80000000 | chainId) >>> 0;
}

// chainId 0 is the default EVM address (ENSIP-19): 2147483648. It answers for
// Ethereum and every EVM chain that has no address of its own.
const DEFAULT_EVM = evmCoinType(0);

const CHAIN_IDS = { base: 8453, optimism: 10, arbitrum: 42161 } as const;

// ---------------------------------------------------------------------------
// 4. Records: each one is checked on its own
// ---------------------------------------------------------------------------
// A bad value drops only its own record. Throwing would fail the whole lookup.

const Text = z.string().trim().min(1).max(4096);

// "@happy" → "happy". A pasted URL or anything that isn't a handle is dropped.
const Handle = z
  .string()
  .trim()
  .transform((value) => value.replace(/^@/, ""))
  .pipe(z.string().regex(/^[A-Za-z0-9_.-]{1,64}$/));

// https only: a javascript: or http: link in a public record is a phishing risk.
const HttpsUrl = Text.refine((value) => URL.canParse(value) && value.startsWith("https://"));

// An image URL, or an NFT in ENSIP-12 form (see nftAvatar).
const Avatar = Text.refine(
  (value) =>
    (URL.canParse(value) && /^(https|ipfs):\/\//.test(value)) ||
    /^eip155:\d+\/(erc721|erc1155):0x[0-9a-fA-F]{40}\/\d+$/.test(value),
);

const Contenthash = Text.regex(/^(ipfs|ipns|ar|bzz|onion3|sia):\/\/.+/);

// Strict isAddress rejects a mixed-case address with a bad checksum: that's
// usually a typo, and re-checksumming it would send money to the typo.
// All-lowercase addresses have no checksum, so they pass and get checksummed.
const EvmAddress = z
  .string()
  .refine((value) => isAddress(value))
  .transform((value) => getAddress(value));

// Format checks for other chains. The gateway does the full decode, and a
// value it rejects fails only that record.
const BitcoinAddress = z
  .string()
  .regex(/^(bc1[02-9ac-hj-np-z]{11,71}|[13][1-9A-HJ-NP-Za-km-z]{25,34})$/); // mainnet only
const SolanaAddress = z.string().regex(/^[1-9A-HJ-NP-Za-km-z]{32,44}$/);

/** The checked value, or undefined when it's missing or invalid. */
function valid<T>(schema: z.ZodType<T>, value: unknown): T | undefined {
  if (value === undefined || value === null) return undefined;
  const result = schema.safeParse(value);
  return result.success ? result.data : undefined;
}

/** Keeps only the entries that have a value. */
function compact(entries: Record<string, string | undefined>): Record<string, string> {
  return Object.fromEntries(
    Object.entries(entries).filter((entry): entry is [string, string] => entry[1] !== undefined),
  );
}

/**
 * An NFT avatar in the form ENS expects (ENSIP-12), built from what the app
 * stores about the NFT. Apps only show it if the name's EVM address owns it.
 *
 * nftAvatar(1, "erc721", "0xb47e…bbb", "0") → "eip155:1/erc721:0xb47e…bbb/0"
 */
export function nftAvatar(
  chainId: number,
  standard: "erc721" | "erc1155",
  contract: string,
  tokenId: string,
): string {
  return `eip155:${chainId}/${standard}:${contract.toLowerCase()}/${tokenId}`;
}

// ---------------------------------------------------------------------------
// 5. The profile
// ---------------------------------------------------------------------------

/** Profile v1. No other top-level fields: the gateway rejects them. */
type Profile = {
  version: 1;
  username: string;
  texts: Record<string, string>;
  addresses: Record<string, string>;
  contenthash?: string;
};

function buildProfile(label: string, user: User): Profile {
  const displayName = valid(Text, user.displayName);
  const wallets = user.wallets ?? {};

  const texts = compact({
    name: displayName, // set both: some apps read `name`, newer ones `alias`
    alias: displayName,
    description: valid(Text, user.bio),
    avatar: valid(Avatar, user.avatar),
    url: valid(HttpsUrl, user.website),
    "com.twitter": valid(Handle, user.twitter),
    "com.github": valid(Handle, user.github),
    // Your own data: your domain reversed, e.g. "com.brand.tier": valid(Text, user.tier)
  });

  // Keys are coin types as decimal strings. Values are plain address strings.
  const addresses = compact({
    [DEFAULT_EVM]: valid(EvmAddress, wallets.evm),
    [evmCoinType(CHAIN_IDS.base)]: valid(EvmAddress, wallets.base),
    [SLIP44.bitcoin]: valid(BitcoinAddress, wallets.bitcoin),
    [SLIP44.solana]: valid(SolanaAddress, wallets.solana),
    // Another EVM chain: [evmCoinType(CHAIN_IDS.optimism)]: valid(EvmAddress, wallets.optimism)
    // Another non-EVM chain: [SLIP44.dogecoin]: valid(DogecoinAddress, wallets.dogecoin)
  });

  const profile: Profile = { version: 1, username: label, texts, addresses };
  const contenthash = valid(Contenthash, user.contenthash);
  if (contenthash) profile.contenthash = contenthash;
  return profile;
}

// ---------------------------------------------------------------------------
// 6. Auth and CORS
// ---------------------------------------------------------------------------

/** Constant-time compare against each accepted token. */
function hasValidToken(request: Request, tokens: string[]): boolean {
  const digest = (value: string) => createHash("sha256").update(value).digest();
  const header = digest(request.headers.get("authorization") ?? "");
  let matched = false;
  for (const token of tokens) {
    if (timingSafeEqual(header, digest(`Bearer ${token}`))) matched = true;
  }
  return matched;
}

// The App's "Test username" button calls from this origin. Real lookups come
// from the Namespace gateway, server to server, and don't need CORS.
const NAMESPACE_APP = "https://app.namespace.ninja";

function corsHeaders(request: Request): Record<string, string> {
  if (request.headers.get("origin") !== NAMESPACE_APP) return {};
  return {
    "Access-Control-Allow-Origin": NAMESPACE_APP,
    "Access-Control-Allow-Headers": "Authorization",
    "Access-Control-Allow-Methods": "GET",
    Vary: "Origin",
  };
}

// ---------------------------------------------------------------------------
// 7. The route: GET {BASE_PATH}/{username}
// ---------------------------------------------------------------------------

/** The username from the path, or null unless it's exactly one segment. */
function usernameFromPath(url: string): string | null {
  const path = new URL(url).pathname;
  if (!path.startsWith(`${BASE_PATH}/`)) return null;
  const segment = path.slice(BASE_PATH.length + 1);
  if (!segment || segment.includes("/")) return null;
  try {
    return decodeURIComponent(segment);
  } catch {
    return null;
  }
}

export default {
  async fetch(request, env): Promise<Response> {
    const cors = corsHeaders(request);
    const reply = (status: number, headers: Record<string, string> = {}) =>
      new Response(null, { status, headers: { ...cors, ...headers } });

    if (request.method === "OPTIONS") return reply(204);
    if (request.method !== "GET") return reply(405, { Allow: "GET" });

    // A token is strongly recommended. Without one, anyone can call this
    // endpoint and walk your user list. Remove this check only on purpose.
    const tokens = [env.ENS_ADAPTER_TOKEN, env.ENS_ADAPTER_TOKEN_PREVIOUS].filter(
      (token): token is string => Boolean(token),
    );
    if (tokens.length === 0) {
      console.error("ENS_ADAPTER_TOKEN is not set");
      return reply(500);
    }
    if (!hasValidToken(request, tokens)) return reply(401);

    // Namespace only sends normalized usernames. Anything else can't be a user.
    const label = usernameFromPath(request.url);
    if (!label || toEnsLabel(label) !== label) return reply(404);

    try {
      const user = await getUser(label);
      // Private, suspended or deleted users look exactly like unknown ones.
      if (!user || !user.isPublic) return reply(404);
      return Response.json(buildProfile(label, user), { headers: cors });
    } catch (error) {
      // Never answer 404 here: that tells wallets the name doesn't exist.
      console.error("Profile lookup failed", { username: label, error });
      return reply(500);
    }
  },
} satisfies ExportedHandler<Env>;
```
