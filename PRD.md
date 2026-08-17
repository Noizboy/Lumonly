# PRD — Lumonly

## 1. Summary

Lumonly will be a central observability platform for multiple systems. Each system or project will send or expose its logs, metrics, and traces through a secure connector. Users will be able to select a project and inspect its activity, errors, performance, resource usage, and data exchanges without mixing information from other projects.

## 2. Problem and opportunity

Teams operate multiple systems with isolated logs and metrics. Diagnosing incidents, comparing consumption, and deciding whether a system should migrate to another server requires manually accessing each infrastructure.

Lumonly centralizes this evidence, preserves project isolation, and turns technical telemetry into operational decisions.

## 3. Goals

- Centralize structured logs from connected systems.
- Display every system as an isolated project in Lumonly.
- Query errors, events, performance, and activity by project and time range.
- Detect, group, prioritize, and resolve web application errors with enough context to reproduce them and measure their user impact.
- Measure process/container CPU, memory, disk, and network usage, plus full-server usage when a host agent is available.
- Record API data inputs and outputs without retaining unnecessary secrets or personal data.
- Enable an AI to implement consistent connectors through the `system-observability-connector` skill.
- Provide documented manual and AI-assisted SDK integration so future code changes preserve error visibility.

## 4. Non-goals for the first version

- Replacing a specialized SIEM, APM, or security platform.
- Capturing full payload bodies, passwords, tokens, or sensitive information.
- Obtaining complete server metrics solely from application code without infrastructure permissions or an agent.
- Creating automatic remediation actions on connected systems.
- Replacing a full session-replay, profiling, or distributed-APM product in the initial error-tracking release.

## 5. Users and primary use cases

| User | Need | Expected outcome |
|---|---|---|
| Operations | Detect failures and degradation | Filters errors and peaks by project and period |
| Development | Investigate a request or incident | Correlates log, trace, endpoint, and duration |
| Technical leadership | Decide on migration or sizing | Compares sustained usage, peaks, and host capacity |

## 6. Functional scope (MVP)

### 6.1 Organizations and projects

- An organization contains multiple projects.
- A project represents a connected system or service.
- Each project has rotatable ingestion credentials, an environment (`production`, `staging`, `development`), and labels.
- Navigation requires selecting a project before viewing data; searches do not cross projects by default.

### 6.2 Logs

- Authenticated HTTP ingestion and paginated query of JSON logs.
- Normalized fields: timestamp, level, service, message, category, environment, `projectId`, `traceId`, and allowed metadata.
- Filters by time range, level, category, service, text, and correlation.
- Configurable retention per project.

### 6.3 Metrics and resources

- Charts for CPU, memory, disk, network, latency, request rate, errors, and throughput.
- A clear distinction between system/process, container, and host metrics.
- Initial alerts for availability, error rate, and resource-consumption thresholds.
- Each metric is labeled with project, service, environment, and instance.

### 6.4 Web error tracking

- A lightweight JavaScript/TypeScript SDK captures unhandled browser exceptions (`window.onerror` and `unhandledrejection`) and exposes `captureException(error, context)` for handled errors.
- Every error event includes a timestamp, stack trace, URL/route, browser and OS details, environment, release, service, allowed tags, and an anonymized user or session identifier when available.
- The SDK records configurable breadcrumbs, including route changes, failed HTTP requests, and relevant business actions. It batches telemetry, retries safely, and must never block or visibly degrade the source application if ingestion is unavailable.
- Ingestion accepts errors using a project-scoped public key, derives the project from that key, validates the schema, applies batch and payload limits, and redacts configured sensitive fields before persistence.
- Errors are fingerprinted and grouped into issues using error type, message, and relevant stack frames. Grouping rules can later merge, split, or ignore known noise.
- The issue list shows status, severity, occurrence count, affected users, first seen, last seen, environment, release, and owner. Statuses are `open`, `acknowledged`, `resolved`, and `ignored`.
- The issue view displays readable stack frames, recent occurrences, tags, breadcrumbs, affected releases, and a link to a correlated trace when available.
- The platform sends a webhook alert when a new production issue appears or when an issue's configured error-rate threshold is exceeded.

