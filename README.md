<div align="center">

# 🍯 Honeypot Suite

**A single, modular Go binary that runs nineteen protocol honeypots side by side.**

Toggle each protocol from `.env`, capture every probe and credential as
structured JSON, and add new protocols as self-contained packages. Every pot
impersonates one shared Windows / Active Directory persona, answers with
randomized timing, and can alert you the moment credentials are captured.

![Go](https://img.shields.io/badge/Go-1.24-00ADD8?logo=go&logoColor=white)
![Honeypots](https://img.shields.io/badge/honeypots-19-orange)
![Tests](https://img.shields.io/badge/tests-unit%20%2B%20integration%20%2B%20race-success)
![Platform](https://img.shields.io/badge/platform-linux%20%7C%20windows-lightgrey)
![License](https://img.shields.io/badge/license-Apache--2.0-blue)

</div>

---

## Overview

Honeypot Suite consolidates the former standalone `DNSpot`, `RDPpot`, and
`SMBpot` tools — plus sixteen more services — into one cohesive, growth-ready
codebase. Every honeypot implements the same small interface, is supervised by
a shared manager, and emits events through one structured logger, so the suite
behaves consistently no matter how many protocols you enable.

It is designed for **defensive use**: deception, early-warning, and attacker
telemetry on networks you own or are authorized to monitor.

## ✨ Features

- **19 protocols, one binary** — SSH, DNS, Kerberos, RDP, SMB, LDAP, NetBIOS,
  FTP, Telnet, SNMP, MySQL, MSSQL, Redis, HTTP, PostgreSQL, VNC, WinRM,
  Elasticsearch, MongoDB.
- **One shared persona** — a single set of identity vars (hostname, domain, OS)
  keeps every pot telling the same Windows Server 2019 / Active Directory story:
  IIS, Microsoft FTP/Telnet/SSH, AD LDAP (with rootDSE), `CONTOSO\WIN-DC01`, …
- **Post-login engagement** — SSH/Telnet drop the attacker into a **safe fake
  cmd.exe** (nothing is ever executed) and record every command and malware
  download URL; FTP serves a fake filesystem and **captures uploaded files**;
  HTTP shows a fake admin panel after a captured login.
- **Crackable hashes** — RDP and SMB drive a full NTLM challenge/response and log
  a hashcat-ready **NetNTLMv2** line, not just the username.
- **Real-time alerting** — high-value captures are pushed to a webhook
  (Discord/Slack) and/or syslog the instant they happen.
- **Built-in metrics** — an optional Prometheus `/metrics` endpoint exposes
  per-pot event/credential counts and unique attacker IPs.
- **Randomized timing** — each pot waits a random per-protocol delay before its
  first byte, with slower built-in pauses for failed logins (300–800 ms) and
  unknown users (500–1500 ms), so constant-latency fingerprinting fails.
- **`.env`-driven** — enable/disable each pot and set its bind address without
  touching code. Real environment variables always override the file.
- **Structured intelligence** — every event is one JSON line (JSONL), ready to
  ship straight into Elasticsearch / Loki / Splunk.
- **Credential capture** — plaintext where the protocol allows (FTP, Telnet,
  LDAP simple bind, MSSQL, Redis, HTTP Basic) and salted hashes/scrambles
  otherwise (RDP/SMB NTLM, MySQL).
- **Resilient by design** — a pot that can't bind its port (common on Windows
  where 445/3389 are taken) is logged and skipped; the rest keep serving.
- **Graceful shutdown** — `SIGINT`/`SIGTERM` cleanly stops every listener.
- **Modular & tested** — each protocol is its own package with unit tests for
  the parsers and an integration test that exercises the real handshake.

## 🎯 Honeypots

| Pot | Port | Protocol | Captures | Plaintext? |
|-----|------|----------|----------|:---------:|
| `ssh` | 22 | SSH | Username + password + **post-login commands** (fake shell) | ✅ |
| `dns` | 53 (UDP+TCP) | DNS | Queried name, record type, source | — |
| `kerberos` | 88 (UDP+TCP) | Kerberos | AS-REQ/TGS-REQ principal + realm (spray/roast intel) | — |
| `rdp` | 3389 | RDP / NLA | Username + domain + **NetNTLMv2** (CredSSP/TLS) | ✗ (hash) |
| `smb` | 445 | SMBv1/v2 | Username + domain + **NetNTLMv2** (session setup) | ✗ (hash) |
| `ldap` | 389 | LDAP | Bind DN + password; serves AD rootDSE | ✅ |
| `netbios` | 137 (UDP) | NBNS | NetBIOS name + suffix/role | — |
| `ftp` | 21 | FTP | `USER`/`PASS` + **uploaded files** (fake filesystem) | ✅ |
| `telnet` | 23 | Telnet | Username + password + **post-login commands** | ✅ |
| `snmp` | 161 (UDP) | SNMP v1/v2c | Community string + PDU type | ✅ |
| `mysql` | 3306 | MySQL | Username + auth scramble (+ salt) | ✗ (scramble) |
| `mssql` | 1433 | TDS | Username + password | ✅ |
| `redis` | 6379 | RESP | `AUTH` credentials + commands | ✅ |
| `http` | 80 | HTTP | Request, Basic-auth creds, form bodies | ✅ |
| `postgres` | 5432 | PostgreSQL | Username + database + cleartext password | ✅ |
| `vnc` | 5900 | RFB | Auth challenge + response (crackable password) | ✗ (DES) |
| `winrm` | 5985 | WinRM/HTTP | Basic creds + **NetNTLMv2** (Negotiate/NTLM) | ✗ (hash) |
| `elasticsearch` | 9200 | HTTP/JSON | Requests, search/exfil queries, Basic creds | ✅ |
| `mongodb` | 27017 | Mongo wire | Commands + SCRAM username | — |

> **Plaintext?** indicates whether the protocol hands the suite a recoverable
> password. RDP/SMB drive an NTLM challenge and log a hashcat **NetNTLMv2** line
> (`-m 5600`); MySQL logs a salted scramble. All are crackable offline.

## 🎭 Persona & deception

All pots share one identity, set once at the top of `.env`:

| Variable | Default | Used by |
|----------|---------|---------|
| `HONEYPOT_HOSTNAME` | `WIN-DC01` | NetBIOS, SNMP sysName, RDP cert, Kerberos |
| `HONEYPOT_NETBIOS_DOMAIN` | `CONTOSO` | NetBIOS, SMB |
| `HONEYPOT_AD_DOMAIN` | `contoso.local` | RDP cert, Kerberos realm, FQDNs |
| `HONEYPOT_OS` | `Windows Server 2019 Datacenter` | SMB, SNMP sysDescr |
| `HONEYPOT_OS_VERSION` | `10.0.17763.1` | DNS version.bind, SNMP |

The result is a consistent Windows DC: **Microsoft-IIS/10.0 + ASP.NET** on HTTP,
**Microsoft FTP Service**, **Microsoft Telnet Service**, **SQL Server 2019
(15.0.4415)**, **MySQL 5.7.44**, **Redis 7.2.4**, AD LDAP, `CONTOSO\WIN-DC01`
over NetBIOS, and a `CONTOSO.LOCAL` Kerberos KDC that answers username
enumeration like real AD (`PREAUTH_REQUIRED` for known accounts vs
`PRINCIPAL_UNKNOWN` for the rest).

**Timing.** Each pot sleeps a random window before its first response (DNS
1–15 ms, RDP 30–150 ms, HTTP 20–300 ms, …) — override any with
`HONEYPOT_<POT>_DELAY_MIN_MS` / `_MAX_MS`. Credential rejections add a longer,
randomized pause (failed login 300–800 ms, unknown user 500–1500 ms) so latency
analysis can't separate the decoy from a real service.

## 🛡️ No amplification, ever

A honeypot must never become a DDoS reflector. Every UDP pot
(`internal/ratelimit`) enforces **per-client-IP Response Rate Limiting (RRL)** —
a spoofed flood from one source is dropped, so there is no high-rate reply stream
to amplify. On top of that:

- **DNS** truncates any UDP reply over 512 bytes (`TC=1`, forcing TCP retry) and
  **refuses `ANY`** queries — so a UDP answer is never meaningfully larger than the
  query, the classic amplification lever.
- **Kerberos** never emits a UDP reply larger than the request.
- **SNMP / NetBIOS** answer only within the rate limit.

With `HONEYPOT_DNS_FORWARD=true` the DNS pot also resolves queries through an
upstream resolver (`HONEYPOT_DNS_UPSTREAM`, default `1.1.1.1:53`) **over TCP** for
real answers, with a bounded cache shielding the upstream — recursion without a
reflection surface. Leave it `false` to return synthesized records instead.

## 🔔 Alerting & monitoring

- **Real-time alerts.** Set `HONEYPOT_ALERT_WEBHOOK_URL` (Discord/Slack/generic)
  and/or `HONEYPOT_ALERT_SYSLOG` to be notified the instant a credential or login
  is captured — a non-blocking worker pushes a one-line summary plus the redacted
  event, dropping under overload rather than ever stalling a pot.
- **Prometheus metrics.** With `HONEYPOT_METRICS_ENABLED=true` a `/metrics`
  endpoint (bind it to localhost/admin via `HONEYPOT_METRICS_LISTEN`) exposes
  `honeypot_events_total{pot}`, `honeypot_credentials_total{pot}`,
  `honeypot_unique_source_ips`, and uptime.
- **Log rotation.** Each log rotates past `HONEYPOT_LOG_MAX_MB`, keeping
  `HONEYPOT_LOG_MAX_BACKUPS` files (`events.log.1`, `.2`, …) so disk never fills.

## 🪤 Engagement & post-login capture

Capturing a password is good; capturing what the attacker does *after* is far
better. Several pots keep the attacker engaged and record their real intent —
**without ever executing anything** (`internal/shell` is a pure canned-output
emulator):

- **SSH / Telnet fake shell** (`HONEYPOT_SSH_SHELL`, `HONEYPOT_TELNET_SHELL`).
  After login the attacker gets a believable Windows `cmd.exe`
  (`C:\Users\Administrator>`). Every command is logged (`SSH_COMMAND`), and
  download attempts (`wget`/`curl`/`certutil`/`powershell …DownloadString`) are
  flagged as `SSH_DOWNLOAD` with the extracted URL — so you capture the malware
  payload location, not just the credentials.
- **FTP fake filesystem** (`HONEYPOT_FTP_FS`). Login is accepted and
  `LIST`/`RETR`/`STOR` work over PASV/EPSV; **uploaded files are captured**
  (`FTP_UPLOAD` with size) — attackers often upload webshells.
- **HTTP fake admin panel.** A captured login POST returns a believable
  dashboard instead of the login form, keeping the attacker clicking.

Accepting any login is a deliberate trade-off that enables command capture; set
the `*_SHELL` / `*_FS` flags to `false` to fall back to credential-only capture.

## 🧱 Production hardening

Built to survive a hostile internet 24/7 (`internal/netx`):

- **Per-IP connection cap + lifetime.** `HONEYPOT_MAX_CONNS_PER_IP` (default 10)
  bounds concurrent connections from one source; `HONEYPOT_MAX_CONN_SECONDS`
  (default 120) is an absolute per-connection deadline that cannot be reset by
  per-read timeouts — defeating slowloris and connection-flood resource exhaustion.
- **Panic recovery in every handler.** A malformed packet that trips a parser is
  logged and the connection dropped — it can never crash the process (and with it
  the other 13 pots).
- **Bounded parsing.** BER lengths and SMB/LDAP session buffers are capped, so a
  declared multi-gigabyte length can't force unbounded buffering.
- **Windows-like TTL.** Listeners emit packets with IP TTL 128 (Linux default is
  64), reducing the OS-fingerprint mismatch with the Windows persona.
- **Spoofable source marking.** UDP-sourced events carry `"spoofable":true` — the
  source IP is forgeable, so **never feed these into RTBH/BGP-blackhole automation**;
  use only TCP-handshake-verified sources for blocklisting.
- **Stable identity & logging.** Persisted SSH host key
  (`HONEYPOT_SSH_HOST_KEY`), size-based log rotation, an optional merged log
  (`HONEYPOT_LOG_COMBINED`), and alert/metrics observers that run off the log lock.

## 🏗️ Architecture

```
                            ┌──────────────────────────┐
                  .env ───▶ │   config.Load()          │  typed Config
                            └────────────┬─────────────┘
                                         │
                            ┌────────────▼─────────────┐
                            │   cmd/honeypot (main)     │  wires enabled pots
                            └────────────┬─────────────┘
                                         │ Register(...)
                            ┌────────────▼─────────────┐
       SIGINT/SIGTERM ────▶ │   honeypot.Manager       │  supervises + shuts down
                            └──┬───────┬───────┬────────┘
                               │       │       │   (one goroutine per pot)
                  ┌────────────▼─┐ ┌───▼───┐ ┌─▼──────────┐
                  │ dns / rdp /  │ │ ftp / │ │ mysql /    │   each implements
                  │ smb / ldap…  │ │ http… │ │ redis…     │   honeypot.Honeypot
                  └──────┬───────┘ └───┬───┘ └─────┬──────┘
                         └─────────────┼───────────┘
                                       │ Event{...}
                            ┌──────────▼───────────────┐
                            │   logging.Logger         │
                            └──────────┬───────────────┘
                       ┌───────────────┼───────────────┐
                       ▼               ▼               ▼
                  events.log      <pot>.log       honeypot.log
                  (all, JSONL)    (per pot)       (operational)
```

Adding a protocol means writing one package that satisfies `honeypot.Honeypot`
— the manager, config, and logger require no changes elsewhere.

## 🚀 Quick start

```sh
cp .env.example .env          # choose which pots to enable
go build -o honeypot ./cmd/honeypot
./honeypot                    # reads ./.env by default
```

Point at a different config file with `-env`:

```sh
./honeypot -env /etc/honeypot/prod.env
```

> **Privileged ports.** Binding ports below 1024 (22, 21, 23, 53, 80, 88, 137,
> 161, 389, 445) requires elevated privileges: run as root/Administrator, or on
> Linux grant the capability once with
> `sudo setcap cap_net_bind_service=+ep ./honeypot`.

> **Windows tip.** `go run ./cmd/honeypot` may fail with *"executable file not
> found"* because Windows Defender quarantines the temporary binary `go run`
> produces. Build to a fixed path (`go build -o honeypot.exe ./cmd/honeypot`)
> and run that instead.

## ⚙️ Configuration

Every setting is an environment variable, optionally seeded from `.env`.
All pots default to **disabled**, so the suite only opens what you ask for.

<details open>
<summary><b>Per-pot settings</b></summary>

Per pot, `HONEYPOT_<POT>_ENABLED` (default `false`), `_LISTEN`, and the optional
`_DELAY_MIN_MS` / `_DELAY_MAX_MS` are always available. The protocol-specific
settings:

| Variable | Default | Description |
|----------|---------|-------------|
| `HONEYPOT_DNS_ANSWER_IPV4` | `192.168.1.1` | Fake A record (when not forwarding) |
| `HONEYPOT_DNS_ANSWER_IPV6` | `2001:db8::1` | Fake AAAA record |
| `HONEYPOT_DNS_DOMAIN` | _(empty)_ | Base zone for synthesized MX/NS/SOA/… (empty = mirror the queried domain) |
| `HONEYPOT_DNS_FORWARD` | `false` | Resolve through an upstream resolver for real answers (safeguards always on) |
| `HONEYPOT_DNS_UPSTREAM` | `1.1.1.1:53` | Upstream recursive resolver (queried over TCP) |
| `HONEYPOT_DNS_VERSION` | _(empty)_ | CHAOS `version.bind`: empty = refuse (Windows-like); set to emulate BIND |
| `HONEYPOT_KERBEROS_*` | `0.0.0.0:88` | Kerberos KDC pot (TCP+UDP) |
| `HONEYPOT_KERBEROS_REALM` | `CONTOSO.LOCAL` | Realm (defaults to upper-cased AD domain) |
| `HONEYPOT_KERBEROS_VALID_USERS` | `administrator,krbtgt,guest` | "Existing" accounts (PREAUTH_REQUIRED vs PRINCIPAL_UNKNOWN) |
| `HONEYPOT_SSH_*` | `0.0.0.0:22` | SSH pot bind address |
| `HONEYPOT_SSH_BANNER` | `SSH-2.0-OpenSSH_for_Windows_8.1` | SSH identification string |
| `HONEYPOT_SSH_HOST_KEY` | _(empty)_ | Path to persist the host key (stable fingerprint); empty = ephemeral |
| `HONEYPOT_SSH_SHELL` | `true` | Accept login + serve fake cmd.exe to capture commands (false = creds only) |
| `HONEYPOT_TELNET_SHELL` | `true` | Same fake shell for Telnet |
| `HONEYPOT_FTP_FS` | `true` | Accept login + fake filesystem; capture uploads (false = creds only) |
| `HONEYPOT_LDAP_BASE_DN` | _from AD domain_ | rootDSE base, e.g. `DC=contoso,DC=local` |
| `HONEYPOT_RDP_CERT_ORG` | `contoso.local` | Org name in the self-signed cert |
| `HONEYPOT_RDP_CERT_CN` | _host FQDN_ | Certificate CN (e.g. `win-dc01.contoso.local`) |
| `HONEYPOT_LDAP_BIND_RESULT` | `invalid` | Bind reply: `invalid` (49) or `success` (0) |
| `HONEYPOT_NETBIOS_ANSWER_IPV4` | `192.168.1.1` | Fake IP in name-query answers |
| `HONEYPOT_NETBIOS_HOSTNAME` | `WIN-DC01` | Computer name in NBSTAT replies |
| `HONEYPOT_NETBIOS_DOMAIN` | `CONTOSO` | Workgroup/domain in NBSTAT replies |
| `HONEYPOT_FTP_BANNER` | `220 Microsoft FTP Service` | FTP greeting banner |
| `HONEYPOT_TELNET_BANNER` | `Microsoft Telnet Service…` | Pre-login banner |
| `HONEYPOT_SNMP_SYSDESCR` | _Windows Server string_ | sysDescr.0 value |
| `HONEYPOT_SNMP_SYSNAME` | `WIN-DC01` | sysName.0 value |
| `HONEYPOT_MYSQL_VERSION` | `5.7.44` | Version advertised in the greeting |
| `HONEYPOT_MSSQL_VERSION` | `15.0.4415` | SQL Server version in the pre-login |
| `HONEYPOT_REDIS_REQUIRE_AUTH` | `true` | Reply `NOAUTH` until an AUTH is seen |
| `HONEYPOT_REDIS_VERSION` | `7.2.4` | `redis_version` in the INFO reply |
| `HONEYPOT_HTTP_SERVER` | `Microsoft-IIS/10.0` | `Server:` response header |
| `HONEYPOT_HTTP_POWERED_BY` | `ASP.NET` | `X-Powered-By:` header (empty omits it) |
| `HONEYPOT_HTTP_REALM` | `Restricted` | Basic-auth realm in 401 responses |
| `HONEYPOT_POSTGRES_*` | `0.0.0.0:5432` | PostgreSQL pot bind address |
| `HONEYPOT_VNC_*` | `0.0.0.0:5900` | VNC pot bind address |
| `HONEYPOT_WINRM_*` | `0.0.0.0:5985` | WinRM pot bind address (NTLM persona from FQDN) |
| `HONEYPOT_ELASTICSEARCH_*` | `0.0.0.0:9200` | Elasticsearch pot bind address |
| `HONEYPOT_ELASTICSEARCH_VERSION` | `7.17.9` | Advertised ES version |
| `HONEYPOT_ELASTICSEARCH_CLUSTER` | `elasticsearch` | Advertised cluster name |
| `HONEYPOT_ELASTICSEARCH_NODE` | _persona hostname_ | Advertised node name |
| `HONEYPOT_MONGODB_*` | `0.0.0.0:27017` | MongoDB pot bind address |
| `HONEYPOT_MONGODB_VERSION` | `6.0.4` | Advertised MongoDB version |

</details>

> **Believable DNS answers.** Synthesized records (MX, NS, SOA, SRV, CNAME, PTR)
> hang their hostnames off a real-looking zone instead of any honeypot-revealing
> string. By default that zone mirrors the domain the client queried
> (`example.com MX` → `mail.example.com`); set `HONEYPOT_DNS_DOMAIN` to pin every
> answer to a fixed domain instead.

<details>
<summary><b>Logging settings</b></summary>

| Variable | Default | Description |
|----------|---------|-------------|
| `HONEYPOT_LOG_DIR` | `./logs` | Log output directory |
| `HONEYPOT_LOG_CONSOLE` | `true` | Also print events to stdout (disable in production) |
| `HONEYPOT_LOG_COMBINED` | `true` | Also write the merged `events.log` (false halves disk use) |
| `HONEYPOT_LOG_LEVEL` | `info` | `info` or `debug` |
| `HONEYPOT_MAX_CONNS_PER_IP` | `10` | Max concurrent connections per source IP (0 disables) |
| `HONEYPOT_MAX_CONN_SECONDS` | `120` | Absolute per-connection lifetime, anti-slowloris (0 disables) |
| `HONEYPOT_LOG_MAX_MB` | `100` | Rotate each log past this size (0 disables) |
| `HONEYPOT_LOG_MAX_BACKUPS` | `5` | Rotated files to keep per log |
| `HONEYPOT_REDACT_IPS` | _(empty)_ | Extra addresses to scrub (interface + bind IPs are auto-scrubbed) |
| `HONEYPOT_REDACT_WITH` | _persona FQDN_ | Replacement string for a redacted address |
| `HONEYPOT_ALERT_WEBHOOK_URL` | _(empty)_ | POST high-value captures here (Discord/Slack/generic) |
| `HONEYPOT_ALERT_SYSLOG` | _(empty)_ | `host:port` (UDP) or `tcp://host:port` |
| `HONEYPOT_METRICS_ENABLED` | `false` | Serve the Prometheus `/metrics` endpoint |
| `HONEYPOT_METRICS_LISTEN` | `127.0.0.1:9100` | Metrics admin bind address |

</details>

> **Hide the host's real IP.** TLS/HTTP handshakes and connection errors can echo
> the server's own address into a logged event (e.g. `read tcp <public-ip>:3389->…`
> or an HTTP `Host: <public-ip>`), which would then leak through the public log
> dashboard. The logger **auto-scrubs every non-loopback interface address and
> every IP the suite binds to** (the latter also appears in scanner probes like
> `MGLNDD_<ip>_21`) from all output — event files, console, and `honeypot.log` —
> with no configuration. Add `HONEYPOT_REDACT_IPS` for any extra address the host
> doesn't own directly (e.g. a NAT/elastic IP), and `HONEYPOT_REDACT_WITH` to
> change the replacement (the persona FQDN by default).

## 📊 Logging

Two streams are written to `HONEYPOT_LOG_DIR`:

| File | Content |
|------|---------|
| `events.log` | Every intelligence event from every pot, one JSON object per line (JSONL). |
| `<pot>.log` | The same events filtered to a single pot (`dns.log`, `ftp.log`, …), created on first use. |
| `honeypot.log` | Operational lifecycle and error messages (startup, bind failures, shutdown). |

Example `events.log` lines:

```json
{"time":"2026-06-10T12:29:50Z","pot":"dns","ip":"185.220.101.4","type":"DNS_QUERY","query":"vpn.corp.local","qtype":"A","proto":"UDP"}
{"time":"2026-06-10T12:30:02Z","pot":"ftp","ip":"45.134.26.7","type":"FTP_CREDENTIAL_CAPTURED","user":"admin","password":"P@ssw0rd123"}
{"time":"2026-06-10T12:31:14Z","pot":"mssql","ip":"103.97.3.21","type":"MSSQL_LOGIN_ATTEMPT","user":"sa","password":"Summer2026!"}
{"time":"2026-06-10T12:33:48Z","pot":"snmp","ip":"91.240.118.9","type":"SNMP_COMMUNITY_CAPTURED","password":"public","detail":"version=v2c pdu=GetRequest"}
{"time":"2026-06-10T12:35:09Z","pot":"http","ip":"193.43.72.5","type":"HTTP_BASIC_AUTH","user":"administrator","password":"hunter2"}
```

Since every line is self-describing JSON, you can grep across all pots — for
example, list every captured password:

```sh
jq -r 'select(.password) | "\(.pot)\t\(.ip)\t\(.user)\t\(.password)"' logs/events.log
```

## 🧪 Tests

```sh
go test ./...            # all unit + integration tests
go test -race ./...      # with the race detector (needs a C toolchain)
gofmt -l .               # should print nothing (formatting check)
```

Tests are self-contained: pure parsers (BER, NTLM, TDS, RESP, NBNS encoding…)
are unit-tested directly, and each pot has an integration test that binds an
ephemeral port and drives the real protocol handshake — no privileges or fixed
ports required.

## 🗂️ Project layout

```
cmd/honeypot/          entrypoint: load config, wire pots, run, shut down
internal/config/       .env loader + typed configuration
internal/logging/      unified event + operational logger
internal/honeypot/     Honeypot interface + Manager (supervision/shutdown)
internal/ber/          shared ASN.1/BER helpers (used by ldap, snmp, kerberos)
internal/ntlm/         shared NTLM challenge/response (used by rdp, smb)
internal/timing/       randomized response-delay windows
internal/ratelimit/    per-IP token bucket (UDP anti-amplification RRL)
internal/netx/         connection guard rails (TTL, per-IP cap, deadline, recover)
internal/shell/        safe fake cmd.exe emulator (used by ssh, telnet)
internal/alert/        real-time webhook/syslog notifier
internal/metrics/      Prometheus counters + /metrics endpoint
internal/testutil/     test helpers (free port, wait-for-listener)
internal/pots/
    ssh/  dns/  kerberos/  rdp/      one self-contained package per protocol
    smb/  ldap/  netbios/  ftp/
    telnet/  snmp/  mysql/  mssql/  redis/  http/
    postgres/  vnc/  winrm/  elasticsearch/  mongodb/
```

## ➕ Adding a new honeypot

1. Create `internal/pots/<name>/` with a type implementing the
   `honeypot.Honeypot` interface:
   ```go
   type Honeypot interface {
       Name() string
       Run(ctx context.Context) error   // block until ctx is cancelled
   }
   ```
2. Add a config block to `internal/config/config.go` and read its env vars in
   `Load`.
3. Register it in `registerPots` in `cmd/honeypot/main.go`.

The manager (supervision, fail-isolation, shutdown) and the logger require no
changes. Reuse `logging.Event` fields where they fit; only extend the struct
when a concept is genuinely new.

## 🛡️ Deployment notes

- **Run as an unprivileged service** behind the capability grant above, or
  inside a container, and forward `logs/` to your SIEM.
- **Port conflicts** on the host (e.g. Windows already serving 445/3389) are
  non-fatal: the affected pot logs an error and the suite keeps running the
  rest. Move the pot to a free port in `.env` or disable it.
- **systemd** example:
  ```ini
  [Unit]
  Description=Honeypot Suite
  After=network.target

  [Service]
  ExecStart=/opt/honeypot/honeypot -env /etc/honeypot/prod.env
  WorkingDirectory=/opt/honeypot
  AmbientCapabilities=CAP_NET_BIND_SERVICE
  Restart=always

  [Install]
  WantedBy=multi-user.target
  ```

## ⚠️ Legal & ethical use

This software is intended for **authorized defensive security** only. Deploy it
solely on systems and networks you own or have explicit written permission to
monitor. Honeypots collect data from hosts that connect to them; ensure your
use complies with applicable laws and privacy regulations in your jurisdiction.
The authors accept no liability for misuse.

## License

Licensed under the **[Apache License 2.0](LICENSE)** — a permissive,
OSI-approved license recognized by GitHub.

- ✅ **Free for any use**, including commercial, with no royalties.
- 📝 **Attribution required** — you must retain the copyright and license
  notices, and any redistribution or derivative work must include the
  [NOTICE](NOTICE) file, so the author is always credited as the source
  (Apache-2.0 §4).
- 🛡️ **Patent grant** — contributors grant an explicit patent license, with
  termination on patent litigation (Apache-2.0 §3).

> Copyright © 2026 **Alptekin Sünnetci** &lt;alptekin@sunnetci.net&gt;
