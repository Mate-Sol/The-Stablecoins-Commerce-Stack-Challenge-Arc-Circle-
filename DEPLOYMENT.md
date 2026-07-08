# Deploy — Ignyte Stablecoins Commerce Stack Challenge (Circle Arc)

Two-service deploy: **Vercel for both clients + Railway for server (with Mongo add-on)**. Total time: ~30 min for a fresh setup, mostly form-clicking. This is the same shape Colosseum used.

Three deployables per repo:
- `client/` — v2 lender UI → Vercel
- `client-legacy/` — PSP + admin portals → Vercel
- `server/` — Node API → Railway (+ Railway Mongo)

## Public URLs (once deployed)

- **Lender UI**: https://defa-arc.vercel.app (or custom domain defa-arc.invoicemate.net)
- **Admin/PSP UI**: https://defa-arc-admin.vercel.app
- **API**: https://defa-arc-api.up.railway.app

## Step 1 — Backend (Railway)

1. Sign in to https://railway.app with GitHub.
2. **New Project → Deploy from GitHub repo** → pick this repo.
3. After the project scaffolds:
   - **Settings → Root Directory**: `server`
   - **Settings → Start Command**: (blank — nixpacks.toml handles it)
4. **Variables** — copy from `server/.env.production.example`. Fill in:

```
PORT                          5050
JWT_SECRET                    <generate with: node -e "console.log(require('crypto').randomBytes(32).toString('base64url'))">
FRONTEND_URL                  https://defa-arc.vercel.app
EXTRA_CORS_ORIGINS            https://defa-arc.vercel.app,https://defa-arc-admin.vercel.app

EVM_CHAIN_ID                  5042002
EVM_RPC_URL                   https://rpc.testnet.arc.network
PAYFI_FACTORY_ADDRESS         0x4e39880B43f9a83586a2aC75a01dff779Eb958c0
PAYFI_STABLECOIN_ADDRESS      0x2b2037760695772770182C84dFeE2b9594526c7f
PAYFI_TREASURY_ADDRESS        0xcC3a9A71532a1402Ab57742C22661eE6e96102e5

AGENT_PRIVATE_KEY             0x692fb6f9b2c22e3d2ad4e0434f22f41617fb65ce1ac89146da2ae21b58443ce9
FAUCET_AUTHORITY_PRIVATE_KEY  0x692fb6f9b2c22e3d2ad4e0434f22f41617fb65ce1ac89146da2ae21b58443ce9
ONCHAIN_ADMIN_WALLETS         0x0b9dDfcdB31aEf5Cde26d0E6DbAc6917B6849f05

SIWE_DOMAIN                   defa-arc.vercel.app
SIWE_ORIGIN                   https://defa-arc.vercel.app

EVM_INDEXER_WINDOW_BLOCKS     100
EVM_INDEXER_INTERVAL_MS       90000
```

5. **Add MongoDB plugin**: Project → New Service → Database → MongoDB. Railway wires it into your service — set `MONGODB_URI` to the auto-injected connection string (`${{Mongo.MONGODB_URI}}`).
6. Click **Deploy**. First build ~3 min. Check logs — should end with:
   `[evmIndexer] starting; interval=90000ms window=100 blocks`
   `MongoDB Connected`
7. **Settings → Networking → Generate Domain**. Note the resulting `defa-arc-api.up.railway.app` URL.

## Step 2 — Frontend (Vercel × 2)

### 2a. v2 lender client

1. https://vercel.com/new → import same GitHub repo.
2. **Configure project**:
   - **Root Directory**: `client`
   - Framework preset: **Vite** (should auto-detect via `client/vercel.json`)
   - Build command / output: leave defaults
3. **Environment Variables** — from `.env.production.example`. Fill:

```
VITE_API_URL                          https://defa-arc-api.up.railway.app
VITE_CHAIN_ID                         5042002
VITE_CHAIN_NAME                       Arc Testnet
VITE_CHAIN_NATIVE_SYMBOL              USDC
VITE_CHAIN_NATIVE_DECIMALS            18
VITE_RPC_URL                          https://rpc.testnet.arc.network
VITE_CHAIN_EXPLORER_URL               https://testnet.arcscan.app
VITE_STABLECOIN_ADDRESS               0x2b2037760695772770182C84dFeE2b9594526c7f
VITE_TREASURY_ADDRESS                 0xcC3a9A71532a1402Ab57742C22661eE6e96102e5
VITE_FACTORY_ADDRESS                  0x4e39880B43f9a83586a2aC75a01dff779Eb958c0
VITE_WALLETCONNECT_PROJECT_ID         defa-demo
```

4. Click **Deploy**. Auto-assigns `defa-arc.vercel.app` (or an ID; you can rename in Settings → Domains).

### 2b. PSP + admin (legacy) client

1. https://vercel.com/new → import same GitHub repo (yes, again — Vercel supports multiple projects per repo).
2. **Configure project**:
   - **Root Directory**: `client-legacy`
   - Framework preset: Vite
3. **Environment Variables** — same list as 2a, plus:

```
VITE_API_URL                          https://defa-arc-api.up.railway.app
```

4. Deploy. Rename domain to `defa-arc-admin.vercel.app`.

## Step 3 — CORS

Go back to Railway. Update the server's env:

```
EXTRA_CORS_ORIGINS   https://defa-arc.vercel.app,https://defa-arc-admin.vercel.app
```

Railway auto-redeploys on env change.

## Step 4 — Seed access code (one time)

Get a Railway Mongo shell (Railway Settings → Data → Query):

```javascript
db.accesscodes.insertOne({
  code: "654321",
  label: "public-demo",
  usedAt: null,
  expiresAt: new Date(Date.now() + 30*24*3600*1000),
  createdBy: "seed"
})
```

## Step 5 — Verify

- `curl https://defa-arc-api.up.railway.app/pools` → 3 pools
- Open `https://defa-arc.vercel.app/enter-access-code` → paste `654321`
- Full lender loop should work on real Arc Testnet.

## Custom domain (optional)

Vercel + Railway both support custom domains via CNAME. Point:
- `defa-arc.invoicemate.net` CNAME → Vercel project 1
- `defa-arc-admin.invoicemate.net` CNAME → Vercel project 2
- `defa-arc-api.invoicemate.net` CNAME → Railway domain

Update the server envs `FRONTEND_URL`, `SIWE_DOMAIN`, `SIWE_ORIGIN`, and Vercel client's `VITE_API_URL` accordingly.

## Redeploy on code change

Vercel + Railway both auto-deploy on `git push`. No manual action required.