### 6.5 Traces and data input/output

- Distributed traces for HTTP requests, queued jobs, and external calls when supported by the source system.
- For APIs: method, normalized route, response code, duration, request/response sizes, and outcome; payloads are opt-in, redacted, and bounded.
- For host network data: received/sent bytes, connections, and errors through the infrastructure agent.

### 6.6 Dashboards

- Home page with project list and last-telemetry status.
- Per-project dashboard: overview, issues, logs, metrics, traces, and connector configuration.
- Capacity dashboard: trends, peaks, and comparisons against configured limits to support migration decisions.

## 7. Proposed architecture

```text
Connected Node.js services ── OTLP/HTTPS ── OpenTelemetry Collector ── Prometheus / Loki / Tempo ── Lumonly API/UI
Web error SDK ── HTTPS ─────────────────── Lumonly API ── PostgreSQL / Redis ────────────────────────────────────────┘
Node Exporter + cAdvisor ── Prometheus scrape ───────────────────────────────────────────────────────────────────────┘
```

1. Connected Node.js services use the OpenTelemetry JavaScript SDK and OTLP/HTTPS for logs, metrics, and traces.
2. The OpenTelemetry Collector authenticates, applies limits, adds project attributes, and forwards logs to Loki, metrics to Prometheus, and traces to Tempo.
3. Node Exporter and cAdvisor run on every monitored Docker host and are scraped by Prometheus for host and container metrics.
4. The Lumonly API queries data labeled with `organizationId` and `projectId`; this boundary also applies to access control.
5. The web error SDK sends error batches directly to the Lumonly API. The API normalizes and redacts each event, creates the fingerprint, stores occurrences and issue state in PostgreSQL, and uses BullMQ workers to evaluate alert rules.

## 8. Selected implementation stack

| Area | Technology | Required use |
|---|---|---|
| Telemetry standard | OpenTelemetry with OTLP/HTTPS | Every connected Node.js service emits logs, metrics, and traces using the OpenTelemetry JavaScript SDK. |
| Telemetry collector | OpenTelemetry Collector | Receives OTLP, enforces limits, injects tenant attributes, and forwards each signal. |
| Host and container metrics | Prometheus Node Exporter and cAdvisor | Run on every monitored Docker host; Prometheus scrapes both exporters. |
| Metrics storage | Prometheus | Stores and queries metrics for the MVP. |
| Log storage | Loki | Stores and queries structured logs. |
| Trace storage | Tempo | Stores and queries distributed traces. |
| Error occurrences and issues | PostgreSQL | Stores error occurrences, fingerprints, issue state, assignments, alert rules, organizations, projects, and credentials. |
| Backend API | TypeScript with NestJS | Hosts ingestion, query, authentication, issue management, and alert APIs. |
| Web application | TypeScript with Next.js | Hosts the Lumonly dashboard. |
| Frontend design system | Magic UI | The exclusive source for layouts and visual components, as defined in section 13. |
| ORM | Prisma | Provides all PostgreSQL access from NestJS. |
| Charts | Apache ECharts | Renders telemetry charts inside Magic UI layouts. |
| Asynchronous jobs | Redis and BullMQ | Runs error grouping, alert evaluation, retention deletion, and webhook delivery. |
| Authentication and authorization | Keycloak using OIDC and RBAC | Authenticates users and issues role claims scoped to organizations and projects. |
| Deployment | Docker Compose | Deploys Lumonly, PostgreSQL, Redis, OpenTelemetry Collector, Prometheus, Loki, Tempo, Node Exporter, and cAdvisor for the MVP. |

## 9. Minimum ingestion contract

