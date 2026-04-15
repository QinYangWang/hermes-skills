---
name: awesome-selfhosted-dataset
description: "Consume and build applications on top of the awesome-selfhosted-data machine-readable dataset. Use when building catalogs, aggregators, or generators that source from https://github.com/awesome-selfhosted/awesome-selfhosted-data."
---

# awesome-selfhosted-data Dataset Integration

## Trigger Conditions
Use this skill when:
- Building a web app, catalog, or tool on top of the Awesome Self Hosted list
- Need to understand the schema, fields, and sync strategy for `awesome-selfhosted/awesome-selfhosted-data`
- Want to avoid common misconceptions about what metadata is available (Docker images, deploy buttons, etc.)

## Repository Structure

```
awesome-selfhosted-data/
├── software/           # ~1000+ YAML files, one per project
├── tags/               # Category definitions
├── platforms/          # Platform definitions
├── licenses.yml        # License definitions
└── .hecat/             # Import/export automation configs
```

## Access Methods

### 1. Raw GitHub Content (simplest for bulk sync)
```bash
# List software files
curl -sL "https://api.github.com/repos/awesome-selfhosted/awesome-selfhosted-data/contents/software?per_page=100"

# Read individual file
curl -sL "https://raw.githubusercontent.com/awesome-selfhosted/awesome-selfhosted-data/master/software/immich.yml"
```

### 2. GitHub API Contents Endpoint
```bash
curl -sL "https://api.github.com/repos/awesome-selfhosted/awesome-selfhosted-data/contents/software/nextcloud.yml"
```
> **Pitfall**: The API returns `content` as **base64-encoded**. You must decode it:
> ```bash
> echo "<base64_string>" | base64 -d
> ```

### 3. Git Trees API (recursive listing)
```bash
curl -sL "https://api.github.com/repos/awesome-selfhosted/awesome-selfhosted-data/git/trees/master?recursive=1"
```

## Software Entry Schema

Each `software/<slug>.yml` contains:

```yaml
name: Immich
website_url: https://immich.app/
description: Photo and video backup solution directly from your mobile phone (alternative to Google Photos).
licenses:
  - AGPL-3.0
platforms:
  - Docker
tags:
  - Photo Galleries
source_code_url: https://github.com/immich-app/immich
demo_url: https://github.com/immich-app/immich#demo
stargazers_count: 96184
updated_at: '2026-04-02'
archived: false
current_release:
  tag: v2.6.3
  published_at: '2026-03-26'
commit_history:
  2025-05: 264
  2025-06: 300
  # ... last 12 months
```

### Field Reference

| Field | Type | Notes |
|-------|------|-------|
| `name` | string | Display name |
| `description` | string | May contain markdown links and `(alternative to X)` phrases |
| `website_url` | string | Official project website |
| `source_code_url` | string | GitHub/GitLab/etc. repo |
| `demo_url` | string | Optional live demo |
| `licenses` | string[] | SPDX identifiers |
| `platforms` | string[] | e.g., `Docker`, `PHP`, `Nodejs`, `deb`, `Python` |
| `tags` | string[] | Categories from `tags/` |
| `stargazers_count` | integer | GitHub stars (updated by bot) |
| `updated_at` | date | Last metadata refresh |
| `archived` | boolean | Whether the upstream repo is archived |
| `current_release.tag` | string | Latest release tag |
| `current_release.published_at` | date | Release publish date |
| `commit_history` | map | `{YYYY-MM: count}` for last 12 months |

## Tags Schema

Each `tags/<slug>.yml`:

```yaml
name: Analytics
description: '[Analytics](https://en.wikipedia.org/wiki/Analytics) is the systematic...'
related_tags:
  - Database Management
  - Personal Dashboards
```

## What's NOT in the Dataset (Critical Pitfalls)

1. **No Docker image names or default ports** — only `platforms: [Docker]` indicates Docker support. You cannot auto-generate `docker-compose.yml` without external curation.
2. **No structured "proprietary alternatives" field** — this info is embedded in `description` as prose, typically `(alternative to X)` or `(alternative to X and Y)`.
3. **No one-click deploy platform mappings** — there is no metadata for Vercel, Railway, Cloudflare, Render, Fly.io, etc.
4. **No primary language breakdown** — `platforms` lists runtime/frameworks, not GitHub-language-stats style primary language.

