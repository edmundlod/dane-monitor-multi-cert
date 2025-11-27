# DANE TLSA Monitoring
## About
This script automates the validation of DANE TLSA records, checking for consistency across authoritative nameservers, DNSSEC validation, and TLS certificate validation for multiple certificate types.

It also handles alerts and logging based on the validation results.

## Requirements and Installation

### System Requirements

### Required Packages

**Debian/Ubuntu:**
```bash
sudo apt install \
    bash \
    openssl \
    bind9-dnsutils \
    postfix \
    mailutils \
    util-linux
```

**Fedora/RHEL:**
```bash
sudo dnf install \
    bash \
    openssl \
    bind-utils \
    postfix \
    mailx \
    util-linux
```

### Package Details

| Package | Provides | Used For |
|---------|----------|----------|
| `bash` | bash 4.0+ | Script execution |
| `openssl` | openssl CLI | TLS certificate validation, DANE support |
| `bind9-dnsutils`/`bind-utils` | dig, delv | DNS queries and DNSSEC validation |
| `postfix` | postconf | Postfix configuration management |
| `mailutils`/`mailx` | mail command | Email alerts |
| `util-linux` | logger | Syslog logging |

### Custom Requirements

1. **danesmtp-extended script**
   - Location: `/usr/local/bin/danesmtp-extended`
   - Source: github.com/edmundlod/danesmtp-extended
   - Must support `-s` flag for signature algorithm selection
   - Must be executable: `chmod +x /usr/local/bin/danesmtp-extended`

2. **Working mail system**
   - Postfix (or other MTA) must be configured to send outbound email

3. **DANE-enabled Postfix**
   - Postfix compiled with DANE support
   - Check with: `postconf -d | grep smtp_dns_support_level`
   - Should show: `smtp_dns_support_level = dnssec`

## Verification

### Check OpenSSL DANE Support
```bash
openssl s_client -help 2>&1 | grep dane
# Should show: -dane_tlsa_domain, -dane_tlsa_rrdata, etc.
```

### Check Postfix DANE Support
```bash
postconf -d | grep smtp_dns_support_level
# Should show: smtp_dns_support_level = dnssec

postconf -d | grep smtp_tls_security_level  
# Shows available security levels including 'dane'
```

### Check danesmtp-extended
Depending on which certificate you have (RSA or ECDSA):
```bash
/usr/local/bin/danesmtp-extended -s ecdsa mail.domain.com
# Should successfully connect and validate
```
or:
```
/usr/local/bin/danesmtp-extended -s rsa mail.domain.com
# Should successfully connect and validate
```


## Installation

### 1. Install the Script
```bash
sudo mv dane-monitor-multi-cert /usr/local/bin/
sudo chmod +x /usr/local/bin/dane-monitor-multi-cert
```

### 2. Configure
Edit the CONFIGURATION section at the top of the script:
```bash
sudo $EDITOR /usr/local/bin/dane-monitor-multi-cert
```

Set:
- `MX_HOST` - Your mail server hostname
- `DOMAIN` - Your domain
- `PRIMARY_EMAIL` - Primary alert email
- `BACKUP_EMAIL` - Backup alert email (different domain recommended)
- `AUTH_NS` - Your authoritative nameservers

### 3. Test
```bash
# Run manually
sudo /usr/local/bin/dane-monitor-multi-cert

# Check output - should show all checks passing
```

### 4. Set Up Automated Monitoring

**Option A: Systemd Timer (Recommended)**

Create service file:
```bash
sudo $EDITOR /etc/systemd/system/dane-monitor-multi-cert.service
```
```ini
[Unit]
Description=DANE TLSA Record Monitoring
After=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/dane-monitor-multi-cert
StandardOutput=journal
StandardError=journal
```

Create timer file:
```bash
sudo $EDITOR /etc/systemd/system/dane-monitor-multi-cert.timer
```
```ini
[Unit]
Description=DANE TLSA Monitoring Timer
Requires=dane-monitor.service

[Timer]
OnBootSec=5min
OnUnitActiveSec=30min
AccuracySec=1min

[Install]
WantedBy=timers.target
```
```bash
# Enable and start
sudo systemctl daemon-reload
sudo systemctl enable dane-monitor.timer
sudo systemctl start dane-monitor.timer

# Verify
systemctl status dane-monitor.timer
systemctl list-timers dane-monitor.timer
```

**Option B: Cron**
```bash
crontab -e
```
Add:
```cron
# DANE monitoring - hourly checks
0 * * * * /usr/local/bin/dane-monitor-multi-cert 2>&1 | logger -t dane-monitor-cron
```

### 5. Monitor Logs
```bash
# Systemd
journalctl -u dane-monitor-multi-cert.service -f

# Syslog
tail -f /var/log/syslog | grep dane-monitor
```

- Check logs: `journalctl -u dane-monitor-multi-cert.service`
- Run manual test: `/usr/local/bin/dane-monitor-multi-cert`
- Test individual components: `danesmtp mail.domain.tld
