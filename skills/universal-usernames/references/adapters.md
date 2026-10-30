# Adapters

Complete handlers for Hono, Express 5, and the Next.js App Router. Each one: answers `GET {base}/{username}`, decodes the username once, checks an optional bearer token in constant time before revealing anything, returns `200` with the profile, `404` for an absent user or unknown path, and `500` with a JSON body when the lookup throws. All three are tested against the same suite (schema v1 body, percent-decoded username, `404`, `500`, `401`).

A complete runnable project with tests: https://github.com/thenamespace/docs/tree/main/examples/universal-usernames-adapter

## Shared pieces

Put these next to the handler (in Next.js, under `lib/`).

```ts profile.ts
// Universal Usernames profile, schema version 1.
export interface UsernameProfileV1 {
  version: 1;
  username: string; // the requested username, echoed exactly
  texts: Record<string, string>; // leave missing records out; never null
  addresses: Record<string, string>; // keyed by decimal ENS coin type
  contenthash?: string; // ipfs://, ipns://, bzz:// or ar://
}
```

```ts bearer-token.ts
import { createHash, timingSafeEqual } from 'node:crypto';

const BEARER = /^Bearer\s+(.+)$/i;

// Constant-time check of an Authorization header against the expected token.
// Hashing both sides gives timingSafeEqual equal-length inputs.
export function isAuthorizedBearer(header: string | null | undefined, expectedToken: string): boolean {
  const presented = header?.match(BEARER)?.[1];
  if (!presented) return false;
  const digest = (value: string) => createHash('sha256').update(value).digest();
  return timingSafeEqual(digest(presented), digest(expectedToken));
}
```

`lookupUser` is the only part that touches the brand's data. Map fields with [records.md](records.md), and return `null` only when the user doesn't exist:

```ts users.ts
import type { UsernameProfileV1 } from './profile.js';
import { findUserByUsername } from './db.js'; // your existing data access

// Replace with a query against your user store, keyed by the ENSIP-15
// normalized username. Return null when the user doesn't exist; let store
// failures throw so the endpoint answers 500, never 404.
export async function lookupUser(username: string): Promise<UsernameProfileV1 | null> {
  const user = await findUserByUsername(username);
  if (!user) return null;

  const texts: Record<string, string> = { name: user.displayName };
  if (user.avatarUrl) texts.avatar = user.avatarUrl;

  const addresses: Record<string, string> = {};
  if (user.walletAddress) addresses['60'] = user.walletAddress; // Ethereum

  return { version: 1, username, texts, addresses };
}
```

## Hono

```ts app.ts
import { Hono } from 'hono';
import { isAuthorizedBearer } from './bearer-token.js';
import { lookupUser } from './users.js';

export function createApp(endpointToken = process.env.ENDPOINT_TOKEN || undefined): Hono {
  const app = new Hono();

  if (endpointToken) {
    app.use('*', async (c, next) => {
      if (!isAuthorizedBearer(c.req.header('Authorization'), endpointToken)) {
        return c.json({ error: 'unauthorized' }, 401);
      }
      await next();
    });
  }

  // c.req.param() is already percent-decoded: "caf%C3%A9" arrives as "café".
  app.get('/:username', async (c) => {
    const profile = await lookupUser(c.req.param('username'));
    if (!profile) return c.json({ error: 'not_found' }, 404);
    return c.json(profile);
  });

  app.notFound((c) => c.json({ error: 'not_found' }, 404));
  app.onError((error, c) => {
    console.error('Username lookup failed', error);
    return c.json({ error: 'internal_error' }, 500);
  });

  return app;
}

// Node.js: serve({ fetch: createApp().fetch, port: 8787 }) from '@hono/node-server'.
// Under a path: new Hono().route('/ens/users', createApp()).
```

Node.js entry point (`npm install hono @hono/node-server`):

```ts
import { serve } from '@hono/node-server';
import { createApp } from './app.js';

serve({ fetch: createApp().fetch, port: Number(process.env.PORT ?? 8787) });
```

## Express 5

Use Express 5 (`npm install express@5`). Express 4 doesn't forward rejected promises from async handlers to the error handler, so a failing lookup would hang instead of answering `500`.

