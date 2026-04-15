---
name: cloudflare-secrets-store
title: Cloudflare Secrets Store
description: Practical guide to managing account-level secrets, bindings, and access control with Cloudflare Secrets Store.
---

# Cloudflare Secrets Store

## What it is
Cloudflare Secrets Store is an **account-level**, encrypted secret vault. Secrets are replicated across all Cloudflare data centers and can be bound to Workers and AI Gateway.  
- **One store per account**, max **100 production secrets** (open beta).  
- Each secret is a string ≤ **1024 bytes**.  
- After creation, the **value is never retrievable** via dashboard or API—only the bound service can access it.  
- **Not available** in the Cloudflare China Network.

## When to use it
- Reuse the same secret across **multiple Workers** without duplicating it per-script.  
- Centrally rotate a secret once and have all bound services pick up the new value automatically.  
- Share secrets with **AI Gateway** (BYOK setups).  

> Prefer per-Worker secrets only when the secret is specific to a single Worker and you don’t need centralized rotation.

## Creating and storing secrets

### Prerequisites
- **Role:** Super Administrator or Secrets Store Admin to create/edit secrets.  
- **API token permission:** `Account Secrets Store Edit` (or `Read` for metadata-only).

### Via Wrangler (CLI)
```bash
# List stores to get your STORE_ID
npx wrangler secrets-store store list

# Create a production secret (requires interactive value input)
npx wrangler secrets-store secret create <STORE_ID> --name MY_SECRET --scopes workers --remote

# Local-only secret (no --remote; does not count toward the 100 limit)
npx wrangler secrets-store secret create <STORE_ID> --name MY_LOCAL_SECRET --scopes workers
```

### Via Dashboard
1. Go to **Secrets Store** in the Cloudflare dashboard.
2. Select **Create secret**, fill in **Name**, **Value**, **Permission scope** (e.g. `workers`), and optional comment.
3. Save. The value will be hidden immediately.

### Via API
```bash
curl "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/secrets_store/stores/$STORE_ID/secrets" \
  --request POST \
  --header "Authorization: Bearer $TOKEN" \
  --json '[
    {
      "name": "MY_SECRET",
      "value": "<SECRET_VALUE>",
      "scopes": ["workers"],
      "comment": ""
    }
  ]'
```

### Updating and deleting
- **Edit (PATCH):** `https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/secrets_store/stores/$STORE_ID/secrets/$SECRET_ID`  
  ⚠️ Editing a value **replaces it in all bound services immediately**.
- **Delete (DELETE):** Same endpoint with `DELETE`.  
  ⚠️ Verify the secret is **not bound** to any Worker or AI Gateway first.

## Binding secrets to Workers

### 1. Configure the binding
**wrangler.jsonc**
```json
{
  "secrets_store_secrets": [
    {
      "binding": "API_KEY",
      "store_id": "<STORE_ID>",
      "secret_name": "MY_SECRET"
    }
  ]
}
```

**wrangler.toml**
```toml
[[secrets_store_secrets]]
binding = "API_KEY"
store_id = "<STORE_ID>"
secret_name = "MY_SECRET"
```

### 2. Access in Worker code
Access is **async** via `env.<BINDING>.get()`:
```js
export default {
  async fetch(request, env) {
    const apiKey = await env.API_KEY.get();
    const res = await fetch("https://api.example.com/data", {
      headers: { "Authorization": `Bearer ${apiKey}` },
    });
    return res;
  },
};
```

### Dashboard alternative
1. Go to **Workers & Pages** → select a Worker → **Settings > Bindings > Add**.
2. Choose **Secrets Store**, set the variable name, and pick the secret.

## Binding secrets to AI Gateway
Secrets Store can be used with AI Gateway for **bring-your-own-key** configurations.  
- In the AI Gateway dashboard, create or edit a gateway and associate a secret from your Secrets Store.  
- Required role: **Super Administrator** or **Secrets Store Deployer**.

## Access control & permissions

| Role | Can CRUD secrets | Can bind to Workers / AI Gateway |
|------|------------------|----------------------------------|
| Super Administrator | ✅ | ✅ |
| Secrets Store Admin | ✅ | ❌ |
| Secrets Store Deployer | ❌ (read metadata only) | ✅ |
| Secrets Store Reporter | ❌ (read metadata only) | ❌ |

**API token permissions**
- `Account Secrets Store Edit` — create, edit, duplicate, delete.
- `Account Secrets Store Read` — view metadata only.

## Audit logs
Logged actions: **Access**, **Create**, **Update**, **Delete**.  
- Duplicate shows as `create` with `duplicated_from_id`.  
- Value edits show `"value_modified": true`.

## Common pitfalls
1. **Irretrievable values** — Once created, you cannot view the secret again. Keep a backup in your own vault if needed.
2. **Async access only** — In Workers, `env.BINDING.get()` is a Promise; forgetting `await` returns the binding object, not the string.
3. **Local dev mismatch** — Production secrets (dashboard/API/`--remote`) are **not available locally**. Use Wrangler without `--remote` for local-only secrets.
4. **Name restrictions** — Secret names **cannot contain spaces**.
5. **Global impact on edit** — Changing a secret value updates it for every bound Worker and gateway instantly.
6. **Delete before unbind** — Deleting a bound secret will break live services. Unbind it first.

## Quick reference commands
```bash
# Create store (if none exists)
npx wrangler secrets-store store create my-store --remote

# List secrets in a store
npx wrangler secrets-store secret list <STORE_ID>

# Update a secret
npx wrangler secrets-store secret update <STORE_ID> <SECRET_NAME> --scopes workers --remote
```
