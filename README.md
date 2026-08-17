# Lumonly

Lumonly is a multi-project observability platform. It centralizes logs, metrics, and traces from connected systems to investigate incidents, understand performance, and support capacity or migration decisions.

## Status

The project is in the product-definition and architecture stage. The initial scope is documented in the [PRD](PRD.md).

## What it will provide

- Isolated projects for viewing each connected system separately.
- Secure ingestion of structured logs, metrics, and traces.
- Per-project dashboards for errors, activity, latency, and resource consumption.
- Correlation between requests, logs, and traces.
- Process, container, and host metrics according to the available integration.
- Historical evidence to determine whether a system should migrate or remain on its current infrastructure.

## Proposed architecture

```text
Source system ── OpenTelemetry / Logs API ── OpenTelemetry Collector ── Lumonly
Host agent ── infrastructure metrics ───────────────────────────────────┘
```

Source systems emit telemetry through OpenTelemetry (preferred) or the Logs API pattern. The Collector receives, limits, enriches, and forwards data. Infrastructure agents provide real host and container metrics when authorized.

## Resource-consumption scope

| Scope | How it is collected | Examples |
|---|---|---|
| Application | Instrumentation inside the system | Process CPU/memory, requests, latency, errors, HTTP bytes |
| Container | Container runtime or collector | CPU, memory, network, and limits per container |
| Server | Host agent or cloud metrics | Full-server CPU, RAM, disk, I/O, and network |

An application alone cannot guarantee complete server visibility. That view requires an authorized agent—such as Node Exporter or Telegraf—or a cloud-provider metrics integration.

## Documentation

- [PRD](PRD.md): requirements, architecture, technology options, and phases.
- [Logs API skill](SKILL.md): structured logging and query-API pattern for Next.js/Express with Prisma and PostgreSQL.
- [Connector skill](skills/system-observability-connector/SKILL.md): telemetry implementation for source systems.
- [Changelog skill](skills/changelog-maintenance/SKILL.md): local rule for documenting changes.
- [Changelog](CHANGELOG.md): project change history.

## Initial technology options

- OpenTelemetry and OpenTelemetry Collector for instrumentation and ingestion.
- Loki for logs, Prometheus/Thanos or Mimir for metrics, and Tempo for traces.
- Node Exporter, Telegraf, cAdvisor, or cloud metrics for infrastructure.
- Next.js/TypeScript for the dashboard and API; PostgreSQL for transactional data.
- Embedded Grafana or ECharts for visualizations.

The final selection must be validated against telemetry volume, retention, data residency, and deployment platform.

## Change conventions

Every relevant modification must update [CHANGELOG.md](CHANGELOG.md) in the same delivery, following the local `changelog-maintenance` skill.
