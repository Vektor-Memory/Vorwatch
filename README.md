# Vörwatch

**Vör's Watch** — lightweight, dependency-free VPS anomaly detection in a single bash script.

Named for Vör, the Old Norse goddess of vigilant awareness, described in the Prose Edda as "wise and inquiring, so that nothing can be concealed from her."

Vörwatch watches a Linux server for the signs that usually show up early in a compromise — new listening ports, changed critical files, first-seen outbound connections, suspicious process trees, SSH attempts from known-bad IPs, nginx traffic patterns that look like scanning or a volumetric attack, vulnerable installed packages, rootkit/backdoor signatures, CIS-style hardening drift, and first-seen outbound DNS queries — and logs what it finds. Optionally, it can rate the reputation of your busiest visitors and email you when something urgent happens.

It does **not** ban, block, or auto-remediate anything. Every alert is a recommendation for you to review. That's a deliberate design choice, not a missing feature — see (#philosophy).

<img width="1074" height="530" alt="Screenshot 2026-07-17 183435" src="https://github.com/user-attachments/assets/7cef88c5-a40b-44e5-a814-895bca7bcffc" />


---

## Features

- **File integrity monitoring** — SHA-256 baseline of critical files (`sshd_config`, `passwd`, `shadow`, `crontab`, `nginx.conf`, `authorized_keys`), alerts on any change
- **Listening port baselining** — flags new ports that weren't there when you last captured a baseline
- **Outbound connection tracking** — alerts the first time your server talks to a new IP, cross-referenced against a public threat blocklist
- **Process tree anomaly detection** — flags suspicious parent/child pairings (e.g. nginx spawning a shell)
- **SSH auth cross-reference** — checks recent SSH connection attempts against the threat blocklist
- **nginx anomaly detection** — high-request-volume and 404-scanning detection per source IP, configurable thresholds
- **fail2ban + nginx stats** — bundled into one report
- **Package vulnerability scanning** — checks your installed package list against [OSV.dev](https://osv.dev)'s free vulnerability database in one batched call, no API key needed. Report output is capped (top packages by CVE count, top CVE IDs per package) so an older box with hundreds of historical findings doesn't blow out the report — full detail always lives in the cache file
- **Rootkit / backdoor scanning** — shells out to `chkrootkit` or `rkhunter` if either is installed, rate-limited independently of the check cadence
- **CIS-style hardening spot-checks** — sshd config (root login, password auth) and critical file permissions (`/etc/shadow`, `/etc/passwd`), only re-alerts when findings actually change
- **DNS query anomaly detection** *(optional, off by default)* — first-seen queried domain tracking, same pattern as outbound IPs, if you point it at a resolver log
- **Optional: IP reputation scoring** — grades your top-5 nginx source IPs 1–5 for risk via [AbuseIPDB](https://www.abuseipdb.com) (bring your own free-tier key)
- **Optional: email notifications** — urgent alerts (blocklist hits, file tampering, attack patterns, rootkit hits) send immediately; everything else lands in a scheduled digest, via [Resend](https://resend.com) (bring your own free-tier key)
- **Zero daemon, zero database** — one bash file, runs off cron, state lives in flat files
- **Text and JSON report output** — pipe the JSON output into whatever you already use for monitoring

## Philosophy

Vörwatch is recommend-only by design. It will never run `ufw deny`, never call `fail2ban-client banip`, never touch iptables. Every alert that suggests an action tells you the exact command to run yourself. This is intentional: automated banning based on heuristics has a real false-positive cost on a single production box, and the unknown risk of unattended firewall changes isn't worth the convenience for most single-server setups. If you want auto-remediation, this isn't that tool — pair it with something like [CrowdSec](https://crowdsec.net) instead.

## Requirements

- Linux with `bash`, coreutils, `iproute2` (`ss`), `procps` (`ps`), `curl`
- `sudo` access (several checks need root to read process/socket tables)
- Optional: `fail2ban-client` (for fail2ban stats in reports), `docker` (baseline captures container names if present), `journalctl` (SSH auth cross-reference), `python3` (package vulnerability scanning), `chkrootkit` or `rkhunter` (rootkit scanning)

Nothing else required for the core checks. No package manager dependencies, no runtime, no compiled binary — the optional checks above degrade gracefully (skip themselves) if their tool isn't installed.

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
vorwatch baseline       # capture current state as "known good"
vorwatch check          # run one detection pass, log findings
vorwatch install        # wire up the cron job(s)
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
| `VORWATCH_EMAIL_FROM` | `Vorwatch <alerts@yourdomain.com>` | Sender address — its domain must be verified in your Resend account |
| `VORWATCH_RESEND_API_KEY` | unset | [Resend](https://resend.com) API key |
| `VORWATCH_EMAIL_FREQUENCY` | `monday` | `daily` or `monday` — controls the scheduled digest cadence |
| `VORWATCH_EMAIL_CRON` | derived from frequency | Raw cron override for the digest send time |
| `VORWATCH_EMAIL_URGENT_COOLDOWN_HOURS` | `4` | Minimum hours between repeat urgent emails of the same category |
| `VORWATCH_VULN_SCAN` | `true` | Package vulnerability scanning via [OSV.dev](https://osv.dev), needs `python3`, no API key |
| `VORWATCH_VULN_CACHE_TTL_HOURS` | `24` | How often to re-scan (OSV.dev is queried once per TTL window, not per check) |
| `VORWATCH_VULN_REPORT_MAX` | `15` | Max packages shown in the report before summarizing the rest |
| `VORWATCH_VULN_IDS_PER_PKG` | `8` | Max CVE/USN IDs shown per package line before summarizing |
| `VORWATCH_ROOTKIT_SCAN` | `true` | Uses `chkrootkit` or `rkhunter` if installed, skipped entirely if neither is present |
| `VORWATCH_ROOTKIT_SCAN_HOURS` | `24` | Minimum hours between rootkit scans, independent of the check cadence |
| `VORWATCH_CIS_CHECKS` | `true` | sshd hardening + critical file permission spot-checks |
| `VORWATCH_DNS_LOG` | unset | Path to a resolver log (dnsmasq/systemd-resolved) — enables first-seen DNS query tracking |
| `VORWATCH_DNS_WINDOW_LINES` | `2000` | How many recent DNS log lines each `check` scans |

## IP reputation scoring

Scoped deliberately narrow: only your nginx **top-5 source IPs by request count** get scored, and only when you run `vorwatch report` — never during the 15-minute `check` cron. That keeps API usage to at most 5 calls per manual report, and a 7-day file cache means repeat visitors (which is most of them — the same scanners hit you weekly) cost zero additional calls after the first lookup.

```
Top 5 source IPs by request count (risk 1-5, 5=critical, via AbuseIPDB):
    115.186.231.43         35 requests  [risk 1]
    3.99.128.211           17 requests  [risk 2]
    216.73.217.6           8 requests  [risk 5]
    34.56.201.30           5 requests  [risk 1]
    40.223.148.196         4 requests  [risk 1]
```

Note the risk score isn't just a function of volume — a low-traffic IP can still come back as high-risk if AbuseIPDB has real abuse reports against it, which raw request counts alone would never catch.

Get a free key at [abuseipdb.com](https://www.abuseipdb.com) (1000 checks/day, no credit card). Without a key configured, top-5 IPs just show as `unrated` — everything else in the report works normally.

## Package vulnerability scanning

Scans your installed package list against [OSV.dev](https://osv.dev)'s free vulnerability database in a single batched HTTP call — not one call per package. Needs `python3` (used for correct JSON parsing, not bash regex against nested data); skipped entirely if it's not present. Results are cached for `VORWATCH_VULN_CACHE_TTL_HOURS` (default 24h), so repeat runs don't re-hit the API.

OSV.dev returns every historical CVE/USN ever filed against a package version, including old, low-severity, or already-patched-elsewhere entries — on an older or heavily-packaged box that can mean dozens of packages with hundreds of IDs each. The report caps this at both levels: `VORWATCH_VULN_REPORT_MAX` (default 15) packages shown, sorted by CVE count, with a "N more not shown" footer; and `VORWATCH_VULN_IDS_PER_PKG` (default 8) CVE IDs per package line, with a "(+N more)" suffix. The full, uncapped list always lives in the cache file (`vuln-cache.txt` in your `VORWATCH_DIR`). Alerts are a single summary line per scan ("N packages have known vulnerabilities"), not one alert per package.

## Rootkit / backdoor scanning

Shells out to `chkrootkit` or `rkhunter` if either is installed — a no-op otherwise. Rate-limited independently of the 15-minute check cron via `VORWATCH_ROOTKIT_SCAN_HOURS` (default 24h), since a filesystem-wide scan is much heavier than everything else `vorwatch check` does. Any hit is treated as urgent: it's included in the "High Priority" section at the top of every report and, if email is configured, sent immediately.

## CIS-style hardening spot-checks

Not a full CIS benchmark suite — a handful of cheap, high-value checks: `PermitRootLogin` and `PasswordAuthentication` in `sshd_config`, and permissions on `/etc/shadow` and `/etc/passwd`. Only re-alerts when the set of findings actually changes, so a known, unfixed issue doesn't re-fire every 15 minutes forever.

## DNS query anomaly detection

Off by default — not every box runs a local resolver that logs queries. Point `VORWATCH_DNS_LOG` at a resolver log (dnsmasq or systemd-resolved) to enable first-seen domain tracking, the same pattern used for first-seen outbound IPs.

## Email notifications

Bring your own [Resend](https://resend.com) account (free tier available, no credit card for low volume). You'll need a domain verified in Resend for the `From` address to deliver — Resend's dashboard walks you through the DNS records.

Two tracks:

- **Urgent** — blocklisted outbound connections, blocklisted SSH source IPs, critical file changes, nginx volume/scan-pattern anomalies, and rootkit hits email immediately when `vorwatch check` finds them, subject to a per-category cooldown (default 4 hours) so a sustained attack sends one email per category per window, not one every 15 minutes.
- **Digest** — everything else (new listening ports, suspicious process trees, routine first-seen outbound IPs, CIS findings, vulnerable packages, first-seen DNS queries) lands in a scheduled report, sent weekly (Monday, configurable) or daily, your choice.

If email fails — bad key, network hiccup, whatever — it fails silently and never affects `check` or `report`'s own exit code. Monitoring itself never depends on the email path working.

## Sample report

```
============================================================
 Vörwatch — VPS Anomaly Detection — Text Report
 Generated: 2026-07-17 08:30:54 UTC
 Range: today
============================================================

-- High Priority (blocklist matches + integrity/rootkit hits) --
  none

-- Alert Summary (today) --
  Total alerts:           103
  New listening ports:    0
  Critical file changed:  0
  First-seen outbound IP: 103
  Blocklisted outbound:   0
  Suspicious proc tree:   0
  Blocklisted SSH source: 0
  CIS hardening findings: 0
  Rootkit findings:       0
  First-seen DNS queries: 0
  Vulnerable packages:    0

-- System Vitals --
  Load average: 0.00, 0.00, 0.05
  Memory:       2402MB/23988MB used (10%)
  Disk (/):     46G/146G used (32%)
  Uptime:       up 9 weeks, 4 days, 20 hours, 42 minutes

-- fail2ban --
  sshd                     total_failed=41605    currently_banned=0      total_banned=1487

-- nginx (current log file only) --
  Requests logged:        449
  Top 5 source IPs by request count (risk 1-5, 5=critical, via AbuseIPDB):
    115.186.231.43       35 requests  [risk 1]
  ...

-- Package Vulnerabilities (OSV.dev, cached) --
  1 package(s), 1 total CVE/USN ID(s) found (via OSV.dev, includes historical/low-severity entries)
  ...
  and 1 more package(s) not shown (see vuln-cache.txt for the full list)
  Look up any ID at: https://osv.dev/vulnerability/<ID>

-- Rootkit Scan --
  Last scan: 3h ago (runs at most every 24h)
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
