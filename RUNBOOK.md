# Plumbline — Operations Runbook v1

**Version:** v1.0 · **Last reviewed:** 2026-08-03 · **Operator:** TSE Reiersen

Practical playbook for keeping Plumbline services healthy. Structured as
**symptom → check → fix → verify**. Aim: any competent operator can bring
a service back from a hard incident in under 30 minutes with only what's
in this document.

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

---

## 1. Infrastructure map

| Component | Host | Path | systemd unit |
| --- | --- | --- | --- |
| Plumbline Faucet | Hetzner Helsinki (157.180.16.39) | `/opt/plumbline-faucet` | `plumbline-faucet.service` |
| Calix (calibration) | same box | `/opt/calix` | `calix.service` |
| Plumbline Monitor (status page) | same box, static via nginx | `/opt/plumbline-monitor/web` | (static, no service) |
| SP `t0143103` | private cluster, dedicated node | — | see cluster docs |
| SP `t0144416` | private cluster, dedicated node | — | see cluster docs |

Nginx serves all three web-facing hosts behind Cloudflare (orange-cloud).
Origin cert is the wildcard `*.reiers.io` Cloudflare Origin CA cert at
`/etc/ssl/cloudflare/reiers.io.crt`.

---

## 2. Fast-path health checks

Run these first on any incident. They take under a minute total.

```bash
# From anywhere on the internet:
curl -sS https://faucet.reiers.io/healthz | jq .
curl -sS https://calix.reiers.io/api/v1/health | jq .
curl -sS https://status.reiers.io/api/summary.json | jq .

# On the box:
ssh root@157.180.16.39
systemctl status plumbline-faucet calix nginx
journalctl -u plumbline-faucet -n 100 --no-pager
journalctl -u calix -n 100 --no-pager
```

`/healthz` returning `{ok: true}` + non-zero dispenser balances is the
canonical "faucet is fine" signal. `/api/v1/health` returning
`chainHeadAgeSeconds < 300` is the canonical "calix is fine" signal.

---

## 3. Faucet — symptoms and fixes

### 3.1 Symptom: `/healthz` returns 502 or times out

1. **Check the service:**
   ```bash
   systemctl status plumbline-faucet
   journalctl -u plumbline-faucet -n 200 --no-pager
   ```
2. **If crashed:** `systemctl restart plumbline-faucet`. Watch logs for the
   next 60s. Common causes: bad `.env` after a config change, `better-sqlite3`
   locking against a stale reader, out-of-disk on `/opt`.
3. **If OOM:** `dmesg | tail -50`. Faucet steady-state is <150MB RSS; if
   it grew past 500MB there's a leak. Restart, file an issue.

### 3.2 Symptom: drips succeed in the UI but tx never appears on-chain

1. **Check dispenser wallet balance:**
   ```bash
   curl -sS https://faucet.reiers.io/api/info | jq '{fil: .filBalanceWei, usdfc: .usdfcBalanceWei, minFil: .minReserveFil, minUsdfc: .minReserveUsdfc}'
   ```
   If either balance is below `minReserve*`, drips of that asset are
   auto-disabled. Refill (§3.4).
2. **Check upstream RPC:**
   ```bash
   curl -sS https://api.calibration.node.glif.io/rpc/v1 \
     -H 'content-type: application/json' \
     -d '{"jsonrpc":"2.0","method":"Filecoin.ChainHead","params":[],"id":1}' | jq .result.Height
   ```
   If glif is down or lagging, switch `RPC_URL` in `/opt/plumbline-faucet/.env`
   to a backup (any Calibration RPC — chainup.io, ChainSafe, or a local
   Lotus).
3. **Check nonce / tx pool:** `journalctl -u plumbline-faucet | grep -i nonce`.
   Nonce mismatch after RPC-provider change requires one restart to resync.

### 3.3 Symptom: rate-limit denials for legitimate builders

The DB is `better-sqlite3` at `/opt/plumbline-faucet/rate-limits.sqlite`.
Never edit while the service is running.

To grant a bypass for a partner:

1. Issue an API key. Add to `.env`:
   ```
   API_KEYS_JSON=[{"name":"synapS3-ci","key":"<32-byte hex>","enabled":true,"limitFil":200,"limitUsdfc":200}]
   ```
2. `systemctl restart plumbline-faucet`.
3. Share the key over Signal or Slack DM — never email.

Per-address caps still apply. To lift a per-address cap, edit the DB
offline:

```bash
systemctl stop plumbline-faucet
sqlite3 /opt/plumbline-faucet/rate-limits.sqlite \
  "DELETE FROM address_limits WHERE address='0x...' AND asset='fil';"
systemctl start plumbline-faucet
```

### 3.4 Symptom: dispenser wallet low reserve

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
```

Balances should be above min-reserve and the healthz endpoint should
report the updated amount.

### 3.5 Deploy a faucet update

```bash
ssh root@157.180.16.39
cd /opt/plumbline-faucet
sudo -u faucet -H git pull
sudo -u faucet -H pnpm install --prod=false
systemctl restart plumbline-faucet
journalctl -u plumbline-faucet -n 50 --no-pager
```

Verify: `curl https://faucet.reiers.io/healthz` returns `ok:true` and
`/api/info` shows the expected balances.

---

## 4. Calix — symptoms and fixes

### 4.1 Symptom: `chainHeadAgeSeconds` climbing past 300