```ts app.ts
import express, { type NextFunction, type Request, type Response } from 'express';
import { isAuthorizedBearer } from './bearer-token.js';
import { lookupUser } from './users.js';

// Express 5: rejected promises from async handlers reach the error handler.
export function createApp(endpointToken = process.env.ENDPOINT_TOKEN || undefined) {
  const app = express();
  app.disable('x-powered-by');

  if (endpointToken) {
    app.use((req: Request, res: Response, next: NextFunction) => {
      if (!isAuthorizedBearer(req.get('Authorization'), endpointToken)) {
        res.status(401).json({ error: 'unauthorized' });
        return;
      }
      next();
    });
  }

  // req.params is already percent-decoded: "caf%C3%A9" arrives as "café".
  app.get('/:username', async (req: Request<{ username: string }>, res: Response) => {
    const profile = await lookupUser(req.params.username);
    if (!profile) {
      res.status(404).json({ error: 'not_found' });
      return;
    }
    res.json(profile);
  });

  app.use((_req: Request, res: Response) => {
    res.status(404).json({ error: 'not_found' });
  });

  app.use((error: unknown, _req: Request, res: Response, _next: NextFunction) => {
    console.error('Username lookup failed', error);
    res.status(500).json({ error: 'internal_error' });
  });

  return app;
}

// Node.js: createApp().listen(8787).
// Under a path: express().use('/ens/users', createApp()).
```

## Next.js App Router

The file location sets the path. With this file, the endpoint URL to connect is `https://your-app.example/ens/users`.

```ts app/ens/users/[username]/route.ts
// app/ens/users/[username]/route.ts
// Connect https://your-app.example/ens/users as the endpoint URL.
import { isAuthorizedBearer } from '@/lib/bearer-token';
import { lookupUser } from '@/lib/users';

export const dynamic = 'force-dynamic';

const endpointToken = process.env.ENDPOINT_TOKEN || undefined;

export async function GET(request: Request): Promise<Response> {
  if (endpointToken && !isAuthorizedBearer(request.headers.get('authorization'), endpointToken)) {
    return Response.json({ error: 'unauthorized' }, { status: 401 });
  }

  // Read the last path segment and decode it once, whatever the framework did.
  const segment = new URL(request.url).pathname.split('/').pop() ?? '';
  let username: string;
  try {
    username = decodeURIComponent(segment);
  } catch {
    return Response.json({ error: 'not_found' }, { status: 404 });
  }

  try {
    const profile = await lookupUser(username);
    if (!profile) return Response.json({ error: 'not_found' }, { status: 404 });
    return Response.json(profile);
  } catch (error) {
    console.error('Username lookup failed', error);
    return Response.json({ error: 'internal_error' }, { status: 500 });
  }
}
```

`force-dynamic` keeps Next.js from caching the route; Namespace already caches each answer for 60 seconds.

## With `@thenamespace/universal-usernames`

When the package is installed, `createUsernameHandler` replaces the handler code: it parses the path after `basePath`, calls your lookup, answers `404` for `null`, and compares the bearer token in constant time. `defineUsernameProfile` accepts chain names (`eth`, `btc`, `sol`, `base`, `evm`) as address keys.

```ts
import { createUsernameHandler, defineUsernameProfile } from '@thenamespace/universal-usernames';

export const handleUsername = createUsernameHandler(
  async (username) => {
    const user = await findUserByUsername(username);
    if (!user) return null;
    return defineUsernameProfile({
      username,
      texts: { name: user.displayName },
      addresses: { eth: user.walletAddress },
    });
  },
  { basePath: '/ens/users', bearerToken: process.env.ENDPOINT_TOKEN },
);
// Fetch API handler: (request: Request) => Promise<Response>.
// Next.js: export const GET = handleUsername; Hono: app.get('/ens/users/:username', (c) => handleUsername(c.req.raw)).
```

## Tests to write

For whichever adapter you use:

- `200`, `Content-Type: application/json`, a body that validates against the schema in [endpoint-contract.md](endpoint-contract.md), and `username` equal to the request.
- A percent-encoded username (`/caf%C3%A9`) arrives decoded.
- `404` for a missing user.
- `500` when the lookup throws.
- With a token configured: `401` with no header, a wrong token, or a wrong token on a missing user.
