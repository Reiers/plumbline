# Plumbline — Public Service-Level Objectives

**Version:** v1.1 · **Effective:** 2026-08-03 · **Revised:** 2026-10-06 · **Operator:** TSE Reiersen (Org. 929 074 912)

Plumbline is stability infrastructure for the Filecoin Calibration testnet. This
document is a public commitment: what we promise, how we measure it, and what
happens when we miss.

Live compliance is published at <https://status.reiers.io>. Monthly
realised numbers, incident post-mortems and reviews are published in
[reports/](./reports/). This document
tracks named targets; the status page tracks realised numbers over the
rolling 30-day window.

---

## 1. Scope

Three services, each with named SLOs:

| Service | URL | What it is |
| --- | --- | --- |
| **Plumbline Faucet** | <https://faucet.reiers.io> | Public tFIL + USDFC drip for Calibration builders and SPs. |
| **Calix** | <https://calix.reiers.io> | Real-time Calibration network stability console + nv-upgrade validation surface. |
| **SP test target** | `t0143103` | Long-running Storage Provider deliberately kept stable across nv boundaries. |

Services out of scope: private/internal tooling, one-off scripts, brand
assets, anything not listed above.

---

## 2. Service Level Indicators (SLIs)

Each SLI is a measurable, reproducible signal. The status page publishes the
raw numbers; this section defines them.

### 2.1 Faucet — drip latency

Time from a successful POST to `/api/drip/{fil,usdfc}` (captcha or API key
accepted) to the drip transaction being included in a Calibration tipset.

- Instrument: end-to-end server time of each successful drip request,
  taken from the faucet request log. The faucet responds only after it has
  seen the transaction receipt, so this is an upper bound on time to
  inclusion. Reported per month in [reports/](./reports/).
- Successful drips on the captcha and API-key paths only. Public
  (`/api/public/drip/*`, `claim_token_all`) drips, and requests rejected by
  captcha, rate limit, malformed input, or downstream RPC 5xx, are excluded.

### 2.2 Faucet — availability

Fraction of 5-minute probes in which `GET /healthz` returned HTTP 200 with
`{"ok": true}`.