Calix reads through a Lotus RPC. If the RPC lags, chain-head age grows.

1. **Check the configured RPC** in `/opt/calix/calix.env` (variable
   `CALIBNET_RPC_URL`).
2. **Test it directly:**
   ```bash
   curl -sS $CALIBNET_RPC_URL -H 'content-type: application/json' \
     -d '{"jsonrpc":"2.0","method":"Filecoin.ChainHead","params":[],"id":1}' | jq .result.Height
   ```
3. **If lagging:** flip to a healthy RPC, `systemctl restart calix`.
4. **If all public RPCs are down:** the outage is upstream. Announce on
   the status page banner and accept the SLO burn.

### 4.2 Symptom: operational pill stuck on DEGRADED

Read `journalctl -u calix -n 200`. Common causes:

- **Manifest CID mismatch:** the local Lotus's manifest CID for the
  active nv differs from `canonicalManifestCIDs` in the calix source. If
  a new nv has activated and the map is stale, update the constant and
  redeploy (§4.4).
- **Migration audit inconclusive:** if the audit has not completed by
  `migrationConfirmEpochs` past activation, calix flags degraded. Check
  the audit output; often this needs a restart after Lotus itself catches
  up.

### 4.3 Symptom: nv-upgrade validation window

When a new nv is scheduled, one working day before the activation epoch:

1. Confirm the local Lotus is running the release candidate covering the
   new nv.
2. Confirm `canonicalManifestCIDs[<nv>]` in calix source matches the
   expected manifest for the new actors bundle.
3. Watch the activation epoch on the status page. Migration audit runs
   `migrationConfirmEpochs` (~5.5 minutes) after activation.
4. Publish the audit result within 24h — see [SLO.md §2.4](./SLO.md#24-calix--nv-upgrade-validation-latency).

### 4.4 Deploy a calix update

Calix is a Go binary built from `github.com/Reiers/calix/api`.

```bash
ssh root@157.180.16.39
cd /opt/calix/src   # git checkout of Reiers/calix
git pull
cd api
go build -o /opt/calix/bin/calix .
systemctl restart calix
journalctl -u calix -n 50 --no-pager
```

Verify: `curl https://calix.reiers.io/api/v1/health` and
`/api/v1/version` return sane values.

---

## 5. SP test targets — symptoms and fixes

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

### 5.3 Roster changes

Adding, removing, or replacing an SP is a public change. Update
`README.md` in this repo, note the change on the next monthly report,
and hold the previous SP in the roster until the new one has passed one
full PoST cycle.

---

## 6. Status page — symptoms and fixes

`plumbline-monitor` is a static site at `/opt/plumbline-monitor/web`
served by nginx at `status.reiers.io`. It polls the endpoints listed in
[SLO.md §8](./SLO.md#8-machine-readable-index) client-side.

### 6.1 Symptom: page loads but all panels show "N/A"

- CORS regression on an upstream. Test the upstream endpoints directly.
  All are behind Cloudflare with `Access-Control-Allow-Origin: *`.
- If CORS was changed intentionally, add
  `status.reiers.io` to the allow-list on the affected service.

### 6.2 Deploy a monitor update

```bash
ssh root@157.180.16.39
cd /opt/plumbline-monitor
git pull
# nothing else — nginx serves ./web/ directly
```

Verify by loading <https://status.reiers.io> in a fresh browser tab.

---

## 7. Cloudflare + DNS

- Zone: `reiers.io` (Cloudflare Registrar).
- All faucet/calix/status DNS records are proxied (orange cloud).
- Origin certs live at `/etc/ssl/cloudflare/` on the box; validity is 15
  years. If a cert is ever revoked or expires, re-issue via the Cloudflare
  dashboard → SSL/TLS → Origin Server → Create Certificate.

If Cloudflare edge is degraded (rare), you can grey-cloud a hostname to
route traffic direct to origin as a temporary bypass. Announce the
change on the status page — origin IP is now exposed.

---

## 8. Backups

| What | Frequency | Where |
| --- | --- | --- |
| Faucet rate-limit DB | daily 03:00 UTC | `/var/backups/plumbline-faucet/*.sqlite.gz` (7-day rotation) |
| Faucet `.env` | on change | encrypted vault (workspace-local `.vault/`) |
| Dispenser private key | at creation | encrypted vault; never on the box in plaintext except via `.env` |
| Calix binary + config | in git | `github.com/Reiers/calix` |
| Status page | in git | `github.com/Reiers/plumbline-monitor` |

Rebuilding faucet from scratch on a new box: follow
[`plumbline-faucet/docs/deploy.md`](https://github.com/Reiers/plumbline-faucet/blob/main/docs/deploy.md).
Dispenser key restore is the sensitive step — use the vault, never
paste over Slack.

---

## 9. Metrics — where they live

Every service exposes Prometheus text-format at `/metrics`.

- Faucet: <https://faucet.reiers.io/metrics>
- Calix: <https://calix.reiers.io/metrics>

Scrape config example:

```yaml
- job_name: plumbline
  scrape_interval: 30s
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

- **v1.0 (2026-08-03):** Initial version. Covers Faucet, Calix, SP test
  targets, status page, DNS/Cloudflare, backups. Companion to SLO.md v1.0.

Update this file whenever a new failure mode is observed and resolved,
or when a service is added or removed from the surface.
