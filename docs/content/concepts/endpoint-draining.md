---
title: "Endpoint Draining"
linkTitle: "Endpoint Draining"
weight: 25
description: "Remove an endpoint from load-balancing rotation at runtime — for upgrades, restarts and maintenance — without configuration edits or gateway restarts."
---

Olla supports runtime endpoint draining: take a backend out of the rotation,
do work on it, and bring it back — with zero configuration changes and zero
gateway downtime.

## The lifecycle

```
POST /internal/endpoints/{name}/drain    # stop routing NEW requests
   → bounce / upgrade / reconfigure the backend
   → verify health & load
POST /internal/endpoints/{name}/undrain  # rejoin rotation
```

- Draining is a **runtime flag** on the endpoint, not a status transition.
  Health checking continues while drained; the flag survives health-checker
  state copies.
- All load balancers (priority, least-connections, round-robin) and the
  sticky-session wrapper filter candidates through `Endpoint.RoutableNow()`.
- **Existing sticky pins are honoured** — in-flight pinned conversations keep
  working; only *new* pins route elsewhere. This preserves KV-cache affinity
  (see [Sticky Sessions]({{< relref "sticky-sessions" >}})) while draining.
- There is no auto-undrain: recovery is an explicit operator decision.

Example:

```bash
curl -X POST http://localhost:40114/internal/endpoints/gpu-a/drain \
     -d '{"reason":"spark 2.1 upgrade"}'
# ... restart gpu-a, watch /internal/status until healthy ...
curl -X POST http://localhost:40114/internal/endpoints/gpu-a/undrain
```

## Staggered fleet maintenance (canonical recipe)

For homogeneous multi-node inference rings (e.g. tensor-parallel clusters
behind one logical endpoint group), drain-bounce-verify-undrain one endpoint
at a time:

1. `drain` endpoint A — the LB shifts all new load to its peers.
2. Bounce A's engine with the new configuration/version.
3. Verify: health `200`, a real completion round-trips, error rate flat.
4. `undrain` A; confirm it accepts load (connections gauge rises).
5. Repeat for B, C, … never two at once, so the ring always has N−1 capacity.

Because drains are runtime state, the whole cycle needs **no config edits, no
comment-block toggling, no gateway restart** — the exact failure mode this
exists to avoid.

## Observability

- `GET /internal/status` includes per-endpoint drain state and reason.
- Prometheus: `olla_endpoint_connections{endpoint=...}` (in-flight requests —
  watch it fall to 0 while drained) and `olla_sticky_pins{endpoint=...}`
  (committed conversations — watch pins migrate on drain/undrain).
- Response header `X-Olla-Endpoint` names the serving backend, useful when
  proving where a request landed during a staggered bounce.
