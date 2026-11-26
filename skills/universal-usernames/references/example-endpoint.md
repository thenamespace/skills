# Example endpoint

One Cloudflare Worker file. It needs no database: `getUser` returns sample data so it runs as is. Point `getUser` at the API or database the app already has, and leave the rest of the file alone.

The file has seven parts, in the order a request uses them:

1. **Your user data:** the `User` shape and `getUser`. This is the only part you change.
2. **Usernames:** `toEnsLabel`, the ENSIP-15 check.
3. **Coin types:** SLIP-44 numbers for non-EVM chains, `evmCoinType(chainId)` for EVM chains.
4. **Validating records:** each record is checked on its own, and a bad one is dropped.
5. **The profile:** builds the v1 JSON.
6. **Auth and CORS:** the bearer token (two during rotation) and the App's test button.
7. **The route:** status codes, and error handling that answers 500, never a fake 404.

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
 */
import { createHash, timingSafeEqual } from "node:crypto";
import { getAddress, isAddress } from "viem";
import { normalize } from "viem/ens";
import { z } from "zod";

interface Env {
  ENS_ADAPTER_TOKEN: string;
  /** Set only while rotating the token, then delete it. */
  ENS_ADAPTER_TOKEN_PREVIOUS?: string;
}

// ---------------------------------------------------------------------------
// 1. Your user data
// ---------------------------------------------------------------------------

/** What this endpoint needs to know about a user. Map your own fields to it. */
type User = {
  isPublic: boolean;
  displayName?: string;
  bio?: string;
  avatarUrl?: string;
  website?: string;
  twitter?: string; // only handles connected through OAuth
  github?: string;
  // Only addresses the user proved they control, e.g. by signing a message.
  wallets: {
    evm?: string; // used on every EVM chain
    base?: string; // only if the user wants a different address on Base
    bitcoin?: string;
    solana?: string;
  };
  ipfsSite?: string; // e.g. "ipfs://bafy..."
};

// Sample data so the Worker runs as is. Delete it once getUser reads your data.
const SAMPLE_USERS: Record<string, User> = {
  happy: {
    isPublic: true,
    displayName: "Happy",
    bio: "Building on brand",
    avatarUrl: "https://cdn.brand.example/avatars/happy.png",
    twitter: "happy",
    wallets: {
      evm: "0xd8da6bf26964af9d7eed9e03e53415d37aa96045",
      bitcoin: "bc1qar0srrr7xfkvy5l643lydnw9re59gtzzwf5mdq",
      solana: "9WzDXwBbmkg8ZTbNMqUxvQRAyrZzDsGYdLVL9zYtAWWM",
    },
  },
};

/**
 * Finds a user by their ENS label (the normalized username). Returns null when
 * there's no such user. Throw on a real failure, so the lookup returns 500.
 *
 * Read from wherever your users already live. Don't add a database for this.
 */