## Deriving Missing Fields

### Extracting Proprietary Alternatives
Use regex on `description`:
```python
import re
match = re.search(r'\(alternative to ([^)]+)\)', description, re.IGNORECASE)
alternatives = match.group(1) if match else None
# e.g., "Google Photos" or "Google Photos and Apple Photos"
```

### Filtering Docker-Only Projects
```python
if "Docker" in entry.get("platforms", []):
    # Project has Docker support
```

## Recommended Sync Architecture

For a Cloudflare-based app:

1. **GitHub Actions cron** (daily):
   - Fetch all `software/*.yml` files
   - Parse YAML
   - Extract proprietary alternatives via regex
   - Merge any external curated data (deploy buttons, Docker images) from your own JSON
   - Bulk upsert into **Cloudflare D1**

2. **Worker API**:
   - `GET /api/software?tag=&search=&sort=&page=` — serve catalog
   - `GET /api/tags` — serve categories

3. **Frontend**:
   - Vite + React + Tailwind
   - Search, tag filter, sort by stars/updated_at

## Full-Stack Implementation Learnings

### Tailwind CSS v4 + Vite
Tailwind v4 no longer uses the PostCSS plugin. Use the Vite plugin instead:
```bash
npm install -D @tailwindcss/vite
```
```typescript
// vite.config.ts
import tailwindcss from '@tailwindcss/vite'
export default { plugins: [tailwindcss()] }
```
```css
/* src/index.css */
@import "tailwindcss";
```

### D1 Bulk Import via `wrangler d1 execute`
**Critical**: `wrangler d1 execute --remote` does **not** accept `BEGIN TRANSACTION` or `COMMIT` in the SQL file. Remove them and run statements sequentially:
```sql
-- WRONG: D1 will reject BEGIN TRANSACTION
-- RIGHT: plain sequential INSERTs/DELETEs
DELETE FROM software;
DELETE FROM tags;
INSERT INTO software VALUES (...);
```

### Admin Auth with Cloudflare Secrets Store
Store a shared admin password in the account-level Secrets Store so multiple Workers can reuse it:
```toml
[[secrets_store_secrets]]
binding = "ADMIN_PASSWORD_STORE"
store_id = "<your-store-id>"
secret_name = "ADMIN_PASSWORD"
```
```typescript
const password = await env.ADMIN_PASSWORD_STORE.get();
const inputHash = await crypto.subtle.digest("SHA-256", new TextEncoder().encode(body.password));
// Compare hex hashes
```
Issue a JWT on successful login for stateless admin sessions.

## Automated Discovery Pipeline (GitHub Trending + LLM Filter)

Beyond the static dataset, you can automatically discover new self-hostable projects from **GitHub Trending** and community forums, then use an LLM to filter noise before human review.

### Architecture
```
GitHub Trending API → fetch metadata + README → Kimi/LLM judge → D1 pending queue → Admin approves → Promoted to main catalog
```

### 1. Fetch GitHub Trending
GitHub has no official Trending API. Use a stable third-party proxy:
```bash
curl -s "https://gtrend.yapie.me/repositories?since=daily"
```
Returns JSON array with `author`, `name`, `stars`, `description`, `language`.

### 2. Enrich with GitHub API
For each repo, fetch:
```bash
curl -s https://api.github.com/repos/<owner>/<repo>
curl -s https://api.github.com/repos/<owner>/<repo>/readme
```
Decode README `content` from base64 and truncate to ~800 chars.

### 3. LLM Filter Prompt (Kimi)
```yaml
model: kimi-k2-5-coder
temperature: 0.2
messages:
  - role: user
    content: |
      You are an expert in identifying self-hostable web services.

      Project: {name}
      Description: {description}
      Topics: {topics}
      Language: {language}
      README excerpt:
      {readme_snippet}

      Is this a complete end-user web application/service that can be self-hosted (not a library, framework, CLI tool, or SDK)?
      Reply in strict JSON:
      {
        "is_selfhostable": true|false,
        "confidence": 0.0-1.0,
        "category": "best category name",
        "has_docker": true|false,
        "reason": "short reason"
      }
```
Parse the JSON block from the response with a regex: `/\{[\s\S]*\}/`.
Only promote items where `is_selfhostable === true && confidence >= 0.6`.

