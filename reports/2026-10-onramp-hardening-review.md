# On-ramp hardening review: faucet + builder API

**Period reviewed:** 2026-08-01 to 2026-10-06 (first two months of builder usage under SLO v1.0)
**Published:** 2026-10-06 · **Milestone:** M4, on-ramp hardening

This is the public record of the rate-limit, abuse and documentation review
for the Plumbline Faucet (<https://faucet.reiers.io>) and its builder API
(<https://faucet.reiers.io/api>, source docs in
[plumbline-faucet/docs/api.md](https://github.com/Reiers/plumbline-faucet/blob/main/docs/api.md)).

## 1. Current limits

| Path | Who it is for | Amount per drip | Limit | Bot protection |
| --- | --- | --- | --- | --- |
| `POST /api/drip/{fil,usdfc}` (web UI) | Humans | 5,000 tFIL / 2,000 USDFC | 2 drips per asset per IP **and** per address per 24 h | Cloudflare Turnstile |
| `POST /api/drip/{fil,usdfc}` with API key | CI, SDK test suites, partners | 5,000 tFIL / 2,000 USDFC | Per-key quota, set per partner (100 to 1,000 per 24 h today) | Key replaces captcha |
| `/api/public/drip/*`, `claim_token_all` | OSS CLIs that cannot embed secrets | 1,000 tFIL / 100 USDFC | 1 drip per asset per IP per 24 h, shares the per-address window with the captcha path | None (smaller amounts instead), best-effort per SLO.md §6 |
| All paths | | | 5 requests/s global | Cloudflare in front |

Safety floor: the faucet refuses with HTTP 503 `faucet_dry` before the
dispenser falls below 50,000 tFIL or 20,000 USDFC, so a burst can never
empty it. Every 429 response names the scope that tripped
(`ip_rate_limited`, `address_rate_limited`, `public_ip_rate_limited`,
`api_key_rate_limited`) and includes `retryAfterSec`, so builders can see
exactly why they were limited and when to retry.

## 2. What real usage looked like

From the faucet request log, 2026-08-01 to 2026-10-06:

| Signal | Count |
| --- | --- |
| Drips sent (on-chain transactions) | 35: 12 tFIL (52,000 tFIL) and 23 USDFC (38,400 USDFC) |
| Drip requests (`POST`) | 33, from 15 distinct IPs (a `claim_token_all` request sends both assets) |
| Rate-limited (429) | 3 (2 captcha path, 1 public path) |
| Rejected as malformed or failed captcha (400) | 1 |
| Errors caused by the upstream public RPC | 2 (outside our control, SLO.md §6) |
| Busiest single IP | 21% of attempts (7), within limits |
| Unauthenticated `GET` to drip URLs (crawlers, links opened in a browser) | 23, all rejected with 404 |
| Dispenser below reserve floor | Never |

## 3. Abuse review

- No IP, address or API key came close to draining the dispenser. The
  busiest source made 7 attempts in two months.
- The 3 rate-limit hits were ordinary repeat requests inside the 24 h
  window, not automated abuse. Each got a clear 429 with a retry time.
- Crawler noise is limited to `GET` requests on drip URLs, which never
  reach the drip logic (drips require `POST`).
- Turnstile is doing its job: one failed captcha in two months, no
  sign of bypass attempts.

## 4. Decisions

- **Rate limits stay as they are.** Two drips per day of 5,000 tFIL is
  enough for normal SP and dApp testing. Teams that need more get an API key
  instead of a looser public limit. Usage did not justify loosening, and
  there was no abuse to justify tightening.
- **Public (no-captcha) path stays smaller and best-effort.** It exists for
  OSS CLIs. The smaller amount is the abuse control.
- **Abuse response is per-source, not faucet-wide.** A single bad source is
  blocked at Cloudflare. Global limits do not change for everyone because of
  one actor. Procedure: [RUNBOOK.md §3.3](../RUNBOOK.md#33-symptom-rate-limit-denials-for-legitimate-builders).

## 5. Documentation changes from this review

- RUNBOOK.md §3.3 rewritten to match how API keys are actually issued and
  revoked (database table, not environment variable), with an abuse-triage
  procedure.
- This document is the reference for current limits. RUNBOOK.md and the
  monthly reports link to it.
- Builder-facing docs:
  [faucet.reiers.io/api](https://faucet.reiers.io/api) and
  [docs/api.md](https://github.com/Reiers/plumbline-faucet/blob/main/docs/api.md)
  cover the endpoints, authentication, and every 429 scope with its
  retry timing.

## 6. Next review

Repeated with the December monthly report, or sooner if any 30-day window
has more than 10% of drip attempts rate-limited or the dispenser hits the
reserve floor.
