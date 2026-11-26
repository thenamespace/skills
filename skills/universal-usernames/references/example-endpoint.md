# Example endpoint

A framework-neutral handler built on the Fetch API `Request` and `Response`, so it runs on Next.js route handlers, Hono, Bun, Deno and Cloudflare Workers (with `nodejs_compat` for `node:crypto`). Swap `findUserByEnsLabel` and the `UserRecord` fields for your own store. Uses viem 2 and zod 4:

```sh
npm install viem zod
```

zod checks each record before it goes out, so a bad stored value drops only that record, and the strict `Profile` schema catches a malformed response before the gateway does.

## Handler

```ts profile-endpoint.ts
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
      avatar: valid(TextValue, user.avatarUrl),
      url: valid(TextValue, user.website),
      "com.twitter": valid(Handle, user.twitterHandle),
      "com.github": valid(Handle, user.githubHandle),
      "com.brand.tier": valid(TextValue, user.tier),
    }),
    addresses: compact({ "2147483648": evmAddress }),
    contenthash: valid(Contenthash, user.siteContenthash),
  });
}

/** Constant-time compare, so response timing doesn't leak the token. */
function hasValidToken(request: Request, token: string): boolean {
  const header = request.headers.get("authorization") ?? "";
  const digest = (value: string) => createHash("sha256").update(value).digest();
  return timingSafeEqual(digest(header), digest(`Bearer ${token}`));
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
  token: string,
): Promise<Response> {
  if (request.method !== "GET") return new Response(null, { status: 405 });
  if (!hasValidToken(request, token)) return new Response(null, { status: 401 });

  const username = lastPathSegment(request.url);
  // Namespace only sends normalized labels; anything else can't match a user.
  if (!username || toEnsLabel(username) !== username) {
    return new Response(null, { status: 404 });
  }

  const user = await findUserByEnsLabel(username);
  if (!user || !user.isPublic) return new Response(null, { status: 404 });

  // Profile.parse throws only on a bug in buildProfile. Let it surface as a
  // 5xx: a failed lookup is better than a profile the gateway rejects.
  return Response.json(buildProfile(username, user));
}
```

## Wiring it up

Next.js App Router, deployed so the route sits at the base URL (`app/[username]/route.ts`):

```ts
import { handleProfileRequest } from '@/lib/profile-endpoint';
import { findUserByEnsLabel } from '@/lib/users';

export function GET(request: Request) {
  const token = process.env.ENS_ADAPTER_TOKEN;
  if (!token) return new Response(null, { status: 500 });
  return handleProfileRequest(request, findUserByEnsLabel, token);
}
```

`findUserByEnsLabel` is one indexed query, for example with Prisma:

```ts
export const findUserByEnsLabel = (label: string) =>
  db.user.findUnique({ where: { ensLabel: label } });
```

If usernames live onchain, keep the handler and make `findUserByEnsLabel` read the registry contract or an indexer.
