---
name: system-observability-connector
description: >
  Adds a secure Lumonly observability connector to an existing system.
  Use when the user asks to connect a system to Lumonly, create telemetry APIs,
  expose logs or metrics, instrument requests, observe resource consumption,
  send logs, metrics, and traces to a central dashboard, install the Lumonly
  web SDK, or review a codebase for try/catch blocks that hide errors.
---

# Lumonly System Observability Connector

Implement a connector that sends normalized telemetry to Lumonly using OpenTelemetry Protocol (OTLP) when possible. Keep the source system reliable: telemetry must be asynchronous or batched, bounded, and unable to break the business request path.

## Required discovery before editing

1. Identify language, framework, deployment model (host, Docker, Kubernetes, cloud) and existing logging/metrics libraries.
2. Identify the data owner and a stable Lumonly project credential. Never hard-code it.
3. Locate secrets and PII fields that must be redacted.
4. Determine whether metrics required are application/process, container, or full-host metrics. Do not claim host-level visibility if an approved host agent or cloud integration is unavailable.

## Web error SDK baseline

Install and initialize the Lumonly browser SDK once per web application. The SDK automatically captures unhandled browser exceptions and unhandled promise rejections. It may capture `console.error` only as an explicitly enabled fallback; console output is not the primary error contract.

```ts
import * as Lumonly from "@lumonly/browser";

Lumonly.init({
  endpoint: process.env.NEXT_PUBLIC_LUMONLY_ENDPOINT,
  projectKey: process.env.NEXT_PUBLIC_LUMONLY_PROJECT_KEY,
  environment: process.env.NODE_ENV,
  release: process.env.NEXT_PUBLIC_APP_VERSION,
});
```

Keep the project key and endpoint in environment configuration. Never place server ingest tokens in a browser bundle. The SDK must batch and retry exports without blocking or failing the application request path.

## AI-assisted error instrumentation workflow

When asked to add or review Lumonly instrumentation, scan the repository before editing:

1. Find `try/catch` and equivalent error handlers in application code. Ignore test fixtures, generated files, and cleanup-only `finally` blocks.
2. Detect existing `Lumonly.captureException`, the project's approved telemetry wrapper, or an equivalent error reporter. Do not add a duplicate report.
3. Classify each handler. A handler that recovers, returns a fallback, shows an error to the user, retries, or rethrows is a candidate for manual reporting. A handler used only for expected control flow (for example optional cache lookup) is not automatically a candidate.
4. Add the smallest reviewed patch at the point where the error is available:

```ts
try {
  await saveReport();
} catch (error) {
  Lumonly.captureException(error, {
    tags: { feature: "report_save", operation: "save" },
  });
  showSaveError();
}
```

5. Do not add capture calls to `finally`, every function, render methods, or ordinary `console.log` statements. Do not report the same error both in a lower-level helper and again at every caller unless the project explicitly requires a second boundary event.
6. Redact secrets and PII from context. Prefer stable tags such as `feature`, `operation`, `route`, and `service`; never attach raw request bodies, cookies, authorization headers, passwords, tokens, or form inputs.
7. Show the diff for review, run the relevant tests, and perform one controlled test error. Confirm that it reaches the intended Lumonly project, is grouped once, and does not alter the business response when export is unavailable.

Manual capture is required only when application code catches the error. Unhandled exceptions and unhandled promise rejections are covered by the one-time SDK initialization. Business events are separate from errors and must use the project's approved event API only when they are explicitly required by the feature.

## Telemetry contract

Use these resource attributes on every signal:

```text
service.name=<stable-service-name>
service.version=<release-version-if-known>
deployment.environment=<production|staging|development>
lumen.project_id=<server-derived-project-id>
```

Emit:

- **Logs**: timestamp, severity, message, category, trace ID, sanitized attributes.
- **Metrics**: request count, error count, request duration, process CPU, process memory, queue metrics where applicable.
- **Traces**: inbound HTTP/API requests, database calls, outbound HTTP calls and background jobs.

For API input/output visibility record only route, method, status, duration, request/response byte counts, outcome and a correlation ID. Do not store raw request/response bodies by default. Redact authorization headers, cookies, access tokens, passwords, financial identifiers and configured PII.

## Preferred implementation order

1. Add OpenTelemetry SDK for the system language and configure OTLP exporter through environment variables:

```text
OTEL_SERVICE_NAME=
OTEL_EXPORTER_OTLP_ENDPOINT=
OTEL_EXPORTER_OTLP_HEADERS=Authorization=Bearer <project-ingest-token>
OTEL_RESOURCE_ATTRIBUTES=deployment.environment=production
```

2. Instrument HTTP server, database client, outbound HTTP client and job runner with maintained official instrumentation packages.
3. Add structured application logs and correlate them with the active trace ID.
4. Add custom business metrics only when they have clear units and bounded label cardinality.
5. Add a health check that verifies local telemetry configuration without exposing credentials.
6. Configure retries, batching, timeouts and a bounded queue; failures to export telemetry must be logged locally and must not fail application requests.
7. Document deployment configuration, including variables needed by every container/worker.

