# Build & Environment Configuration — {{SERVICE_NAME}}

> **Role:** The authoritative bridge between source code and the runtime environment.
> **Purpose:** Defines abstract requirements for dependencies, environment variables, and secrets.

---

## 1. Runtime Specifications
*The technical constraints of the execution environment.*

| Specification | Value |
|:---|:---|
| **Runtime Engine** | `{{e.g., Node.js 20.x / Java 17 / Go 1.22}}` |
| **Package Manager** | `{{e.g., npm / Maven / Go Modules}}` |
| **Entry Point** | `{{e.g., src/index.ts / com.pkg.App / main.go}}` |
| **Default Port** | `{{e.g., 8080}}` |
| **Artifact Type** | `{{e.g., Docker Image / Zip Archive / Executable Binary}}` |

---

## 2. Dependency Manifest (Abstract)
*Core capabilities required from external libraries.*

| Category | Capability Required | Preferred Standard |
|:---|:---|:---|
| **Web Framework** | REST Routing, Middleware, JSON Parsing | `{{OpenAPI 3.0 Compatible}}` |
| **Data Access** | Connection Pooling, Migration Engine | `{{e.g., SQL-based / NoSQL}}` |
| **Communication** | HTTP Client, Retry Logic, Circuit Breaking | `{{Standardized Error Handling}}` |
| **Validation** | Schema validation for Request/Response | `{{Type-safe schemas}}` |
| **Observability** | Structured Logging, Metrics, Tracing | `{{OpenTelemetry Standard}}` |

---

## 3. Configuration Schema (Environment Variables)
*The settings injected at runtime.*

### 3.1 System Metadata

| Variable | Values | Purpose |
|----------|--------|---------|
| `APP_ENV` | `local \| stage \| prod` | Active environment |
| `LOG_LEVEL` | `DEBUG \| INFO \| WARN \| ERROR` | Log verbosity |

### 3.2 Connectivity (Downstreams)

| Variable | Example | Purpose |
|----------|---------|---------|
| `{{DEPENDENCY}}_BASE_URL` | `https://api.example.com` | Base URL for upstream dependency |
| `{{DEPENDENCY}}_TIMEOUT_MS` | `3000` | Request timeout in milliseconds |
| `{{DEPENDENCY}}_RETRIES` | `3` | Maximum retry attempts |

### 3.3 Persistence & Caching

| Variable | Purpose |
|----------|---------|
| `DATABASE_URL` | Full connection string (or use host/port/db components) |
| `CACHE_HOST` / `CACHE_PORT` | Endpoint for volatile storage |

---

## 4. Secrets & Sensitive Data
*Inputs that must be retrieved from a secure store (Vault/Secret Manager). Never hardcoded.*

| Secret Key | Consumer | Purpose |
|:---|:---|:---|
| `PRIMARY_AUTH_KEY` | Identity Layer | Token signing and validation. |
| `DB_CREDENTIAL` | Data Layer | Database authentication credentials. |
| `EXTERNAL_API_KEY` | Connector Layer | Third-party service integration key. |

---

## 5. Build Interface
*The abstract triggers for the CI/CD pipeline.*

*   **Setup:** `{{Command to install dependencies}}`
*   **Build:** `{{Command to compile or bundle code}}`
*   **Verify:** `{{Command to run tests and linters}}`
*   **Package:** `{{Command to create the final artifact}}`

---

## 6. Cross-Environment Mappings
*Target URLs across lifecycle stages.*

| Target Service | Stage Endpoint | Prod Endpoint |
|:---|:---|:---|
| `{{Internal_API_1}}` | `https://{{service}}-stage.example.com` | `https://{{service}}.example.com` |
| `{{External_Vendor}}` | `https://sandbox.vendor.com` | `https://api.vendor.com` |