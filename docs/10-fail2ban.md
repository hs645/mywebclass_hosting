# Chapter 10: Fail2Ban

**Automated Intrusion Prevention and IP Banning**

---

## Learning Objectives

By the end of this chapter, you'll be able to:
- ✅ Understand what Fail2Ban does
- ✅ Install and configure Fail2Ban
- ✅ Set up SSH protection
- ✅ Configure jails for different services
- ✅ Monitor banned IPs
- ✅ Manually ban/unban IP addresses
- ✅ Integrate with UFW

**Time Required:** 30-40 minutes

---

## What is Fail2Ban?

### Basic Concept

**Fail2Ban = Automated security guard**

**What it does:**
1. Monitors log files
2. Detects failed login attempts
3. Bans IP addresses automatically
4. Unbans after timeout

**Real-world analogy:**
- **Security guard** = Fail2Ban
- **Guest list** = Log files
- **Failed entry** = Failed SSH login
- **Banned** = IP blocked by firewall

---

### How Fail2Ban Works

**The process:**

```
1. Someone tries to SSH with wrong password
   ↓
2. SSH logs the failed attempt
   ↓
3. Fail2Ban reads the log file
   ↓
4. Sees 5 failures from same IP in 10 minutes
   ↓
5. Adds UFW rule to block that IP
   ↓
6. Attacker is blocked for 10 minutes (or permanently)
```

---

### Why You Need Fail2Ban

**Without Fail2Ban:**
```
Attacker tries 1000 passwords per hour
Eventually might guess correct one
Your SSH rate limiting helps but not enough
```

**With Fail2Ban:**
```
Attacker tries 5 wrong passwords
Fail2Ban blocks their IP
Can't try anymore
Must wait or find new IP
```

**Benefits:**
- ✅ Automatic protection
- ✅ Stops brute force attacks
- ✅ No manual intervention needed
- ✅ Works 24/7
- ✅ Protects multiple services

---

## Fail2Ban Components

### Jails

**Jail = Protection for specific service**

**Common jails:**
- `sshd` - Protects SSH
- `apache` - Protects Apache web server
- `nginx` - Protects Nginx web server
- `postfix` - Protects email server

**Each jail:**
- Monitors specific log file
- Detects specific patterns
- Has own ban rules
- Can be enabled/disabled independently

---

### Filters

**Filter = Pattern to detect attacks**

**Example SSH filter:**
- "Failed password for"
- "Invalid user"
- "Connection closed by authenticating user"

**Fail2Ban searches logs for these patterns.**

**Filters located in:** `/etc/fail2ban/filter.d/`

---

### Actions

**Action = What to do when attack detected**

**Common actions:**
- Ban IP using iptables
- Ban IP using UFW
- Send email notification
- Execute custom script
- Log to file

**Actions located in:** `/etc/fail2ban/action.d/`

---

## Installation

### Install Fail2Ban

```bash
sudo apt update
sudo apt install fail2ban -y
```

**Output:**
```
Reading package lists... Done
Building dependency tree... Done
The following NEW packages will be installed:
  fail2ban
0 upgraded, 1 newly installed, 0 to remove
```

---

### Verify Installation

```bash
sudo systemctl status fail2ban
```

**Should show:**
```
● fail2ban.service - Fail2Ban Service
     Loaded: loaded (/lib/systemd/system/fail2ban.service; enabled)
     Active: active (running) since Mon 2025-11-12 10:00:00 UTC
```

**If not running:**
```bash
sudo systemctl start fail2ban
sudo systemctl enable fail2ban
```

---

### Check Version

```bash
fail2ban-client version
```

**Output:**
```
Fail2Ban v1.0.2
```

---

## Configuration Files

### Understanding the Structure

