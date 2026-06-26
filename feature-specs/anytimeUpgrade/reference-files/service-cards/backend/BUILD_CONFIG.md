# Build Configuration — bridge

---

## 1. Stack

| Category | Choice | Version |
|----------|--------|---------|
| Build tool | npm | per package-lock.json |
| Language | TypeScript | 5.4.2 |
| Runtime | Node.js | 20 (Dockerfile) |
| Framework | Next.js (Pages Router) | 15.2.6 |
| HTTP client | Native `fetch` (built-in) | Node.js 20 |
| Validation | Zod | 3.22.4 |
| Logging | Pino + pino-pretty | 9.9.5 + 13.1.1 |
| Monitoring | New Relic | 13.6.2 |
| Auth | IMS OAuth2 — jsonwebtoken (decode only) | 9.0.2 |
| ORM / Persistence | none — stateless | — |
| Test framework | Jest + ts-jest + jest-environment-jsdom | 30.1.3 + 29.4.1 + 30.1.2 |

---

## 2. Key Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `next` | 15.2.6 | Full-stack framework (pages router + API routes) |
| `react` | 18.2.0 | UI library |
| `@react-spectrum/s2` | ^0.9.1 | Adobe Spectrum 2 design system |
| `@tanstack/react-query` | ^5.77.2 | Client-side data fetching and caching |
| `zod` | ^3.22.4 | Schema validation (both request and response) |
| `pino` | ^9.9.5 | Structured logging |
| `newrelic` | ^13.6.2 | APM monitoring |
| `jsonwebtoken` | ^9.0.2 | JWT decode for IMS token inspection |
| `uuid` | ^11.1.0 | Generate `X-Correlation-Id` and `X-Request-Id` per call |
| `axios` | ^1.6.8 | Declared but unused in server-side controllers (controllers use native fetch) |
| `node-fetch` | ^3.3.2 | Used only in integration test files |
| `cookie` | ^0.6.0 | Cookie parsing |
| `js-cookie` | ^3.0.5 | Browser cookie access |
| `react-hook-form` | ^7.51.2 | Form state management in UI |
| `dotenv` | ^16.5.0 | `.env` file loading |

---

## 3. Runtime Configuration

All configuration is supplied via environment variables. No config files other than `.env`.

| Env var | Required | Default | Description |
|---------|----------|---------|-------------|
| `PARTNER_API_BASE_URL` | YES | — | Base URL for Adobe Partner API (e.g. `https://partners.adobe.io`) |
| `IMS_TOKEN_URL` | NO | `https://ims-na1-stg1.adobelogin.com/ims/token/v2` | IMS token endpoint (staging default — change for prod) |
| `PARTNER_CLIENT_ID` | YES | — | OAuth2 client ID; also sent as `x-api-key` header on every Partner API call |
| `PARTNER_CLIENT_SECRET` | YES | — | OAuth2 client secret |
| `PARTNER_NAME` | NO | `""` | Partner display name shown in UI |
| `MARKET_SEGMENTS` | YES | — | JSON array of market segment codes, e.g. `["COM","EDU","GOV"]` |
| `CURRENCIES` | YES | — | JSON array of currency codes, e.g. `["USD"]` |
| `REGION` | YES | — | Region code, e.g. `"NA"` |
| `IMS_SCOPES` | NO | `openid,AdobeID,read_organizations` | OAuth2 scopes for IMS token |
| `NODE_ENV` | NO | `development` | `development` / `production` / `test` |
| `LOG_LEVEL` | NO | `debug` (dev) / `info` (prod) | Pino log level |
| `NEXT_PUBLIC_APP_ENV` | NO | — | App environment label, surfaced in logs and client |

See `.env.sample` for the minimal required set.

---

## 4. Security Headers (next.config.js)

Applied to all routes (`/:path*`):

| Header | Value |
|--------|-------|
| `X-Frame-Options` | `DENY` |
| `X-Content-Type-Options` | `nosniff` |

---

## 5. Build Scripts

| Script | Command |
|--------|---------|
| Dev server | `next dev` |
| Dev with custom server | `NODE_ENV=development NEXT_PUBLIC_APP_ENV=local node server.js` |
| Production build | `next build` |
| Production start | `node server.js` |
| Test (all) | `jest` |
| Test (unit only) | `jest tests/unit` |
| Test (coverage) | `jest --coverage` |
| Lint | `next lint` |
| Format | `prettier --write .` |
