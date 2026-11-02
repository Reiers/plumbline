# Plumbline: Operations Runbook

**Version:** v1.3 · **Last reviewed:** 2026-11-02 · **Operator:** TSE Reiersen

Practical playbook for keeping Plumbline services healthy. Structured as
**symptom → check → fix → verify**. Aim: any competent operator can bring
a service back from a hard incident in under 30 minutes with only what's
in this document and the access listed in §0.1.

v1.2 was checked by a successor dry run on 2026-11-02: every service was
stood up on a fresh Ubuntu 24.04 host from the public repos alone, and
the recovery steps below were executed there. Findings and timings:
[reports/2026-11-successor-operability-dryrun.md](./reports/2026-11-successor-operability-dryrun.md).

Companion doc: [SLO.md](./SLO.md) defines what "healthy" means. This
document defines how to get there.

---

## 0. On-call ownership

- **Primary:** Nicklas Reiersen (`@Reiers` on Filecoin Slack,
  `nicklas@reiers.io`).
- **Escalation for network-side issues:** `#fil-net-calibration-discuss`
  on Filecoin Slack.

There is no secondary on-call today. Extended unavailability is announced
on the status page ≥24h in advance and reflected as a maintenance
window (see [SLO.md §2.2](./SLO.md#22-faucet--availability)).

### 0.1 Access a second operator needs

Everything else is in the public repos. These are held by the primary
operator and handed over through the encrypted vault, never chat or email:

| Access | Needed for |
| --- | --- |
| Root SSH key for `<plumbline-host>` | Everything on the box |
| Cloudflare account with the `reiers.io` zone | DNS, origin certs, WAF rules (§7) |
| GitHub write access to `Reiers/plumbline*` and `Reiers/calix` | Deploys, docs |
| Faucet `.env` (contains `FAUCET_PK`, `TURNSTILE_SECRET`) | Faucet rebuild, key rotation |
| Off-host copies of the faucet DB backups (§8) | Restore after host loss |
| Turnstile site in the Cloudflare account | Captcha keys |
| Curio cluster web UI and SSH for `t0143103` | SP recovery (§5) |
| A Calibration Lotus node with admin API (SSH) | nv-upgrade audit (§4.3) |

Without the last two, a successor can still observe the SP (Calix,
Filfox) and announce on the status page, but cannot fix it.

---

## 1. Infrastructure map

| Component | Host | Path | systemd unit | Listens |
| --- | --- | --- | --- | --- |
| Plumbline Faucet | Hetzner Helsinki (`<plumbline-host>`) | `/opt/plumbline-faucet` (git checkout) | `plumbline-faucet.service` | `127.0.0.1:<PORT>` from `.env` |
| Faucet DB backup | same box | `/var/backups/plumbline-faucet` | `plumbline-faucet-backup.timer` | |
| Calix (calibration) | same box | `/opt/calix/calix-api` + `/opt/calix/web` (deployed by `deploy/deploy.sh`, not a checkout) | `calix.service` | `127.0.0.1:8765` |
| Plumbline Monitor (status page) | same box | `/opt/plumbline-monitor` (git checkout), nginx serves `web/` | none (static) | |
| Uptime collector | same box | `/opt/plumbline-monitor/collector`, data in `/var/lib/plumbline-monitor` | `plumbline-uptime.timer` (every 5 min) | |
| Mainnet preview faucet (not yet activated) | same box | `/opt/plumbline-faucet-mainnet` (second checkout, `NETWORK=mainnet`, user `faucet-mainnet`) | `plumbline-faucet-mainnet.service` | `127.0.0.1:8006`, served at `faucet.reiers.io/mainnet/` |
| SP `t0143103` | private cluster, dedicated node | n/a | see cluster docs | |

Nginx serves all three web-facing hosts behind Cloudflare (orange cloud).
Vhosts live in `/etc/nginx/sites-available/`, templates in each repo:
`plumbline-faucet/ops/plumbline-faucet.nginx`, `calix/deploy/calix.nginx`,
`plumbline-monitor/deploy/plumbline-monitor.nginx`. Origin certs are
Cloudflare Origin CA certs in `/etc/ssl/cloudflare/` (the wildcard
`reiers.io.crt`, plus a faucet-specific cert).

Host packages: `nginx`, `git`, `python3`, `sqlite3`, `jq`,
`build-essential`, Node.js 22 (NodeSource) with pnpm 10. Go is not
needed on the host; Calix is built on a workstation.

---

## 2. Fast-path health checks

Run these first on any incident. They take under a minute total.

```bash
# From anywhere on the internet:
curl -sS https://faucet.reiers.io/healthz | jq .
curl -sS https://calix.reiers.io/api/v1/status | jq '{level, headAgeSeconds, epoch}'
curl -sS https://status.reiers.io/uptime.json | jq '.generatedAt, (.components | map_values(.uptime_90d))'

# On the box:
ssh root@<plumbline-host>
systemctl status plumbline-faucet calix nginx plumbline-uptime.timer
journalctl -u plumbline-faucet -n 100 --no-pager
journalctl -u calix -n 100 --no-pager
```

`/healthz` returning `{ok: true}` + dispenser balances above reserve is
the canonical "faucet is fine" signal. Note `ok` stays `true` even at
zero balance; `plumbline_faucet_low_reserve` in `/metrics` is the
low-balance signal. `/api/v1/status` returning `headAgeSeconds < 300`
is the canonical "calix is fine" signal; `/api/v1/health` only says the
process is up.

---

## 3. Faucet: symptoms and fixes

### 3.1 Symptom: `/healthz` returns 502 or times out

1. **Check the service:**
   ```bash
   systemctl status plumbline-faucet
   journalctl -u plumbline-faucet -n 200 --no-pager
   ```
2. **If crashed:** systemd restarts it on its own (`Restart=always`,
   back within about 10 s in the dry run). If it keeps failing,
   `systemctl restart plumbline-faucet` and watch logs for 60 s. Common
   causes: bad `.env` after a config change (`Missing required env
   var`), `better-sqlite3` locking against a stale reader, out-of-disk on
   `/opt`.
3. **If OOM:** `dmesg | tail -50`. Faucet steady-state is <150MB RSS; if
   it grew past 500MB there's a leak. Restart, file an issue.

### 3.2 Symptom: drips succeed in the UI but tx never appears on-chain

1. **Check dispenser wallet balance:**
   ```bash
   curl -sS https://faucet.reiers.io/api/info | jq '{fil: .filBalanceWei, usdfc: .usdfcBalanceWei, minFil: .minReserveFil, minUsdfc: .minReserveUsdfc}'
   ```
   If either balance is below `minReserve*` + one drip, drips of that
   asset fail with `503 faucet_dry`. Refill (§3.4).
2. **Check upstream RPC:**
   ```bash
   curl -sS https://api.calibration.node.glif.io/rpc/v1 \
     -H 'content-type: application/json' \
     -d '{"jsonrpc":"2.0","method":"Filecoin.ChainHead","params":[],"id":1}' | jq .result.Height
   ```
   If Glif is down or lagging, set `RPC_URL` in
   `/opt/plumbline-faucet/.env` to another Calibration RPC and
   `systemctl restart plumbline-faucet`. `https://rpc.ankr.com/filecoin_testnet`
   answered `Filecoin.ChainHead` and served faucet balance reads in the
   2026-11 dry run (drips through it were not tested). A local Lotus
   also works.
3. **Check nonce / tx pool:** `journalctl -u plumbline-faucet | grep -i nonce`.
   Nonce mismatch after RPC-provider change requires one restart to resync.
4. A response with `"ok": true, "verified": false` usually means the
   balance re-read raced the RPC; check the tx hash on an explorer
   before assuming a loss.

### 3.3 Symptom: rate-limit denials for legitimate builders

Current limits are documented in
[reports/2026-10-onramp-hardening-review.md](./reports/2026-10-onramp-hardening-review.md).
A denied request returns HTTP 429 with `error` set to one of
`ip_rate_limited`, `address_rate_limited`, `public_ip_rate_limited` or
`api_key_rate_limited`, plus `retryAfterSec`. Check which one first.

State lives in `better-sqlite3` at `/opt/plumbline-faucet/rate-limits.sqlite`
(tables `ip_drips`, `address_drips`, `api_keys`, `api_key_drips`,
`recent_drips`, `stats`). Never edit while the service is running.

**Builder needs more than the captcha limits (CI, SDK test suites):** issue
an API key. Keys bypass Turnstile and replace the per-IP window with a
per-key quota. The CLI works while the faucet runs:

```bash
cd /opt/plumbline-faucet
sudo -u faucet -H pnpm run keygen create --name "<partner>-ci" --max-drips 200 --window-sec 86400 --notes "requested by ..."
sudo -u faucet -H pnpm run keygen list
sudo -u faucet -H pnpm run keygen revoke pk_...
```

No `--` before `create`: pnpm passes it through and the CLI prints its
help instead. Share the key over Signal or Slack DM, never email.

**Lift a per-address window early** (one-off, e.g. a builder hit the cap
during a demo):

```bash
systemctl stop plumbline-faucet
sqlite3 /opt/plumbline-faucet/rate-limits.sqlite \
  "DELETE FROM address_drips WHERE address='0x...' AND asset='fil';"
systemctl start plumbline-faucet
```

Addresses are stored lower-case.

**Suspected abuse** (one IP or address range draining the dispenser):

```bash
sqlite3 /opt/plumbline-faucet/rate-limits.sqlite \
  "SELECT address, asset, count(*) FROM recent_drips GROUP BY 1,2 ORDER BY 3 DESC LIMIT 20;"
journalctl -u plumbline-faucet --since '24 hours ago' -o cat | grep '"statusCode":429' | wc -l
```

If a single source is responsible, block it at Cloudflare (WAF custom
rule) rather than changing faucet-wide limits. The reserve floor
(`MIN_RESERVE_*`) guarantees the faucet stops with HTTP 503 `faucet_dry`
before the dispenser is emptied.

### 3.4 Symptom: dispenser wallet low reserve

**Check:** `curl -s https://faucet.reiers.io/metrics | grep low_reserve`
(`1` = below reserve), or drips returning `503 faucet_dry` with a
`reason` such as `dispenser tFIL low: 10 (reserve 20 + drip 5)`.

**tFIL top-up:**

- From a personal tFIL wallet with headroom, send to
  `0x1aB747560deFa40D27627AA507c6bd3680c28c6B`. Target refill: 5 million tFIL.
- Public fallbacks if personal is dry:
  - <https://faucet.calibnet.chainsafe-fil.io/funds.html> (1000 tFIL/24h)
  - Any other community Calibration faucet

**USDFC top-up:**

- Requires collateralising tFIL in a Trove at <https://stg.usdfc.net>.
- Steps: connect MetaMask to dispenser wallet → deposit tFIL → mint
  USDFC → confirm balance visible via `/api/info`.
- Target refill: 100 000 USDFC.
- If USDFC minting is broken (mainnet USDFC issues do cascade to
  Calibration occasionally), announce degraded USDFC on the status page
  and continue tFIL service.

Verify:

```bash
curl -sS https://faucet.reiers.io/api/info | jq '{fil: .filBalanceWei, usdfc: .usdfcBalanceWei}'
curl -sS https://faucet.reiers.io/metrics | grep low_reserve
```

Balances should be above min-reserve and `low_reserve` back to `0`. No
restart is needed; the faucet reads balances live.

### 3.5 Deploy a faucet update

```bash
ssh root@<plumbline-host>
cd /opt/plumbline-faucet
sudo -u faucet -H git pull
sudo -u faucet -H pnpm install --prod=false
systemctl restart plumbline-faucet
journalctl -u plumbline-faucet -n 50 --no-pager
```

Verify: `curl https://faucet.reiers.io/healthz` returns `ok:true` and
`/api/info` shows the expected balances.

### 3.6 Rotate the dispenser key

Do this if the key may have leaked, and on any change of operator.

1. Generate a new key and store it in the vault
   ([deploy guide](https://github.com/Reiers/plumbline-faucet/blob/main/docs/deploy.md#bootstrap-a-dispenser-wallet)
   has a one-liner that needs no extra tools).
2. Set the new `FAUCET_PK` in `/opt/plumbline-faucet/.env`, then
   `systemctl restart plumbline-faucet`.
3. Check `/healthz` shows the new dispenser address. Until it is funded,
   drips return `503 faucet_dry`.
4. Move the tFIL from the old key with the viem snippet in the
   [deploy guide](https://github.com/Reiers/plumbline-faucet/blob/main/docs/deploy.md#rotate-the-dispenser-key)
   (no Foundry needed). Move USDFC with an ERC-20 transfer, or mint
   fresh against the new key.
5. Update the dispenser address in §3.4 and on any public page that
   shows it.

Dry run (2026-11-02): restart to new key, drain transaction and first
drip from the new key took about 3 minutes, all on-chain waits.

---

## 4. Calix: symptoms and fixes

### 4.1 Symptom: `headAgeSeconds` climbing past 300

Calix reads through a Lotus RPC. If the RPC lags, chain-head age grows.

1. **Check the configured RPC:** `systemctl show calix -p Environment`
   (variable `CALIX_LOTUS_RPC`, set in `/etc/systemd/system/calix.service`).
2. **Test it directly:**
   ```bash
   curl -sS https://api.calibration.node.glif.io/rpc/v1 -H 'content-type: application/json' \
     -d '{"jsonrpc":"2.0","method":"Filecoin.ChainHead","params":[],"id":1}' | jq .result.Height
   ```
3. **If lagging:** flip to a healthy RPC with a drop-in, so the next
   deploy does not overwrite it:
   ```bash
   systemctl edit calix
   # [Service]
   # Environment=CALIX_LOTUS_RPC=https://rpc.ankr.com/filecoin_testnet
   systemctl restart calix
   ```
   Remove the drop-in (`systemctl revert calix`) when the primary is
   back. With Ankr the finality panel shows "unavailable" (no v2 API);
   everything else works.
4. **If all public RPCs are down:** the outage is upstream. Announce on
   the status page banner and accept the SLO burn.

### 4.2 Symptom: operational pill stuck on DEGRADED

Read `journalctl -u calix -n 200`. Common causes:

- **Manifest CID mismatch:** the manifest CID for the active nv differs
  from `canonicalManifestCIDs` in `api/main.go`. If a new nv has
  activated and the map is stale, update the constant and redeploy (§4.4).
- **Migration audit inconclusive:** if the audit has not completed by
  `migrationConfirmEpochs` (11 epochs) past activation, calix flags
  degraded. Check the audit output; often this needs a restart after
  Lotus itself catches up.

### 4.3 nv-upgrade validation window (last executed: nv29 Solstice, 2026-09-28)

Needs a Calibration Lotus node with admin API reachable over SSH
(§0.1). When a new nv is scheduled, one working day before the
activation epoch:

1. Confirm that Lotus node runs the release covering the new nv.
2. In one Calix commit, add the upgrade to every schedule table:
   `calibUpgrades` and `canonicalManifestCIDs` in `api/main.go`,
   `UPGRADES` in `web/index.html`, and both `case` tables
   (activation epoch, canonical manifest) in `scripts/refresh-audit.sh`.
   Take the epoch and manifest CID from the Lotus release notes and the
   builtin-actors release. Deploy (§4.4).
3. Watch the activation epoch on the status page. Migration audit runs
   `migrationConfirmEpochs` (~5.5 minutes) after activation.
4. Run `CALIX_REMOTE_SSH='ssh <lotus-node>' ./scripts/refresh-audit.sh <nv>`.
   Cross-check every actor code CID against an independent
   `lotus state actor-cids` output. The refresh script refuses to write
   `audit.json` if any CID diverges.
5. Deploy so `web/data/audit.json` goes live, then publish the audit
   result within 24h, see [SLO.md §2.4](./SLO.md#24-calix--nv-upgrade-validation-latency).

### 4.4 Deploy a calix update

Calix is a Go binary built from `github.com/Reiers/calix/api` on a
workstation (Go 1.22+, rsync, root SSH to the host). The host has no Go
toolchain and no Calix checkout.

```bash
git clone https://github.com/Reiers/calix && cd calix   # or git pull
ssh root@<plumbline-host> 'cp -p /opt/calix/calix-api /opt/calix/calix-api.bak-$(date +%Y%m%d)'
CALIX_HOST=root@<plumbline-host> ./deploy/deploy.sh
```

The script builds linux/amd64, rsyncs the binary and `web/`, installs
the unit and nginx vhost, restarts, and curls `/api/v1/health` and
`/api/v1/status`. It overwrites `/etc/systemd/system/calix.service`;
keep host-specific settings in a drop-in (§4.1).

Verify: `curl https://calix.reiers.io/api/v1/version` shows the new
commit and `/api/v1/status` a sane `headAgeSeconds`.

For about 10 seconds after any restart, `/api/v1/miners/status`
reports every miner as `down` with `epoch: 0` while the cache fills,
and an uptime probe in that window records a false SP failure. Restart
right after a probe: `systemctl list-timers plumbline-uptime.timer`.

Rollback: `cp -p /opt/calix/calix-api.bak-<date> /opt/calix/calix-api && systemctl restart calix`.

---

## 5. SP test targets: symptoms and fixes

Fixing the SP needs Curio cluster access (§0.1). Without it: confirm
with Calix and Filfox, announce on the status page, escalate to the
primary operator.

### 5.1 Symptom: SP misses a WindowPoST

- Check the miner's PoST scheduler in the local Curio dashboard.
- If it missed one window and recovered on the next, no action beyond
  logging.
- If it misses two consecutive windows, escalate: restart the specific
  Curio node responsible for that partition, verify sector state, file
  an issue on the private cluster docs.

### 5.2 Symptom: sector loss / fault ceiling exceeded

Do not try to fix at 3am. Announce degraded on the status page, gather
the state, and address in a scheduled window. Faulted sectors are
recoverable if caught within a proving deadline.

**Planned SP maintenance** (storage server, proving node, network): post
it on the status page and in `#fil-net-calibration-discuss` at least 24
hours before it starts, with the expected window. Only announced windows
are excluded from availability under [SLO.md §2.2](./SLO.md#22-faucet--availability).
Before the window, record the current storage paths (`curio cli storage list`)
so the post-maintenance check in §5.5 has something to compare against.

### 5.3 Symptom: miner stops producing blocks, Curio nodes look healthy

Observed 2026-10-01 ([post-mortem](./reports/incidents/2026-10-01-t0143103-block-production.md)).
Both Curio nodes were running but had lost their machine registration in
the cluster database, so the task scheduler could not claim WinningPoSt or
WindowPoSt work.

- **Check:** Calix shows t0143103 status not `ok` and `lastBlockAgeSec`
  climbing. In the Curio UI the machine list is empty or missing nodes, and
  WinningPoSt tasks queue without an owner.
- **Fix:** restart Curio on each node, one at a time. Each restart
  re-registers the machine. Expect a burst of stale election tasks that
  drains within about 15 minutes.
- **Verify:** blocks land again (Calix `blocksLast60` > 0), and after the
  next proving window `Filecoin.StateMinerSectorCount` shows `Faulty: 0`.
- If WindowPoSt was missed, faulted sectors recover automatically at the
  next proving window for that deadline. Confirm and note it on the status
  page.

### 5.4 Roster changes

Adding, removing, or replacing an SP is a public change. Update
`README.md` in this repo, note the change on the next monthly report,
and hold the previous SP in the roster until the new one has passed one
full PoST cycle. The SP address is also hardcoded in
`plumbline-monitor` (`web/app.js`, `web/index.html`, collector) and in
the Calix dashboard; update those in the same change.

### 5.5 Symptom: WindowPoSt fails after storage maintenance, sectors unreachable

Observed 2026-10-12 to 2026-10-16 ([post-mortem](./reports/incidents/2026-10-12-t0143103-storage-outage.md)).
The storage server came back with a different export path, the mount on
the proving node pointed at nothing, and a malformed Curio storage
configuration hid the problem.

- **Check:** Calix shows t0143103 not `ok`; Curio logs WindowPoSt errors
  about missing sector files. On the proving node, confirm the mount is
  read-write and at the expected path (`mount | grep -i calibration`,
  then list a known sealed sector as root). `curio cli storage list`
  should show the sealed-sector store with its full Sealed/Cache counts.
  Open `~/.curio/storage.json`: every path must be its own object, e.g.
  `{"StoragePaths":[{"Path":"/a/"},{"Path":"/b/"}]}`. Two `Path` keys
  in one object means only the last one is read.
- **Fix:** remount at the server's current export path. Back up and
  correct `storage.json`, restart Curio, and if the store is still
  missing from `storage list`, attach it explicitly:
  `curio cli --machine <node>:12300 storage attach <path>`.
- **Verify:** `storage list` shows the store with the expected counts,
  blocks land again (Calix `blocksLast60` > 0), and after the next
  proving windows `Filecoin.StateMinerSectorCount` shows `Faulty: 0`.
  Close the status-page entry and write the post-mortem if the outage
  passed about 1h 48m.

---

## 6. Status page: symptoms and fixes

`plumbline-monitor` is a static site at `/opt/plumbline-monitor/web`
served by nginx at `status.reiers.io`. The browser polls the endpoints
listed in [SLO.md §8](./SLO.md#8-machine-readable-index); the uptime bars
come from `/uptime.json`, written every 5 minutes by the collector.

### 6.0 Symptom: all three services fail at the same time

Observed 2026-08-24 ([post-mortem](./reports/incidents/2026-08-24-host-network-loss.md)).
The collector probes the public endpoints from the same host, so a host
network loss looks like faucet, Calix and SP all failing together.

- **Check:** from outside, `curl -sS https://faucet.reiers.io/healthz`.
  On the box, `journalctl --since '-30 min' | grep -c 'deadline exceeded\|i/o timeout'`.
  Many outbound timeouts plus no inbound faucet requests means network, not
  services.
- **Fix:** do not restart services. Check the hosting provider status page
  and Cloudflare status. Open a provider ticket if it lasts more than 15
  minutes.
- **Verify:** probes succeed again, then post the incident on the status page.

### 6.1 Symptom: page loads but all panels show "N/A"

- CORS regression on an upstream. Test the upstream endpoints directly.
  All are behind Cloudflare with `Access-Control-Allow-Origin: *`.
- If CORS was changed intentionally, add
  `status.reiers.io` to the allow-list on the affected service.

### 6.2 Symptom: uptime bars empty or stale

- `curl -sI https://status.reiers.io/uptime.json`. A 403 means nginx
  cannot read `/var/lib/plumbline-monitor`: `chmod 755` it (a fresh
  `useradd --create-home` makes it 0750).
- Stale `generatedAt`: `systemctl list-timers plumbline-uptime.timer`
  and `journalctl -u plumbline-uptime -n 20`. Run once by hand:
  `sudo -u plumbline-monitor python3 /opt/plumbline-monitor/collector/plumbline_uptime.py`.

### 6.3 Deploy a monitor update

```bash
ssh root@<plumbline-host>
cd /opt/plumbline-monitor
git pull
# nginx serves ./web/ directly; the collector timer uses the new script on its next run
```

Make status-page edits (maintenance banners, overrides) in git and push
them to GitHub. Edits that exist only on the host are lost in a rebuild.

Verify by loading <https://status.reiers.io> in a fresh browser tab.

---

## 7. Cloudflare + DNS

Needs the Cloudflare account (§0.1).

- Zone: `reiers.io` (Cloudflare Registrar).
- All faucet/calix/status DNS records are proxied (orange cloud) A
  records pointing at `<plumbline-host>`.
- SSL/TLS mode for the zone: Full (strict). Origin CA certs are only
  trusted by Cloudflare, so grey-clouding a host breaks TLS for visitors
  until a public cert is installed.
- Origin certs live at `/etc/ssl/cloudflare/` on the box; validity is 15
  years. If a cert is ever revoked or expires, re-issue via the Cloudflare
  dashboard → SSL/TLS → Origin Server → Create Certificate, cover
  `*.reiers.io` and `reiers.io`, save as `reiers.io.crt` /
  `reiers.io.key` (key mode 600), `nginx -t && systemctl reload nginx`.
- Turnstile: the faucet's site and secret key come from the same account
  (Turnstile → site → Settings). Rotating the secret means updating
  `TURNSTILE_SECRET` in the faucet `.env` and restarting.

**New origin host:** issue or copy the origin cert, install the three
vhosts (§1), then change the three A records to the new address. With
orange cloud, the switch takes effect within a minute or two.

If Cloudflare edge is degraded (rare), you can grey-cloud a hostname to
route traffic direct to origin as a temporary bypass. Announce the
change on the status page; the origin IP is now exposed, and visitors
see a certificate error unless a publicly trusted cert is installed.

---

## 8. Backups and restore

### 8.1 What is kept where

| What | Frequency | Where |
| --- | --- | --- |
| Faucet DB (rate limits, API keys, stats) | daily 03:00 UTC, `plumbline-faucet-backup.timer` | `/var/backups/plumbline-faucet/rate-limits-<UTC>.sqlite.gz` (7 days), plus an off-host copy pulled by the operator |
| Faucet `.env` | on change | operator's encrypted vault |
| Dispenser private key | at creation | encrypted vault; never on the box in plaintext except via `.env` |
| Calix binary + config | in git | `github.com/Reiers/calix`; host-only RPC drop-ins are listed in §4.1 |
| Status page | in git | `github.com/Reiers/plumbline-monitor` |
| Uptime history | not backed up | rebuilt from scratch; monthly reports in `reports/` are the durable record |

Install the backup job (once per host):

```bash
cd /opt/plumbline-faucet
cp ops/plumbline-faucet-backup.service ops/plumbline-faucet-backup.timer /etc/systemd/system/
systemctl daemon-reload
systemctl enable --now plumbline-faucet-backup.timer
systemctl start plumbline-faucet-backup.service && ls -l /var/backups/plumbline-faucet/
```

Off-host copy, from the operator's machine:
`rsync -a root@<plumbline-host>:/var/backups/plumbline-faucet/ <vault-backed dir>/`.

### 8.2 Restore the faucet DB

Use when the DB is lost or corrupt. Took 6 s in the dry run.

```bash
systemctl stop plumbline-faucet
cd /opt/plumbline-faucet
mkdir -p /root/faucet-db-pre-restore && mv rate-limits.sqlite* /root/faucet-db-pre-restore/
gunzip -c /var/backups/plumbline-faucet/rate-limits-<UTC>.sqlite.gz > rate-limits.sqlite
chown faucet:faucet rate-limits.sqlite && chmod 644 rate-limits.sqlite
sqlite3 rate-limits.sqlite 'PRAGMA integrity_check;'      # must print ok
systemctl start plumbline-faucet
```

Verify: `/api/stats` totals and `pnpm run keygen list` match what you
expect. Drips since the backup are missing from stats and cooldowns,
which is acceptable.

### 8.3 Rebuild the host from scratch

Order that worked in the dry run on a fresh Ubuntu 24.04 host (about 10
minutes of commands plus a 4 to 5 minute `better-sqlite3` compile):

1. Base packages and Node 22 + pnpm: faucet
   [deploy guide](https://github.com/Reiers/plumbline-faucet/blob/main/docs/deploy.md#install-on-the-box).
2. Origin cert to `/etc/ssl/cloudflare/` (§7).
3. Faucet: install per the deploy guide, put `.env` back from the vault,
   restore the latest off-host DB backup (§8.2), start, install the
   faucet vhost and the backup timer.
4. Calix: from a workstation, `CALIX_HOST=root@<new-host> ./deploy/deploy.sh`
   (§4.4). Re-apply any RPC drop-in.
5. Status page + collector:
   [plumbline-monitor README](https://github.com/Reiers/plumbline-monitor#deploy).
6. Point the DNS records at the new host (§7).
7. Run the §2 checks from outside, and one real drip.

Dispenser key restore is the sensitive step: use the vault, never
paste over Slack.

---

## 9. Metrics: where they live

Every service exposes Prometheus text-format at `/metrics`.

- Faucet: <https://faucet.reiers.io/metrics>
- Calix: <https://calix.reiers.io/metrics>

Scrape config example:

```yaml
- job_name: plumbline
  scrape_interval: 30s
  scheme: https
  static_configs:
    - targets:
        - faucet.reiers.io
        - calix.reiers.io
      labels:
        env: calibration
```

The status page uses the JSON endpoints (not `/metrics`) for public
consumption.

---

## 10. Change log for the runbook

- **v1.3 (2026-11-02):** Added §5.5 (sectors unreachable after storage
  maintenance, from the 2026-10-12 window) and the 24-hour announcement
  rule for SP maintenance in §5.2. Added the mainnet preview faucet
  (ships not yet activated) to the infrastructure map; its activation
  runbook is `plumbline-faucet/docs/mainnet-activation-runbook.md`.
- **v1.2 (2026-11-02):** Successor dry run. Added §0.1 (access a second
  operator needs), §3.6 (key rotation), §6.2 (uptime.json), §8.2 restore
  and §8.3 host rebuild. Fixed: status-page URL in §2 (`/uptime.json`),
  the Calix freshness signal (`/api/v1/status` `headAgeSeconds`), the
  Calix RPC setting (`CALIX_LOTUS_RPC` via drop-in, not a `calix.env`
  file), the Calix deploy (workstation `deploy.sh`, binary
  `/opt/calix/calix-api`), API keys via the `keygen` CLI, the backup job
  (now shipped as `ops/backup.sh` + timer), Cloudflare origin steps,
  infrastructure map with ports and data paths.
- **v1.1 (2026-10-06):** Removed retired SP t0144416 from the
  infrastructure map. Rewrote §3.3 for the API-key table, added abuse
  triage. Added §5.3 (Curio machine registration lost, from the
  2026-10-01 incident) and §6.0 (host network loss, from the 2026-08-24
  incident). nv-upgrade checklist updated after the nv29 run. Host
  address replaced with a placeholder.
- **v1.0 (2026-08-03):** Initial version. Covers Faucet, Calix, SP test
  targets, status page, DNS/Cloudflare, backups. Companion to SLO.md v1.0.

Update this file whenever a new failure mode is observed and resolved,
or when a service is added or removed from the surface. Re-run the
successor dry run after any major change to how a service is deployed.