**Main files:**
```
/etc/fail2ban/
├── fail2ban.conf          # Main config (don't edit)
├── fail2ban.local         # Override main config (create this)
├── jail.conf              # Jail definitions (don't edit)
├── jail.local             # Override jails (create this)
├── filter.d/              # Filter definitions
│   └── sshd.conf
├── action.d/              # Action definitions
│   └── ufw.conf
└── jail.d/                # Additional jail configs
    └── defaults-debian.conf
```

**Golden rule:** Never edit `.conf` files, create `.local` files instead.

**Why?**
- `.conf` files are overwritten on updates
- `.local` files override `.conf` settings
- `.local` files persist through updates

---

## Basic Configuration

### Create jail.local

**Create custom configuration file:**
```bash
sudo nano /etc/fail2ban/jail.local
```

**Add this configuration:**
```ini
[DEFAULT]
# Ban settings
bantime  = 10m
findtime = 10m
maxretry = 5

# Email notifications (optional, configure later)
destemail = your@email.com
sender = fail2ban@mywebclass.org
action = %(action_)s

# Ignore our own IP (optional)
ignoreip = 127.0.0.1/8 ::1

[sshd]
enabled = true
port = 2222
logpath = %(sshd_log)s
backend = %(sshd_backend)s
```

**Save and exit:** Ctrl+X, Y, Enter

---

### Understanding the Configuration

**[DEFAULT] section:**

**bantime = 10m**
- How long to ban (10 minutes)
- Can use: s (seconds), m (minutes), h (hours), d (days)
- Example: `1h`, `1d`, `forever`

**findtime = 10m**
- Time window to count failures
- If 5 failures in 10 minutes = ban

**maxretry = 5**
- Number of failures before ban
- 5 is reasonable (allows typos)
- Lower = more strict

**ignoreip**
- IPs never to ban
- Add your office IP if you want
- Localhost always safe

---

**[sshd] section:**

**enabled = true**
- Activate SSH protection

**port = 2222**
- Your SSH port
- Change to match your port!

**logpath**
- Where SSH logs are
- Default works for Ubuntu

**backend**
- How to read logs
- Auto-detected by systemd

---

### Restart Fail2Ban

**Apply configuration:**
```bash
sudo systemctl restart fail2ban
```

**Check status:**
```bash
sudo systemctl status fail2ban
```

**Should show:**
```
● fail2ban.service - Fail2Ban Service
     Active: active (running)
```

---

## Monitoring Fail2Ban

### Check Jail Status

**See all jails:**
```bash
sudo fail2ban-client status
```

**Output:**
```
Status
|- Number of jail:      1
`- Jail list:   sshd
```

**See specific jail details:**
```bash
sudo fail2ban-client status sshd
```

**Output:**
```
Status for the jail: sshd
|- Filter
|  |- Currently failed: 0
|  |- Total failed:     12
|  `- File list:        /var/log/auth.log
`- Actions
   |- Currently banned: 2
   |- Total banned:     5
   `- Banned IP list:   192.0.2.50 192.0.2.51
```

**Shows:**
- How many IPs are banned now
- Total bans since start
- Which IPs are banned

---

### View Fail2Ban Logs

**Fail2Ban log location:**
```bash
sudo tail -f /var/log/fail2ban.log
```

**Example entries:**
```
2025-11-12 10:30:15 fail2ban.filter  [12345]: INFO    [sshd] Found 192.0.2.50 - 2025-11-12 10:30:15
2025-11-12 10:30:20 fail2ban.filter  [12345]: INFO    [sshd] Found 192.0.2.50 - 2025-11-12 10:30:20
2025-11-12 10:30:25 fail2ban.actions [12345]: NOTICE  [sshd] Ban 192.0.2.50
```

**Translation:**
- Detected failed attempts from 192.0.2.50
- Reached max retries
- Banned the IP

---

### Watch Bans in Real-Time

**Monitor for new bans:**
```bash
sudo tail -f /var/log/fail2ban.log | grep Ban
```

**See unbans:**
```bash
sudo tail -f /var/log/fail2ban.log | grep Unban
```

