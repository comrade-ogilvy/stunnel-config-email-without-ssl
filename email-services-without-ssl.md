# Working Email Services for old phones and devices
> **Problem:** Due to expired certificates, smartphones running WP7.x, Android <=4.4, some old non-smartphones and other devices cannot connect to any public email services (Gmail, Yahoo, mail.com, Hotmail/Outlook, etc.). A service that works **without SSL** is required.

---

## 1. eno.one — Free

**Website:** https://eno.one

A social network reminiscent of early Twitter. Registration requires no personal data (no phone number). An alternative email for password reset is optional.

After registration you get a username which is also your email address: `FirstName.LastName@eno.one`

- **Webmail:** https://eno.one/mail/
- **Client setup info:** https://eno.one/mail/client/

| Setting | Value |
|---|---|
| Username | your full email address |
| Password | your account password |
| Servers (POP, IMAP, SMTP) | `mail.eno.one` |
| IMAP port | `143` (with or without SSL/TLS) |
| POP port | `110` (with or without SSL/TLS) |
| SMTP port | `587` (with or without SSL/TLS) |
| Encryption | Optional — TLS (STARTTLS) |

Everything is **free**.

---

## 2. purelymail.com — $10/year

**Website:** https://purelymail.com

Price includes one email address in the `purelymail.com` domain plus email hosting for an **unlimited number of custom domains**.

| Setting | Value |
|---|---|
| Login | your email address (including custom domain addresses) |
| Password | your mailbox password |
| IMAP | `imap.purelymail.com:993` (SSL/TLS) or `:143` (no SSL) |
| SMTP | `smtp.purelymail.com:465` (SSL/TLS) or `:587` (STARTTLS) or `:587` (no SSL) |

> **Tip:** Numeric domains of 6–9 digits in the `.xyz` zone cost **$1/year permanently** (not just the first year): https://gen.xyz/number

---

## 3. juno.com — POP3/SMTP only (no IMAP)

**Website:** https://juno.com

> **Note:** Requires a US phone number for registration.

| Setting | Value |
|---|---|
| Protocol | POP3 and SMTP only — no IMAP |

---

## Summary

| Service | IMAP | POP3 | SMTP | No-SSL ports | Price | Notes |
|---|---|---|---|---|---|---|
| eno.one | ✅ | ✅ | ✅ | ✅ | Free | No personal data required |
| purelymail.com | ✅ | ✅ | ✅ | ✅ | $10/year | Custom domains supported |
| juno.com | ❌ | ✅ | ✅ | ✅ | Free | US phone number required |

---

*Originally published at [symnok Blog](https://symnok.blogspot.com/) — [July 30, 2024](https://symnok.blogspot.com/2024/07/working-email-services-for-windows.html)*
