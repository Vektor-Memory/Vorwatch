# Vörwatch

**Vör's Watch** — lightweight, dependency-free VPS anomaly detection in a single bash script.

Named for Vör, the Old Norse goddess of vigilant awareness, described in the Prose Edda as "wise and inquiring, so that nothing can be concealed from her."

Vörwatch watches a Linux server for the signs that usually show up early in a compromise — new listening ports, changed critical files, first-seen outbound connections, suspicious process trees, SSH attempts from known-bad IPs, and nginx traffic patterns that look like scanning or a volumetric attack — and logs what it finds. Optionally, it can rate the reputation of your busiest visitors and email you when something urgent happens.

It does **not** ban, block, or auto-remediate anything. Every alert is a recommendation for you to review. That's a deliberate design choice, not a missing feature — see (#philosophy).

---

## Features

- **File integrity monitoring** — SHA-256 baseline of critical files (`sshd_config`, `passwd`, `shadow`, `crontab`, `nginx.conf`, `authorized_keys`), alerts on any change
- **Listening port baselining** — flags new ports that weren't there when you last captured a baseline
- **Outbound connection tracking** — alerts the first time your server talks to a new IP, cross-referenced against a public threat blocklist
- **Process tree anomaly detection** — flags suspicious parent/child pairings (e.g. nginx spawning a shell)
- **SSH auth cross-reference** — checks recent SSH connection attempts against the threat blocklist
- **nginx anomaly detection** — high-request-volume and 404-scanning detection per source IP, configurable thresholds
- **fail2ban + nginx stats** — bundled into one report
- **Optional: IP reputation scoring** — grades your top-5 nginx source IPs 1–5 for risk via [AbuseIPDB](https://www.abuseipdb.com) (bring your own free-tier key)
- **Optional: email notifications** — urgent alerts (blocklist hits, file tampering, attack patterns) send immediately; everything else lands in a scheduled digest, via [Resend](https://resend.com) (bring your own free-tier key)
- **Zero daemon, zero database** — one bash file, runs off cron, state lives in flat files
- **Text and JSON report output** — pipe the JSON output into whatever you already use for monitoring

## Philosophy

Vörwatch is recommend-only by design. It will never run `ufw deny`, never call `fail2ban-client banip`, never touch iptables. Every alert that suggests an action tells you the exact command to run yourself. This is intentional: automated banning based on heuristics has a real false-positive cost on a single production box, and the unknown risk of unattended firewall changes isn't worth the convenience for most single-server setups. If you want auto-remediation, this isn't that tool — pair it with something like [CrowdSec](https://crowdsec.net) instead.

## Requirements

- Linux with `bash`, coreutils, `iproute2` (`ss`), `procps` (`ps`), `curl`
- `sudo` access (several checks need root to read process/socket tables)
- Optional: `fail2ban-client` (for fail2ban stats in reports), `docker` (baseline captures container names if present), `journalctl` (SSH auth cross-reference)

Nothing else. No package manager dependencies, no runtime, no compiled binary.

## Install

```bash
git clone https://github.com/vektormemory/vorwatch.git
cd vorwatch
sudo bash install.sh
```

The installer walks you through setup: data directory, check interval, blocklist source, optional IP reputation key, optional email notifications. Press Enter on any prompt to accept the sensible default.

Prefer no prompts? `sudo bash install.sh --defaults` copies `vorwatch.conf.example` as-is — edit `/etc/vorwatch/vorwatch.conf` by hand afterward.

### npm

```bash
npm install -g vorwatch
sudo vorwatch-install
```

(The npm package is a thin wrapper around the same bash script — Linux only.)

## Quick start

```bash
vorwatch baseline     # capture current state as "known good"
vorwatch check         # run one detection pass, log findings
vorwatch install       # wire up the cron job(s)
vorwatch status         # confirm everything's live
vorwatch report today   # see what's happened
```

## Commands

| Command | Description |
|---|---|
| `vorwatch baseline` | Capture current state (ports, processes, file hashes) as "known good" |
| `vorwatch check` | Run all detection scenarios, log findings, send urgent emails if configured |
| `vorwatch report [--range R] [--json] [--email]` | Text or JSON report. `R` = `all` \| `today` \| `24h` \| `7d`. `--email` sends the report instead of printing it |
| `vorwatch status` | One-line health check of the monitor itself |
| `vorwatch install` | Install cron job(s) — check always installs; email digest installs only if configured |
| `vorwatch uninstall` | Remove cron job(s) |
| `vorwatch version` | Print version |
| `vorwatch help` | Full usage |

## Configuration

All settings live in `/etc/vorwatch/vorwatch.conf` — plain shell variable assignments, sourced directly. Anything unset falls back to a built-in default. See [`vorwatch.conf.example`](./vorwatch.conf.example) for the full annotated template.

| Variable | Default | Purpose |
|---|---|---|
| `VORWATCH_DIR` | `$HOME/.vorwatch` | Where baseline/alerts/state files live |
| `VORWATCH_BLOCKLIST_URL` | FireHOL level1 netset | Public threat intel feed (Spamhaus DROP + DShield aggregate, free, no key) |
| `VORWATCH_BLOCKLIST_AGE_MAX` | `86400` | Seconds before the cached blocklist re-downloads |
| `VORWATCH_CRON_SCHEDULE` | `*/15 * * * *` | How often `check` runs |
| `VORWATCH_CRITICAL_FILES` | see script | Space-separated list of files to hash and watch |
| `VORWATCH_NGINX_LOG` | `/var/log/nginx/access.log` | nginx access log path |
| `VORWATCH_NGINX_REQ_THRESHOLD` | `200` | Requests-per-IP threshold in the check window before flagging |
| `VORWATCH_NGINX_404_THRESHOLD` | `20` | Distinct-404s-per-IP threshold before flagging as a scan |
| `VORWATCH_NGINX_WINDOW_LINES` | `3000` | How many recent log lines each `check` scans |
| `VORWATCH_ABUSEIPDB_KEY` | unset | [AbuseIPDB](https://www.abuseipdb.com) key — enables 1–5 risk scoring on your nginx top-5 source IPs. Free tier: 1000 checks/day. Only called during `report`, never `check`, and results are file-cached for a week — see [IP reputation scoring](#ip-reputation-scoring) |
| `VORWATCH_IP_REP_CACHE_TTL_DAYS` | `7` | How long a cached reputation score is trusted before re-checking |
| `VORWATCH_EMAIL_TO` | unset | Comma-separated recipient list. Unset disables email entirely | 
| `VORWATCH_EMAIL_FROM` | `Vorwatch <hello@yourdomain.com>` | Sender address — its domain must be verified in your Resend account |
| `VORWATCH_RESEND_API_KEY` | unset | [Resend](https://resend.com) API key |
| `VORWATCH_EMAIL_FREQUENCY` | `monday` | `daily` or `monday` — controls the scheduled digest cadence |
| `VORWATCH_EMAIL_CRON` | derived from frequency | Raw cron override for the digest send time |
| `VORWATCH_EMAIL_URGENT_COOLDOWN_HOURS` | `4` | Minimum hours between repeat urgent emails of the same category |

## IP reputation scoring

Scoped deliberately narrow: only your nginx **top-5 source IPs by request count** get scored, and only when you run `vorwatch report` — never during the 15-minute `check` cron. That keeps API usage to at most 5 calls per manual report, and a 7-day file cache means repeat visitors (which is most of them — the same scanners hit you weekly) cost zero additional calls after the first lookup.

```
Top 5 source IPs by request count (risk 1-5, 5=critical, via AbuseIPDB):
    115.186.231.43       35 requests  [risk 1]
    3.99.128.211         17 requests  [risk 2]
    216.73.217.6          8 requests  [risk 5]
    34.56.201.30           5 requests  [risk 1]
    40.223.148.196          4 requests  [risk 1]
```

Note the risk score isn't just a function of volume — a low-traffic IP can still come back as high-risk if AbuseIPDB has real abuse reports against it, which raw request counts alone would never catch.

Get a free key at [abuseipdb.com](https://www.abuseipdb.com) (1000 checks/day, no credit card). Without a key configured, top-5 IPs just show as `unrated` — everything else in the report works normally.

## Email notifications

Bring your own [Resend](https://resend.com) account (free tier available, no credit card for low volume). You'll need a domain verified in Resend for the `From` address to deliver — Resend's dashboard walks you through the DNS records.

Two tracks:

- **Urgent** — blocklisted outbound connections, blocklisted SSH source IPs, critical file changes, and nginx volume/scan-pattern anomalies email immediately when `vorwatch check` finds them, subject to a per-category cooldown (default 4 hours) so a sustained attack sends one email per category per window, not one every 15 minutes.
- **Digest** — everything else (new listening ports, suspicious process trees, routine first-seen outbound IPs) lands in a scheduled report, sent weekly (Monday, configurable) or daily, your choice.

If email fails — bad key, network hiccup, whatever — it fails silently and never affects `check` or `report`'s own exit code. Monitoring itself never depends on the email path working.

## Sample report

```
============================================================
 Vörwatch — VPS Anomaly Detection — Text Report
 Generated: 2026-07-17 02:42:54 UTC
 Range: today
============================================================

-- High Priority (blocklist matches + integrity changes) --
  none

-- Alert Summary (today) --
  Total alerts:           12
  New listening ports:    0
  Critical file changed:  0
  First-seen outbound IP: 12
  Blocklisted outbound:   0
  Suspicious proc tree:   0
  Blocklisted SSH source: 0

-- System Vitals --
  Load average: 0.03, 0.03, 0.00
  Memory:       2407MB/23988MB used (10%)
  Disk (/):     29G/146G used (20%)
  Uptime:       up 9 weeks, 4 days

-- fail2ban --
  sshd                     total_failed=41605    currently_banned=0      total_banned=1487

-- nginx (current log file only) --
  Requests logged:        167
  Top 5 source IPs by request count (risk 1-5, 5=critical, via AbuseIPDB):
    115.186.231.43       32 requests  [risk 1]
  ...
```

High-priority findings are always at the top; the raw per-IP alert log is always at the bottom. Everything that matters most is closest to your eyes first.

## Log rotation

`install.sh` writes `/etc/logrotate.d/vorwatch` automatically, pointed at whatever `VORWATCH_DIR` you configured — weekly rotation, 8 weeks retained, `copytruncate` so the running check doesn't lose its file handle mid-write.

## Uninstalling

```bash
vorwatch uninstall        # removes cron job(s)
sudo rm /usr/local/bin/vorwatch /etc/logrotate.d/vorwatch
sudo rm -rf /etc/vorwatch ~/.vorwatch   # or wherever VORWATCH_DIR pointed
```

## Contributing

Issues and PRs welcome. A few things worth knowing before you dig in:

- No targeted `sed`/sudo patches on deployed installs — this is a single-file script, edit it whole
- New checks should follow the existing `alert()` pattern and stay recommend-only
- If a check might reasonably run every 15 minutes forever, make sure it's either genuinely idempotent (only fires on real state change) or has a dedup/cooldown mechanism — a couple of the early checks didn't and it's a known rough edge

## License

Apache License 2.0 — see [LICENSE](./LICENSE).
