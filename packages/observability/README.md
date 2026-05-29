# Observability

**Status:** Architecture Phase  
**Implementation Status:** Not Started

## Purpose

Shared logging, metrics, and tracing utilities: standard labels, correlation ID propagation, Prometheus metric registration helpers, OpenTelemetry setup.

## Planned Contents

- Structured log formatter
- `correlation_id` middleware
- Metric name constants aligned with [observability-and-reliability.md](../../docs/architecture/observability-and-reliability.md)

## Implemented

Nothing.

## Planned

Library package consumed by all services.

## Future Work

- Auto-instrumentation for HTTP clients
- Transition event exporter helper