### 4. D1 Schema for Discovery Queue
```sql
CREATE TABLE discovered_projects (
  id TEXT PRIMARY KEY,
  source TEXT NOT NULL,          -- 'github_trending' | 'hackernews' | 'reddit'
  source_url TEXT,
  name TEXT,
  description TEXT,
  github_url TEXT,
  stargazers_count INTEGER,
  discovered_at TEXT,
  llm_confidence REAL,
  llm_category TEXT,
  llm_has_docker INTEGER,
  status TEXT DEFAULT 'pending', -- 'pending' | 'approved' | 'rejected'
  merged_into_software_id TEXT
);
```

### 5. GitHub Actions Cron Workflow
```yaml
name: Sync and Discover
on:
  schedule:
    - cron: '0 4 * * *'
  workflow_dispatch:

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: npm install
      - name: Sync official dataset
        run: node scripts/sync-data.mjs
      - name: Fetch trending
        run: node scripts/fetch-trending.mjs
      - name: AI filter
        env:
          KIMI_API_KEY: ${{ secrets.KIMI_API_KEY }}
        run: node scripts/llm-filter.mjs
      - name: Apply to D1
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
        run: |
          npx wrangler d1 execute my-db --file=./schema.sql --remote
          npx wrangler d1 execute my-db --file=./sync.sql --remote
          [ -s discovered.sql ] && npx wrangler d1 execute my-db --file=./discovered.sql --remote
```

### 6. Admin Review API Pattern
- `GET /api/discovered?status=pending` — requires JWT, returns pending items
- `POST /api/discovered/:id/approve` — copies row into `software` table, sets `status='approved'`
- `POST /api/discovered/:id/reject` — sets `status='rejected'`
- `GET /api/discovered?status=approved` — public read-only feed of approved discoveries

### Cloudflare D1 Free Tier Limits
The free plan has a hard limit on the number of D1 databases. If you hit:
> "You have reached the maximum number of D1 databases for your account"

List and delete unused databases before creating a new one:
```bash
npx wrangler d1 list
npx wrangler d1 delete <unused-db-name>
```

### wrangler.toml Static Assets
Use the table syntax `[assets]`, not `[[assets]]`:
```toml
[assets]
directory = "./dist"
```
Using `[[assets]]` causes wrangler warnings.

### Reference Sync Script (Node.js)
```javascript
import { parse } from "yaml";
import fs from "fs/promises";
import path from "path";
import { execSync } from "child_process";

const REPO = "https://github.com/awesome-selfhosted/awesome-selfhosted-data.git";
const TMP = "/tmp/awesome-selfhosted-data";

function slugify(name) {
  return name.toLowerCase().replace(/[^a-z0-9]+/g, "-").replace(/^-|-$/g, "");
}

function extractAlternative(desc) {
  if (!desc) return null;
  const m = desc.match(/\(alternative to ([^)]+)\)/i);
  return m ? m[1] : null;
}

// Clone or shallow pull
execSync(`git clone --depth 1 ${REPO} ${TMP}`, { stdio: "inherit" });

const files = await fs.readdir(path.join(TMP, "software"));
let sql = "DELETE FROM software;\n";
for (const f of files) {
  if (!f.endsWith(".yml")) continue;
  const data = parse(await fs.readFile(path.join(TMP, "software", f), "utf8"));
  const alt = extractAlternative(data.description);
  sql += `INSERT INTO software (...) VALUES (...);\n`;
}
await fs.writeFile("sync.sql", sql);
// Then run: wrangler d1 execute <db> --file=./sync.sql --remote
```

## Pitfalls

- **Case-sensitive slugs**: Filenames use lowercase with hyphens, but `name` field preserves capitalization.
- **API rate limits**: Bulk reading via GitHub API contents endpoint hits rate limits quickly. Use raw.githubusercontent.com URLs for bulk sync instead.
- **Missing files**: Not every project you expect exists under the filename you guess. Always list directory contents first.
- **Description parsing**: `description` may contain markdown links like `[more](https://...)` — render or strip markdown appropriately.
- **Proprietary alternatives extraction is heuristic**: Some descriptions use variations like "(open-source alternative to X)" or "alternative to X, Y, and Z".
- **GitHub API content is base64-encoded**: The `contents` API returns `content` as base64. Decode it before parsing YAML.
