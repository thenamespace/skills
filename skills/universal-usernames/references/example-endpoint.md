# Example endpoint

A framework-neutral handler built on the Fetch API `Request` and `Response`, so it runs on Cloudflare Workers, Hono, Bun, Deno and Node. The wiring below is a Cloudflare Worker with a D1 database. Swap `findUserByEnsLabel` and the `UserRecord` fields for your own store. Uses viem 2 and zod 4:

```sh
npm install viem zod
```

zod checks each record before it goes out, so a bad stored value drops only that record, and the strict `Profile` schema catches a malformed response before the gateway does.

## Handler

```ts src/profile-endpoint.ts
import { createHash, timingSafeEqual } from "node:crypto";
import { getAddress, isAddress } from "viem";
import { normalize } from "viem/ens";
import { z } from "zod";

// Replace with your own user store. `ensLabel` is the normalized copy of the
// username, computed once with toEnsLabel when the username is set.
export type UserRecord = {
  ensLabel: string;
  isPublic: boolean;
  displayName: string | null;
  bio: string | null;
  avatarUrl: string | null;
  website: string | null;
  twitterHandle: string | null;
  githubHandle: string | null;
  tier: string | null;
  verifiedEvmAddress: string | null;
  siteContenthash: string | null;
};

export type FindUserByEnsLabel = (label: string) => Promise<UserRecord | null>;

/** The ENS label for a username, or null when it can't be one. */
export function toEnsLabel(username: string): string | null {
  try {
    const label = normalize(username);
    return label && !label.includes(".") ? label : null;
  } catch {
    return null;
  }
}

// One record. Empty strings fail, so blank fields are left out instead of
// resolving as "".
const TextValue = z.string().trim().min(1).max(4096);
const Handle = TextValue.transform((handle) => handle.replace(/^@/, ""));
// https only: a javascript: or http: URL in a public record is a phishing vector.
const HttpsUrl = TextValue.refine((value) => URL.canParse(value) && value.startsWith("https://"));
const ImageUrl = TextValue.refine(
  (value) => URL.canParse(value) && /^(https|ipfs):\/\//.test(value),
);
const EvmAddress = z
  .string()
  .refine((value) => isAddress(value, { strict: false }))
  .transform((value) => getAddress(value));
const Contenthash = z
  .string()
  .max(4096)
  .regex(/^(ipfs|ipns|ar|bzz|onion3|sia):\/\/.+/);

// Profile v1. strictObject rejects unknown top-level fields, which the
// gateway would reject too.
const Profile = z.strictObject({
  version: z.literal(1),
  username: z.string().min(1),
  texts: z
    .record(z.string().min(1).max(256), z.string().max(4096))
    .refine((texts) => Object.keys(texts).length <= 128),
  addresses: z
    .record(z.string().regex(/^(0|[1-9][0-9]*)$/), z.string().min(1).max(1024))
    .refine((addresses) => Object.keys(addresses).length <= 128),
  contenthash: z.string().optional(),
});
type Profile = z.infer<typeof Profile>;

/** The parsed value, or undefined when it's missing or invalid. */
function valid<T>(schema: z.ZodType<T>, value: unknown): T | undefined {
  const parsed = schema.safeParse(value);
  return parsed.success ? parsed.data : undefined;
}

/** Drops undefined entries, so a bad field leaves out only its own record. */
function compact(entries: Record<string, string | undefined>) {
  return Object.fromEntries(
    Object.entries(entries).filter(
      (entry): entry is [string, string] => entry[1] !== undefined,
    ),
  );
}

function buildProfile(username: string, user: UserRecord): Profile {
  const displayName = valid(TextValue, user.displayName);
  const evmAddress = valid(EvmAddress, user.verifiedEvmAddress);
  if (user.verifiedEvmAddress && !evmAddress) {
    console.error("Invalid stored EVM address", { label: username });
  }

  return Profile.parse({
    version: 1,
    username,
    texts: compact({
      name: displayName,
      alias: displayName,
      description: valid(TextValue, user.bio),
      avatar: valid(ImageUrl, user.avatarUrl),
      url: valid(HttpsUrl, user.website),
      "com.twitter": valid(Handle, user.twitterHandle),
      "com.github": valid(Handle, user.githubHandle),
      "com.brand.tier": valid(TextValue, user.tier),
    }),
    addresses: compact({ "2147483648": evmAddress }),
    contenthash: valid(Contenthash, user.siteContenthash),
  });
}

/**
 * Constant-time compare, so response timing doesn't leak a token. Takes a list
 * so the old and new token both work while you rotate.
 */
function hasValidToken(request: Request, tokens: string[]): boolean {
  const digest = (value: string) => createHash("sha256").update(value).digest();
  const header = digest(request.headers.get("authorization") ?? "");
  let matched = false;
  for (const token of tokens) {
    // No early exit, so timing doesn't reveal which token matched.
    if (timingSafeEqual(header, digest(`Bearer ${token}`))) matched = true;
  }
  return matched;
}

// The App's "Test username" button calls the endpoint from this origin. Real
// lookups come from the gateway, server to server, and don't need CORS.
const NAMESPACE_APP_ORIGIN = "https://app.namespace.ninja";

function corsHeaders(request: Request): Record<string, string> {
  if (request.headers.get("origin") !== NAMESPACE_APP_ORIGIN) return {};
  return {
    "Access-Control-Allow-Origin": NAMESPACE_APP_ORIGIN,
    "Access-Control-Allow-Headers": "Authorization",
    "Access-Control-Allow-Methods": "GET",
    Vary: "Origin",
  };
}

function lastPathSegment(url: string): string | null {
  const segment = new URL(url).pathname.split("/").pop();
  if (!segment) return null;
  try {
    return decodeURIComponent(segment);
  } catch {
    return null;
  }
}

export async function handleProfileRequest(
  request: Request,
  findUserByEnsLabel: FindUserByEnsLabel,
  tokens: string[],
): Promise<Response> {
  const headers = corsHeaders(request);
  const empty = (status: number) => new Response(null, { status, headers });

  if (request.method === "OPTIONS") return empty(204);
  if (request.method !== "GET") return empty(405);
  if (!hasValidToken(request, tokens)) return empty(401);

  const username = lastPathSegment(request.url);
  // Namespace only sends normalized labels; anything else can't match a user.
  if (!username || toEnsLabel(username) !== username) return empty(404);

  try {
    const user = await findUserByEnsLabel(username);
    if (!user || !user.isPublic) return empty(404);
    return Response.json(buildProfile(username, user), { headers });
  } catch (error) {
    // A failed lookup, never a fake 404: a 404 tells wallets the name doesn't exist.
    console.error("Profile lookup failed", { username, error });
    return empty(500);
  }
}
```

