# Platform — bridge

---

## 1. Deployment Topology

| Field | Value |
|-------|-------|
| Container base | `node:20-alpine` |
| Entry point | `node server.js` |
| HTTP port (primary) | 9000 |
| HTTP port (secondary, Dockerfile EXPOSE) | 8080 |
| Health check | `GET /ping` → 200 `OK` (handled in `server.js` before Next.js router) |
| Process manager | none — single Node.js process |
| Kubernetes / Terraform | NOT CAPTURED — no deployment spec located in repo |
| Helm | NOT CAPTURED |

---

## 2. Server Bootstrap

`server.js` creates a plain Node.js `http.Server` wrapping the Next.js request handler:

```javascript
app.prepare().then(() => {
  http.createServer(requestHandler).listen(9000, '0.0.0.0');
});
```

`/ping` is intercepted before Next.js and returns `200 OK` — suitable for load-balancer health checks.

---

## 3. Logging

| Field | Value |
|-------|-------|
| Library | Pino 9.x |
| Base fields | `service: bridge-app`, `version`, `environment: NEXT_PUBLIC_APP_ENV`, `pid`, `hostname` |
| Production format | Structured JSON (stdout) |
| Development format | pino-pretty (colourised, human-readable) |
| Level control | `LOG_LEVEL` env var (default: `debug` in dev, `info` in prod) |

---

## 4. Monitoring

| System | Details |
|--------|---------|
| New Relic | `newrelic` npm package (13.6.2) loaded via `newrelic.js` at app root. Standard Node.js APM — automatic instrumentation of HTTP requests and outbound fetch. |

---

## 5. Caching

| Cache | Location | TTL | Notes |
|-------|----------|-----|-------|
| IMS access token | `utils/imsTokenService.ts` module-level variables | `expires_in - 300s` | In-process only; single pod; lost on restart |
| `/api/env` response | `pages/api/env.ts` module-level variables | 60s | Also sets `Cache-Control: public, max-age=60` |

---

## 6. Feature Flags

NOT CAPTURED — no feature flag infrastructure detected in codebase.

---

## 7. Database / Persistence

None. Bridge is stateless — all data lives in the Adobe Partner API.

---

## 8. Active Migrations

None — stateless service.

---

## 9. Runbooks

NOT CAPTURED — no runbook links in codebase.

---

## 10. Security Headers

Set by `next.config.js` on all routes:

| Header | Value |
|--------|-------|
| `X-Frame-Options` | `DENY` |
| `X-Content-Type-Options` | `nosniff` |
