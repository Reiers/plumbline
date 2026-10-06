# Plumbline reports

Public operations record for Plumbline, as committed in [SLO.md §4](../SLO.md#4-public-reporting).

## Monthly operations reports

| Month | Report | Headline |
| --- | --- | --- |
| September 2026 | [2026-09.md](./2026-09.md) | nv29 Solstice validated 37 min after activation. All availability SLOs met. |
| August 2026 | [2026-08.md](./2026-08.md) | First month under SLO v1.0. All availability SLOs met despite a 5h host network loss. |

## Incident post-mortems

Required for any single event consuming more than 25% of a 30-day error budget (about 1h 48m against a 99.0% target).

| Date | Post-mortem | Services |
| --- | --- | --- |
| 2026-10-01 | [t0143103 stopped producing blocks](./incidents/2026-10-01-t0143103-block-production.md) | SP test target |
| 2026-08-24 | [Host network loss](./incidents/2026-08-24-host-network-loss.md) | Faucet, Calix (SP unobservable) |

## How the numbers are produced

- **Availability** comes from the uptime collector in [plumbline-monitor](https://github.com/Reiers/plumbline-monitor/tree/main/collector): one probe per service every 5 minutes against the public endpoints listed in SLO.md §8. A month's availability is successful probes divided by total probes. Each failed probe counts as 5 minutes of downtime.
- **Drip latency** is currently measured as server-side response time of successful `POST /api/drip/{fil,usdfc}` requests. The faucet only responds after it has seen the transaction receipt, so this is an upper bound on time-to-inclusion. The dedicated histogram described in SLO.md §2.1 is not exposed yet (see planned changes in each report).
- **nv-validation latency** is the time from the Calibration activation epoch to the Calix post-upgrade audit going live.
- Calendar-month figures are reported here. The SLO itself is evaluated over a rolling 30-day window; where that differs materially it is called out.
