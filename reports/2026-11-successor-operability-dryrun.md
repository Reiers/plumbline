# Successor-operability dry run

**Date:** 2026-11-02 · **Milestone:** M5(c), successor-operability checkpoint
**Runbook under test:** [RUNBOOK.md](../RUNBOOK.md) v1.1, updated to v1.2 by this exercise

## 1. Question

Could a second operator, with only the public repos and the access a
handover provides, stand up and recover every Plumbline service?

Docs in scope:
[plumbline](https://github.com/Reiers/plumbline) README, RUNBOOK, SLO;
[plumbline-faucet](https://github.com/Reiers/plumbline-faucet) README and
`docs/deploy.md`; [calix](https://github.com/Reiers/calix) README and
`deploy/`; [plumbline-monitor](https://github.com/Reiers/plumbline-monitor)
README and `collector/README.md`.

## 2. Answer

**Yes, after the fixes in this report.** With the docs as they stood on
the morning of 2026-11-02, a new operator would have got all three
services running, but only by working around a wrong Node.js install
step, an nginx setup the faucet docs did not describe, a status-page
permission problem, and a runbook that described a Calix layout and a
backup job that did not exist. Recovery from host loss was not possible
from the docs: there was no restore procedure and no backups to restore.

All gaps found are fixed in the docs (§5). A second, timed rebuild from
the corrected docs went through without improvising (§4).

## 3. Method

- Clean environment: a disposable Ubuntu 24.04 container with systemd,
  matching what a cloud provider hands out (systemd, ssh, curl, git,
  python3, sudo; no Node, nginx, jq or sqlite3). No production files or
  secrets were used.
- Hostnames: the public names (`faucet.`, `calix.`, `status.reiers.io`)
  were pointed at the container in its own `/etc/hosts`, and a
  throwaway local CA stood in for the Cloudflare Origin CA certificate.
  This runs the shipped nginx configs unmodified. Cloudflare itself was
  not exercised (§6).
- Faucet: three throwaway Calibration keys (dispenser, rotation target,
  recipient), Cloudflare's published Turnstile test keys, rate-limit
  values exactly as in `.env.example`. The dispenser was funded with
  one normal 1,000 tFIL public drip from <https://faucet.reiers.io>.
  No mainnet funds were involved.
- Calix was deployed with its own `deploy/deploy.sh` from a separate
  workstation, the way the README describes.
- Every step was timed; every place where the docs were wrong, silent,
  or assumed knowledge was logged as a gap.

## 4. Results

### 4.1 Stand-up from the docs as published (2026-11-02, times UTC)

| Step | Result | Time |
| --- | --- | --- |
| Faucet tooling per `deploy.md` | **Gap.** `apt-get install nodejs` gives Node 18 and no npm; faucet needs Node 20+. Switched to NodeSource Node 22. | 37 s + 35 s |
| Faucet user, clone, `pnpm install` | OK. `better-sqlite3` compiles from source. | 4 min 29 s |
| `.env` | **Gap.** Doc says create an empty file; `.env.example` was missing 5 variables the code reads. | 1 min |
| systemd unit | OK. `/healthz` up 6 s after start. | 10 s |
| Faucet TLS | **Gap.** `deploy.md` described Caddy, the runbook nginx; no faucet vhost in the repo. Wrote one. | 5 min |
| Dry drip (unfunded dispenser) | OK. `503 faucet_dry` for tFIL and USDFC, `400 captcha` without token, Turnstile test token accepted. | 3 s |
| Calix via `deploy.sh` | **Gap.** Prerequisites (nginx running, origin cert) unstated; the script's final check queried the production site, not the new host. Ran clean once nginx was up. | 8 s |
| Calix API | OK. `health`, `version`, `status` (head age 33 s), `signals`, `upgrade`, `finality`, `miners/status?addrs=t0143103`, dashboard, `/metrics`. | 1 min |
| Status page + collector per README | **Gap.** `/uptime.json` returned 403: `useradd --create-home` makes the data dir 0750 on Ubuntu 24.04. Fixed with `chmod 755`. | 6 s + 1 min |
| End-to-end drip from the new faucet | OK. 5 tFIL, `verified: true`, 57.7 s. | 58 s |

### 4.2 Recovery scenarios

| Scenario | Runbook section | Result |
| --- | --- | --- |
| Faucet process killed (SIGKILL) | §3.1 | systemd restarted it; `/healthz` back in 10.0 s. nginx returns 502 meanwhile. |
| Calix process killed | §4 | Back in 5.6 s. A clean `systemctl stop` is not restarted (`Restart=on-failure`), as intended. |
| Calix restart seen by the uptime collector | §4.4 | **Finding.** For about 7 s after start, `miners/status` reports the SP `down` with `epoch: 0`; one probe in that window recorded a false SP failure. Documented; code fix tracked separately. |
| Faucet DB lost, restore from backup | §8 | **Gap.** v1.1 listed a daily backup to `/var/backups/plumbline-faucet`, but no job for it was in any repo and there was no restore procedure. Added `ops/backup.sh` + timer; DB deleted, restored in 6 s, stats and API keys back. |
| API key issuance | §3.3 | **Gap.** `pnpm run keygen -- create` (as in the source comment) prints help under pnpm 10. Without `--` it works. Raw SQL path from v1.1 also worked. |
| Dispenser key rotation | `deploy.md` | **Gap.** Doc relied on Foundry (`cast`), which is not installed. Replaced with a viem one-liner using the faucet's own dependencies. Rotated, drained 983.99 tFIL old to new (63 s), drip from the new key with an API key `verified: true` (79 s). |
| Low reserve | §3.4 | OK. `503 faucet_dry` names the shortfall; `plumbline_faucet_low_reserve` flips to 1. **Gap:** `/healthz` stays `ok: true` at zero balance, so uptime alone does not catch it. Documented. |
| Upstream RPC failover | §3.2, §4.1 | **Gap.** v1.1 pointed at `/opt/calix/calix.env` and `CALIBNET_RPC_URL`; neither exists. Real setting is `CALIX_LOTUS_RPC`, now changed with a systemd drop-in. Tested against Ankr's public Calibration RPC: Calix fine except the finality panel; faucet balance reads fine. |
| Faucet update deploy | §3.5 | OK as written. |
| Calix update deploy | §4.4 | **Gap.** v1.1 described a source checkout and `go build` on the host; production has neither. Rewritten around `deploy.sh` from a workstation, with backup and rollback of the binary. |
| Host rebuild from scratch | §8.3 (new) | Ran on a second fresh container, from the corrected docs only, restoring `.env` and the latest DB backup. See §4.3. |
| nginx and Cloudflare origin | §7 | nginx side tested with a stand-in cert. Cloudflare steps (DNS, Full (strict), Origin CA issue) documented, not exercised: needs the account. |
| nv-upgrade checklist (tabletop) | §4.3 | **Gap.** A new network version touches four schedule tables in three Calix files, and the audit script needs SSH to a Lotus node with admin API. Neither was written down. Now listed step by step. |
| SP recovery (tabletop) | §5 | **Gap.** Every fix needs Curio cluster access that only the primary operator has. Now stated up front, with what a successor can do without it. |

### 4.3 Timed rebuild from the corrected docs

The first host was stopped to simulate its loss. A second fresh
container was rebuilt following RUNBOOK §8.3 and the corrected deploy
docs, with only the faucet `.env` and the latest DB backup carried over
(as they would come from the vault and the off-host backup copy).

| Step | Time (UTC) | Duration |
| --- | --- | --- |
| Base packages, Node 22, pnpm | 11:52:45 | 54 s |
| Faucet user, clone, `pnpm install` | 11:53:39 | 3 min 35 s |
| `.env` from vault, DB restore, systemd | 11:57:14 | 6 s |
| Origin cert, faucet vhost, backup timer | 11:57:20 | 5 s |
| Status page + collector | 11:57:25 | 3 s |
| Calix `deploy.sh` from a workstation | 11:57:34 | 9 s |
| All checks green | 11:58:05 | |
| Real drip from the rebuilt faucet | 11:59:07 | 62 s, `verified: true` |

**5 minutes 21 seconds** from an empty host to every service serving,
most of it the `better-sqlite3` compile. Restored state checked: same
dispenser address and balance, lifetime drip count (4), both API keys,
Calix on the deployed commit with head age 26 s, SP `ok`, status page
and `/uptime.json` 200. The first collector run fired before Calix was
deployed and logged two failures; §8.3 now deploys the status page
last. DNS cut-over is the remaining step on a real rebuild.

## 5. Gaps found and fixed

| # | Where | Gap | Fix |
| --- | --- | --- | --- |
| 1 | faucet `deploy.md` | Ubuntu `nodejs` is v18 without npm | NodeSource Node 22, pnpm 10 |
| 2 | faucet `deploy.md`, README | Caddy vs nginx conflict, no faucet vhost | `ops/plumbline-faucet.nginx`, nginx + Cloudflare section |
| 3 | faucet `.env.example` | `PUBLIC_FIL_DRIP`, `PUBLIC_USDFC_DRIP`, `MAX_PUBLIC_DRIPS_PER_IP`, `CALL_ACTOR_FORWARDER`, `LOG_LEVEL` missing; `HOST=0.0.0.0` | Added (code defaults); `HOST=127.0.0.1`; full `.env` reference in `deploy.md`. Rate-limit values unchanged. |
| 4 | faucet `deploy.md` | Verify step used the retired `/api/drip` and an old log format | Current endpoints and log line |
| 5 | faucet `deploy.md` | "Reset rate limits" deleted the whole DB, including API keys and stats | Targeted `DELETE` |
| 6 | faucet `deploy.md`, RUNBOOK | Key rotation needed Foundry; not in the runbook | viem snippet; RUNBOOK §3.6 |
| 7 | faucet `keygen.ts` comment | `--` before the subcommand breaks under pnpm | Corrected; RUNBOOK §3.3 uses the CLI |
| 8 | faucet | `MIN_RESERVE_*` must be whole numbers (parsed as integers for `/metrics`) | Documented |
| 9 | faucet README | Said native `t1`/`t3`/`t0` tFIL sends were unsupported; they are | Address table updated |
| 10 | RUNBOOK §8 | Backup job did not exist; no restore procedure | `ops/backup.sh` + daily timer in the faucet repo; §8.1 to §8.3 |
| 11 | RUNBOOK §2 | `status.reiers.io/api/summary.json` returns the HTML page; Calix health has no head-age field | `/uptime.json`; `/api/v1/status` `headAgeSeconds` |
| 12 | RUNBOOK §4.1, §4.4 | Calix env file, variable name, paths and build location wrong | Drop-in for `CALIX_LOTUS_RPC`; `deploy.sh` from a workstation |
| 13 | RUNBOOK §4.3 | nv checklist missed three of four places to edit and the Lotus admin requirement | Rewritten |
| 14 | RUNBOOK §0, §5, §7 | Access a successor needs was implicit; no origin-cert steps | §0.1 access table; §7 origin and Turnstile steps |
| 15 | calix README, `deploy.sh` | Prerequisites and three env vars undocumented; verify step hit production | Prerequisites, config table, `CALIX_VERIFY_URL` |
| 16 | monitor README | `/uptime.json` 403 on fresh Ubuntu 24.04; commands assumed the checkout as working directory | `chmod 755`; explicit `cd` |
| 17 | monitor collector | Probe URLs hard-coded, so it could not watch a rebuilt or staging host | `PLUMBLINE_FAUCET_URL`, `PLUMBLINE_CALIX_URL`, `PLUMBLINE_SP_URL` (defaults unchanged) |
| 18 | monitor README | Status-page edits made only on the host are lost in a rebuild | Push-to-GitHub rule; RUNBOOK §6.3 |

Fixes are in pull requests on each repo, branch
`docs/successor-dryrun-2026-11`.

## 6. Not covered

- **Cloudflare.** DNS records, SSL mode, Origin CA issuance and WAF rules
  need the account; the steps are written (RUNBOOK §7) but were not run.
- **SP recovery.** Tabletop only; no access to the Curio cluster.
- **nv-upgrade audit refresh.** Tabletop only; `refresh-audit.sh` needs
  a Lotus node with admin API.
- **USDFC.** Not funded; only the USDFC low-reserve path was exercised.
- **Drips through a backup RPC.** Ankr served reads; a drip through it
  was not attempted.

## 7. Follow-ups

- Install the backup timer and the updated faucet vhost on the
  production host, and start off-host backup copies.
- Calix: report "warming up" instead of `down` while the tipset ring is
  empty, and make `/api/v1/health` reflect ring freshness.
- Repeat this dry run after any change to how a service is deployed.
