# Function selection heuristics

## Prioritize

- Request/response boundaries (API handlers, queue consumers, cron jobs).
- Agent or workflow entrypoints and their major stages.
- External calls not already auto-instrumented (tool calls, proprietary APIs).
- Cross-service boundaries where context propagation matters.

## Avoid

- Low-level utilities called in tight loops.
- Small pure functions that add noise and little context.
- Per-item spans in large fanout loops (prefer a single span around the loop).

## Span design

- Start with one root span per user request or job.
- Add 2-6 child spans for the most important steps.
- Use tags for route, job name, model, or environment; use session_id/user_id for correlation.

## Privacy controls

- Ignore or format inputs/outputs that contain PII or secrets.
- Keep metadata small and JSON-serializable.
