# Changelog

All notable changes to Vörwatch are documented here.

## [0.10.2] - 2026-09-12

### Fixed
- **File integrity check false positive on authorized_keys** -
  `CRITICAL_FILES` resolved `$HOME/.ssh/authorized_keys` using the
  shell's `$HOME`, which differs depending on invocation context
  (`/home/ubuntu` under cron, `/root` under `sudo vorwatch check`).
  Running the check manually with `sudo` after a baseline captured
  without it (or vice versa) compared two genuinely different file
  paths and alerted a false "critical file changed". Now resolves the
  real user's home directory once via
  `getent passwd "${SUDO_USER:-$(id -un)}"`, stable regardless of how
  the check is invoked.

## [0.10.1] - 2026-09-12

### Added
- **Cloudflare edge-blocked probe detector** - queries the Cloudflare
  GraphQL Analytics API (`httpRequestsAdaptiveGroups`, free on all
  plans) for requests Cloudflare blocked at the edge (4xx) that never
  reached origin nginx logs, and matches them against the same
  `HIGH_VALUE_404_TARGETS` signature list used by the weighted
  secret-filename scoring check. Closes a real blind spot: an attacker
  sweep can be fully edge-blocked and completely invisible to every
  log-based check in this project. Configured via
  `VORWATCH_CF_API_TOKEN` / `VORWATCH_CF_ZONE_ID` (both unset by
  default - the check is a no-op with no network calls until both are
  set). Degrades gracefully on API/auth failure, same pattern as the
  CrowdSec integration.

## [0.10.0] - 2026-09-12

### Added
- **Credential & AI API key exposure scan** — sweeps `.git-credentials`,
  `.gitconfig`, shell history, SSH config, and cloud CLI cred files
  (AWS/Azure/GCP) for embedded Anthropic/OpenAI/GitHub key patterns and
  `user:token@` git URLs.
- **SSRF / cloud-metadata payload detector** — flags nginx requests
  targeting `169.254.169.254`, GCP/Azure metadata endpoints, and
  `file://`/`dict://`/`gopher://` wrapper abuse as a dedicated CRITICAL
  category, separate from generic 404 noise.
- **UA-rotation / crawler-impersonation detector** — flags a single IP
  presenting 2+ distinct known-crawler identities (reuses
  `KNOWN_BOT_UA_REGEX`), with per-IP trusted-CIDR suppression.
- **Weighted high-value secret-filename scoring** — `.env`,
  `terraform.tfstate`, `service-account.json`, `id_rsa`, etc. bypass
  count-threshold logic and alert on first hit.
- **Docker/containerd port-churn filter** — ephemeral-range,
  127.0.0.1-only ports owned by `docker-proxy`/`containerd`/`kubelet`
  no longer trigger new-listening-port alerts.
- **Process-linked outbound connection tracking** — first-seen outbound
  IP alerts now include the owning process/binary, not just the IP.
- **Polymorphic dropper detection** — flags new executable files dropped
  in `/tmp`, `/dev/shm`, `/var/tmp` within the last 24h.
- **Adaptive scan velocity detector** — flags a single IP hitting 15+
  distinct 400/403/404 paths in one log window (autonomous/AI-driven
  scanning signature, distinct from single-path brute force).
- **Build-tool / exploit-foundry detector** — flags non-root execution
  of compilers/build tools (`gcc`, `make`, `npm`, `cargo`, etc.) on the
  bare host, with a configurable path exception
  (`VORWATCH_BUILD_TOOL_EXCLUDE_PATHS`) for legitimate app-level builds.
- **Shell history tampering detection** — flags `.bash_history`/
  `.zsh_history` symlinked to `/dev/null` or truncated to 0 bytes.
- **fail2ban bridge** — SSRF, high-value-secret, and adaptive-scan hits
  now write to `/var/log/vorwatch-actionable.log`, picked up by a
  dedicated `vorwatch-actionable` jail for automatic banning.
- **CrowdSec blocklist integration** (optional, off by default) — merges
  a CrowdSec Service API blocklist into the same IP-reputation check as
  the existing firehol list via `VORWATCH_CROWDSEC_API_KEY` /
  `VORWATCH_CROWDSEC_BLOCKLIST_ID`.

### Fixed
- UA-rotation detector no longer false-positives on real crawlers whose
  UA string drifts slightly between requests (Googlebot, Applebot) —
  dedupes on matched bot name, not raw UA string.

### Notes
- All new checks degrade gracefully with zero new required dependencies
  — still pure coreutils/awk/grep, consistent with the project's
  dependency-free design goal.
- Design informed by Anthropic's September 2026 Threat Intelligence
  Report (GTG-50014, GTG-50020, GTG-50021, GTG-20006, GTG-10007) and
  live findings on this project's own production VPS.

## [0.9.1] and earlier
- See git history — no changelog was kept prior to 0.10.0.