- Probe: the Plumbline uptime collector
  ([source](https://github.com/Reiers/plumbline-monitor/tree/main/collector)),
  5-minute interval. Each failed probe counts as 5 minutes of downtime.
- Dispenser balances are tracked separately. Below the configured reserve
  (`MIN_RESERVE_FIL` / `MIN_RESERVE_USDFC`) the faucet refuses drips with
  HTTP 503 `faucet_dry`, and `plumbline_faucet_low_reserve` on `/metrics`
  flips to 1. Any such period is reported in the monthly report.
- Excluded: planned maintenance windows announced ≥24h in advance on the
  status page and in `#fil-net-calibration-discuss`.

### 2.3 Calix — availability

Fraction of 5-minute probes in which `GET /api/v1/health` returned HTTP
200 with `ok: true`. Same collector as §2.2. Chain-head freshness is shown
separately on the status page (head age signal).

### 2.4 Calix — nv-upgrade validation latency

Time from a Calibration `nv{N}` activation epoch to Calix publishing its
post-upgrade migration audit for that upgrade at
<https://calix.reiers.io/data/audit.json> (`networkVersion = N`,
`actors.match = true`). The audit cross-checks every actor code CID against
an independent `lotus state actor-cids` output and is not published if any
CID diverges. Timestamp: the publishing commit in
[Reiers/calix](https://github.com/Reiers/calix).

### 2.5 SP test-target availability

Fraction of 5-minute probes in which
`GET https://calix.reiers.io/api/v1/miners/status?addrs=t0143103` reports
the miner as `ok` (active in the Calibration state tree and producing
blocks). Same collector as §2.2. If Calix itself is unreachable, the
window counts as SP downtime too.

_Roster history: a second SP (`t0144416`) previously participated as a
redundant backup; it was retired 2026-07-15. If a second test target is
re-added, this SLI is re-defined against the roster as a whole._

---

## 3. Service Level Objectives (SLOs)

All SLOs are measured over a **rolling 30-day window**. Targets:

| SLO | Target | Window |
| --- | --- | --- |
| Faucet drip latency, tFIL, p99 | ≤ 60 seconds | 30 days |
| Faucet drip latency, USDFC, p99 | ≤ 90 seconds | 30 days |
| Faucet availability | ≥ 99.0% | 30 days |
| Calix availability | ≥ 99.0% | 30 days |
| Calix nv-validation latency | ≤ 24 hours from activation | per upgrade |
| SP test-target availability | ≥ 99.0% | 30 days |

The **error budget** for each 99.0% target is roughly 7h 12m of downtime
per rolling 30 days. Anything above that is a breach and triggers the
process below.

---

## 4. Public reporting

- **Live status page:** <https://status.reiers.io>. Publishes current SLO
  compliance for each SLI. Live signals refresh every 30 seconds; the
  90-day uptime history refreshes every 5 minutes. Incidents are posted on
  the page as they happen, each linking its post-mortem.
- **Monthly report:** a short markdown post at
  <https://github.com/Reiers/plumbline/tree/main/reports/> (index:
  [reports/README.md](./reports/README.md)) summarising
  the month's realised numbers, any SLO breaches, root-cause notes, and
  planned changes. Published within the first 5 working days of the
  following month.
- **Incident post-mortems:** any incident that consumes >25% of a
  30-day error budget in a single event gets a public write-up in
  `reports/incidents/YYYY-MM-DD-<slug>.md` within 5 working days.

---

## 5. Escalation on breach

If a live SLO is projected to breach its target inside the current
30-day window, in order:

1. **Investigate**: root cause, mitigation, ETA.
2. **Announce**: notice on the status page banner + a post to
   `#fil-net-calibration-discuss` on Filecoin Slack. Include: which SLO,
   current burn rate, planned mitigation.
3. **Mitigate**: apply the fix. Log actions in the runbook.
4. **Post-mortem**: publish per §4.

Contact for anything blocking: DM `@Reiers` on Filecoin Slack, or
`nicklas@reiers.io`.

---

## 6. What is deliberately not promised

- Anonymous public drips are best-effort. The `/api/public/drip/*`
  endpoints exist for OSS CLIs that cannot embed secrets; they are
  rate-limited more aggressively and are outside the SLO window.
- Custom-amount drips (larger than the default `filDrip` / `usdfcDrip`)
  are best-effort and handled out-of-band.
- Mainnet Plumbline features are experimental and out of scope. The
  mainnet console (calix-mainnet.reiers.io) is not covered here.
- Third-party dependencies (Cloudflare Turnstile, `glif.io` RPC, filfox)
  are not part of the target — if they are down, we mark the affected
  minute as degraded on the status page but do not count it against the
  budget when the downstream is provably at fault.

---

## 7. Change process

Changes to this document (new SLOs, adjusted targets, added services)
land as PRs against `github.com/Reiers/plumbline`. Each change is
reviewed against three questions:

1. Is the new commitment measurable with an SLI we already expose?
2. Are we currently meeting the proposed target on 30 days of history?
3. Would tightening this target require infrastructure we do not yet
   have funded?

Loosening a target requires a public note on the next monthly report
explaining why.

---

## 8. Machine-readable index

Endpoints used by the status page and by third-party monitors:

| SLI | Endpoint |
| --- | --- |
| Faucet up | `GET https://faucet.reiers.io/healthz` |
| Faucet stats | `GET https://faucet.reiers.io/api/stats` |
| Faucet metrics | `GET https://faucet.reiers.io/metrics` |
| Calix up | `GET https://calix.reiers.io/api/v1/health` |
| Calix status | `GET https://calix.reiers.io/api/v1/status` |
| Calix metrics | `GET https://calix.reiers.io/metrics` |
| SP roster | `GET https://calix.reiers.io/api/v1/miners/status?addrs=t0143103` |
| nv-upgrade audit | `GET https://calix.reiers.io/data/audit.json` |
| Uptime history (90 days) | `GET https://status.reiers.io/uptime.json` |

All endpoints return JSON except `/metrics`, which returns Prometheus
text-format.

---

_Change history is `git log SLO.md`. v1.1 (2026-10-06) aligned the SLI
measurement descriptions (§2) with the instrumentation actually running and
linked the published reports. No target in §3 changed._
