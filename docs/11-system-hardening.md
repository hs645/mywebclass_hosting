# Chapter 11: System Hardening

**Additional Security Measures and Automated Updates**

---

## Learning Objectives

By the end of this chapter, you'll be able to:
- ✅ Configure automatic security updates
- ✅ Install and configure rootkit detection
- ✅ Set up file integrity monitoring
- ✅ Enable system auditing
- ✅ Configure secure shared memory
- ✅ Harden network parameters
- ✅ Monitor system security

**Time Required:** 40-50 minutes

---

## Automatic Security Updates

### Why Automatic Updates?

**The problem:**
- New vulnerabilities discovered daily
- Manual updates take time
- Easy to forget
- Delay = risk

**Real example:**
- **Day 0:** Security flaw discovered in SSH
- **Day 1:** Fix released
- **Day 2:** Attackers exploit unpatched systems
- **Day 30:** You remember to update

**Solution:** Automatic security updates!

---

### Understanding Update Types

**Security updates:**
- Critical security fixes
- Should be applied immediately
- Low risk of breaking things
- **Auto-install these!**

**Regular updates:**
- Feature improvements
- Bug fixes
- Non-critical
- **Review these manually**

**Kernel updates:**
- Operating system core
- Require reboot
- Test first
- **Auto-install but schedule reboot**

---

### Install Unattended-Upgrades

**Install the package:**
```bash
sudo apt update
sudo apt install unattended-upgrades -y
```

**Verify installation:**
```bash
dpkg -l | grep unattended-upgrades
```

**Should show package installed.**

---

### Configure Automatic Updates

**Edit configuration:**
```bash
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
```

**Key settings to enable/verify:**

**1. Enable security updates:**
```
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}-security";
//  "${distro_id}:${distro_codename}-updates";
};
```

