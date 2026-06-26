# Connectors — {{SERVICE_NAME}}

> **Role:** Registry and implementation guide for all outbound dependencies.
> **Purpose:** Documents the interaction contract, resilience posture, and auth requirements for every downstream the service calls.
> **Last Updated:** {{DATE}}

---

## 1. Overview

| Field | Value |
|-------|-------|
| Total connectors | `{{N}}` |
| Shared auth mechanism | `{{e.g., Bearer token / API key / mTLS / mixed}}` |
| Shared transport | `{{e.g., HTTPS / gRPC / mixed}}` |

---

## 2. Connector Summary

| # | Dependency | Protocol | Auth Type | Connect Timeout | Retry | Circuit Breaker | Criticality |
|---|------------|----------|-----------|-----------------|-------|-----------------|-------------|
| 1 | `{{Name}}` | `{{REST / gRPC}}` | `{{Token / Key / mTLS}}` | `{{ms}}` | `{{N}}` | `{{Y / N}}` | `{{CRITICAL / HIGH / LOW}}` |

---

## 3. Connector Details

<!-- One `### <N>. <ConnectorName> — <one-line desc>` per connector.
     The heading slug becomes the anchor referenced by SERVICE_CARD.md → ## 3. Contracts
     and MODULE_INDEX.md → ## 4. Capability Catalogue. -->

### 1. {{ConnectorName}} — {{one-line description}}

**Purpose:** {{Why this service calls this dependency}}

**Operations:**

| Operation | Input | Output | Error codes |
|-----------|-------|--------|-------------|
| `{{OperationName}}` | `{{params / body}}` | `{{response shape}}` | `{{4xx/5xx meanings}}` |

**Auth & transport:**

| Field | Value |
|-------|-------|
| Auth mechanism | `{{e.g., OAuth2 client credentials / API key / mTLS}}` |
| Credential source | `{{env var or secret store key}}` |
| Base URL source | `{{config property or env var}}` |
| Mandatory headers | `{{e.g., Authorization, x-request-id}}` |

**Resilience:**

| Aspect | Value |
|--------|-------|
| Connect timeout | `{{ms}}` |
| Request timeout | `{{ms}}` |
| Retry strategy | `{{e.g., exponential backoff, max N attempts}}` |
| Circuit breaker | `{{threshold / none}}` |

**Error handling:**

| Downstream signal | Internal classification | Action |
|-------------------|------------------------|--------|
| `404 Not Found` | `NOT_FOUND` | Return empty / propagate |
| `401 / 403` | `AUTH_ERROR` | Log and fail-fast |
| `429 Too Many Requests` | `THROTTLED` | Honor Retry-After or backoff |
| `5xx / Timeout` | `UNAVAILABLE` | Retry or fail-fast |

**Unavailability posture:** `{{e.g., fail-fast / fallback value / circuit breaker trips}}`

<!-- Repeat `### <N>. <ConnectorName> — <desc>` per connector. -->

---

## 4. Integration Patterns

| Component | Implementation | Config prefix |
|-----------|---------------|---------------|
| `{{ConnectorName}}` | `{{SDK / custom wrapper}}` | `{{e.g., app.connectors.payment}}` |

---

## 5. Observability

| Signal | Requirement |
|--------|-------------|
| Latency | Emit `latency_ms` per connector call |
| Call count | Emit `call_count` tagged with `target_service` |
| Error count | Emit `error_count` tagged with `status_code` |
| Trace context | Propagate W3C `traceparent` (or equivalent) to all downstreams |
| Log policy | No PII, credentials, or full request/response bodies in production logs |

---

## 6. Adding a New Connector

1. Document it here (`CONNECTORS.md`) — auth, base URL, operations, error posture
2. Identify the credential and its injection method (env var / secret store)
3. Align timeouts with the target service's P99 latency
4. Map downstream HTTP/gRPC codes to internal domain exceptions
5. Verify the connector appears in service dashboards after deployment
