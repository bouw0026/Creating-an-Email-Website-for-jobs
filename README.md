# Creating-an-Email-Website-for-jobsOwn Your Domain + Email on Rocky Linux 9 — A Practical Guide (with One‑Shot Autoconfig Script)

A copy‑pasteable, vendor‑agnostic walkthrough to get a personal website + real inboxes (IMAP/SMTP) online on a single VPS. Designed for Rocky Linux 9. Tested with OVH/Hetzner/DigitalOcean/Linode. Includes a single Bash script that can bring up Apache + Let’s Encrypt + Postfix + Dovecot + OpenDKIM (+ optional Roundcube webmail) in ~10–15 minutes.

0) What you’ll build

Website: https://yourdomain (and www) served by Apache with auto‑renewed Let’s Encrypt TLS.

Email: real inboxes like you@yourdomain delivered by Postfix (SMTP) and read via Dovecot (IMAPS/POP3S). Auth over submission (587) with STARTTLS.

Deliverability: SPF, DKIM signing (OpenDKIM), DMARC policy, and reverse DNS guidance.

Optional webmail: Roundcube at https://yourdomain/webmail.

Firewall: sane defaults with firewalld.

Why this stack? It’s modern, minimal, and has clean defaults on Rocky 9.

1) Buying the pieces (my recommendation)

A. Domain

Any mainstream registrar works (Namecheap, Porkbun, Google Domains (migrated), Cloudflare Registrar, etc.). Priorities:

DNS editor that lets you add A/AAAA, MX, TXT records quickly.

WHOIS privacy.

Budget: ~$10–$20/yr for a .com or comparable TLD.

B. VPS plan (starter → comfy)

Tier

vCPU

RAM

NVMe

Traffic

Why

Starter

2

4 GB

40–60 GB

1–2 TB

Fine for a personal site + 2–3 inboxes

Comfort (your example)

4

8 GB

75 GB + 50 GB

2–4 TB

Plenty of headroom for Roundcube, logs, backups

Pick Rocky Linux 9 image. Choose the datacenter closest to your audience. Enable automated backups/snapshots if the provider offers a cheap toggle.

If your provider allows setting reverse DNS (PTR) in panel, plan to set it to mail.yourdomain after MX is live.

2) DNS you’ll need (summary table)

Create these at your registrar (or Cloudflare DNS). Replace yourdomain with the real domain and SERVER_IP with your VPS public IPv4.

Type

Name

Value

Purpose

A

@

SERVER_IP

apex → your site

A

www

SERVER_IP

www → your site

A

mail

SERVER_IP

mail host (MTA & IMAP)

MX

@

10 mail.yourdomain.

route mail to your server

TXT

@

v=spf1 a mx ~all

SPF allow web + mail host

TXT

_dmarc

v=DMARC1; p=quarantine; rua=mailto:dmarc@yourdomain; fo=1; adkim=s; aspf=s; pct=100

DMARC policy & reports

TXT

default._domainkey

(add after script prints the DKIM key)

DKIM public key

Optional (nice for clients):

autodiscover + autoconfig → A to SERVER_IP (or CNAME to mail).

SRV records for IMAP/Submission (not strictly required; most clients work fine without).

Reverse DNS (PTR): In your VPS panel, set the PTR for your IPv4 to mail.yourdomain. This plus DKIM/SPF/DMARC will do 90% of deliverability heavy lifting.

3) Ports to open (firewalld services)

ssh (22)

http (80), https (443)

smtp (25) — required to receive mail from the world

submission (587) — users send mail with auth

imaps (993) and/or pop3s (995)

4) Quick start (tl;dr)

Point @, www, mail, and MX to your server (Section 2). Wait a few minutes.

SSH to your fresh Rocky 9 server as root (or a sudoer).

Copy this script into /root/autoinstall.sh, edit the VARIABLES at the top, then run:

