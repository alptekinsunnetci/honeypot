<div align="center">

# 🍯 Honeypot Suite

**A single, modular Go binary that runs twelve protocol honeypots side by side.**

Toggle each protocol from `.env`, capture every probe and credential as
structured JSON, and add new protocols as self-contained packages.

![Go](https://img.shields.io/badge/Go-1.24-00ADD8?logo=go&logoColor=white)
![Honeypots](https://img.shields.io/badge/honeypots-12-orange)
![Tests](https://img.shields.io/badge/tests-unit%20%2B%20integration%20%2B%20race-success)
![Platform](https://img.shields.io/badge/platform-linux%20%7C%20windows-lightgrey)
![License](https://img.shields.io/badge/license-Apache--2.0-blue)

</div>

---

## Overview

Honeypot Suite consolidates the former standalone `DNSpot`, `RDPpot`, and
`SMBpot` tools — plus nine more services — into one cohesive, growth-ready
codebase. Every honeypot implements the same small interface, is supervised by
a shared manager, and emits events through one structured logger, so the suite
behaves consistently no matter how many protocols you enable.

It is designed for **defensive use**: deception, early-warning, and attacker
telemetry on networks you own or are authorized to monitor.

## ✨ Features

- **12 protocols, one binary** — DNS, RDP, SMB, LDAP, NetBIOS, FTP, Telnet,
  SNMP, MySQL, MSSQL, Redis, HTTP.
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

| Variable | Default | Description |
|----------|---------|-------------|
| `HONEYPOT_DNS_ENABLED` | `false` | Enable the DNS pot |
| `HONEYPOT_DNS_LISTEN` | `0.0.0.0:53` | DNS bind address |
| `HONEYPOT_DNS_ANSWER_IPV4` | `192.168.1.1` | Fake A record |
| `HONEYPOT_DNS_ANSWER_IPV6` | `2001:db8::1` | Fake AAAA record |
| `HONEYPOT_RDP_ENABLED` | `false` | Enable the RDP pot |
| `HONEYPOT_RDP_LISTEN` | `0.0.0.0:3389` | RDP bind address |
| `HONEYPOT_RDP_CERT_ORG` | `sifirgun.tr` | Org name in the self-signed cert |
| `HONEYPOT_SMB_ENABLED` | `false` | Enable the SMB pot |
| `HONEYPOT_SMB_LISTEN` | `0.0.0.0:445` | SMB bind address |
| `HONEYPOT_LDAP_ENABLED` | `false` | Enable the LDAP pot |
| `HONEYPOT_LDAP_LISTEN` | `0.0.0.0:389` | LDAP bind address |
| `HONEYPOT_LDAP_BIND_RESULT` | `invalid` | Bind reply: `invalid` (49) or `success` (0) |
| `HONEYPOT_NETBIOS_ENABLED` | `false` | Enable the NetBIOS pot |
| `HONEYPOT_NETBIOS_LISTEN` | `0.0.0.0:137` | NetBIOS (UDP) bind address |
| `HONEYPOT_NETBIOS_ANSWER_IPV4` | `192.168.1.1` | Fake IP in name-query answers |
| `HONEYPOT_FTP_ENABLED` | `false` | Enable the FTP pot |
| `HONEYPOT_FTP_LISTEN` | `0.0.0.0:21` | FTP bind address |
| `HONEYPOT_FTP_BANNER` | `220 (vsFTPd 3.0.3)` | FTP greeting banner |
| `HONEYPOT_TELNET_ENABLED` | `false` | Enable the Telnet pot |
| `HONEYPOT_TELNET_LISTEN` | `0.0.0.0:23` | Telnet bind address |
| `HONEYPOT_TELNET_BANNER` | _(empty)_ | Optional pre-login banner |
| `HONEYPOT_SNMP_ENABLED` | `false` | Enable the SNMP pot |
| `HONEYPOT_SNMP_LISTEN` | `0.0.0.0:161` | SNMP (UDP) bind address |
| `HONEYPOT_MYSQL_ENABLED` | `false` | Enable the MySQL pot |
| `HONEYPOT_MYSQL_LISTEN` | `0.0.0.0:3306` | MySQL bind address |
| `HONEYPOT_MYSQL_VERSION` | `5.7.40` | Version advertised in the greeting |
| `HONEYPOT_MSSQL_ENABLED` | `false` | Enable the MSSQL pot |
| `HONEYPOT_MSSQL_LISTEN` | `0.0.0.0:1433` | MSSQL bind address |
| `HONEYPOT_REDIS_ENABLED` | `false` | Enable the Redis pot |
| `HONEYPOT_REDIS_LISTEN` | `0.0.0.0:6379` | Redis bind address |
| `HONEYPOT_REDIS_REQUIRE_AUTH` | `true` | Reply `NOAUTH` until an AUTH is seen |
| `HONEYPOT_HTTP_ENABLED` | `false` | Enable the HTTP pot |
| `HONEYPOT_HTTP_LISTEN` | `0.0.0.0:80` | HTTP bind address |
| `HONEYPOT_HTTP_SERVER` | `nginx` | `Server:` response header |
| `HONEYPOT_HTTP_REALM` | `Restricted` | Basic-auth realm in 401 responses |

</details>

<details>
<summary><b>Logging settings</b></summary>

| Variable | Default | Description |
|----------|---------|-------------|
| `HONEYPOT_LOG_DIR` | `./logs` | Log output directory |
| `HONEYPOT_LOG_CONSOLE` | `true` | Also print events to stdout |
| `HONEYPOT_LOG_LEVEL` | `info` | `info` or `debug` |

</details>

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
internal/ber/          shared ASN.1/BER helpers (used by ldap, snmp)
internal/testutil/     test helpers (free port, wait-for-listener)
internal/pots/
    dns/  rdp/  smb/  ldap/        one self-contained package per protocol
    netbios/  ftp/  telnet/  snmp/
    mysql/  mssql/  redis/  http/
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
