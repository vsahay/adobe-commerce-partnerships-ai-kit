# Code Patterns — {{SERVICE_NAME}}

> **Role:** The coding conventions in force for this service — what the skill scans from the codebase and what a code generator must respect.
> **Last Updated:** {{DATE}}

---

## 1. Naming & Package Layout

| Concern | Convention |
|---------|------------|
| Package / module root | `{{e.g., com.example.service / src/ / app/}}` |
| Controllers / handlers | `{{naming pattern — e.g., *Controller, *Handler, *Router}}` |
| Services / use cases | `{{naming pattern — e.g., *Service, *UseCase}}` |
| Connectors / clients | `{{naming pattern — e.g., *Connector, *Client}}` |
| Repositories / DAOs | `{{naming pattern — e.g., *Repository, *Dao}}` |
| DTOs | `{{naming pattern — e.g., *Request, *Response, *Dto}}` |
| Constants | `{{naming pattern — e.g., *Constants, UPPER_SNAKE_CASE fields}}` |

---

## 2. DTO Patterns

| Aspect | Convention |
|--------|------------|
| Immutability | `{{e.g., immutable via constructor / mutable setters / record types}}` |
| Null handling | `{{e.g., Optional<> / nullable annotations / empty collections}}` |
| Serialization | `{{e.g., Jackson annotations / Gson / built-in}}` |
| Validation annotations | `{{e.g., placed on DTO fields / separate validator class}}` |

---

## 3. Error Handling

| Aspect | Convention |
|--------|------------|
| Error type hierarchy | `{{e.g., extends RuntimeException / custom sealed classes}}` |
| Error codes | `{{e.g., enum ErrorCode / string constants}}` |
| Global handler | `{{e.g., @ControllerAdvice class / middleware / exception_handler}}` |
| Response shape | `{{e.g., { code, message, traceId } / RFC 7807 ProblemDetail}}` |
| Logging on error | `{{e.g., log at ERROR with context / structured fields}}` |

---

## 4. Validation

| Aspect | Convention |
|--------|------------|
| Library | `{{e.g., Bean Validation / Zod / Pydantic / govalidator}}` |
| Invocation point | `{{e.g., controller layer / service boundary / both}}` |
| Custom validators | `{{e.g., ConstraintValidator / custom Zod refinement}}` |
| Failure response | `{{e.g., 400 with field-level errors / single message}}` |

---

## 5. Cross-cutting Concerns

*Populated by scan 2.8 — one row per middleware / interceptor / filter found in the codebase.*

| Name | Purpose | Scope | Reads | Writes | Order |
|------|---------|-------|-------|--------|-------|
| `{{ComponentName}}` | `{{e.g., auth check / correlation ID injection}}` | `{{global / route-specific}}` | `{{e.g., Authorization header}}` | `{{e.g., sets auth context}}` | `{{e.g., 1st}}` |

---

## 6. Test Patterns

| Aspect | Convention |
|--------|------------|
| Framework | `{{e.g., JUnit 5 / Jest / pytest / testing package}}` |
| Mocking library | `{{e.g., Mockito / Jest mocks / unittest.mock / testify}}` |
| Test naming | `{{e.g., should_<action>_when_<condition> / describe/it blocks}}` |
| Connector mocking | `{{e.g., @MockBean / nock / responses / httptest}}` |
| Integration tests | `{{e.g., Spring Boot test slice / supertest / TestClient}}` |

---

## 7. Universal Principles

*Language-agnostic rules applied across all capabilities.*

### Naming
- Methods/functions describe actions (`calculateTotal`, `fetchUserById`)
- Boolean predicates prefixed with `is`, `has`, `can`, or `should`
- Use the domain's ubiquitous language consistently — no synonyms across layers

### Design
- Composition over inheritance — avoid deep hierarchies
- Single Responsibility — one reason to change per class/module
- Stateless logic — externalize state to DB/cache
- No upward dependencies — Controller → Service → Connector → Domain only

### Data integrity
- DTOs immutable after instantiation
- Never return `null` for collections — return empty
- Use Optional/Result wrappers for potentially missing single values

### Error handling
- Never swallow errors silently — log with context or re-throw
- Always chain the original cause when wrapping exceptions
- Validate inputs at the outermost boundary before business logic