## Cloudflare Worker

Deploy the Worker on its own route, such as `api.brand.example`, and save that as the base URL. `nodejs_compat` is required for `node:crypto`.

```jsonc wrangler.jsonc
{
  "name": "ens-profile-endpoint",
  "main": "src/worker.ts",
  "compatibility_date": "2026-11-01",
  "compatibility_flags": ["nodejs_compat"],
  "routes": [{ "pattern": "api.brand.example", "custom_domain": true }],
  "d1_databases": [
    { "binding": "DB", "database_name": "users", "database_id": "<your D1 database id>" }
  ]
}
```

The table needs a unique index on `ens_label`, so each lookup is one indexed read:

```sql
CREATE UNIQUE INDEX users_ens_label ON users (ens_label);
```

Store the token as a secret, never in `wrangler.jsonc`. To rotate, put the old value in `ENS_ADAPTER_TOKEN_PREVIOUS`, the new one in `ENS_ADAPTER_TOKEN`, save the new one in the App, then delete the previous secret.

```sh
npx wrangler secret put ENS_ADAPTER_TOKEN
npx wrangler deploy
```

```ts src/worker.ts
import {
  type FindUserByEnsLabel,
  handleProfileRequest,
  type UserRecord,
} from "./profile-endpoint";

interface Env {
  DB: D1Database;
  ENS_ADAPTER_TOKEN: string;
  /** Set only while rotating, then delete it. */
  ENS_ADAPTER_TOKEN_PREVIOUS?: string;
}

type UserRow = {
  ens_label: string;
  is_public: number;
  display_name: string | null;
  bio: string | null;
  avatar_url: string | null;
  website: string | null;
  twitter_handle: string | null;
  github_handle: string | null;
  tier: string | null;
  verified_evm_address: string | null;
  site_contenthash: string | null;
};

// Name the columns: SELECT * reads every column, including ones added later.
const USER_QUERY = `
  SELECT ens_label, is_public, display_name, bio, avatar_url, website,
         twitter_handle, github_handle, tier, verified_evm_address, site_contenthash
  FROM users WHERE ens_label = ?`;

function toUserRecord(row: UserRow): UserRecord {
  return {
    ensLabel: row.ens_label,
    isPublic: row.is_public === 1,
    displayName: row.display_name,
    bio: row.bio,
    avatarUrl: row.avatar_url,
    website: row.website,
    twitterHandle: row.twitter_handle,
    githubHandle: row.github_handle,
    tier: row.tier,
    verifiedEvmAddress: row.verified_evm_address,
    siteContenthash: row.site_contenthash,
  };
}

export default {
  async fetch(request, env): Promise<Response> {
    if (!env.ENS_ADAPTER_TOKEN) return new Response(null, { status: 500 });

    // One indexed read by the normalized label.
    const findUserByEnsLabel: FindUserByEnsLabel = async (label) => {
      const row = await env.DB.prepare(USER_QUERY)
        .bind(label)
        .first<UserRow>();
      return row ? toUserRecord(row) : null;
    };

    const tokens = [env.ENS_ADAPTER_TOKEN, env.ENS_ADAPTER_TOKEN_PREVIOUS].filter(
      (token): token is string => Boolean(token),
    );
    return handleProfileRequest(request, findUserByEnsLabel, tokens);
  },
} satisfies ExportedHandler<Env>;
```

If usernames live somewhere other than D1, keep the Worker and replace `findUserByEnsLabel` with a read from Hyperdrive (Postgres), KV, or an indexer for onchain registries. Keep it to one read per lookup.