chmod +x /root/autoinstall.sh
sudo /root/autoinstall.sh | tee /root/setup.log

When the script prints your DKIM TXT value, paste it into DNS as default._domainkey.yourdomain.

Verify:

Visit https://yourdomain (auto‑HTTPS redirect).

IMAP: use mail.yourdomain, port 993, SSL/TLS, login with the Linux user you created.

SMTP: use mail.yourdomain, port 587, STARTTLS, same login.

Optional: https://yourdomain/webmail for Roundcube.

5) The one‑shot autoconfiguration script (Rocky Linux 9)

Note: Readable, idempotent-ish, and safe on a fresh Rocky 9 image. It:

Secures firewall, installs Apache + Certbot, Postfix, Dovecot, OpenDKIM, and optional Roundcube.

Reuses your Let’s Encrypt certs for Postfix/Dovecot.

Emits the DKIM DNS TXT you must add.

Creates one initial mailbox user.

#!/usr/bin/env bash
set -euo pipefail

#############################
# >>> EDIT ME <<<
#############################
DOMAIN="yourdomain"            # e.g., redxtale.com (no trailing dot)
WEB_HOST="${DOMAIN}"          # website host (keep as apex)
MAIL_HOST="mail.${DOMAIN}"    # mail host FQDN
ADMIN_EMAIL="admin@${DOMAIN}" # for Let’s Encrypt notifications
CREATE_USER="you"             # first mailbox user (a Linux user)
TZ_REGION="Etc/UTC"           # e.g., America/Toronto
INSTALL_WEBMAIL=true           # Roundcube at /webmail (true/false)
#############################

log() { echo -e "\e[36m[INFO]\e[0m $*"; }
warn(){ echo -e "\e[33m[WARN]\e[0m $*"; }
err() { echo -e "\e[31m[ERR ]\e[0m $*"; }
need() { command -v "$1" >/dev/null 2>&1 || { err "Missing $1"; exit 1; }; }

require_root() { [ "$(id -u)" -eq 0 ] || { err "Run as root"; exit 1; }; }

require_root

# 0) Sanity
hostnamectl set-hostname "${MAIL_HOST}"
timedatectl set-timezone "${TZ_REGION}" || true

log "Updating system & enabling EPEL"
dnf -y update
rpm -q epel-release >/dev/null 2>&1 || dnf -y install epel-release

log "Installing base packages"
dnf -y install firewalld policycoreutils-python-utils curl tar git vim
systemctl enable --now firewalld

# 1) Apache + Certbot
log "Installing Apache + Certbot"
dnf -y install httpd mod_ssl certbot python3-certbot-apache
systemctl enable --now httpd

