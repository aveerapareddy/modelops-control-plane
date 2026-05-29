# Prometheus Infrastructure

**Status:** Architecture Phase  
**Implementation Status:** Not Started

## Purpose

Prometheus scrape configuration, recording rules, and alert rules for lifecycle, training, deployment, drift, and workflow failure metrics.

## Planned Contents

- `prometheus.yml` scrape targets for platform services
- Alert rules: stuck `DEPLOYING`, `FAILED` spike, scrape down
- Recording rules for SLO-style aggregates (when defined)

## Implemented

Nothing.

## Planned

Config added alongside first service metrics exporters.

## Future Work

- Thanos or remote write for long-term retention
