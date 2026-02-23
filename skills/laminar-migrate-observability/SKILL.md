---
name: laminar-migrate-observability
description: "Migrate an existing observability/tracing setup (Langfuse, LangSmith, Helicone, or OpenTelemetry) to Laminar with minimal diffs. Use when a user wants to replace their current tracer/exporter, keep trace boundaries, or map tags/metadata to Laminar."
---

# Laminar Migrate Observability

## Workflow

1. Identify the current provider, runtime, and entrypoints. Capture trace boundaries and existing context (user/session IDs, tags, metadata, redaction rules).
2. Choose the migration approach:
   - If OpenTelemetry is already in use, keep instrumentation and reconfigure the OTLP exporter to Laminar.
   - Otherwise, replace wrappers/decorators with Laminar `observe()` / `@observe()` or manual spans, and enable provider auto-instrumentation.
   - Avoid double-instrumentation.
3. Install Laminar with the repo's package manager. Add `LMNR_PROJECT_API_KEY` (and base URL if self-hosted) to env examples.
4. Map concepts and naming: stable span names, low-cardinality tags, IDs in metadata.
5. Verify: run a representative flow and confirm root span, child spans, and tags in the UI. Ensure trace context is preserved end-to-end.

## References

- `references/migration-mapping.md` for provider-to-Laminar concept mapping and minimal-diff patterns.
- `references/otel-exporter.md` for OTLP exporter setup guidance.
