# ADR 024: Semantic Annotations — Rejected (Defer)

**Status:** Rejected  
**Date:** 2026-02-26

## Context

OSI/MetricFlow and Malloy classify fields as `entity`, `dimension`, or `measure`. This metadata helps BI tools, AI agents, and documentation generators understand the semantic role of each output field. The [external landscape research](../research/external-landscape-2026-02.md) and [Malloy gap analysis](../research/malloy-gap-analysis.md) identified optional semantic annotations as a candidate feature.

## Decision

Reject for now. No annotation syntax, comment convention, or external API.

Three options were evaluated:

1. **Inline DSL annotation** (`SET id = o.id @entity`) — adds non-functional grammar; the parser and compiler gain complexity for metadata that doesn't affect SQL output.
2. **Structured comment convention** (`-- @entity`) — zero grammar cost but fragile, informal, and not queryable.
3. **External PG API** (`pg_weave_annotate(...)`) — keeps DSL clean but disconnects metadata from the definition it describes.

None of these justify their cost at current project maturity. The core DSL is still stabilising, and there are no downstream consumers (BI tools, AI agents) that would use the metadata today.

## Rationale

- No compile-time or runtime effect — annotations are pure documentation.
- No downstream consumer exists yet to validate the design.
- Adding annotations now risks baking in a vocabulary that may not match real usage.
- Can be revisited once the core language is stable and there's a concrete integration target.

## Consequences

- Fields carry no semantic classification beyond their name and SQL type.
- Users who need this metadata can maintain it externally (e.g. PG `COMMENT ON COLUMN`).
- When a BI/AI integration materialises, this decision should be revisited.
