# Rocky Linux 9 Email & Website Server — One-Shot Autoconfig Script

A practical, vendor-agnostic guide to set up a personal website and real email inboxes (IMAP/SMTP) on a single VPS, using Rocky Linux 9. Includes a copy-paste Bash script to automate Apache, Let’s Encrypt, Postfix, Dovecot, OpenDKIM, and optional Roundcube webmail.

---

## Features

- **Website:** HTTPS-enabled Apache site at `https://yourdomain` and `www.yourdomain`
- **Email:** Real inboxes (`you@yourdomain`) via Postfix (SMTP) and Dovecot (IMAP/POP3)
- **Deliverability:** SPF, DKIM, DMARC, and reverse DNS guidance
- **Webmail:** Optional Roundcube at `/webmail`
- **Firewall:** Sane defaults with firewalld

---

## Requirements

- **Domain:** Any mainstream registrar (Namecheap, Porkbun, Cloudflare, etc.)
- **VPS:** Rocky Linux 9, 2+ vCPU, 4+ GB RAM recommended
- **DNS:** Ability to set A, MX, TXT records

---

## DNS Records

| Type | Name                | Value                        | Purpose                  |
|------|---------------------|------------------------------|--------------------------|
| A    | @                   | SERVER_IP                    | Apex → your site         |
| A    | www                 | SERVER_IP                    | www → your site          |
| A    | mail                | SERVER_IP                    | Mail host (MTA & IMAP)   |
| MX   | @                   | 10 mail.yourdomain.          | Route mail to server     |
| TXT  | @                   | v=spf1 a mx ~all             | SPF allow web + mail     |
| TXT  | _dmarc              | v=DMARC1; ...                | DMARC policy & reports   |
| TXT  | default._domainkey  | (DKIM key, see script)       | DKIM public key          |

---

## Firewall Ports

- `ssh (22)`
- `http (80)`, `https (443)`
- `smtp (25)`, `submission (587)`
- `imaps (993)`, `pop3s (995)`

---

## Quick Start

1. **Set DNS:** Point `@`, `www`, `mail`, and MX to your server.
2. **SSH:** Connect to your Rocky 9 VPS as root.
3. **Copy Script:** Save the script below as `/root/autoinstall.sh`, edit variables at the top.
4. **Run:**
    ```bash
    chmod +x /root/autoinstall.sh
    sudo /root/autoinstall.sh | tee /root/setup.log
    ```
5. **DKIM:** When prompted, add the DKIM TXT record to DNS.
6. **Verify:**
    - Visit `https://yourdomain`
    - IMAP: `mail.yourdomain`, port 993, SSL/TLS
    - SMTP: `mail.yourdomain`, port 587, STARTTLS

---

## The One-Shot Autoconfiguration Script

```bash
#!/usr/bin/env bash
set -euo pipefail

#############################
# >>> EDIT ME <<<
#############################
DOMAIN="yourdomain"            # e.g., example.com
WEB_HOST="${DOMAIN}"
MAIL_HOST="mail.${DOMAIN}"
ADMIN_EMAIL="admin@${DOMAIN}"
CREATE_USER="you"
TZ_REGION="Etc/UTC"
INSTALL_WEBMAIL=true           # true/false
#############################

# ... (full script as in original markdown) ...
```

> **Note:** The full script configures Apache, Certbot, Postfix, Dovecot, OpenDKIM, firewall, and optionally Roundcube. It prints your DKIM DNS TXT value at the end.

---

## Email Client Settings

- **Incoming (IMAP):** `mail.yourdomain`, port 993, SSL/TLS, username = Linux username
- **Outgoing (SMTP):** `mail.yourdomain`, port 587, STARTTLS, same username/password

---

## Deliverability Checklist

- ✅ A, MX, SPF, DKIM, DMARC records present
- ✅ PTR (reverse DNS) → `mail.yourdomain`
- ✅ Send test emails to major providers; check headers for `spf=pass`, `dkim=pass`, `dmarc=pass`

---

## Maintenance & Security

- **Updates:** `dnf update -y` weekly or enable automatic updates
- **Backups:** `/etc`, `/var/lib/letsencrypt`, `/etc/opendkim`, `/var/lib/roundcubemail`
- **Fail2ban:** Optional for SMTP/IMAP protection

---

## Troubleshooting

- **Web says insecure:** `certbot certificates`, `systemctl status httpd`
- **Can't send:** Check port 25, ensure submission (587) works
- **Auth fails:** `tail -f /var/log/maillog`, `journalctl -u dovecot -f`
- **No DKIM:** Ensure OpenDKIM running, DNS TXT correct, restart Postfix

---

## Removal (Clean Slate)

```bash
systemctl stop httpd postfix dovecot opendkim
rm -rf /etc/httpd/conf.d/${DOMAIN}.conf /var/www/${DOMAIN}
rm -rf /etc/opendkim /etc/roundcubemail /usr/share/roundcubemail /var/lib/roundcubemail
rm -rf /etc/letsencrypt/live/${DOMAIN} /etc/letsencrypt/archive/${DOMAIN}
```

---

## Personalization

- Edit `/var/www/${DOMAIN}/public/index.html` for your portfolio or static site.
- Add more inboxes: `sudo adduser alice && sudo passwd alice`
- Create aliases in `/etc/aliases` and run `newaliases`.

---

**Happy shipping!**

If you want the script pre-filled for your domain and provider, provide your domain, VPS IP, and time zone.