**Keep `-security` uncommented.**
**Comment out `-updates` (has //)** unless you want all updates.

---

**2. Enable automatic reboot if needed:**
```
Unattended-Upgrade::Automatic-Reboot "true";
```

**Uncomment and set to "true".**

---

**3. Set reboot time:**
```
Unattended-Upgrade::Automatic-Reboot-Time "03:00";
```

**Reboots at 3 AM if kernel updated.**
**Change to your low-traffic time.**

---

**4. Send email notifications (optional):**
```
Unattended-Upgrade::Mail "your@email.com";
Unattended-Upgrade::MailReport "on-change";
```

**Requires mail setup (postfix).**
**Options:** `always`, `only-on-error`, `on-change`

---

**5. Remove unused dependencies:**
```
Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";
Unattended-Upgrade::Remove-New-Unused-Dependencies "true";
Unattended-Upgrade::Remove-Unused-Dependencies "true";
```

**Keeps system clean automatically.**

---

### Enable the Service

**Enable automatic updates:**
```bash
sudo dpkg-reconfigure -plow unattended-upgrades
```

**Select:** `Yes`

**Verify enabled:**
```bash
sudo systemctl status unattended-upgrades
```

**Should show:**
```
● unattended-upgrades.service - Unattended Upgrades Shutdown
     Active: active (running)
```

---

### Configure Update Frequency

**Edit periodic settings:**
```bash
sudo nano /etc/apt/apt.conf.d/20auto-upgrades
```

**Recommended settings:**
```
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Download-Upgradeable-Packages "1";
APT::Periodic::AutocleanInterval "7";
APT::Periodic::Unattended-Upgrade "1";
```

**Explanation:**
- Update package lists: Daily
- Download upgrades: Daily
- Clean old packages: Weekly
- Install updates: Daily

---

### Test Automatic Updates

**Dry run (test without installing):**
```bash
sudo unattended-upgrades --dry-run --debug
```

**Shows what would be installed.**

**Manual run (for testing):**
```bash
sudo unattended-upgrade -d
```

**Check logs:**
```bash
sudo tail -f /var/log/unattended-upgrades/unattended-upgrades.log
```

---

### View Update History

**See what was auto-installed:**
```bash
sudo cat /var/log/unattended-upgrades/unattended-upgrades.log
```

**Recent updates:**
```bash
sudo tail -50 /var/log/unattended-upgrades/unattended-upgrades.log
```

**Package logs:**
```bash
sudo cat /var/log/apt/history.log
```

---

## Rootkit Detection

### What is a Rootkit?

**Rootkit = Hidden malware**

**What it does:**
- Hides itself from normal detection
- Gives attacker backdoor access
- Modifies system commands
- Steals data silently

**How you get infected:**
- Compromised package
- Exploited vulnerability
- Malicious script
- Social engineering

**Detection is critical!**

---

### Install rkhunter

**rkhunter = Rootkit Hunter**

**Install:**
```bash
sudo apt install rkhunter -y
```

**Update signatures:**
```bash
sudo rkhunter --update
```

**Output:**
```
[ Rootkit Hunter version 1.4.6 ]
Checking rkhunter data files...
  Checking file mirrors.dat                  [ OK ]
  Checking file programs_bad.dat             [ OK ]
  ...
```

---

### Configure rkhunter

**Edit configuration:**
```bash
sudo nano /etc/rkhunter.conf
```

**Key settings:**

**Enable email alerts:**
```
MAIL-ON-WARNING="your@email.com"
```

**Update after each check:**
```
UPDATE_MIRRORS=1
MIRRORS_MODE=0
```

**Allow SSH root login warning (we disabled it):**
```
ALLOW_SSH_ROOT_USER=no
```

**Check for hidden files:**
```
HIDDEN_FILE_CHECK=1
```

---

### Update Baseline

**After installing, create baseline:**
```bash
sudo rkhunter --propupd
```

**What this does:**
- Records current file signatures
- Creates baseline for comparison
- Future scans check against this

**Run after every legitimate system change!**

---

### Run Your First Scan

**Full system scan:**
```bash
sudo rkhunter --check --skip-keypress
```

**This scans:**
- System commands
- Network interfaces
- Hidden files
- Startup files
- Rootkit signatures

**Takes 2-5 minutes.**

**View results:**
```bash
sudo cat /var/log/rkhunter.log
```

---

### Interpret Results

**Green (OK):**
```
[OK] Check 1: System commands
```

**Yellow (Warning):**
```
[Warning] Check 2: SSH root login
```

**Red (Alert):**
```
[Alert] File '/usr/bin/ls' has been modified
```

**Common warnings (usually false positives):**
- SSH configuration warnings (expected after hardening)
- Package manager warnings
- System updates

**Real alerts need investigation!**

---

### Automate rkhunter Scans

**Create daily cron job:**
```bash
sudo nano /etc/cron.daily/rkhunter-scan
```

**Add:**
```bash
#!/bin/bash
/usr/bin/rkhunter --cronjob --update --quiet
```

**Make executable:**
```bash
sudo chmod +x /etc/cron.daily/rkhunter-scan
```

**Now runs daily automatically!**

---

## File Integrity Monitoring

### Install AIDE

**AIDE = Advanced Intrusion Detection Environment**

**What it does:**
- Creates database of all files
- Monitors for changes
- Detects unauthorized modifications
- Alerts on suspicious activity

**Install:**
```bash
sudo apt install aide -y
```

---

### Initialize AIDE Database

**Create initial database:**
```bash
sudo aideinit
```

**This takes 5-10 minutes!**

**Output:**
```
Running aide --init...
AIDE, version 0.17

### AIDE database at /var/lib/aide/aide.db.new initialized.
```

**Move database to active location:**
```bash
sudo mv /var/lib/aide/aide.db.new /var/lib/aide/aide.db
```

---

### Run AIDE Check

**Check for changes:**
```bash
sudo aide --check
```

**First run after init shows:**
```
AIDE, version 0.17

### All files match AIDE database. Looks okay!
```

**If files changed:**
```
### Files changed:
f = File
changed: /etc/ssh/sshd_config
```

---

### Configure AIDE

**Edit configuration:**
```bash
sudo nano /etc/aide/aide.conf
```

**Common exclusions (don't monitor these):**
```
!/var/log/.*
!/var/spool/.*
!/var/cache/.*
!/tmp/
```

**These change frequently and are expected.**

---

### Automate AIDE Checks

**Create daily check:**
```bash
sudo nano /etc/cron.daily/aide-check
```

**Add:**
```bash
#!/bin/bash
/usr/bin/aide --check | mail -s "AIDE Check $(hostname)" your@email.com
```

**Make executable:**
```bash
sudo chmod +x /etc/cron.daily/aide-check
```

**Update database after legitimate changes:**
```bash
sudo aide --update
sudo mv /var/lib/aide/aide.db.new /var/lib/aide/aide.db
```

---

## System Auditing with auditd

### What is auditd?

**auditd = System audit daemon**

**Logs:**
- File access
- System calls
- User actions
- Network connections
- Process execution

**Use cases:**
- Security compliance
- Forensics after incident
- Tracking privileged operations
- Compliance requirements (PCI-DSS, HIPAA)

---

### Install auditd

```bash
sudo apt install auditd audispd-plugins -y
```

**Start and enable:**
```bash
sudo systemctl start auditd
sudo systemctl enable auditd
```

**Verify:**
```bash
sudo systemctl status auditd
```

---

### Configure Basic Rules

**View current rules:**
```bash
sudo auditctl -l
```

**Add rules to config:**
```bash
sudo nano /etc/audit/rules.d/audit.rules
```

**Monitor important files:**
```bash
# Monitor /etc/passwd for changes
-w /etc/passwd -p wa -k passwd_changes

# Monitor /etc/shadow
-w /etc/shadow -p wa -k shadow_changes

# Monitor SSH config
-w /etc/ssh/sshd_config -p wa -k sshd_config_changes

# Monitor sudo config
-w /etc/sudoers -p wa -k sudoers_changes
-w /etc/sudoers.d/ -p wa -k sudoers_changes

# Monitor network changes
-w /etc/network/ -p wa -k network_changes
-w /etc/hosts -p wa -k hosts_changes

# Monitor failed login attempts
-w /var/log/faillog -p wa -k login_failures
-w /var/log/lastlog -p wa -k login_changes

# Monitor user changes
-w /usr/bin/passwd -p x -k passwd_modification
-w /usr/sbin/usermod -p x -k user_modification
-w /usr/sbin/adduser -p x -k user_modification
-w /usr/sbin/userdel -p x -k user_modification

# Monitor privilege escalation
-w /usr/bin/sudo -p x -k sudo_usage
-w /usr/bin/su -p x -k su_usage
```

**Flags explained:**
- `-w` = Watch file/directory
- `-p wa` = Watch for write and attribute changes
- `-p x` = Watch for execution
- `-k` = Key (tag) for searching logs

---

### Reload Audit Rules

```bash
sudo auditctl -R /etc/audit/rules.d/audit.rules
```

**Verify rules loaded:**
```bash
sudo auditctl -l
```

---

### Search Audit Logs

**Search by key:**
```bash
sudo ausearch -k passwd_changes
```

**Search recent events:**
```bash
sudo ausearch -ts today
```

**Search by user:**
```bash
sudo ausearch -ua deploy
```

**Search for sudo usage:**
```bash
sudo ausearch -k sudo_usage
```

**Generate report:**
```bash
sudo aureport --summary
```

---

## Network Hardening

### Disable IPv6 (Optional)

**If not using IPv6:**

**Edit sysctl config:**
```bash
sudo nano /etc/sysctl.conf
```

**Add:**
```
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
```

**Apply:**
```bash
sudo sysctl -p
```

**⚠️ Only disable if you don't need IPv6!**

---

### Protect Against SYN Floods

**Edit sysctl:**
```bash
sudo nano /etc/sysctl.conf
```

**Add:**
```
# SYN flood protection
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 2048
net.ipv4.tcp_synack_retries = 2
```

**Apply:**
```bash
sudo sysctl -p
```

---

### Disable ICMP Redirects

**Prevents routing attacks:**

```
# Ignore ICMP redirects
net.ipv4.conf.all.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0

# Do not send redirects
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
```

---

### Enable Reverse Path Filtering

**Prevents IP spoofing:**

```
# Reverse path filtering
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
```

---

### Complete Network Hardening

**Full secure sysctl configuration:**

```bash
sudo nano /etc/sysctl.d/99-security.conf
```

**Add:**
```
# IP forwarding (disabled - we're not a router)
net.ipv4.ip_forward = 0
net.ipv6.conf.all.forwarding = 0

# Ignore ICMP ping requests (optional - makes server "invisible")
# net.ipv4.icmp_echo_ignore_all = 1

# Ignore broadcast pings
net.ipv4.icmp_echo_ignore_broadcasts = 1

# Ignore bogus ICMP errors
net.ipv4.icmp_ignore_bogus_error_responses = 1

# Disable source packet routing
net.ipv4.conf.all.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0

# Log Martians (packets with impossible addresses)
net.ipv4.conf.all.log_martians = 1

# Ignore ICMP redirects
net.ipv4.conf.all.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0

# Reverse path filtering
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# SYN flood protection
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 2048
net.ipv4.tcp_synack_retries = 2
net.ipv4.tcp_syn_retries = 5

# Increase TCP queue length
net.ipv4.tcp_max_tw_buckets = 1440000
net.ipv4.tcp_tw_reuse = 1
```

**Apply:**
```bash
sudo sysctl -p /etc/sysctl.d/99-security.conf
```

**Verify:**
```bash
sudo sysctl net.ipv4.tcp_syncookies
# Should output: net.ipv4.tcp_syncookies = 1
```

---

## Secure Shared Memory

### What is Shared Memory?

**Shared memory:**
- /run/shm and /dev/shm
- Allows processes to share data
- Can be exploited for attacks
- Should be secured

---

### Secure /dev/shm

**Edit fstab:**
```bash
sudo nano /etc/fstab
```

**Add at end:**
```
tmpfs /dev/shm tmpfs defaults,noexec,nosuid 0 0
```

**Explanation:**
- `noexec` - Cannot execute programs
- `nosuid` - Cannot use setuid/setgid

**Remount:**
```bash
sudo mount -o remount /dev/shm
```

**Verify:**
```bash
mount | grep shm
```

**Should show:** `noexec,nosuid`

---

## Additional Hardening

### Disable Unnecessary Services

**List running services:**
```bash
sudo systemctl list-unit-files --state=enabled
```

**Disable unnecessary ones:**
```bash
# Example (check if you need these first):
sudo systemctl disable bluetooth
sudo systemctl disable cups
```

**⚠️ Only disable services you don't need!**

---

### Limit Core Dumps

**Core dumps can contain sensitive data.**

**Disable:**
```bash
sudo nano /etc/security/limits.conf
```

**Add:**
```
* hard core 0
```

**In sysctl:**
```bash
sudo nano /etc/sysctl.d/99-security.conf
```

**Add:**
```
kernel.core_uses_pid = 1
fs.suid_dumpable = 0
```

**Apply:**
```bash
sudo sysctl -p /etc/sysctl.d/99-security.conf
```

---

### Set Password Policies

**Install password quality tool:**
```bash
sudo apt install libpam-pwquality -y
```

**Configure:**
```bash
sudo nano /etc/security/pwquality.conf
```

**Recommended settings:**
```
minlen = 12
dcredit = -1
ucredit = -1
ocredit = -1
lcredit = -1
```

**Explanation:**
- Minimum 12 characters
- At least 1 digit
- At least 1 uppercase
- At least 1 special character
- At least 1 lowercase

---

### Configure Login Timeouts

**Auto-logout inactive SSH sessions:**

```bash
sudo nano /etc/profile.d/timeout.sh
```

**Add:**
```bash
# Logout after 15 minutes of inactivity
TMOUT=900
readonly TMOUT
export TMOUT
```

**Make executable:**
```bash
sudo chmod +x /etc/profile.d/timeout.sh
```

---

## Monitoring and Maintenance

### Daily Security Checks

**Create daily security script:**
```bash
sudo nano /usr/local/bin/daily-security-check.sh
```

**Add:**
```bash
#!/bin/bash

echo "=== Daily Security Check - $(date) ==="
echo

echo "1. Failed SSH attempts:"
grep "Failed password" /var/log/auth.log | tail -10
echo

echo "2. Fail2Ban status:"
fail2ban-client status sshd
echo

echo "3. Listening ports:"
ss -tlnp
echo

echo "4. Active connections:"
ss -tnp | grep ESTAB
echo

echo "5. Last logins:"
last -10
echo

echo "6. Disk usage:"
df -h /
echo

echo "=== Check Complete ==="
```

**Make executable:**
```bash
sudo chmod +x /usr/local/bin/daily-security-check.sh
```

**Run manually:**
```bash
sudo /usr/local/bin/daily-security-check.sh
```

---

### Automated Security Reports

**Create weekly cron job:**
```bash
sudo nano /etc/cron.weekly/security-report
```

**Add:**
```bash
#!/bin/bash
{
    echo "Weekly Security Report - $(hostname)"
    echo "=================================="
    echo
    
    echo "Firewall Status:"
    ufw status verbose
    echo
    
    echo "Fail2Ban Stats:"
    fail2ban-client status sshd
    echo
    
    echo "System Updates Available:"
    apt list --upgradable 2>/dev/null | grep -v Listing
    echo
    
    echo "Top Failed Login IPs:"
    grep "Failed password" /var/log/auth.log | grep -oP 'from \K[\d.]+' | sort | uniq -c | sort -rn | head -10
    echo
    
    echo "Recent Root Commands:"
    grep "sudo" /var/log/auth.log | tail -20
    
} | mail -s "Weekly Security Report - $(hostname)" your@email.com
```

**Make executable:**
```bash
sudo chmod +x /etc/cron.weekly/security-report
```

---

## Verification Checklist

**Before finishing this chapter:**

- ✅ Automatic security updates configured
- ✅ rkhunter installed and running
- ✅ AIDE file integrity monitoring set up
- ✅ auditd logging system events
- ✅ Network hardening applied
- ✅ Shared memory secured
- ✅ Password policies enforced
- ✅ Security monitoring scripts created
- ✅ All configurations tested

---

## Practice Exercises

### Exercise 1: Test Automatic Updates

```bash
# Check configuration
sudo cat /etc/apt/apt.conf.d/50unattended-upgrades | grep "security"

# Dry run
sudo unattended-upgrades --dry-run

# Check logs
sudo tail -50 /var/log/unattended-upgrades/unattended-upgrades.log
```

---

### Exercise 2: Run Security Scans

```bash
# Rootkit scan
sudo rkhunter --check --skip-keypress

# File integrity check
sudo aide --check

# View audit logs
sudo ausearch -ts today
sudo aureport --summary
```

---

### Exercise 3: Review System Security

```bash
# Network hardening settings
sudo sysctl -a | grep -E "tcp_syncookies|rp_filter|accept_redirects"

# Check shared memory
mount | grep shm

# View security rules
sudo auditctl -l

# Run daily security check
sudo /usr/local/bin/daily-security-check.sh
```

---

## Key Takeaways

**Remember:**

1. **Automate security updates**
   - Security patches applied automatically
   - Reduces exposure window
   - Minimal maintenance required

2. **Monitor for intrusions**
   - rkhunter for rootkits
   - AIDE for file changes
   - auditd for system events

3. **Harden the network**
   - Disable unused protocols
   - Enable SYN flood protection
   - Configure reverse path filtering

4. **Layer your security**
   - Each measure adds protection
   - No single point of failure
   - Defense in depth

5. **Regular monitoring essential**
   - Automated checks
   - Review logs
   - Respond to alerts

6. **Document everything**
   - Configuration decisions
   - Baseline values
   - Expected vs. suspicious activity

---

## Next Steps

**You now have:**
- ✅ Comprehensive server security
- ✅ Automated updates and monitoring
- ✅ Intrusion detection
- ✅ Network hardening
- ✅ System auditing

**Security Part Complete!**

**In Part 3 (Docker):**
- Chapter 12: Docker concepts and architecture
- Chapter 13: Docker installation and configuration
- Learn containerization
- Prepare for application deployment

**Your server is now very secure!**

---

## Quick Reference

### Automatic Updates

```bash
# Configure
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades

# Test
sudo unattended-upgrades --dry-run

# Logs
sudo tail -f /var/log/unattended-upgrades/unattended-upgrades.log
```

### Security Scans

```bash
# Rootkit scan
sudo rkhunter --check

# Update baseline
sudo rkhunter --propupd

# File integrity
sudo aide --check
sudo aide --update

# Audit logs
sudo ausearch -ts today
sudo aureport
```

### Network Hardening

```bash
# Apply settings
sudo sysctl -p /etc/sysctl.d/99-security.conf

# Verify
sudo sysctl -a | grep tcp_syncookies
```

### Monitoring

```bash
# Daily check
sudo /usr/local/bin/daily-security-check.sh

# Failed logins
sudo grep "Failed password" /var/log/auth.log | tail -20

# Audit events
sudo ausearch -k sudo_usage
```

---

[← Previous: Chapter 10 - Fail2Ban](10-fail2ban.md) | [Next: Chapter 12 - Docker Concepts →](12-docker-concepts.md)
