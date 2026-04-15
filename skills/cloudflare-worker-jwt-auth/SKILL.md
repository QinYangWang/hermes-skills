---
name: cloudflare-worker-jwt-auth
title: Cloudflare Worker JWT Auth with D1
description: Implement user registration, login, and JWT-protected API routes inside a Cloudflare Worker using Web Crypto and D1 SQLite.
triggers:
  - Adding user authentication to a Cloudflare Worker
  - JWT signing inside a Worker without external libraries
  - Password hashing and user sessions with D1
---

# Cloudflare Worker JWT Auth with D1

## When to use

Use this when you need simple username/password auth for a Worker-based app and don't want to bring in external services like Auth0 or Clerk.

## Schema

```sql
CREATE TABLE IF NOT EXISTS users (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  username TEXT UNIQUE NOT NULL,
  password_hash TEXT NOT NULL,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

Initialize with:
```bash
npx wrangler d1 execute <db-name> --remote --file=worker/schema.sql
```

## Worker Code

```typescript
export interface Env {
  DB: D1Database;
  JWT_SECRET: string;
}

const corsHeaders = {
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Methods': 'POST, OPTIONS',
  'Access-Control-Allow-Headers': 'Content-Type, Authorization',
};

function base64UrlEncode(str: string) {
  return btoa(str).replace(/\+/g, '-').replace(/\//g, '_').replace(/=/g, '');
}

function base64UrlDecode(str: string) {
  // CRITICAL FIX: old code `new Array(5 - (str.length % 4)).join('=')`
  // incorrectly adds 4 '=' when length is divisible by 4, breaking atob().
  const padding = (4 - (str.length % 4)) % 4;
  str += '='.repeat(padding);
  return atob(str.replace(/\-/g, '+').replace(/\_/g, '/'));
}

async function signJWT(payload: object, secret: string) {
  const header = base64UrlEncode(JSON.stringify({ alg: 'HS256', typ: 'JWT' }));
  const payloadB64 = base64UrlEncode(JSON.stringify(payload));
  const data = `${header}.${payloadB64}`;
  const key = await crypto.subtle.importKey(
    'raw',
    new TextEncoder().encode(secret),
    { name: 'HMAC', hash: 'SHA-256' },
    false,
    ['sign']
  );
  const sig = await crypto.subtle.sign('HMAC', key, new TextEncoder().encode(data));
  const sigB64 = base64UrlEncode(String.fromCharCode(...new Uint8Array(sig)));
  return `${data}.${sigB64}`;
}

async function verifyJWT(token: string, secret: string) {
  const parts = token.split('.');
  if (parts.length !== 3) throw new Error('invalid token');
  const [h, p, s] = parts;
  const data = `${h}.${p}`;
  const key = await crypto.subtle.importKey(
    'raw',
    new TextEncoder().encode(secret),
    { name: 'HMAC', hash: 'SHA-256' },
    false,
    ['verify']
  );
  const sig = Uint8Array.from(base64UrlDecode(s), (c) => c.charCodeAt(0));
  const valid = await crypto.subtle.verify('HMAC', key, sig, new TextEncoder().encode(data));
  if (!valid) throw new Error('invalid token');
  return JSON.parse(base64UrlDecode(p)) as { sub: string };
}

async function hashPassword(password: string, secret: string) {
  const data = new TextEncoder().encode(password + secret);
  const buf = await crypto.subtle.digest('SHA-256', data);
  return Array.from(new Uint8Array(buf))
    .map((b) => b.toString(16).padStart(2, '0'))
    .join('');
}

async function requireAuth(request: Request, env: Env) {
  const auth = request.headers.get('Authorization') || '';
  const token = auth.replace(/^Bearer\s+/i, '');
  if (!token) throw new Error('Unauthorized');
  return await verifyJWT(token, env.JWT_SECRET);
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);

    if (request.method === 'OPTIONS') {
      return new Response(null, { headers: corsHeaders });
    }

    if (url.pathname === '/api/register' && request.method === 'POST') {
      const body = (await request.json()) as { username?: string; password?: string };
      const { username, password } = body;
      if (!username || !password) return json({ error: 'required' }, 400);
      const hash = await hashPassword(password, env.JWT_SECRET);
      await env.DB.prepare('INSERT INTO users (username, password_hash) VALUES (?, ?)')
        .bind(username, hash).run();
      const token = await signJWT({ sub: username }, env.JWT_SECRET);
      return json({ success: true, token });
    }

    if (url.pathname === '/api/login' && request.method === 'POST') {
      const body = (await request.json()) as { username?: string; password?: string };
      const { username, password } = body;
      if (!username || !password) return json({ error: 'required' }, 400);
      const row = await env.DB.prepare('SELECT password_hash FROM users WHERE username = ?')
        .bind(username).first<{ password_hash: string }>();
      if (!row) return json({ error: 'invalid' }, 401);
      const hash = await hashPassword(password, env.JWT_SECRET);
      if (row.password_hash !== hash) return json({ error: 'invalid' }, 401);
      const token = await signJWT({ sub: username }, env.JWT_SECRET);
      return json({ success: true, token });
    }

    if (url.pathname === '/api/protected' && request.method === 'POST') {
      try {
        await requireAuth(request, env);
        return json({ ok: true });
      } catch (e) {
        return json({ error: (e as Error).message }, 401);
      }
    }

    return new Response('Not found', { status: 404 });
  },
};

function json(body: object, status = 200) {
  return new Response(JSON.stringify(body), {
    status,
    headers: { ...corsHeaders, 'Content-Type': 'application/json' },
  });
}
```

## Wrangler setup

Add `JWT_SECRET` as a secret (never commit it):
```bash
echo "your-secret" | npx wrangler secret put JWT_SECRET
```

## Pitfalls

- **base64url padding bug**: The common snippet `new Array(5 - (str.length % 4)).join('=')` breaks when the JWT segment length is divisible by 4 because it adds 4 extra `=` chars. Use `const padding = (4 - (str.length % 4)) % 4; str += '='.repeat(padding);` instead.
- **Do not** store `JWT_SECRET` in `wrangler.toml` `[vars]`; use `wrangler secret put` so it is encrypted.
- `crypto.subtle` is async; all JWT and password functions return Promises.
- D1 `UNIQUE constraint failed` on usernames must be caught and returned as a 409.
- Always include `Authorization` in `Access-Control-Allow-Headers` if your frontend sends Bearer tokens.
- If using Cloudflare Secrets Store for `JWT_SECRET`, remember to `await env.JWT_SECRET_STORE.get()` — forgetting `await` returns the binding object, not the string.