---

## Manual IP Management

### Manually Ban an IP

**Ban specific IP:**
```bash
sudo fail2ban-client set sshd banip 192.0.2.50
```

**Output:**
```
192.0.2.50
```

**Use cases:**
- Suspicious activity
- Known attacker
- Preemptive blocking

---

### Manually Unban an IP

**Unban specific IP:**
```bash
sudo fail2ban-client set sshd unbanip 192.0.2.50
```

**Output:**
```
192.0.2.50
```

**Use cases:**
- Accidentally banned yourself
- False positive
- Partner/colleague banned

---

### List Currently Banned IPs

**See all banned IPs:**
```bash
sudo fail2ban-client status sshd | grep "Banned IP"
```

**Or:**
```bash
sudo fail2ban-client get sshd banip
```

---

## Advanced Configuration

### Permanent Bans

**Ban attackers forever:**

**Edit jail.local:**
```bash
sudo nano /etc/fail2ban/jail.local
```

**Change bantime:**
```ini
[DEFAULT]
bantime = -1
# OR
bantime = forever
```

**⚠️ Consider carefully:**
- IPs can change
- You might ban legitimate users
- Better: longer timeout (1d, 7d)

**Recommended:**
```ini
bantime = 1d    # 1 day is reasonable
```

---

### Email Notifications

**Get notified when IPs are banned:**

**Install mail utilities:**
```bash
sudo apt install mailutils -y
```

**Edit jail.local:**
```bash
sudo nano /etc/fail2ban/jail.local
```

**Update [DEFAULT]:**
```ini
[DEFAULT]
destemail = your@email.com
sender = fail2ban@mywebclass.org
action = %(action_mw)s
```

**Action types:**
- `%(action_)s` - Ban only (no email)
- `%(action_mw)s` - Ban + email with log excerpt
- `%(action_mwl)s` - Ban + email with full logs

**Restart:**
```bash
sudo systemctl restart fail2ban
```

---

### Increase Security

**More aggressive banning:**

```ini
[DEFAULT]
bantime  = 1d       # Ban for 1 day
findtime = 1h       # Look at last hour
maxretry = 3        # Only 3 attempts allowed
```

**Very strict (use with caution):**
```ini
[DEFAULT]
bantime  = 7d       # Ban for 1 week
findtime = 1h
maxretry = 2        # Only 2 mistakes allowed
```

**Recommended balance:**
```ini
[DEFAULT]
bantime  = 1h       # 1 hour ban
findtime = 10m      # 10 minute window
maxretry = 5        # 5 attempts
```

---

### Whitelist Your IP

**Never ban your own IP:**

**Find your public IP:**
```bash
curl ifconfig.me
```

**Edit jail.local:**
```bash
sudo nano /etc/fail2ban/jail.local
```

**Add to [DEFAULT]:**
```ini
[DEFAULT]
ignoreip = 127.0.0.1/8 ::1 YOUR_IP_HERE
```

**Example:**
```ini
ignoreip = 127.0.0.1/8 ::1 203.0.113.50
```

**Restart:**
```bash
sudo systemctl restart fail2ban
```

---

## Additional Jails

### Protect Caddy/Nginx

**If using Caddy or Nginx, protect web server too:**

**Create filter for HTTP auth failures:**
```bash
sudo nano /etc/fail2ban/filter.d/caddy-auth.conf
```

**Add:**
```ini
[Definition]
failregex = ^.*\[ERROR\].*<HOST>.*401
ignoreregex =
```

**Add jail:**
```bash
sudo nano /etc/fail2ban/jail.local
```

**Add:**
```ini
[caddy-auth]
enabled = true
port = http,https
logpath = /var/log/caddy/access.log
maxretry = 10
findtime = 10m
bantime = 1h
```

---

### Protect Against DDoS

**Rate limit HTTP requests:**

