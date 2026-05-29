# Infrastructure

**Status:** Architecture Phase  
**Implementation Status:** Not Started

## Purpose

Operational infrastructure configuration: database, containers, Kubernetes, metrics, and dashboards.

## Directories

| Directory | Role |
|-----------|------|
| `db/` | PostgreSQL and migrations |
| `docker/` | Local compose and images |
| `k8s/` | Kubernetes manifests (post-local demo) |
| `prometheus/` | Scrape and alert config |
| `grafana/` | Dashboards and datasources |

## Implemented

Nothing.

## Planned

Introduced incrementally as services become runnable.