`POST /v1/ingest/logs` will receive event batches through the OpenTelemetry Collector using a project-exclusive credential. The Collector derives the project from the credential; the client cannot arbitrarily choose another `projectId`.

```json
{
  "resource": { "service.name": "billing-api", "deployment.environment": "production" },
  "logs": [{
    "timestamp": "2026-08-17T15:00:00.000Z",
    "severity": "ERROR",
    "message": "Payment provider timeout",
    "traceId": "...",
    "attributes": { "http.route": "/payments", "http.status_code": 504, "duration_ms": 8200 }
  }]
}
```

The query API always applies the authorized user's `organizationId` and `projectId` and provides pagination, filters, size limits, and rate limiting.

## 10. Can an API observe server resource consumption?

Yes, with an important distinction:

- **Application consumption**: the application can expose process CPU and memory, latency, requests, errors, input/output sizes, and custom metrics. In Node.js, for example, it can report `process.cpuUsage()`, `process.memoryUsage()`, and instrumented HTTP metrics.
- **Container consumption**: cAdvisor attributes CPU, memory, network, and limits to each Docker container.
- **Full server**: Node Exporter provides host CPU, RAM, disk, I/O, and network metrics. An API inside the application cannot reliably or safely promise this complete visibility.

Therefore, Lumonly can support a decision to migrate or remain: it must collect application/container and host metrics over a representative period and compare them with limits and peaks. It must not rely on a single instantaneous reading.

## 11. Web error event contract

`POST /v1/ingest/errors` receives batches authenticated by a project-scoped public key. The server derives `projectId` from the key and creates an immutable occurrence plus an issue fingerprint; the client cannot set either field.

```json
{
  "release": "web@2026.08.17.1",
  "environment": "production",
  "errors": [{
    "timestamp": "2026-08-17T15:00:00.000Z",
    "type": "TypeError",
    "message": "Cannot read properties of undefined",
    "stack": "...",
    "url": "https://app.example.com/checkout",
    "user": { "id": "user_opaque_id" },
    "tags": { "feature": "checkout" },
    "breadcrumbs": [{ "category": "http", "statusCode": 500, "route": "/api/payment" }]
  }]
}
```

The server validates field types and maximum sizes, masks configured keys and query parameters, and rejects payloads that violate project quotas or rate limits. Source maps are outside the MVP; the UI explicitly identifies minified stack frames rather than presenting them as source locations.

## 12. Security and privacy

- Mandatory TLS, per-project keys with rotation and expiry.
- RBAC by organization and project; never trust a client-provided `projectId`.
- Redact `Authorization`, cookies, tokens, passwords, PII, and configured payloads before persistence.
- Encryption at rest, access audit trails, and retention/deletion policies.
- Batch limits, per-project quotas, rate limiting, and backpressure to protect source systems.
- Host agents are installed only with explicit authorization and the least possible privilege.
- Do not collect request/response bodies, form input, DOM snapshots, cookies, authorization headers, or raw IP addresses by default. User identity must be pseudonymous unless an explicit, documented policy authorizes more.

## 13. Frontend design constraint

