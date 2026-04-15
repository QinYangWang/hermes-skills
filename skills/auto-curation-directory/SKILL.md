---
name: auto-curation-directory
description: "Build an auto-curated software directory with daily GitHub trending ingestion, LLM-based filtering, admin approval queue, and D1 persistence on a single Cloudflare Worker."
---

# Auto-Curation Directory Pipeline

## Trigger Conditions
Use when building a catalog/directory/showcase site that needs:
- Automatic discovery of new items from external feeds (GitHub, HN, Reddit)
- LLM-based quality/filtering to reject noise
- Human-in-the-loop admin approval before public display
- Daily cron automation with minimal infrastructure

## Architecture

```
GitHub Trending  →  fetch-trending.mjs
        ↓
GitHub API metadata  →  repo info, topics, README
        ↓
LLM filter (Kimi/OpenAI)  →  is_selfhostable? confidence? category?
        ↓
D1 discovered_projects (status=pending)  ←  Admin reviews
        ↓
Admin approves  →  INSERT INTO software  →  Public catalog
```

All running on **one Cloudflare Worker + D1 + Static Assets**.

## Step-by-Step

### 1. D1 Schema
```sql
CREATE TABLE software (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  description TEXT,
  website_url TEXT,
  source_code_url TEXT,
  stargazers_count INTEGER,
  tags_json TEXT,
  -- ... your main catalog fields
);

CREATE TABLE discovered_projects (
  id TEXT PRIMARY KEY,
  source TEXT NOT NULL,            -- 'github_trending' | 'hackernews' | 'reddit'
  source_url TEXT,
  name TEXT,
  description TEXT,
  github_url TEXT,
  stargazers_count INTEGER,
  discovered_at TEXT,
  llm_confidence REAL,
  llm_category TEXT,
  llm_has_docker INTEGER,
  status TEXT DEFAULT 'pending',   -- pending | approved | rejected
  merged_into_software_id TEXT
);

CREATE INDEX idx_discovered_status ON discovered_projects(status);
```

### 2. Scraper Script (`scripts/fetch-trending.mjs`)
GitHub Trending has no official API. Use a stable third-party endpoint:
```javascript
const ENDPOINT = "https://gtrend.yapie.me/repositories?since=daily";
const res = await fetch(ENDPOINT);
const items = (await res.json()).map(r => ({
  id: `${r.author}/${r.name}`,
  source: "github_trending",
  github_url: `https://github.com/${r.author}/${r.name}`,
  stargazers_count: r.stars || 0,
  discovered_at: new Date().toISOString(),
}));
await fs.writeFile("trending_raw.json", JSON.stringify(items, null, 2));
```

**Pitfall**: If the endpoint returns 404, try `https://api.gitterapp.com/repositories?since=daily` or fallback to cheerio scraping.

### 3. LLM Filter (`scripts/llm-filter.mjs`)
Fetch repo metadata + README, then ask LLM:
```javascript
const prompt = `Project: ${name}
Description: ${description}
Topics: ${topics.join(", ")}
README excerpt: ${readme.slice(0, 600)}

Is this a complete end-user web application that can be self-hosted?
Reply in strict JSON:
{
  "is_selfhostable": true|false,
  "confidence": 0.0-1.0,
  "category": "best category",
  "has_docker": true|false,
  "reason": "..."
}`;
```

Only items with `is_selfhostable && confidence >= 0.6` are written to `discovered.sql`.

**Rate limiting**: Sleep 500ms between LLM calls to avoid hitting TPM limits.

### 4. Admin Auth (Worker)
Store one admin password in **Cloudflare Secrets Store**:
```toml
[[secrets_store_secrets]]
binding = "ADMIN_PASSWORD_STORE"
store_id = "<your-store-id>"
secret_name = "ADMIN_PASSWORD"
```

Worker login endpoint:
```typescript
const adminPassword = await env.ADMIN_PASSWORD_STORE.get();
const inputHash = await sha256(body.password);
const expectedHash = await sha256(adminPassword);
if (inputHash !== expectedHash) return new Response("Unauthorized", { status: 401 });
const token = await new SignJWT({ role: "admin" }).setExpirationTime("7d").sign(secret);
```

### 5. Approval API
```typescript
// POST /api/discovered/:id/approve
if (!admin) return forbidden();

// 1. copy fields from discovered → software
await env.DB.prepare(`INSERT OR REPLACE INTO software ...`).bind(...).run();

// 2. mark discovered as approved
await env.DB.prepare(`UPDATE discovered_projects SET status = 'approved', merged_into_software_id = ? WHERE id = ?`).bind(id, id).run();
```

### 6. Daily Cron (GitHub Actions)
```yaml
on:
  schedule:
    - cron: '0 4 * * *'

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm install
      - run: node scripts/fetch-trending.mjs
      - run: node scripts/llm-filter.mjs
        env:
          KIMI_API_KEY: ${{ secrets.KIMI_API_KEY }}
      - run: |
          npx wrangler d1 execute my-db --file=./discovered.sql --remote
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
```

**Pitfall**: Do NOT wrap the discovered SQL in `BEGIN TRANSACTION; ... COMMIT;`. D1's `execute` command may reject transactions in some modes. Run plain `INSERT` statements instead.

## Frontend Patterns

### Discovered Tab (read-only for guests)
```
GET /api/discovered?status=approved&page=1&limit=24
```

### Admin view (with token)
```
GET /api/discovered?status=pending&page=1&limit=24
Headers: Authorization: Bearer <token>
```
Response includes `isAdmin: true` so UI can render Approve/Reject buttons.

## Cost Estimates
| Source | Daily volume | LLM cost |
|--------|-------------|----------|
| GitHub Trending | ~100 repos | ~¥0.5-2 |
| HN + Reddit | ~50 posts | ~¥0.5-1 |

GitHub API has 5,000 req/h free tier. Cloudflare D1/Worker free tier is plenty.

## Pitfalls
1. **GitHub Trending endpoint dies** — always have a fallback scraper or mock mode.
2. **LLM returns malformed JSON** — regex-extract the JSON block; wrap in try/catch.
3. **Same file re-upload doesn't trigger onChange** — reset `e.target.value = ''` in the handler.
4. **D1 transaction errors in CI** — avoid `BEGIN/COMMIT` in scripts executed via `wrangler d1 execute --file`.
5. **Secrets Store binding returns a Promise** — always `await env.BINDING.get()`, never direct string comparison.

## Variations
- Replace GitHub Trending with HN / Product Hunt / Reddit by changing the scraper.
- Use a lower confidence threshold (e.g. 0.4) and let admin do more manual filtering.
- Add email/webhook notifications when pending queue grows above N items.