## Resource-observability rules

- Application process metrics can be exported from application code.
- Container metrics require the platform runtime or collector integration (for example cAdvisor/Kubernetes metrics).
- Full-server CPU, memory, disk, filesystem and network metrics require an approved host agent (such as Node Exporter or Telegraf) or the cloud provider metrics API.
- Prefer an OpenTelemetry Collector between the application/agent and Lumonly. It owns authentication, enrichment, filtering, batching and routing.
- Do not add privileged Docker socket or host filesystem access to an application container merely to obtain metrics. Install/configure a dedicated infrastructure agent only with explicit infrastructure approval.

## Security checklist

- Store tokens only in environment/secret management; add placeholders to `.env.example`.
- Use TLS endpoint URLs.
- Do not let the source system set another project's identity; derive it from the ingest credential at Lumonly.
- Apply redaction before export and verify it with tests or sample telemetry.
- Add rate limits and payload limits to custom telemetry APIs.

## Completion evidence

- A test request creates a correlated trace, log and request metric under the intended project.
- A failing request appears with sanitized attributes and an error status.
- Process/container/host metric scope is explicitly documented; only verified scopes are presented in dashboards.
- Exporter outage does not change the source endpoint's success behavior or latency materially.
- Deployment instructions include all required variables for every app and worker process.

## Reference implementation — Node.js / Express

Use this baseline when the source system is Node.js. Choose versions compatible with the existing runtime; do not introduce a second logger or metrics library if the project already has an approved equivalent.

### 1. Install packages

```bash
npm install @opentelemetry/api @opentelemetry/sdk-node @opentelemetry/auto-instrumentations-node \
  @opentelemetry/exporter-trace-otlp-proto @opentelemetry/exporter-metrics-otlp-proto \
  @opentelemetry/sdk-metrics @opentelemetry/resources prom-client pino
```

### 2. Bootstrap telemetry before application imports

Create `instrumentation.ts` in the server entry-point area. It must be loaded before Express, database and HTTP client imports (for example, `node --require ./dist/instrumentation.js ./dist/server.js`).

```ts
import { NodeSDK } from "@opentelemetry/sdk-node";
import { getNodeAutoInstrumentations } from "@opentelemetry/auto-instrumentations-node";
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-proto";
import { OTLPMetricExporter } from "@opentelemetry/exporter-metrics-otlp-proto";
import { PeriodicExportingMetricReader } from "@opentelemetry/sdk-metrics";
import { resourceFromAttributes } from "@opentelemetry/resources";

const endpoint = process.env.OTEL_EXPORTER_OTLP_ENDPOINT;

const sdk = new NodeSDK({
  resource: resourceFromAttributes({
    "service.name": process.env.OTEL_SERVICE_NAME ?? "unnamed-service",
    "deployment.environment": process.env.DEPLOYMENT_ENVIRONMENT ?? "development",
  }),
  traceExporter: endpoint ? new OTLPTraceExporter({ url: `${endpoint}/v1/traces` }) : undefined,
  metricReader: endpoint
    ? new PeriodicExportingMetricReader({
        exporter: new OTLPMetricExporter({ url: `${endpoint}/v1/metrics` }),
        exportIntervalMillis: 15_000,
      })
    : undefined,
  instrumentations: [getNodeAutoInstrumentations()],
});

void sdk.start();
for (const signal of ["SIGTERM", "SIGINT"] as const) {
  process.once(signal, () => void sdk.shutdown().finally(() => process.exit(0)));
}
```

Keep endpoint URLs and credentials in deployment secrets, never in this file:

```dotenv
OTEL_SERVICE_NAME=billing-api
DEPLOYMENT_ENVIRONMENT=production
OTEL_EXPORTER_OTLP_ENDPOINT=https://collector.example.internal
OTEL_EXPORTER_OTLP_HEADERS=Authorization=Bearer <project-ingest-token>
```

If the exporter package/configuration in the target version does not consume `OTEL_EXPORTER_OTLP_HEADERS`, pass sanitized headers through its documented constructor configuration. Verify this with a real export; never print the token while diagnosing it.

### 3. Export application/process and API I/O metrics

Create one internal Prometheus endpoint only when the collector can reach it privately. It reports the process runtime and API-level request/response counts and sizes; it is not a substitute for host metrics.