```ini
[http-get-dos]
enabled = true
port = http,https
filter = http-get-dos
logpath = /var/log/caddy/access.log
maxretry = 300
findtime = 5m
bantime = 10m
action = %(action_)s
```

**Create filter:**
```bash
sudo nano /etc/fail2ban/filter.d/http-get-dos.conf
```

```ini
[Definition]
failregex = ^<HOST> -.*"GET
ignoreregex =
```

**⚠️ Use carefully - might ban legitimate users!**

---

## Integration with UFW

### Verify UFW Integration

**Fail2Ban should use UFW for bans.**

**Check action:**
```bash
sudo nano /etc/fail2ban/jail.local
```

**In [DEFAULT], ensure:**
```ini
banaction = ufw
```

**Or specify per jail:**
```ini
[sshd]
banaction = ufw
```

---

### Check UFW for Banned IPs

**See UFW rules created by Fail2Ban:**
```bash
sudo ufw status numbered
```

**Look for Fail2Ban rules:**
```
[ 7] 2222/tcp                   DENY IN     192.0.2.50
[ 8] 2222/tcp                   DENY IN     192.0.2.51
```

**These are Fail2Ban bans!**

---

### Manual UFW Cleanup

**If Fail2Ban removed but rules remain:**
```bash
sudo ufw status numbered
# Note numbers of Fail2Ban rules

sudo ufw delete 7
sudo ufw delete 8
# Etc.
```

---

## Testing Fail2Ban

### Test SSH Jail (Safely!)

**⚠️ Warning:** You can ban yourself! Have backup access ready.

**Method 1: From a different computer/IP**

**Try wrong passwords multiple times:**
```bash
ssh deploy@YOUR_SERVER_IP
# Enter wrong password 5+ times
```

**After 5 failures, check if banned:**
```bash
# On server:
sudo fail2ban-client status sshd
```

**Should see the IP in banned list.**

---

**Method 2: Whitelist your IP first**

**Add your IP to ignoreip (as shown earlier).**

**Then test from a VPS or cloud instance:**
```bash
# From different server:
ssh root@YOUR_TARGET_SERVER
# Try wrong passwords
```

---

### Simulate Attack

**Generate fake log entries (for testing):**

```bash
# Create test log
sudo bash -c 'cat >> /var/log/auth.log << EOF
Nov 12 10:00:00 server sshd[12345]: Failed password for invalid user test from 192.0.2.99 port 12345 ssh2
Nov 12 10:00:05 server sshd[12346]: Failed password for invalid user test from 192.0.2.99 port 12346 ssh2
Nov 12 10:00:10 server sshd[12347]: Failed password for invalid user test from 192.0.2.99 port 12347 ssh2
Nov 12 10:00:15 server sshd[12348]: Failed password for invalid user test from 192.0.2.99 port 12348 ssh2
Nov 12 10:00:20 server sshd[12349]: Failed password for invalid user test from 192.0.2.99 port 12349 ssh2
EOF'
```

**Wait a few seconds, then check:**
```bash
sudo fail2ban-client status sshd
```

**Should see 192.0.2.99 banned!**

**Unban test IP:**
```bash
sudo fail2ban-client set sshd unbanip 192.0.2.99
```

---

## Troubleshooting

### Problem: Fail2Ban Not Starting

**Check logs:**
```bash
sudo journalctl -u fail2ban -n 50
```

**Common issues:**
- Syntax error in jail.local
- Invalid regex in filter
- Log file doesn't exist

**Test configuration:**
```bash
sudo fail2ban-client -t
```

**Shows config errors if any.**

---

### Problem: Jail Not Active

**Check jail status:**
```bash
sudo fail2ban-client status
```

**If sshd not listed:**

**Check configuration:**
```bash
sudo nano /etc/fail2ban/jail.local
```

**Ensure:**
```ini
[sshd]
enabled = true
```

**Restart:**
```bash
sudo systemctl restart fail2ban
```

