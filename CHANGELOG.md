# Changelog

## Fleet patches (appletree) — vs upstream main

### Added
- Runtime drain/undrain API: `POST /internal/endpoints/{name}/drain|undrain` — endpoint leaves rotation, health-checking continues, existing sticky pins preserved; no auto-undrain. (#3)
- Load-aware sticky pin placement: new conversations pin to least-loaded routable endpoint, ties via inner balancer; existing pins untouched. (#4)
- Metrics: `olla_sticky_pins{endpoint}` gauge; `sticky_key_source` logging + `X-Olla-Sticky-Key-Source` header. (#2/#4)
- `X-Olla-Endpoint` response header naming the serving backend. (#2)
- Root-level OpenAI-compatible proxy routes. (#1)

### Fixed
- Exact `Content-Length` on buffered request bodies. (#1)
- Dashboard route test accepts root-proxy fallback (503).
