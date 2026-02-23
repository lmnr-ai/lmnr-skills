# OTLP Exporter to Laminar (Guidance)

Use this when a codebase already emits OpenTelemetry spans and you want to keep
instrumentation but redirect export to Laminar.

## Checklist

1. Keep the existing OTEL instrumentation in place.
2. Configure the OTLP exporter to point at your Laminar instance.
3. Provide the Laminar API key via the exporter auth header/metadata.
4. Validate by running a representative flow and checking the trace in the UI.

## Notes

- Use the OTLP endpoint documented for your Laminar deployment (cloud vs self-hosted).
- Ensure the authorization header/metadata is set exactly as Laminar expects.
- If you see duplicate spans, remove any second tracer/auto-instrumentation.