```ts
import type { Request, Response, NextFunction } from "express";
import client from "prom-client";

client.collectDefaultMetrics({ prefix: "lumen_" });

const httpRequests = new client.Counter({
  name: "lumen_http_server_requests_total",
  help: "HTTP requests completed by this service",
  labelNames: ["method", "route", "status"] as const,
});
const httpDuration = new client.Histogram({
  name: "lumen_http_server_duration_seconds",
  help: "HTTP request duration",
  labelNames: ["method", "route", "status"] as const,
});
const httpResponseBytes = new client.Histogram({
  name: "lumen_http_server_response_bytes",
  help: "HTTP response size in bytes",
  labelNames: ["method", "route", "status"] as const,
});

export function observeHttp(req: Request, res: Response, next: NextFunction) {
  const started = process.hrtime.bigint();
  res.once("finish", () => {
    // A route template avoids unbounded label cardinality from IDs and query strings.
    const route = req.route?.path ?? req.baseUrl ?? "unmatched";
    const labels = { method: req.method, route, status: String(res.statusCode) };
    httpRequests.inc(labels);
    httpDuration.observe(labels, Number(process.hrtime.bigint() - started) / 1e9);
    const contentLength = Number(res.getHeader("content-length") ?? 0);
    if (Number.isFinite(contentLength) && contentLength >= 0) httpResponseBytes.observe(labels, contentLength);
  });
  next();
}

export async function metricsHandler(_req: Request, res: Response) {
  res.set("Content-Type", client.register.contentType);
  res.send(await client.register.metrics());
}
```

Mount `observeHttp` after route matching middleware when possible, and mount `GET /internal/metrics` only behind private networking or strong service authentication. Do not expose it on the public internet. Configure the Collector/Prometheus to scrape it rather than having browsers query it.

### 4. Emit sanitized structured logs

```ts
import pino from "pino";

const redact = [
  "req.headers.authorization", "req.headers.cookie", "password", "token",
  "accessToken", "refreshToken", "creditCard",
];

export const log = pino({ level: process.env.LOG_LEVEL ?? "info", redact });

// Safe attributes only: no raw body, credentials, cookie, or full URL query.
log.info({ category: "api", method: "POST", route: "/payments", status: 201, durationMs: 42 }, "Payment created");
```

Ship stdout JSON logs with the Collector or an existing log pipeline. When adding direct log export, confirm the chosen Node.js OpenTelemetry logs API/exporter is stable and compatible with the installed SDK; log export support can differ from traces and metrics.

## Reference deployment — Collector and host metrics

### OpenTelemetry Collector

Run the collector as a separate service/daemon. It accepts OTLP, limits memory, batches data, enriches it with a server-controlled project identifier, and forwards it to Lumonly or the selected telemetry backends.

```yaml
# otel-collector.yaml — template; endpoint and credentials are deployment secrets
receivers:
  otlp:
    protocols:
      grpc: {}
      http: {}

processors:
  memory_limiter:
    check_interval: 1s
    limit_mib: 512
  resource/lumen-project:
    attributes:
      - key: lumen.project_id
        value: ${env:LUMEN_PROJECT_ID}
        action: upsert
  batch:
    timeout: 5s

exporters:
  otlphttp/lumen:
    endpoint: ${env:LUMEN_INGEST_ENDPOINT}
    headers:
      Authorization: Bearer ${env:LUMEN_INGEST_TOKEN}

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, resource/lumen-project, batch]
      exporters: [otlphttp/lumen]
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, resource/lumen-project, batch]
      exporters: [otlphttp/lumen]
    logs:
      receivers: [otlp]
      processors: [memory_limiter, resource/lumen-project, batch]
      exporters: [otlphttp/lumen]
```

In production, authenticate source applications at the Collector boundary or keep that endpoint on a private network. The `lumen.project_id` above is an example for a collector dedicated to one project. A shared collector must derive the project identity from validated credentials, not from client-controlled resource attributes.

### Full-host and container scope

Install the following as infrastructure components only after approval from the server/platform owner:

| Runtime | Component | Verified scope |
|---|---|---|
| Linux VM/bare metal | Prometheus Node Exporter or Telegraf | host CPU, RAM, filesystem, disk I/O and network |
| Docker | cAdvisor plus Node Exporter | per-container resources plus host resources |
| Kubernetes | kubelet/cAdvisor metrics, kube-state-metrics and OTel Collector | pods, nodes, workload state and cluster context |
| Managed cloud | Provider monitoring API/exporter | provider-exposed instance, database and load-balancer metrics |

Do not bake this agent into an application image. Its configuration, credentials, network exposure, upgrades and retention are infrastructure responsibilities.

## Framework adaptation

- **Next.js**: initialize server-only instrumentation through the framework-supported instrumentation entry point; never instrument browser bundles with server tokens. Apply custom metrics only in the Node runtime, not Edge runtime.
- **Python/FastAPI**: use the official OpenTelemetry Python SDK plus FastAPI, requests/httpx and database instrumentations; expose process metrics through the Prometheus client on a private endpoint.
- **Java/Spring Boot**: prefer the OpenTelemetry Java agent or Spring Boot starter; configure it with environment variables and propagate trace headers through async executors.
- **.NET/ASP.NET Core**: use the OpenTelemetry .NET SDK and ASP.NET Core/HttpClient/SQL instrumentation; export via OTLP.

For every framework, preserve existing behavior and naming conventions. Generate the smallest compatible implementation after inspecting the actual project; do not copy the Node template into another language.