---

### Problem: Not Detecting Failures

**Check if log file is correct:**
```bash
sudo fail2ban-client get sshd logpath
```

**Should show:** `/var/log/auth.log`

**Check log file exists:**
```bash
ls -l /var/log/auth.log
```

**Test filter manually:**
```bash
sudo fail2ban-regex /var/log/auth.log /etc/fail2ban/filter.d/sshd.conf
```

**Shows how many matches found.**

---

### Problem: Banned Yourself

**Can't SSH to server!**

**Solution 1: Wait for timeout**
- If bantime = 10m, wait 10 minutes
- IP automatically unbanned

**Solution 2: Console access**
- DigitalOcean console
- Login via web browser
- Unban yourself:
  ```bash
  sudo fail2ban-client set sshd unbanip YOUR_IP
  ```

**Solution 3: Restart Fail2Ban**
```bash
# Via console:
sudo systemctl restart fail2ban
# Clears all bans
```

**Prevention:**
- Add your IP to ignoreip
- Use longer bantime with higher maxretry
- Test from different IP first

---

## Monitoring and Maintenance

### Check Ban Statistics

**How many bans total:**
```bash
sudo fail2ban-client status sshd | grep "Total banned"
```

**Currently banned count:**
```bash
sudo fail2ban-client status sshd | grep "Currently banned"
```

**Failed attempts today:**
```bash
sudo grep "Failed password" /var/log/auth.log | grep "$(date +%b\ %d)" | wc -l
```

---

### Review Attack Patterns

**Top attacking IPs:**
```bash
sudo grep "Ban" /var/log/fail2ban.log | grep -oP '\d+\.\d+\.\d+\.\d+' | sort | uniq -c | sort -rn | head -10
```

**When attacks happen (hourly):**
```bash
sudo grep "Failed password" /var/log/auth.log | cut -d' ' -f3 | cut -d: -f1 | sort | uniq -c
```

**Most attempted usernames:**
```bash
sudo grep "Failed password for" /var/log/auth.log | grep -oP 'for \K\w+' | sort | uniq -c | sort -rn | head -10
```

---

### Regular Maintenance

**Weekly:**
```bash
# Check ban stats
sudo fail2ban-client status sshd

# Review recent bans
sudo grep "Ban" /var/log/fail2ban.log | tail -20

# Check for unusual patterns
sudo grep "Failed password" /var/log/auth.log | tail -50
```

**Monthly:**
```bash
# Rotate logs if needed
sudo logrotate -f /etc/logrotate.d/fail2ban

# Review configuration
sudo cat /etc/fail2ban/jail.local

# Update Fail2Ban
sudo apt update && sudo apt upgrade fail2ban -y
```

---

## Best Practices

### Configuration Recommendations

**Balanced security:**
```ini
[DEFAULT]
bantime  = 1h
findtime = 10m
maxretry = 5
ignoreip = 127.0.0.1/8 ::1 YOUR_IP
```

**High security:**
```ini
[DEFAULT]
bantime  = 1d
findtime = 1h
maxretry = 3
```

**Development/testing:**
```ini
[DEFAULT]
bantime  = 5m
findtime = 10m
maxretry = 10
ignoreip = 127.0.0.1/8 ::1 YOUR_IP
```

---

### Documentation

**Document your Fail2Ban setup:**

```bash
sudo nano /root/fail2ban-config.txt
```

**Add:**
```
Fail2Ban Configuration
Server: mywebclass-prod
Date: 2025-11-12

Settings:
- Ban time: 1 hour
- Find time: 10 minutes
- Max retry: 5 attempts
- Whitelist IPs: 203.0.113.50

Active jails:
- sshd (port 2222)

Actions:
- Ban using UFW
- Email on ban: admin@example.com

Notes:
- Integrated with UFW firewall
- Logs to /var/log/fail2ban.log
- SSH failures from /var/log/auth.log
```

