# Mail Proxy on Ubuntu 24.04 — Complete Manual

> **Purpose:** A secure mail proxy on a public VPS that accepts plain (unencrypted) IMAP and SMTP connections from old devices, forwards them over TLS to Gmail and Yahoo, and protects against unauthorized use and spam abuse.

---

## Table of Contents

1. [Architecture](#architecture)
2. [Port Map](#port-map)
3. [Install Nginx](#install-nginx)
4. [Install stunnel](#install-stunnel)
5. [Install Other Dependencies](#install-other-dependencies)
6. [MaxMind GeoIP Database](#maxmind-geoip-database)
7. [Python Auth Backend](#python-auth-backend)
8. [Nginx Mail Proxy Config](#nginx-mail-proxy-config)
9. [stunnel TLS Backend Config](#stunnel-tls-backend-config)
10. [fail2ban](#fail2ban)
11. [Firewall (UFW)](#firewall-ufw)
12. [Start All Services](#start-all-services)
13. [Managing Users](#managing-users)
14. [Managing Blocked Countries](#managing-blocked-countries)
15. [Monitoring & Logs](#monitoring--logs)
16. [Troubleshooting](#troubleshooting)
17. [Quick Reference](#quick-reference)

---

## Architecture

```
[Old phone — plain IMAP/SMTP, no TLS]
        │
        │  plain TCP — public internet
        ▼
[UFW firewall — open ports 1143/1025/1144/1026]
        │
        ▼
[Nginx mail proxy — port 1143/1025/1144/1026]
        │  HTTP auth request per connection
        ▼
[Python auth backend — 127.0.0.1:8080]
        │  Step 1: GeoIP check — block CN, VN, KP, RU ...
        │  Step 2: Username whitelist check
        │  Step 3: Allow → return stunnel backend port
        │
        ▼ (allowed connections only)
[stunnel backend — 127.0.0.1:11430/10250/11440/10260]
        │
        │  TLS (verified against system CA bundle)
        ▼
[Gmail / Yahoo]
```

---

## Port Map

| Service | Client connects to | Internal handoff | Remote host |
|---|---|---|---|
| Gmail IMAP | `VPS_IP:1143` | `127.0.0.1:11430` | `imap.gmail.com:993` |
| Gmail SMTP | `VPS_IP:1025` | `127.0.0.1:10250` | `smtp.gmail.com:465` |
| Yahoo IMAP | `VPS_IP:1144` | `127.0.0.1:11440` | `imap.mail.yahoo.com:993` |
| Yahoo SMTP | `VPS_IP:1026` | `127.0.0.1:10260` | `smtp.mail.yahoo.com:465` |

> Ports `11430`, `10250`, `11440`, `10260` are localhost-only — not reachable from outside.

---

## Install Nginx

### 3.1 Install

```bash
sudo apt update
sudo apt install nginx nginx-extras
```

The `nginx-extras` package includes the mail proxy module required for IMAP/SMTP proxying.

### 3.2 Verify the mail module is present

```bash
nginx -V 2>&1 | grep mail
```

Expected output includes:
```
--with-mail --with-mail_ssl_module
```

If the mail module is missing, install the full package:

```bash
sudo apt install nginx-full
```

### 3.3 Verify Nginx is running

```bash
sudo systemctl status nginx
```

### 3.4 Key Nginx paths

| Path | Purpose |
|---|---|
| `/etc/nginx/nginx.conf` | Main configuration file |
| `/var/log/nginx/mail-access.log` | Mail proxy access log |
| `/var/log/nginx/mail-error.log` | Mail proxy error log |
| `/var/log/nginx/error.log` | General error log |

---

## Install stunnel

### 4.1 Install

```bash
sudo apt install stunnel4
```

### 4.2 Enable the service

By default stunnel4 is installed but not enabled:

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

### 4.3 Verify stunnel version

```bash
stunnel -version
```

### 4.4 Key stunnel paths

| Path | Purpose |
|---|---|
| `/etc/stunnel/` | Configuration directory |
| `/etc/default/stunnel4` | Service startup settings |
| `/var/log/stunnel4/stunnel.log` | stunnel log file |
| `/var/run/stunnel4/stunnel.pid` | PID file |

---

## Install Other Dependencies

```bash
sudo apt install fail2ban geoipupdate python3 python3-pip

# GeoIP2 Python library
sudo pip3 install geoip2 --break-system-packages
```

---

## MaxMind GeoIP Database

### 6.1 Create a free MaxMind account

Register at: https://www.maxmind.com/en/geolite2/signup

After registration, go to **My Account → Manage License Keys** and generate a license key.

### 6.2 Configure geoipupdate

Edit `/etc/GeoIP.conf`:

```ini
AccountID YOUR_ACCOUNT_ID
LicenseKey YOUR_LICENSE_KEY
EditionIDs GeoLite2-Country
```

### 6.3 Download the database

```bash
sudo geoipupdate
# Database saved to: /var/lib/GeoIP/GeoLite2-Country.mmdb
```

Verify:

```bash
ls -lh /var/lib/GeoIP/GeoLite2-Country.mmdb
```

### 6.4 Auto-update weekly

```bash
sudo crontab -e
# Add:
0 3 * * 1 geoipupdate
```

---

## Python Auth Backend

### 7.1 Create the script

Create `/etc/nginx/auth-backend.py`:

```python
#!/usr/bin/env python3
from http.server import HTTPServer, BaseHTTPRequestHandler
import geoip2.database
import logging

logging.basicConfig(
    filename="/var/log/mail-auth.log",
    level=logging.INFO,
    format="%(asctime)s %(message)s"
)

# ============================================================
# WHITELIST — approved email addresses
# ============================================================
ALLOWED_USERS = {
    "john@gmail.com",
    "alice@gmail.com",
    "bob@yahoo.com",
    "mary@yahoo.com",
}

# ============================================================
# BLOCKED COUNTRIES — ISO 3166-1 alpha-2 codes
# ============================================================
BLOCKED_COUNTRIES = {
    "KP",  # North Korea
    # Add more as needed
}

# ============================================================
# Route map — Nginx auth URL path → stunnel backend port
# ============================================================
ROUTE_MAP = {
    "/auth-gmail-imap": "11430",
    "/auth-gmail-smtp": "10250",
    "/auth-yahoo-imap": "11440",
    "/auth-yahoo-smtp": "10260",
}

# Load GeoIP database once at startup
geo_reader = geoip2.database.Reader("/var/lib/GeoIP/GeoLite2-Country.mmdb")


def get_country(ip):
    try:
        response = geo_reader.country(ip)
        return response.country.iso_code
    except Exception:
        return None  # Unknown IP — let username check handle it


class AuthHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        user      = self.headers.get("Auth-User", "").lower()
        client_ip = self.headers.get("Client-IP", "")
        port      = ROUTE_MAP.get(self.path)

        # Step 1: GeoIP check
        country = get_country(client_ip)
        if country in BLOCKED_COUNTRIES:
            logging.warning(f"BLOCKED country={country} ip={client_ip} user={user}")
            self.deny("Access denied")
            return

        # Step 2: Username whitelist check
        if not port or user not in ALLOWED_USERS:
            logging.warning(f"BLOCKED unknown user={user} ip={client_ip} country={country}")
            self.deny("Invalid login")
            return

        # Step 3: Allow
        logging.info(f"ALLOWED user={user} ip={client_ip} country={country}")
        self.send_response(200)
        self.send_header("Auth-Status", "OK")
        self.send_header("Auth-Server", "127.0.0.1")
        self.send_header("Auth-Port", port)
        self.end_headers()

    def deny(self, reason):
        self.send_response(200)  # Nginx requires HTTP 200 always
        self.send_header("Auth-Status", reason)
        self.end_headers()

    def log_message(self, format, *args):
        pass  # suppress default HTTP server logs


if __name__ == "__main__":
    server = HTTPServer(("127.0.0.1", 8080), AuthHandler)
    print("Auth backend running on 127.0.0.1:8080")
    server.serve_forever()
```

### 7.2 Create systemd service

Create `/etc/systemd/system/mail-auth.service`:

```ini
[Unit]
Description=Nginx Mail Auth Backend
After=network.target

[Service]
ExecStart=/usr/bin/python3 /etc/nginx/auth-backend.py
Restart=always
User=www-data

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable mail-auth
sudo systemctl start mail-auth
```

---

## Nginx Mail Proxy Config

Edit `/etc/nginx/nginx.conf`:

```nginx
user www-data;
worker_processes auto;
pid /run/nginx.pid;

events {
    worker_connections 1024;
}

mail {
    access_log /var/log/nginx/mail-access.log;
    error_log  /var/log/nginx/mail-error.log info;

    # ---------------------------------------------------
    # Gmail IMAP — port 1143
    # ---------------------------------------------------
    server {
        listen    1143;
        protocol  imap;
        proxy     on;
        starttls  off;
        auth_http 127.0.0.1:8080/auth-gmail-imap;
        auth_http_header Client-IP $remote_addr;
    }

    # ---------------------------------------------------
    # Gmail SMTP — port 1025
    # ---------------------------------------------------
    server {
        listen    1025;
        protocol  smtp;
        proxy     on;
        starttls  off;
        smtp_auth login plain;
        auth_http 127.0.0.1:8080/auth-gmail-smtp;
        auth_http_header Client-IP $remote_addr;
    }

    # ---------------------------------------------------
    # Yahoo IMAP — port 1144
    # ---------------------------------------------------
    server {
        listen    1144;
        protocol  imap;
        proxy     on;
        starttls  off;
        auth_http 127.0.0.1:8080/auth-yahoo-imap;
        auth_http_header Client-IP $remote_addr;
    }

    # ---------------------------------------------------
    # Yahoo SMTP — port 1026
    # ---------------------------------------------------
    server {
        listen    1026;
        protocol  smtp;
        proxy     on;
        starttls  off;
        smtp_auth login plain;
        auth_http 127.0.0.1:8080/auth-yahoo-smtp;
        auth_http_header Client-IP $remote_addr;
    }
}
```

Test and reload:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## stunnel TLS Backend Config

Create `/etc/stunnel/backend.conf`:

```ini
; stunnel backend — forwards to Gmail/Yahoo over TLS
; Listens on localhost only — not reachable from outside

client = yes

; Verify Gmail/Yahoo certificates using Ubuntu system CA bundle
CAfile = /etc/ssl/certs/ca-certificates.crt
verify = 2

[gmail-imap-out]
accept  = 127.0.0.1:11430
connect = imap.gmail.com:993

[gmail-smtp-out]
accept  = 127.0.0.1:10250
connect = smtp.gmail.com:465

[yahoo-imap-out]
accept  = 127.0.0.1:11440
connect = imap.mail.yahoo.com:993

[yahoo-smtp-out]
accept  = 127.0.0.1:10260
connect = smtp.mail.yahoo.com:465
```

```bash
sudo systemctl enable stunnel4
sudo systemctl start stunnel4
```

---

## fail2ban

### 10.1 Create Nginx mail auth filter

Create `/etc/fail2ban/filter.d/nginx-mail-auth.conf`:

```ini
[Definition]
failregex = \[info\] \d+#\d+: \*\d+ client login failed:.*<HOST>
ignoreregex =
```

### 10.2 Create jail for mail proxy

Create `/etc/fail2ban/jail.d/nginx-mail.conf`:

```ini
[nginx-mail-auth]
enabled   = true
port      = 1143,1025,1144,1026
filter    = nginx-mail-auth
logpath   = /var/log/nginx/mail-error.log

# Ban after 5 failed attempts within 10 minutes
maxretry  = 5
findtime  = 600

# Ban for 1 hour
bantime   = 3600

# Never ban localhost or your own LAN
ignoreip  = 127.0.0.1/8 192.168.1.0/24
```

### 10.3 Create jail for SSH

Create `/etc/fail2ban/jail.d/sshd.conf`:

```ini
[sshd]
enabled  = true
port     = ssh
maxretry = 5
findtime = 600
bantime  = 3600
ignoreip = 127.0.0.1/8 192.168.1.0/24
```

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

---

## Firewall (UFW)

```bash
# Allow SSH first — never lock yourself out
sudo ufw allow ssh

# Allow mail proxy ports from public internet
sudo ufw allow 1143/tcp comment "Mail proxy Gmail IMAP"
sudo ufw allow 1025/tcp comment "Mail proxy Gmail SMTP"
sudo ufw allow 1144/tcp comment "Mail proxy Yahoo IMAP"
sudo ufw allow 1026/tcp comment "Mail proxy Yahoo SMTP"

# Enable UFW
sudo ufw enable
sudo ufw status verbose
```

> Ports `11430`, `10250`, `11440`, `10260` (stunnel backends) and `8080` (auth backend) are localhost-only — **do not open them in UFW**.

---

## Start All Services

```bash
sudo systemctl start mail-auth
sudo systemctl start nginx
sudo systemctl start stunnel4
sudo systemctl start fail2ban

# Verify all are running
sudo systemctl status mail-auth nginx stunnel4 fail2ban
```

---

## Managing Users

### Add a user

Edit `/etc/nginx/auth-backend.py`, add their email to `ALLOWED_USERS`:

```python
ALLOWED_USERS = {
    "john@gmail.com",
    "alice@gmail.com",
    "newuser@gmail.com",   # ← added
}
```

Restart the auth backend:

```bash
sudo systemctl restart mail-auth
```

### Remove a user

Delete or comment out their email from `ALLOWED_USERS`, then restart:

```bash
sudo systemctl restart mail-auth
```

---

## Managing Blocked Countries

Edit `BLOCKED_COUNTRIES` in `/etc/nginx/auth-backend.py`:

```python
BLOCKED_COUNTRIES = {
    "CN",  # China
    "VN",  # Vietnam
    "KP",  # North Korea
    "RU",  # Russia
    "ID",  # Indonesia  ← add
    "NG",  # Nigeria    ← add
}
```

Restart:

```bash
sudo systemctl restart mail-auth
```

### Common country ISO codes

| Country | Code | Country | Code |
|---|---|---|---|
| China | `CN` | Russia | `RU` |
| Vietnam | `VN` | Indonesia | `ID` |
| North Korea | `KP` | Nigeria | `NG` |
| Iran | `IR` | Bangladesh | `BD` |
| Brazil | `BR` | Pakistan | `PK` |

Full list: https://www.iso.org/obp/ui/#search

---

## Monitoring & Logs

### Auth backend log (most useful)

```bash
sudo tail -f /var/log/mail-auth.log
```

Example output:
```
2024-07-30 12:01:03 BLOCKED country=CN ip=1.2.3.4 user=spammer@gmail.com
2024-07-30 12:01:15 BLOCKED unknown user=nobody@gmail.com ip=5.6.7.8 country=US
2024-07-30 12:02:44 ALLOWED user=john@gmail.com ip=9.10.11.12 country=DE
```

### Nginx mail logs

```bash
sudo tail -f /var/log/nginx/mail-error.log
sudo tail -f /var/log/nginx/mail-access.log
```

### fail2ban status

```bash
# All jails
sudo fail2ban-client status

# Mail proxy jail
sudo fail2ban-client status nginx-mail-auth

# SSH jail
sudo fail2ban-client status sshd

# Watch bans in real time
sudo tail -f /var/log/fail2ban.log
```

### stunnel log

```bash
sudo tail -f /var/log/stunnel4/stunnel.log
```

### Unban an IP manually

```bash
sudo fail2ban-client set nginx-mail-auth unbanip 203.0.113.10
```

---

## Troubleshooting

### Service fails to start

```bash
sudo journalctl -u mail-auth -n 50
sudo journalctl -u nginx -n 50
sudo journalctl -u stunnel4 -n 50
```

### Common issues

| Symptom | Cause | Fix |
|---|---|---|
| Nginx won't start | Config syntax error | `sudo nginx -t` |
| Mail module missing | Wrong nginx package | `sudo apt install nginx-extras` |
| Auth backend not responding | Python script crashed | `sudo systemctl restart mail-auth` |
| GeoIP lookup fails | Database not downloaded | `sudo geoipupdate` |
| stunnel can't verify Gmail cert | Outdated CA bundle | `sudo update-ca-certificates` |
| Connection refused from phone | UFW blocking | `sudo ufw status` |
| Auth backend can't read GeoIP DB | Wrong path or permissions | `ls -la /var/lib/GeoIP/` |

### Test auth backend manually

```bash
# Test allowed user from a non-blocked country
curl -v "http://127.0.0.1:8080/auth-gmail-imap" \
  -H "Auth-User: john@gmail.com" \
  -H "Auth-Pass: password" \
  -H "Auth-Protocol: imap" \
  -H "Client-IP: 9.10.11.12"
```

Expected response for allowed user:
```
Auth-Status: OK
Auth-Server: 127.0.0.1
Auth-Port: 11430
```

Expected response for blocked user or country:
```
Auth-Status: Invalid login
```

### Test stunnel backend

```bash
# Should connect to Gmail IMAP
telnet 127.0.0.1 11430
# Expected: * OK Gimap ready...
```

### Check all listening ports

```bash
sudo ss -tlnp | grep -E '1143|1025|1144|1026|8080|11430|10250|11440|10260'
```

---

## Quick Reference

### Mail client settings (on phone)

| Service | Protocol | Server | Port | Encryption |
|---|---|---|---|---|
| Gmail | IMAP | `YOUR.VPS.IP` | `1143` | None |
| Gmail | SMTP | `YOUR.VPS.IP` | `1025` | None |
| Yahoo | IMAP | `YOUR.VPS.IP` | `1144` | None |
| Yahoo | SMTP | `YOUR.VPS.IP` | `1026` | None |

> Use your Gmail/Yahoo **app password** if 2FA is enabled:
> - Gmail: https://myaccount.google.com/apppasswords
> - Yahoo: https://login.yahoo.com/account/security

### Service management

```bash
# Restart everything after config changes
sudo systemctl restart mail-auth nginx stunnel4 fail2ban

# Check all statuses at once
sudo systemctl status mail-auth nginx stunnel4 fail2ban
```

### Defence layers summary

| Layer | What it blocks |
|---|---|
| **GeoIP check** (Python) | Connections from blocked countries |
| **Username whitelist** (Python) | Anyone not manually approved |
| **fail2ban** | IPs with repeated failed attempts |
| **stunnel CAfile** | Man-in-the-middle attacks toward Gmail/Yahoo |
| **UFW** | Everything except the four mail ports and SSH |

---

*Manual covers: Ubuntu 24.04 LTS · Nginx mail proxy · stunnel4 · Python 3 · GeoIP2 · fail2ban*