- The Lumonly frontend must be designed exclusively with [Magic UI](https://magicui.design/) components and layouts.
- Do not invent, recreate, or introduce a custom layout that does not belong to the Magic UI framework.
- When Magic UI does not provide a suitable component or layout, the product requirement must be revised or deferred; implementing a bespoke visual alternative is not permitted.

## 14. SDK documentation and AI-assisted instrumentation

- The project must publish one integration guide for the Lumonly browser SDK covering installation, environment variables, automatic error capture, manual `Lumonly.captureException(error, context)`, business events, redaction, testing, and troubleshooting.
- The SDK is initialized once per web application. It automatically captures unhandled browser exceptions and unhandled promise rejections; `console.error` is an optional fallback and is not the primary error contract.
- Manual capture is required when application code catches an error with `try/catch` and recovers, shows a fallback, retries, or rethrows it. `finally` blocks and ordinary `console.log` calls do not require error capture.
- The `system-observability-connector` skill scans application source for `try/catch` and equivalent handlers, ignores generated code and tests, detects existing reporters, and proposes the smallest non-duplicating Lumonly patch.
- The skill must not instrument every function, cleanup-only handlers, expected control-flow exceptions, or `finally` blocks. It must never add raw bodies, cookies, authorization headers, passwords, tokens, form inputs, or other prohibited data to context.
- The skill must show a reviewable diff, run relevant tests, execute one controlled test error, and verify that the event reaches the correct project and is grouped once. It must not apply changes silently.
- A new feature may add explicit business events only when the event is needed for a defined product metric; errors remain covered by the existing pipeline.

## 15. Phases and acceptance criteria

### Phase 1 — Foundations

- [ ] Data model for organizations, projects, environments, and credentials.
- [ ] NestJS, PostgreSQL, Prisma, Redis, BullMQ, Keycloak, and Docker Compose are running as the Lumonly foundation.
- [ ] Authenticated JSON log ingestion and project-isolated query.
- [ ] UI with project selector, log list, and filters.
- [ ] Tests proving that a credential or user cannot query another project's data.
- [ ] Integration guide documents browser SDK setup and the distinction between automatic capture and manual `try/catch` capture.

### Phase 2 — Web error tracking

- [ ] A reference Next.js application sends an unhandled exception and a manually captured exception to Lumonly in under 10 seconds.
- [ ] Ten equivalent occurrences are shown as one issue with an occurrence count of ten; distinct root causes remain separate.
- [ ] An issue displays stack trace, release, environment, URL, anonymized user/session, and breadcrumbs without prohibited sensitive data.
- [ ] A new production issue and a controlled error-rate spike trigger a webhook alert.
- [ ] If Lumonly is unavailable, the reference application continues to operate without a user-visible error or blocking request.
- [ ] An authorized user can mark an issue acknowledged, resolved, or ignored; a later occurrence changes the last-seen timestamp and preserves the issue history.
- [ ] The `system-observability-connector` skill identifies an unreported recovery `catch`, proposes one `Lumonly.captureException` patch with sanitized context, and leaves an already reported or expected-control-flow `catch` unchanged.

### Phase 3 — Metrics and traces

- [ ] A Node.js reference service using the OpenTelemetry JavaScript SDK sends logs, metrics, and traces through the OpenTelemetry Collector to Loki, Prometheus, and Tempo.
- [ ] Per-project dashboard displays latency, error rate, and application/container CPU and memory.
- [ ] Correlation from an error to its associated `traceId`.

### Phase 4 — Infrastructure and capacity

- [ ] Node Exporter and cAdvisor deliver host and Docker-container CPU, RAM, disk, and network metrics, labeled by instance.
- [ ] Capacity dashboard displays trend, peaks, limits, and analyzed period.
- [ ] Alerts are tested with a controlled threshold breach.

## 16. Fixed operational constraints and risks

- The MVP supports Chromium, Firefox, and Safari releases that are current or one major release behind; the error SDK targets those browsers only.
- The first source platform is Node.js services and browser applications written in JavaScript or TypeScript. Other languages and platforms are outside the MVP.
- The web error SDK has a maximum gzipped bundle contribution of 20 KB and samples no unhandled errors by default; project quotas are 100,000 error occurrences per calendar month.
- Retention is fixed at 30 days for logs, 14 days for traces, 90 days for raw error occurrences, and one year for issue metadata and aggregated counts.
- Source-map upload, session replay, backend exception SDKs, release/issue integrations, Kubernetes deployment, and automatic remediation are outside the MVP.
- Installing Node Exporter and cAdvisor on a monitored host requires explicit infrastructure authorization; without it, Lumonly does not display host or container metrics for that host.