---

## Verification Checklist

**Before finishing this chapter:**

- ✅ Fail2Ban installed
- ✅ SSH jail configured and enabled
- ✅ Configuration tested
- ✅ Can view banned IPs
- ✅ Can manually ban/unban
- ✅ Integrated with UFW
- ✅ Logs are readable
- ✅ Whitelisted your IP (if needed)
- ✅ Fail2Ban starts on boot

---

## Practice Exercises

### Exercise 1: Monitor Activity

```bash
# Check status
sudo fail2ban-client status
sudo fail2ban-client status sshd

# Watch logs
sudo tail -f /var/log/fail2ban.log

# Check banned IPs
sudo fail2ban-client status sshd | grep "Banned IP"
```

---

### Exercise 2: Test Banning

```bash
# Simulate attack (safe test)
sudo fail2ban-client set sshd banip 192.0.2.99

# Verify ban
sudo fail2ban-client status sshd

# Check UFW
sudo ufw status | grep 192.0.2.99

# Unban
sudo fail2ban-client set sshd unbanip 192.0.2.99
```

---

### Exercise 3: Analyze Attacks

```bash
# Count total failed attempts
sudo grep "Failed password" /var/log/auth.log | wc -l

# Top attacking IPs
sudo grep "Failed password" /var/log/auth.log | grep -oP 'from \K[\d.]+' | sort | uniq -c | sort -rn | head -10

# Check Fail2Ban effectiveness
echo "Total failures: $(sudo grep 'Failed password' /var/log/auth.log | wc -l)"
echo "Total bans: $(sudo fail2ban-client status sshd | grep 'Total banned' | grep -oP '\d+')"
```

---

## Key Takeaways

**Remember:**

1. **Fail2Ban automates security**
   - Monitors logs continuously
   - Bans attackers automatically
   - No manual intervention needed

2. **Configure thresholds carefully**
   - Too strict: ban legitimate users
   - Too loose: attackers succeed
   - 5 failures in 10 minutes is balanced

3. **Whitelist your own IPs**
   - Prevent locking yourself out
   - Have backup access ready
   - Test from different IP if possible

4. **Monitor regularly**
   - Check ban statistics
   - Review attack patterns
   - Adjust configuration as needed

5. **Integrates with firewall**
   - Uses UFW to block IPs
   - Creates temporary rules
   - Removes rules after timeout

6. **Layered security**
   - SSH hardening (Chapter 7)
   - Firewall (Chapters 8-9)
   - Fail2Ban (this chapter)
   - System hardening (Chapter 11)

---

## Next Steps

**You now have:**
- ✅ Automated intrusion prevention
- ✅ SSH brute force protection
- ✅ IP-based banning
- ✅ Attack monitoring
- ✅ Integration with UFW

**In Chapter 11:**
- Automatic security updates
- Rootkit detection
- File integrity monitoring
- System auditing
- Additional hardening

**Your server is becoming very secure!**

---

## Quick Reference

### Essential Commands

```bash
# Status
sudo fail2ban-client status
sudo fail2ban-client status sshd

# Ban/unban
sudo fail2ban-client set sshd banip IP
sudo fail2ban-client set sshd unbanip IP

# Logs
sudo tail -f /var/log/fail2ban.log
sudo grep "Ban" /var/log/fail2ban.log

# Service
sudo systemctl restart fail2ban
sudo systemctl status fail2ban

# Test configuration
sudo fail2ban-client -t
```

### Configuration File

**Location:** `/etc/fail2ban/jail.local`

```ini
[DEFAULT]
bantime  = 1h
findtime = 10m
maxretry = 5
ignoreip = 127.0.0.1/8 ::1

[sshd]
enabled = true
port = 2222
```

---

[← Previous: Chapter 9 - Firewall Configuration](09-firewall-configuration.md) | [Next: Chapter 11 - System Hardening →](11-system-hardening.md)