# Web root & simple page
WEBROOT="/var/www/${WEB_HOST}/public"
mkdir -p "$WEBROOT"
chown -R apache:apache "/var/www/${WEB_HOST}"
cat >"$WEBROOT"/index.html <<HTML
<!doctype html><meta charset="utf-8"><title>${DOMAIN}</title>
<style>body{font-family:system-ui;margin:4rem auto;max-width:48rem;padding:0 1rem}code{background:#f3f3f3;padding:.2rem .4rem;border-radius:.2rem}</style>
<h1>${DOMAIN}</h1>
<p>If you can read this, Apache is serving your site.</p>
<p>Next: email is on <code>${MAIL_HOST}</code>.</p>
HTML

# Apache vhost (apex + www)
cat >/etc/httpd/conf.d/${WEB_HOST}.conf <<CONF
<VirtualHost *:80>
    ServerName ${WEB_HOST}
    ServerAlias www.${DOMAIN}
    DocumentRoot ${WEBROOT}
    ErrorLog  /var/log/httpd/${WEB_HOST}-error.log
    CustomLog /var/log/httpd/${WEB_HOST}-access.log combined
</VirtualHost>
CONF

apachectl configtest
systemctl reload httpd

# Open firewall for web + ssh now
for svc in ssh http https; do firewall-cmd --permanent --add-service=${svc}; done
firewall-cmd --reload

# Let’s Encrypt for web + mail hostnames
log "Requesting Let’s Encrypt certificates"
certbot --apache \
  -d ${WEB_HOST} -d www.${DOMAIN} -d ${MAIL_HOST} \
  --non-interactive --agree-tos -m ${ADMIN_EMAIL} --redirect

# 2) Postfix (SMTP) + Dovecot (IMAP/POP3)
log "Installing Postfix + Dovecot"
dnf -y install postfix dovecot dovecot-pigeonhole cyrus-sasl cyrus-sasl-plain mailx
systemctl enable postfix dovecot

# Postfix main.cf tweaks
postconf -e "myhostname = ${MAIL_HOST}"
postconf -e "mydomain = ${DOMAIN}"
postconf -e "myorigin = \$mydomain"
postconf -e "inet_interfaces = all"
postconf -e "inet_protocols = ipv4"
postconf -e "mydestination = \$myhostname, localhost.\$mydomain, localhost, \$mydomain"
postconf -e "home_mailbox = Maildir/"
postconf -e "smtp_tls_security_level = may"
postconf -e "smtpd_tls_security_level = may"
postconf -e "smtpd_tls_cert_file = /etc/letsencrypt/live/${WEB_HOST}/fullchain.pem"
postconf -e "smtpd_tls_key_file  = /etc/letsencrypt/live/${WEB_HOST}/privkey.pem"
postconf -e "smtpd_sasl_type = dovecot"
postconf -e "smtpd_sasl_path = private/auth"
postconf -e "smtpd_sasl_auth_enable = yes"
postconf -e "smtpd_recipient_restrictions = permit_mynetworks, permit_sasl_authenticated, reject_unauth_destination"
postconf -e "mynetworks = 127.0.0.0/8"

# Enable submission (587) + smtps (465) in master.cf if not present
postconf -M submission/inet || true
postconf -P submission/inet/syslog_name=postfix/submission
postconf -P submission/inet/smtpd_tls_security_level=encrypt
postconf -P submission/inet/smtpd_sasl_auth_enable=yes
postconf -P submission/inet/smtpd_client_restrictions=permit_sasl_authenticated,reject

postconf -M smtps/inet || true
postconf -P smtps/inet/syslog_name=postfix/smtps
postconf -P smtps/inet/smtpd_tls_wrappermode=yes
postconf -P smtps/inet/smtpd_sasl_auth_enable=yes

# Dovecot config (IMAP/POP3 over TLS)
cat >/etc/dovecot/conf.d/10-mail.conf <<'DOV'
mail_location = maildir:~/Maildir
namespace inbox {
  inbox = yes
}
DOV

cat >/etc/dovecot/conf.d/10-auth.conf <<'DOV'
auth_mechanisms = plain login
!include auth-system.conf.ext
DOV

cat >/etc/dovecot/conf.d/10-ssl.conf <<SSL
ssl = required
ssl_cert = </etc/letsencrypt/live/${WEB_HOST}/fullchain.pem
ssl_key  = </etc/letsencrypt/live/${WEB_HOST}/privkey.pem
SSL

# Allow Postfix to auth via Dovecot socket
mkdir -p /var/spool/postfix/private
cat >/etc/dovecot/conf.d/10-master.conf <<'DOV'
service auth {
  unix_listener /var/spool/postfix/private/auth {
    mode = 0660
    user = postfix
    group = postfix
  }
}
DOV

systemctl restart dovecot postfix

# Open firewall for mail
for svc in smtp submission imaps pop3s; do firewall-cmd --permanent --add-service=$svc; done
firewall-cmd --reload

# SELinux: allow Apache/PHP to talk to local SMTP (Roundcube)
setsebool -P httpd_can_network_connect 1 || true

# 3) OpenDKIM (DKIM signing)
log "Installing and configuring OpenDKIM"
dnf -y install opendkim opendkim-tools

cat >/etc/opendkim.conf <<ODK
Syslog                  yes
UMask                   002
Mode                    sv
Socket                  inet:8891@localhost
OversignHeaders         From
Selector                default
Domain                  ${DOMAIN}
KeyFile                 /etc/opendkim/keys/${DOMAIN}/default.private
AutoRestart             yes
AutoRestartRate         10/1h
Canonicalization        relaxed/simple
SignatureAlgorithm      rsa-sha256
ODK

mkdir -p /etc/opendkim/keys/${DOMAIN}
cd /etc/opendkim/keys/${DOMAIN}
opendkim-genkey -s default -d ${DOMAIN}
chown -R opendkim:opendkim /etc/opendkim
chmod 600 /etc/opendkim/keys/${DOMAIN}/default.private

# Hook OpenDKIM into Postfix
postconf -e "milter_default_action = accept"
postconf -e "milter_protocol = 6"
postconf -e "smtpd_milters = inet:localhost:8891"
postconf -e "non_smtpd_milters = inet:localhost:8891"

systemctl enable --now opendkim
systemctl restart postfix

# 4) First mailbox user
if ! id -u "${CREATE_USER}" >/dev/null 2>&1; then
  log "Creating user ${CREATE_USER} (you’ll be prompted for a password)"
  adduser "${CREATE_USER}"
  passwd  "${CREATE_USER}"
fi

# Ensure Maildir exists
sudo -u "${CREATE_USER}" bash -lc 'test -d ~/Maildir || (mkdir -p ~/Maildir/{cur,new,tmp})'

# 5) Optional: Roundcube webmail
if [ "${INSTALL_WEBMAIL}" = true ]; then
  log "Installing Roundcube webmail"
  dnf -y install roundcubemail php-intl php-mbstring php-json php-xml php-gd php-imap

  # Minimal config: point to IMAP/SMTP on this host
  cat >/etc/roundcubemail/config.inc.php <<RC
<?php
\$config = [];
\$config['db_dsnw'] = 'sqlite:////var/lib/roundcubemail/roundcube.db?mode=0640';
\$config['default_host'] = 'tls://${MAIL_HOST}';
\$config['default_port'] = 993;
\$config['smtp_server'] = 'tls://${MAIL_HOST}';
\$config['smtp_port'] = 587;
\$config['smtp_user'] = '%u';
\$config['smtp_pass'] = '%p';
\$config['product_name'] = 'Webmail';
\$config['des_key'] = '$(head -c24 /dev/urandom | base64)';
RC

  chown -R apache:apache /var/lib/roundcubemail
  chmod -R 0750 /var/lib/roundcubemail

  # Apache alias at /webmail
  cat >/etc/httpd/conf.d/roundcube.conf <<AP
Alias /webmail /usr/share/roundcubemail
<Directory /usr/share/roundcubemail>
  Options FollowSymLinks
  AllowOverride All
  Require all granted
</Directory>
AP

  apachectl configtest
  systemctl reload httpd
fi

# 6) Final output: DKIM + next steps
PUBKEY=$(awk '/p=/{print $0}' /etc/opendkim/keys/${DOMAIN}/default.txt | sed 's/.*(\(.*\)).*/\1/' || true)

cat <<EOS

=======================================
SETUP COMPLETE
=======================================

1) Add DKIM TXT in DNS:
   Name: default._domainkey.${DOMAIN}
   Type: TXT
   Value: $(tr -d '\n' </etc/opendkim/keys/${DOMAIN}/default.txt | sed 's/.*(//; s/).*//')

