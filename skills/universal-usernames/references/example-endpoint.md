# Example endpoint

A framework-neutral handler built on the Fetch API `Request` and `Response`, so it runs on Next.js route handlers, Hono, Bun, Deno and Cloudflare Workers (with `nodejs_compat` for `node:crypto`). Swap `findUserByEnsLabel` and the `UserRecord` fields for your own store. Tested with Node 24 and viem 2.

## Handler

```ts profile-endpoint.ts
import { createHash, timingSafeEqual } from "node:crypto";
import { getAddress } from "viem";
import { normalize } from "viem/ens";

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

function buildTexts(user: UserRecord): Record<string, string> {
  const texts: Record<string, string> = {};
  // Omit empty fields: an empty string still resolves and renders as blank.
  const set = (key: string, value: string | null) => {
    const trimmed = value?.trim();
    if (trimmed) texts[key] = trimmed;
  };
  set("name", user.displayName);
  set("alias", user.displayName);
  set("description", user.bio);
  set("avatar", user.avatarUrl);
  set("url", user.website);
  set("com.twitter", user.twitterHandle?.replace(/^@/, "") ?? null);
  set("com.github", user.githubHandle?.replace(/^@/, "") ?? null);
  set("com.brand.tier", user.tier);
  return texts;
}

function buildAddresses(user: UserRecord): Record<string, string> {
  const addresses: Record<string, string> = {};
  if (!user.verifiedEvmAddress) return addresses;
  try {
    addresses["2147483648"] = getAddress(user.verifiedEvmAddress);
  } catch {
    // A malformed stored address drops only this record, not the profile.
    console.error("Invalid stored EVM address", { label: user.ensLabel });
  }
  return addresses;
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

  const profile: Record<string, unknown> = {
    version: 1,
    username,
    texts: buildTexts(user),
    addresses: buildAddresses(user),
  };
  if (user.siteContenthash) profile.contenthash = user.siteContenthash;

  return Response.json(profile);
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

## Tests

Every endpoint ships with at least these cases. Run with `node --test endpoint.test.ts`.

```ts endpoint.test.ts
import assert from "node:assert/strict";
import { test } from "node:test";
import {
  handleProfileRequest,
  toEnsLabel,
  type UserRecord,
} from "./profile-endpoint.ts";

const TOKEN = "test-token";

const happy: UserRecord = {
  ensLabel: "happy",
  isPublic: true,
  displayName: "Happy",
  bio: "",
  avatarUrl: "https://cdn.brand.example/a.png",
  website: null,
  twitterHandle: "@happy",
  githubHandle: null,
  tier: "gold",
  verifiedEvmAddress: "0xd8da6bf26964af9d7eed9e03e53415d37aa96045",
  siteContenthash: null,
};

const users = new Map<string, UserRecord>([
  ["happy", happy],
  ["hidden", { ...happy, ensLabel: "hidden", isPublic: false }],
  ["héllo", { ...happy, ensLabel: "héllo", verifiedEvmAddress: "0xnotanaddress" }],
]);

const findUserByEnsLabel = async (label: string) => users.get(label) ?? null;

function lookup(path: string, authorization: string | null = `Bearer ${TOKEN}`) {
  const headers = authorization ? { authorization } : undefined;
  const request = new Request(`https://api.brand.example${path}`, { headers });
  return handleProfileRequest(request, findUserByEnsLabel, TOKEN);
}

test("a found user returns the exact v1 shape", async () => {
  const res = await lookup("/happy");
  assert.equal(res.status, 200);
  assert.equal(res.headers.get("content-type")?.split(";")[0], "application/json");
  assert.deepEqual(await res.json(), {
    version: 1,
    username: "happy",
    texts: {
      name: "Happy",
      alias: "Happy",
      avatar: "https://cdn.brand.example/a.png",
      "com.twitter": "happy",
      "com.brand.tier": "gold",
    },
    addresses: { "2147483648": "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045" },
  });
});

test("unknown and private users are 404", async () => {
  assert.equal((await lookup("/nobody")).status, 404);
  assert.equal((await lookup("/hidden")).status, 404);
});

test("a missing or wrong token is 401", async () => {
  assert.equal((await lookup("/happy", null)).status, 401);
  assert.equal((await lookup("/happy", "Bearer wrong")).status, 401);
});

test("a label that isn't normalized is 404", async () => {
  assert.equal((await lookup("/Happy")).status, 404);
  assert.equal((await lookup("/happy_doe")).status, 404);
});

test("a URL-encoded unicode label resolves, and a bad stored address is dropped", async () => {
  const res = await lookup(`/${encodeURIComponent("héllo")}`);
  assert.equal(res.status, 200);
  const profile = await res.json();
  assert.equal(profile.username, "héllo");
  assert.deepEqual(profile.addresses, {});
});

test("toEnsLabel maps usernames to labels or null", () => {
  assert.equal(toEnsLabel("Happy"), "happy");
  assert.equal(toEnsLabel("_happy"), "_happy");
  assert.equal(toEnsLabel("happy_doe"), null);
  assert.equal(toEnsLabel("happy.doe"), null);
  assert.equal(toEnsLabel("@happy"), null);
});
```
