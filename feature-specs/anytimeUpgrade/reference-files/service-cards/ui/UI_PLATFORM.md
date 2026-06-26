# UI_PLATFORM — Bridge

## §1 Runtime

| Property | Value |
|---|---|
| Framework | Next.js 15.2.6 |
| React version | React 18.2.0 |
| Router | Next.js file-based router (pages directory) |
| Rendering strategy | CSR — all data fetched client-side via TanStack Query after mount; no `getServerSideProps` or `getStaticProps` in use |
| Build tool | Next.js (Webpack + SWC) |
| TypeScript version | 5.4.2 |
| Node.js version | 18+ (per README) |
| State library | React Context (CartContext, PartnerContext) + TanStack Query v5 for server state |
| Data fetch library | TanStack Query v5 (`@tanstack/react-query@5.77.2`) + native `fetch` |
| Component library | `@react-spectrum/s2@0.9.1` (Adobe Spectrum v2) |
| Form library | `react-hook-form@7.51.2` + `zod@3.22.4` |
| Language | TypeScript |

## §2 Build

```json
"scripts": {
  "dev:local": "NODE_ENV=development node server.js",   // Custom Node server on port 9000 (used for local dev)
  "dev": "next dev -p 3000",                            // Standard Next.js dev server on port 3000
  "build": "next build",                                // Production build
  "start": "next start",                                // Start production server
  "lint": "next lint",                                  // ESLint via eslint-config-next
  "format": "prettier --write .",                       // Format all files with Prettier
  "format:check": "prettier --check .",                 // Check formatting without writing
  "test": "jest",                                       // Run all tests
  "test:watch": "jest --watch",                         // Watch mode
  "test:unit": "jest --testPathPattern=tests/unit",     // Unit tests only
  "test:coverage": "jest --coverage"                    // With coverage report
}
```

**Key Dependencies (version-sensitive):**

| Package | Version | Why it matters |
|---|---|---|
| `@react-spectrum/s2` | 0.9.1 | Adobe Spectrum v2 — breaking changes are common pre-1.0; pin tightly |
| `@tanstack/react-query` | 5.77.2 | v5 API differs significantly from v4; `gcTime` replaces `cacheTime` |
| `next` | 15.2.6 | App Router available but not used; stays on Pages Router |
| `react` | 18.2.0 | Required for Spectrum v2 concurrent features |
| `zod` | 3.22.4 | Runtime schema validation for models and form schemas |
| `newrelic` | 13.6.2 | Server-side APM; requires `newrelic.js` at repo root |

**Build plugins required:**
- `unplugin-parcel-macros@0.1.1` — registered as Webpack plugin in `next.config.js`
- Spectrum transpilation — `next.config.js` auto-globs and transpiles all `@adobe/react-spectrum` and `@react-spectrum/*` packages via `transpilePackages`

## §3 Environment Variables

| Variable | Browser-accessible | Description |
|---|---|---|
| `NODE_ENV` | No | Runtime environment (`development` / `production`) |
| `PARTNER_API_BASE_URL` | No | Base URL for VIP Marketplace partner API |
| `IMS_TOKEN_URL` | No | Adobe IMS OAuth token endpoint |
| `PARTNER_CLIENT_ID` | No | OAuth client ID for partner API authentication |
| `PARTNER_CLIENT_SECRET` | No | OAuth client secret |
| `PARTNER_NAME` | No | Display name of the partner (e.g. "Ingram US") |
| `MARKET_SEGMENTS` | No | JSON array of active market segments (e.g. `["COM","EDU","GOV"]`) |
| `CURRENCIES` | No | JSON array of active currencies (e.g. `["USD"]`) |
| `REGION` | No | Partner region (e.g. `NA`) |

**Critical:** No variables use the `NEXT_PUBLIC_` prefix, so **none are exposed to the browser**. All sensitive config is server-side only, accessed via `/api/*` routes.

## §4 Local Development

```bash
# 1. Clone and install
npm install

# 2. Copy env template
cp .env.sample .env
# Fill in PARTNER_API_BASE_URL, IMS_TOKEN_URL, PARTNER_CLIENT_ID, PARTNER_CLIENT_SECRET,
# PARTNER_NAME, MARKET_SEGMENTS, CURRENCIES, REGION

# 3. Start dev server (custom server on port 9000)
npm run dev:local

# Or standard Next.js dev server on port 3000
npm run dev
```

HTTPS is not configured locally. The custom server (`server.js`) uses plain Node `http`. A `/ping` health endpoint is available at `http://localhost:9000/ping`.

## §5 Routing Table (Quick Reference)

| Route | Page file | Auth required | Notes |
|---|---|---|---|
| `/` | `pages/index.tsx` | No | Redirects to `/resellers` |
| `/resellers` | `pages/resellers.tsx` | No | Reseller list with pagination |
| `/customers` | `pages/customers.tsx` | No | Requires `?resellerId=` query param |
| `/catalog` | `pages/catalog.tsx` | No | Product catalog; requires partner contract loaded |
| `/checkout` | `pages/checkout.tsx` | No | Requires non-empty cart + customer in cart |
| `/customerdetails` | `pages/customerdetails.tsx` | No | Requires `?customerId=` + `?resellerId=` |
| `/orderConfirmation` | `pages/orderConfirmation.tsx` | No | Requires query params from checkout |

See `ROUTES.md` for full details.

## §6 Browser Storage (Quick Reference)

| Key | Storage | Contents | Persistence |
|---|---|---|---|
| `bridge_cart` | `localStorage` | `{ cartItemIdToQuantityMap, cartItemIdToCartItemDetailsMap, customerInfoInCart }` | Until explicitly cleared |

See `STATE_MANAGEMENT.md §1` for full shape details.
