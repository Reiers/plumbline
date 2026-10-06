# Post-mortem: host network loss, 2026-08-24

**Status:** Resolved · **Published:** 2026-10-06 (late, required by SLO.md §4 within 5 working days)

## Impact

- **Window:** 2026-08-24 10:00 to 15:10 UTC, about 5h 10m.
- **Faucet:** unavailable. Zero requests reached the faucet during the window.
- **Calix:** unavailable to the public. Internally it served stale data, as its upstream RPC calls were also timing out.
- **Status page:** unavailable (same host).
- **SP test target t0143103:** not affected as far as we can tell, but unobservable. Its availability is measured through Calix, so the window is counted as downtime against the SP SLO too.
- **Error budget:** about 72% of the 30-day budget for the faucet and Calix in one event. No SLO breach. The worst rolling 30-day window ended at 74% of budget.

## Timeline (UTC)

- 09:55: last fully successful probe round.
- 10:00: outbound requests from the host start timing out (Calix upstream RPC, the uptime collector's probes of the public endpoints, other services on the box).
- 10:00 to 15:05: no inbound requests reach the faucet. Every collector probe times out.
- 15:05 to 15:10: connectivity returns and probes succeed again. No restart or operator action was needed.

## Root cause

Loss of network connectivity at the hosting box, both inbound and outbound. The host did not reboot and the services kept running, which points to the network path rather than the services. We could not pin down whether the fault was at the hosting provider or the edge in front of it.

## What went wrong in our response

- The incident was not posted to the status page while it was happening, and this post-mortem was not published within 5 working days as SLO.md requires.
- Every probe runs from the same box it is measuring, so a network loss looks the same as all three services failing together, and nothing alerts anyone off-box.

## Follow-ups

1. Add an external probe from a second network, with alerting to the operator. *(open)*
2. Post incidents to the status page as they happen. *(process, adopted from 2026-10-01 onward)*