2) Confirm existing DNS:
   - A records: @, www, mail → your server IP
   - MX: @ → mail.${DOMAIN}
   - SPF (TXT @): v=spf1 a mx ~all
   - DMARC (TXT _dmarc): v=DMARC1; p=quarantine; rua=mailto:dmarc@${DOMAIN}; fo=1; adkim=s; aspf=s; pct=100
   - PTR (Reverse DNS): ${MAIL_HOST}

3) Test mail:
   - IMAP (SSL): server mail.${DOMAIN}, port 993, user ${CREATE_USER}
   - SMTP (STARTTLS): server mail.${DOMAIN}, port 587, user ${CREATE_USER}

4) Web:
   - https://${WEB_HOST} (site)
   - ${INSTALL_WEBMAIL:+https://${WEB_HOST}/webmail (Roundcube)}

Logs if you need them:
   - Apache: /var/log/httpd/*
   - Postfix: journalctl -u postfix -f
   - Dovecot: journalctl -u dovecot -f
   - OpenDKIM: journalctl -u opendkim -f

Pro tip: Send a message to a Gmail/Outlook address and check headers for SPF/DKIM/DMARC = PASS.
EOS

6) After the script: finishing touches

Add the DKIM TXT the script prints as default._domainkey.yourdomain.

In your VPS panel, set reverse DNS (PTR) to mail.yourdomain.

Create more inboxes: sudo adduser alice && sudo passwd alice (first login creates Maildir/).

Optional: create aliases in /etc/aliases then run newaliases (e.g., postmaster: you).

7) Email client settings (IMAP/SMTP)

Incoming (IMAP): mail.yourdomain, port 993, SSL/TLS, username = Linux username.

Outgoing (SMTP): mail.yourdomain, port 587, STARTTLS, same username/password.

POP3S (995) is enabled too, but IMAP is recommended.

8) Deliverability checklist (keep you out of spam)

✅ A, MX, SPF, DKIM, DMARC records present

✅ PTR (reverse DNS) → mail.yourdomain

✅ Don’t send bulk. Warm up slowly, keep complaint rates low.

✅ Send test emails to major providers; in the headers, look for spf=pass, dkim=pass, dmarc=pass.

Advanced (optional):

Set p=none in DMARC for a few days to collect reports, then move to quarantine → reject.

Add autodiscover/autoconfig DNS if you want mail apps to auto‑configure.

9) Security & maintenance

Updates: dnf update -y weekly or enable automatic updates (dnf-automatic).

Backups: enable VPS snapshots; back up /etc, /var/lib/letsencrypt, /etc/opendkim, /var/lib/roundcubemail.

Fail2ban (optional): protects SMTP/IMAP.

Logs: skim journalctl -u postfix -u dovecot -u opendkim -f when debugging.

10) Troubleshooting quickies

Web says insecure: run certbot certificates and systemctl status httpd.

Can’t send: check port 25 isn’t blocked by provider; ensure submission (587) works and auth succeeds.

Auth fails: tail -f /var/log/maillog (Postfix) & journalctl -u dovecot -f.

No DKIM: ensure OpenDKIM running and DNS TXT pasted exactly (no quotes doubled). Restart Postfix.

Deliverability meh: confirm PTR set; use a new, low‑volume domain, send to self first.

11) Appendix — Removing everything (if you need a clean slate)

systemctl stop httpd postfix dovecot opendkim
rm -rf /etc/httpd/conf.d/${DOMAIN}.conf /var/www/${DOMAIN}
rm -rf /etc/opendkim /etc/roundcubemail /usr/share/roundcubemail /var/lib/roundcubemail
rm -rf /etc/letsencrypt/live/${DOMAIN} /etc/letsencrypt/archive/${DOMAIN}
# Keep Postfix/Dovecot packages installed unless you want to purge

12) Personalizing for your portfolio

Edit /var/www/${DOMAIN}/public/index.html to put your portfolio or drop in your static site build.

Add alice@yourdomain, jobs@yourdomain (aliases), and a contact form that submits via SMTP to mail.${DOMAIN} with auth.

Happy shipping! If you want me to pre‑fill the script variables for your domain and provider, tell me your domain, VPS IP, and time zone.

