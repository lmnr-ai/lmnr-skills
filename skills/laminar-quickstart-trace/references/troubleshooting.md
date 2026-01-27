# Quickstart troubleshooting

- No traces: verify `LMNR_PROJECT_API_KEY`, or pass `base_url`/`baseUrl` with the correct ports for self-hosted Laminar.
- Trace delayed: set `disable_batch`/`disableBatch` and call `Laminar.flush()`; keep the process alive for a second.
- Hard to find the trace: filter by tag `quickstart` or metadata key `run_id`.
- Self-hosted UI missing data: confirm backend services are running and reachable on HTTP/GRPC ports (defaults 8000/8001).
