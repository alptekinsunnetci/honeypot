<div align="center">

# 🍯 Honeypot Suite

**A single, modular Go binary that runs thirteen protocol honeypots side by side.**

Toggle each protocol from `.env`, capture every probe and credential as
structured JSON, and add new protocols as self-contained packages. Every pot
impersonates one shared Windows / Active Directory persona and answers with
randomized, human-plausible timing.

![Go](https://img.shields.io/badge/Go-1.24-00ADD8?logo=go&logoColor=white)
![Honeypots](https://img.shields.io/badge/honeypots-13-orange)
![Tests](https://img.shields.io/badge/tests-unit%20%2B%20integration%20%2B%20race-success)
![Platform](https://img.shields.io/badge/platform-linux%20%7C%20windows-lightgrey)
![License](https://img.shields.io/badge/license-Apache--2.0-blue)

</div>

---

## Overview

Honeypot Suite consolidates the former standalone `DNSpot`, `RDPpot`, and
`SMBpot` tools — plus ten more services — into one cohesive, growth-ready
codebase. Every honeypot implements the same small interface, is supervised by
a shared manager, and emits events through one structured logger, so the suite
behaves consistently no matter how many protocols you enable.

It is designed for **defensive use**: deception, early-warning, and attacker
telemetry on networks you own or are authorized to monitor.

## ✨ Features

- **13 protocols, one binary** — DNS, Kerberos, RDP, SMB, LDAP, NetBIOS, FTP,
  Telnet, SNMP, MySQL, MSSQL, Redis, HTTP.
- **One shared persona** — a single set of identity vars (hostname, domain, OS)
  keeps every pot telling the same Windows Server 2019 / Active Directory story:
  IIS, Microsoft FTP/Telnet, AD LDAP, `CONTOSO\WIN-DC01`, SQL Server 2019, …
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
| `dns` | 53 (UDP+TCP) | DNS | Queried name, record type, source | — |
| `kerberos` | 88 (UDP+TCP) | Kerberos | AS-REQ/TGS-REQ principal + realm (spray/roast intel) | — |
| `rdp` | 3389 | RDP / NLA | NTLM username + domain (via CredSSP/TLS) | ✗ (NTLM) |
| `smb` | 445 | SMBv1/v2 | Session-setup username + domain | ✗ (NTLM) |
| `ldap` | 389 | LDAP | Bind DN + password (simple auth) | ✅ |
| `netbios` | 137 (UDP) | NBNS | NetBIOS name + suffix/role | — |
| `ftp` | 21 | FTP | `USER` / `PASS` + issued commands | ✅ |
| `telnet` | 23 | Telnet | Username + password (IAC-aware) | ✅ |
| `snmp` | 161 (UDP) | SNMP v1/v2c | Community string + PDU type | ✅ |
| `mysql` | 3306 | MySQL | Username + auth scramble (+ salt) | ✗ (scramble) |
| `mssql` | 1433 | TDS | Username + password | ✅ |
| `redis` | 6379 | RESP | `AUTH` credentials + commands | ✅ |
| `http` | 80 | HTTP | Request, Basic-auth creds, form bodies | ✅ |

> **Plaintext?** indicates whether the protocol hands the suite a recoverable
> password. RDP/SMB use NTLM and MySQL uses a salted scramble — those yield
> hashes suitable for offline cracking, logged alongside their salt where
> applicable.

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
over NetBIOS, and a `CONTOSO.LOCAL` Kerberos KDC.

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

> **Privileged ports.** Binding ports below 1024 (21, 23, 53, 80, 137, 161,
> 389, 445) requires elevated privileges: run as root/Administrator, or on
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
| `HONEYPOT_DNS_VERSION` | `Microsoft DNS <os_version>` | CHAOS `version.bind` reply |
| `HONEYPOT_KERBEROS_*` | `0.0.0.0:88` | Kerberos KDC pot (TCP+UDP) |
| `HONEYPOT_KERBEROS_REALM` | `CONTOSO.LOCAL` | Realm (defaults to upper-cased AD domain) |
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
| `HONEYPOT_LOG_CONSOLE` | `true` | Also print events to stdout |
| `HONEYPOT_LOG_LEVEL` | `info` | `info` or `debug` |
| `HONEYPOT_REDACT_IPS` | _(empty)_ | Extra addresses to scrub (local interface IPs are auto-detected and always scrubbed) |
| `HONEYPOT_REDACT_WITH` | _persona FQDN_ | Replacement string for a redacted address |

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
internal/timing/       randomized response-delay windows
internal/ratelimit/    per-IP token bucket (UDP anti-amplification RRL)
internal/testutil/     test helpers (free port, wait-for-listener)
internal/pots/
    dns/  kerberos/  rdp/  smb/     one self-contained package per protocol
    ldap/  netbios/  ftp/  telnet/
    snmp/  mysql/  mssql/  redis/  http/
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