async function getUser(label: string): Promise<User | null> {
  // Option A: your own API.
  // const res = await fetch(
  //   `https://api.brand.example/internal/users/${encodeURIComponent(label)}`,
  //   { signal: AbortSignal.timeout(2000) }, // stay well inside the 5 s limit
  // );
  // if (res.status === 404) return null;
  // if (!res.ok) throw new Error(`User API answered ${res.status}`);
  // return toUser(await res.json());

  // Option B: your database, one indexed read on the normalized username.
  // const row = await db.query(
  //   "SELECT ... FROM users WHERE ens_label = $1", [label],
  // );
  // return row ? toUser(row) : null;

  return SAMPLE_USERS[label] ?? null;
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

// Non-EVM chains use their SLIP-44 number:
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
// 4. Validating records
// ---------------------------------------------------------------------------
// A bad value drops only its own record. Throwing would fail the whole lookup.

const Text = z.string().trim().min(1).max(4096);
const Handle = Text.transform((handle) => handle.replace(/^@/, ""));
// https only: a javascript: or http: link in a public record is a phishing risk.
const HttpsUrl = Text.refine((v) => URL.canParse(v) && v.startsWith("https://"));
const ImageUrl = Text.refine((v) => URL.canParse(v) && /^(https|ipfs):\/\//.test(v));
const Contenthash = Text.regex(/^(ipfs|ipns|ar|bzz|onion3|sia):\/\/.+/);

// EVM addresses go out checksummed.
const EvmAddress = z
  .string()
  .refine((v) => isAddress(v, { strict: false }))
  .transform((v) => getAddress(v));
// Format checks for other chains. The gateway does the full decode, and a
// value it rejects fails only that record.
const BitcoinAddress = z.string().regex(/^(bc1[02-9ac-hj-np-z]{11,71}|[13][1-9A-HJ-NP-Za-km-z]{25,34})$/);
const SolanaAddress = z.string().regex(/^[1-9A-HJ-NP-Za-km-z]{32,44}$/);

/** The parsed value, or undefined when it's missing or invalid. */
function valid<T>(schema: z.ZodType<T>, value: unknown): T | undefined {
  if (value === undefined || value === null) return undefined;
  const parsed = schema.safeParse(value);
  return parsed.success ? parsed.data : undefined;
}

/** Keeps only the entries that have a value. */
function compact(entries: Record<string, string | undefined>): Record<string, string> {
  return Object.fromEntries(
    Object.entries(entries).filter((entry): entry is [string, string] => entry[1] !== undefined),
  );
}

// ---------------------------------------------------------------------------
// 5. The profile
// ---------------------------------------------------------------------------

// Profile v1. strictObject rejects extra top-level fields, as the gateway does.
const Profile = z.strictObject({
  version: z.literal(1),
  username: z.string().min(1),
  texts: z.record(z.string().min(1).max(256), z.string().max(4096)),
  addresses: z.record(z.string().regex(/^(0|[1-9][0-9]*)$/), z.string().min(1).max(1024)),
  contenthash: z.string().optional(),
});

function buildProfile(label: string, user: User) {
  const displayName = valid(Text, user.displayName);

  const texts = compact({
    name: displayName, // set both: many apps still read only `name`
    alias: displayName,
    description: valid(Text, user.bio),
    avatar: valid(ImageUrl, user.avatarUrl),
    url: valid(HttpsUrl, user.website),
    "com.twitter": valid(Handle, user.twitter),
    "com.github": valid(Handle, user.github),
    // Your own data: reverse your domain, e.g. "com.brand.tier": user.tier
  });

  // Keys are coin types as decimal strings. Values are plain address strings.
  const addresses = compact({
    [DEFAULT_EVM]: valid(EvmAddress, user.wallets.evm),
    [evmCoinType(CHAIN_IDS.base)]: valid(EvmAddress, user.wallets.base),
    [SLIP44.bitcoin]: valid(BitcoinAddress, user.wallets.bitcoin),
    [SLIP44.solana]: valid(SolanaAddress, user.wallets.solana),
    // More EVM chains: [evmCoinType(CHAIN_IDS.optimism)]: valid(EvmAddress, user.wallets.optimism)
    // More non-EVM chains: [SLIP44.dogecoin]: valid(DogecoinAddress, user.wallets.dogecoin)
  });

  return Profile.parse({
    version: 1,
    username: label, // exactly the requested label, never the display name
    texts,
    addresses,
    contenthash: valid(Contenthash, user.ipfsSite),
  });
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
// 7. The route: GET /{username}
// ---------------------------------------------------------------------------

export default {
  async fetch(request, env): Promise<Response> {
    const headers = corsHeaders(request);
    const reply = (status: number) => new Response(null, { status, headers });

    if (request.method === "OPTIONS") return reply(204);
    if (request.method !== "GET") return reply(405);

    if (!env.ENS_ADAPTER_TOKEN) {
      console.error("ENS_ADAPTER_TOKEN is not set");
      return reply(500);
    }
    const tokens = [env.ENS_ADAPTER_TOKEN, env.ENS_ADAPTER_TOKEN_PREVIOUS].filter(
      (token): token is string => Boolean(token),
    );
    if (!hasValidToken(request, tokens)) return reply(401);

    // The username is the last path segment, URL-encoded.
    let label: string;
    try {
      label = decodeURIComponent(new URL(request.url).pathname.split("/").pop() ?? "");
    } catch {
      return reply(404);
    }
    // Namespace only sends normalized labels. Anything else can't be a user.
    if (!label || toEnsLabel(label) !== label) return reply(404);

    try {
      const user = await getUser(label);
      // Private, suspended or deleted users look exactly like unknown ones.
      if (!user || !user.isPublic) return reply(404);
      return Response.json(buildProfile(label, user), { headers });
    } catch (error) {
      // Never answer 404 here: that tells wallets the name doesn't exist.
      console.error("Profile lookup failed", { label, error });
      return reply(500);
    }
  },
} satisfies ExportedHandler<Env>;
```
