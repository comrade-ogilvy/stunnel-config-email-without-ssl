# stunnel on Ubuntu 24.04 — Complete Manual

> **Use case covered:** Proxy server on Ubuntu 24.04 that accepts plain (unencrypted) IMAP/SMTP connections from LAN clients (e.g. mobile phones) and forwards them over TLS to Gmail and Yahoo Mail.

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Installation](#installation)
4. [Enable the Service](#enable-the-service)
5. [Certificates](#certificates)
6. [Server Configuration](#server-configuration)
7. [Firewall Rules](#firewall-rules)
8. [Start & Verify](#start--verify)
9. [Mail Client Setup (Mobile)](#mail-client-setup-mobile)
10. [Security Considerations](#security-considerations)
11. [Troubleshooting](#troubleshooting)
12. [Quick Reference](#quick-reference)

---

## Overview

**stunnel** is an SSL/TLS encryption wrapper. It sits between a client and a remote server, handling the TLS handshake transparently — so legacy or limited clients that have no TLS support can still communicate securely with TLS-only services.

**Key concepts:**

| Term | Meaning |
|---|---|
| `client = yes` | stunnel *initiates* TLS (acts as TLS client toward the remote server) |
| `client = no` | stunnel *accepts* TLS (acts as TLS server toward the connecting client) |
| `accept` | Local address:port stunnel listens on |
| `connect` | Remote address:port stunnel forwards to |
| `CAfile` | Certificate Authority bundle used to verify the remote server |
| `cert` | Your own certificate (needed only for mutual TLS) |

In this use case `client = yes` because the Ubuntu box initiates TLS toward Gmail/Yahoo.

---

## Architecture

```
Mobile phone (plain IMAP/SMTP, no TLS)
        │
        │  plain TCP on LAN
        ▼
┌─────────────────────────────────┐
│  Ubuntu 24.04 — stunnel server  │
│                                 │
│  listens: 0.0.0.0:1143  (IMAP)  │
│  listens: 0.0.0.0:1025  (SMTP)  │
│  listens: 0.0.0.0:1144  (IMAP)  │
│  listens: 0.0.0.0:1026  (SMTP)  │
└─────────────┬───────────────────┘
              │  TLS (encrypted) over Internet
              ▼
   imap.gmail.com:993      smtp.gmail.com:465
   imap.mail.yahoo.com:993 smtp.mail.yahoo.com:465
```

> **Why non-standard ports (1143, 1025, etc.)?**  
> Standard ports 143 and 25 require root to bind. Ports above 1024 can be bound by the `stunnel4` system user, which is safer.

---

## Installation

```bash
sudo apt update
sudo apt install stunnel4

# Confirm installed version
stunnel -version
```

After installation, the package creates:

| Path | Purpose |
|---|---|
| `/etc/stunnel/` | Configuration directory |
| `/etc/default/stunnel4` | Service startup settings |
| `/lib/systemd/system/stunnel4.service` | systemd unit file |
| `/var/log/stunnel4/` | Log directory |

---

## Enable the Service

By default the service is disabled. Enable it before starting:

```bash
sudo nano /etc/default/stunnel4
```

Change:

```
ENABLED=0
```

to:

```
ENABLED=1
```

Save and close.

---

## Certificates

### Use case: client = yes (this manual)

When stunnel acts as a TLS **client** toward Gmail/Yahoo, it does **not** need its own certificate. It only needs to *verify* the remote server's certificate.

Ubuntu ships with a system CA bundle that already trusts Gmail and Yahoo:

```
/etc/ssl/certs/ca-certificates.crt
```

No manual certificate generation is required for this use case.

---

### Optional: generate a self-signed cert (for use case #2 — accepting TLS from clients)

If you ever need stunnel to *accept* TLS connections (e.g. clients that *do* support TLS), generate a self-signed certificate:

```bash
sudo openssl req -new -x509 -days 3650 -nodes \
  -out /etc/stunnel/stunnel.pem \
  -keyout /etc/stunnel/stunnel.pem

sudo chmod 600 /etc/stunnel/stunnel.pem
```

> Both the certificate and the private key are stored in the same `.pem` file. Keep it readable only by root/stunnel4.

---

## Server Configuration

Create (or replace) the main config file:

```bash
sudo nano /etc/stunnel/stunnel.conf
```

Paste the following:

```ini
; ============================================================
; Global settings
; ============================================================
pid    = /var/run/stunnel4/stunnel.pid
output = /var/log/stunnel4/stunnel.log
setuid = stunnel4
setgid = stunnel4

; This machine initiates TLS toward Gmail/Yahoo
client = yes

; Verify remote server certificates using Ubuntu's CA bundle
CAfile = /etc/ssl/certs/ca-certificates.crt
verify = 2

; ============================================================
; GMAIL
; ============================================================

[gmail-imap]
; Mobile connects here (plain, no TLS)
accept  = 0.0.0.0:1143
; stunnel connects here over TLS
connect = imap.gmail.com:993

[gmail-smtp]
accept  = 0.0.0.0:1025
connect = smtp.gmail.com:465

; ============================================================
; YAHOO MAIL
; ============================================================

[yahoo-imap]
accept  = 0.0.0.0:1144
connect = imap.mail.yahoo.com:993

[yahoo-smtp]
accept  = 0.0.0.0:1026
connect = smtp.mail.yahoo.com:465
```

### Configuration options explained

| Directive | Value | Meaning |
|---|---|---|
| `client = yes` | — | stunnel connects to remote as TLS client |
| `verify = 2` | — | Verify remote cert against CAfile (recommended) |
| `verify = 0` | — | Skip verification (not recommended, use only for testing) |
| `accept = 0.0.0.0:PORT` | — | Listen on all interfaces (reachable from LAN) |
| `accept = 127.0.0.1:PORT` | — | Listen on localhost only (not reachable from LAN) |
| `connect` | host:port | Remote server to forward to |

---

## Firewall Rules

Open the four ports so LAN devices can reach the proxy:

```bash
sudo ufw allow 1143/tcp comment "stunnel Gmail IMAP"
sudo ufw allow 1025/tcp comment "stunnel Gmail SMTP"
sudo ufw allow 1144/tcp comment "stunnel Yahoo IMAP"
sudo ufw allow 1026/tcp comment "stunnel Yahoo SMTP"

# Reload and verify
sudo ufw reload
sudo ufw status
```

### Restrict to LAN subnet only (recommended)

Replace `0.0.0.0` rules above with subnet-scoped rules, e.g. for `192.168.1.0/24`:

```bash
sudo ufw allow from 192.168.1.0/24 to any port 1143
sudo ufw allow from 192.168.1.0/24 to any port 1025
sudo ufw allow from 192.168.1.0/24 to any port 1144
sudo ufw allow from 192.168.1.0/24 to any port 1026
```

Or restrict the `accept` lines in `stunnel.conf` to the server's own LAN IP:

```ini
accept = 192.168.1.50:1143
```

---

## Start & Verify

```bash
# Enable on boot and start now
sudo systemctl enable stunnel4
sudo systemctl start stunnel4

# Check status
sudo systemctl status stunnel4

# Watch logs in real time
sudo tail -f /var/log/stunnel4/stunnel.log
```

### Test connectivity from the Ubuntu server itself

```bash
# Should connect to Gmail IMAP through the tunnel
telnet 127.0.0.1 1143

# You should see an IMAP banner like:
# * OK Gimap ready for requests ...
```

Press `Ctrl+]` then type `quit` to exit telnet.

---

## Mail Client Setup (Mobile)

Find the Ubuntu server's LAN IP first:

```bash
ip -4 addr show | grep inet
# e.g. 192.168.1.50
```

Configure the mail app on the phone as follows:

### Gmail

| Setting | Value |
|---|---|
| Incoming server (IMAP) | `192.168.1.50` |
| IMAP port | `1143` |
| Outgoing server (SMTP) | `192.168.1.50` |
| SMTP port | `1025` |
| Encryption | **None** |
| Username | your full Gmail address |
| Password | Gmail app password (see note below) |

### Yahoo Mail

| Setting | Value |
|---|---|
| Incoming server (IMAP) | `192.168.1.50` |
| IMAP port | `1144` |
| Outgoing server (SMTP) | `192.168.1.50` |
| SMTP port | `1026` |
| Encryption | **None** |
| Username | your full Yahoo address |
| Password | Yahoo app password (see note below) |

> **App Passwords:** Both Gmail and Yahoo require an *app-specific password* when connecting via IMAP/SMTP if you have 2-factor authentication enabled (which is strongly recommended). Generate one at:
> - Gmail: [myaccount.google.com/apppasswords](https://myaccount.google.com/apppasswords)
> - Yahoo: [login.yahoo.com/account/security](https://login.yahoo.com/account/security)

---

## Security Considerations

### What is protected

- **Internet traffic** (Ubuntu → Gmail/Yahoo) is fully TLS-encrypted and certificate-verified (`verify = 2`).
- Your credentials travel inside that TLS tunnel toward Google/Yahoo — they are safe on the internet leg.

### What is NOT protected

- **LAN traffic** (phone → Ubuntu stunnel) is **plain text**. Anyone on the same LAN can sniff credentials.

### Mitigations for LAN exposure

1. **Subnet restriction in UFW** (shown above) — limits who can connect to the proxy ports.
2. **Trusted LAN only** — only use this setup on a private, trusted network (home/office). Never on public Wi-Fi.
3. **Use case #2 alternative** — if you want end-to-end encryption even on LAN, run stunnel in server mode too (accepting TLS from clients). Requires clients that support TLS with custom certificates.

---

## Troubleshooting

### Service fails to start

```bash
sudo journalctl -u stunnel4 -n 50
sudo cat /var/log/stunnel4/stunnel.log
```

Common causes:

| Symptom | Fix |
|---|---|
| `ENABLED=0` | Edit `/etc/default/stunnel4`, set `ENABLED=1` |
| Port already in use | `sudo ss -tlnp \| grep 1143` — kill conflicting process |
| Permission denied on cert | `sudo chmod 600 /etc/stunnel/stunnel.pem` |
| `CAfile` not found | Verify path: `ls /etc/ssl/certs/ca-certificates.crt` |

### Certificate verification fails

If stunnel cannot verify Gmail/Yahoo certs:

```bash
# Update the CA bundle
sudo apt install --reinstall ca-certificates
sudo update-ca-certificates
```

### Connection refused from mobile

```bash
# Check stunnel is listening on all interfaces
sudo ss -tlnp | grep stunnel

# Check firewall
sudo ufw status verbose
```

### Test from another LAN machine

```bash
telnet 192.168.1.50 1143
# Should show IMAP banner from Gmail
```

---

## Quick Reference

### Port map

| Service | Protocol | Plain port (LAN) | TLS port (Internet) | Remote host |
|---|---|---|---|---|
| Gmail | IMAP | `1143` | `993` | `imap.gmail.com` |
| Gmail | SMTP | `1025` | `465` | `smtp.gmail.com` |
| Yahoo | IMAP | `1144` | `993` | `imap.mail.yahoo.com` |
| Yahoo | SMTP | `1026` | `465` | `smtp.mail.yahoo.com` |

### Useful commands

```bash
# Restart after config changes
sudo systemctl restart stunnel4

# Check listening ports
sudo ss -tlnp | grep stunnel

# View logs
sudo tail -f /var/log/stunnel4/stunnel.log

# Test config file syntax
sudo stunnel -fd 0 < /etc/stunnel/stunnel.conf
```

---

## License

This document is provided as-is for educational and operational use. stunnel is open-source software licensed under the GNU GPL v2.
