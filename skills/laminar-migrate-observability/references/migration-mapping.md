# Migration Mapping

Use this as a checklist to preserve semantics while switching to Laminar.

## Common goals

- Keep the same trace boundary (request/job/turn).
- Keep span names stable and low-cardinality.
- Put IDs (user, session, document) in metadata, not span names.
- Prefer Laminar’s trace-level `user_id` / `session_id` fields (separate from metadata) when you have those IDs.
- Keep tags low-cardinality (environment, feature flags, outcome).
- Avoid double-instrumentation.

## Langfuse

- Trace → Laminar trace
- Observation → Laminar span
- Tags → Laminar span tags
- Metadata → Laminar trace metadata

Migration pattern:
1. Replace Langfuse wrappers with `observe()` / `@observe()` or manual spans.
2. Set user/session IDs early in the trace so downstream spans inherit.
3. Preserve existing tags as Laminar tags and move IDs into metadata.

## LangSmith

- Run → Laminar trace
- Run ID → trace metadata
- Tags → Laminar span tags

Migration pattern:
1. Replace `@traceable` or run wrappers with Laminar `observe()` / `@observe()`.
2. Preserve run tags as span tags.
3. Store run IDs and inputs/outputs in metadata when needed.

## Helicone

Helicone relies on proxy headers rather than SDK spans. Migration typically means:

1. Remove Helicone proxy or middleware.
2. Initialize Laminar and enable provider auto-instrumentation.
3. Add manual spans for orchestration code to preserve structure.

## OpenTelemetry

If you already emit OTEL spans, keep instrumentation and swap the exporter to Laminar:

1. Keep existing spans and attributes.
2. Point the OTLP exporter to Laminar.
3. Set authorization metadata/headers for Laminar.

Use the OTEL exporter reference for exact header and endpoint details.
