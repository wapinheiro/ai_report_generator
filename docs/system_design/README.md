# System Design Documentation

This folder captures the current system architecture and supporting assets for the Medical Operations Data Pipeline initiative.

## Contents
- `system_architecture.md` — detailed project brief, stage-by-stage architecture narrative, non-functional requirements, and operational roadmap.
- `PXL_20251111_172644715.jpg` — reference photo from initial whiteboard session.
- `PXL_20251111_191139958.MP.jpg` — supplementary whiteboard snapshot.

## Architecture Overview
The pipeline follows four orchestrated stages sharing a persistent `doc_id` for lineage:
1. **Dispatcher** ingests heterogeneous documents, normalizes them into JSON, enriches metadata, and stores both raw and extracted artifacts in the data lake.
2. **Classifier** leverages NLP models (with HITL fallback) to assign document types and persists confidence metrics to the operational store.
3. **Analyzer** executes versioned “recipes” per document type, extracting structured fields, emitting observability signals, and writing analysis tables to the warehouse.
4. **Reporter** serves curated data through a semantic layer into dashboards and optional NLG summaries, backed by SLA-driven refresh and data quality checks.

Key cross-cutting concerns include encryption in transit/at rest, lineage tracking, monitoring for model/recipe drift, and disaster recovery objectives.

For the full description and Mermaid architecture diagram, open `system_architecture.md`.

## Next Steps
- Convert the diagram to a static asset (PNG/SVG) for slide decks if required.
- Fill in TBD targets in the non-functional requirements (volumes, latency, RPO/RTO).
- Extend documentation with sequence diagrams, data schemas, and runbooks as designs mature.

