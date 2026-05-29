# Registry Service

**Status:** Architecture Phase  
**Implementation Status:** Not Started

## Purpose

Store model version metadata, artifact URIs, checksums, and links to evaluation results and optional MLflow run IDs. Support queries by `model_id` and `version_id`.

## Boundaries

| Owns | Does not own |
|------|--------------|
| Version metadata and artifact references | Lifecycle state (read-only mirror if cached) |
| Registration API after eval gate | Binary artifact bytes (object storage) |
| Checksum verification at register | Promotion decisions |

## Implemented

Nothing.

## Planned

Register API, version query API, artifact integrity checks.

## Future Work

- Model card metadata
- Schema/version compatibility matrix
