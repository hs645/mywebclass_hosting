# Chapter 9: Firewall Configuration

**Installing and Configuring UFW**

---

## Learning Objectives

By the end of this chapter, you'll be able to:
- ✅ Install UFW on your server
- ✅ Configure firewall rules safely
- ✅ Allow SSH, HTTP, and HTTPS traffic
- ✅ Enable firewall without locking yourself out
- ✅ Test firewall configuration
- ✅ View and interpret firewall logs

**Time Required:** 30-40 minutes

**⚠️ CRITICAL:** Keep existing SSH connection open during this chapter!

---

## Safety First

### The Golden Rule (Again!)

**🔥 DO NOT CLOSE YOUR SSH CONNECTION UNTIL VERIFIED! 🔥**

**Workflow:**
1. Keep existing SSH session open in Terminal 1
2. Add firewall rules in Terminal 1
3. Enable firewall
4. Test NEW connection in Terminal 2
5. If test succeeds, close old connection
6. If test fails, disable firewall in Terminal 1

**This prevents lockouts!**

---

### Terminal Setup

**Before starting:**

**Terminal 1: Working connection**
```bash
ssh deploy@YOUR_SERVER_IP
# Or with custom port:
ssh -p 2222 deploy@YOUR_SERVER_IP
```

**Keep this open the ENTIRE chapter!**

**Terminal 2: For testing**
```bash
# We'll use this to test connectivity
# After firewall is configured
```

---

## Check UFW Installation

### Verify UFW is Installed

**UFW comes pre-installed on Ubuntu.**

**Check if installed:**
```bash
sudo ufw version
```

**Output:**
```
ufw 0.36.2
Copyright 2008-2023 Canonical Ltd.
```

**If not installed:**
```bash
sudo apt update
sudo apt install ufw -y
```

---

### Check UFW Status

**Check current status:**
```bash
sudo ufw status
```

**Likely output:**
```
Status: inactive
```

**This is good!** Firewall is installed but not enabled yet.

**If it says "active":**
```bash
sudo ufw status verbose
```

**Check what rules exist before continuing.**

---

## Understanding Current State

### Check Listening Ports

**Before configuring firewall, see what's listening:**

```bash
sudo ss -tlnp
```

**Example output:**
```
State    Recv-Q   Send-Q   Local Address:Port   Peer Address:Port
LISTEN   0        128      0.0.0.0:2222         0.0.0.0:*         users:(("sshd",pid=1234))
LISTEN   0        128      [::]:2222            [::]:*            users:(("sshd",pid=1234))
```

**What this shows:**
- SSH listening on port 2222
- On all interfaces (0.0.0.0)
- Both IPv4 and IPv6

**Your output might also show:**
- Port 80/443 if Caddy is running
- Other ports from Docker containers

**We need to allow these ports before enabling firewall!**

---

## Configure Default Policies

### Set Default Rules

**Step 1: Deny all incoming (default)**
```bash
sudo ufw default deny incoming
```

**Output:**
```
Default incoming policy changed to 'deny'
(be sure to update your rules accordingly)
```

**This means:** Block everything by default.

---

**Step 2: Allow all outgoing**
```bash
sudo ufw default allow outgoing
```

**Output:**
```
Default outgoing policy changed to 'allow'
(be sure to update your rules accordingly)
```

**This means:** Server can make outbound connections.

---

**Step 3: Allow routed traffic (optional)**
```bash
sudo ufw default deny routed
```

**This controls traffic routed THROUGH your server.**

**For our setup:** Not routing traffic, so deny it.

---

### Verify Default Policies

**Check settings:**
```bash
sudo ufw status verbose
```

**Should show:**
```
Status: inactive
Default: deny (incoming), allow (outgoing), deny (routed)
```

**Perfect!** Secure defaults configured.

---

## Add Firewall Rules

### Allow SSH Port

**⚠️ MOST IMPORTANT RULE - DO THIS FIRST!**

**If you're using port 22:**
```bash
sudo ufw allow 22/tcp
```

**If you changed SSH port (like to 2222):**
```bash
sudo ufw allow 2222/tcp
```

**Use YOUR actual SSH port!**

**Output:**
```
Rule added
Rule added (v6)
```

**What this does:**
- Allows TCP connections on port 2222
- Both IPv4 and IPv6
- From any IP address

---

### Allow HTTP (Port 80)

**For web traffic and Let's Encrypt:**
```bash
sudo ufw allow 80/tcp
```

**Output:**
```
Rule added
Rule added (v6)
```

**Why allow HTTP when using HTTPS?**
- Let's Encrypt certificate validation
- Redirect HTTP to HTTPS
- Some automation tools

---

### Allow HTTPS (Port 443)

**For encrypted web traffic:**
```bash
sudo ufw allow 443/tcp
```

**Output:**
```
Rule added
Rule added (v6)
```

**This is your main web traffic port.**

---

### Review Rules Before Enabling

**Check what rules are configured:**
```bash
sudo ufw show added
```

**Output:**
```
Added user rules (see 'ufw status' for running firewall):
ufw allow 2222/tcp
ufw allow 80/tcp
ufw allow 443/tcp
```

**Verify:**
- ✅ SSH port is included
- ✅ HTTP (80) is included
- ✅ HTTPS (443) is included

**If SSH port is missing, add it NOW!**

---

## Enable the Firewall

### Enable UFW

**🔥 Final check: Is SSH port allowed? 🔥**

```bash
sudo ufw show added | grep 2222
# Or your SSH port number
```

**If you see your SSH port, proceed:**

```bash
sudo ufw enable
```

**You'll see a WARNING:**
```
Command may disrupt existing ssh connections. Proceed with operation (y|n)?
```

**Type:** `y` and press Enter

**Output:**
```
Firewall is active and enabled on system startup
```

---

### Verify Firewall is Active

**Check status:**
```bash
sudo ufw status
```

**Output:**
```
Status: active

To                         Action      From
--                         ------      ----
2222/tcp                   ALLOW       Anywhere
80/tcp                     ALLOW       Anywhere
443/tcp                    ALLOW       Anywhere
2222/tcp (v6)              ALLOW       Anywhere (v6)
80/tcp (v6)                ALLOW       Anywhere (v6)
443/tcp (v6)                ALLOW       Anywhere (v6)
```

**Good signs:**
- Status: active ✅
- SSH port listed ✅
- HTTP/HTTPS listed ✅

---

### Detailed Status

**More verbose output:**
```bash
sudo ufw status verbose
```

**Output:**
```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), deny (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
2222/tcp                   ALLOW IN    Anywhere
80/tcp                     ALLOW IN    Anywhere
443/tcp                    ALLOW IN    Anywhere
2222/tcp (v6)              ALLOW IN    Anywhere (v6)
80/tcp (v6)                ALLOW IN    Anywhere (v6)
443/tcp (v6)               ALLOW IN    Anywhere (v6)
```

**Shows:**
- Logging is enabled
- Default policies
- Direction (IN = incoming)

---

## Test Firewall Configuration

### Test 1: Current Connection Still Works

**In Terminal 1 (your original connection):**

```bash
# You should still be connected!
whoami
hostname
```

**If this works, good!** Firewall didn't break existing connection.

---

### Test 2: New SSH Connection

**In Terminal 2, connect to server:**

**With custom port:**
```bash
ssh -p 2222 deploy@YOUR_SERVER_IP
```

**Or default port:**
```bash
ssh deploy@YOUR_SERVER_IP
```

**Should connect successfully!** ✅

**If connection fails:**
1. Keep Terminal 1 open (don't panic!)
2. In Terminal 1, disable firewall:
   ```bash
   sudo ufw disable
   ```
3. Fix the rule, re-enable
4. Test again

---

### Test 3: Blocked Ports

**Try to connect to a blocked port (Terminal 2):**

```bash
# Try to connect to PostgreSQL (should be blocked)
nc -zv YOUR_SERVER_IP 5432
```

**Expected output:**
```
nc: connect to YOUR_SERVER_IP port 5432 (tcp) failed: Connection refused
```

**Or:**
```
Connection timed out
```

**This is GOOD!** Port is blocked.

---

### Test 4: Allowed Ports

**Test web ports (Terminal 2):**

```bash
# Test HTTP
nc -zv YOUR_SERVER_IP 80

# Test HTTPS  
nc -zv YOUR_SERVER_IP 443
```

**Expected output:**
```
Connection to YOUR_SERVER_IP 80 port [tcp/http] succeeded!
Connection to YOUR_SERVER_IP 443 port [tcp/https] succeeded!
```

**Or if services aren't running yet:**
```
Connection refused
```

**Either is fine!** Important is firewall isn't blocking.

---

### Test 5: Port Scan

**From your LOCAL computer (not server):**

```bash
nmap YOUR_SERVER_IP
```

**Should show:**
```
PORT     STATE    SERVICE
80/tcp   open     http
443/tcp  open     https
2222/tcp open     unknown
```

**Everything else should be "filtered" or not shown.**

**If you see other ports open:**
- These may be services you haven't secured yet
- Review what's running: `sudo ss -tlnp`

---

## Advanced Rules

### Allow From Specific IP

**Restrict SSH to your IP only:**

**First, get your public IP:**
```bash
curl ifconfig.me
```

**Then allow only from your IP:**
```bash
# Delete current SSH rule first
sudo ufw delete allow 2222/tcp

# Add restricted rule
sudo ufw allow from YOUR_IP to any port 2222
```

**Example:**
```bash
sudo ufw allow from 203.0.113.50 to any port 2222
```

**Now only your IP can SSH!**

**⚠️ Warning:** If your IP changes, you'll be locked out!

---

### Allow IP Ranges

**Allow entire subnet:**
```bash
sudo ufw allow from 192.168.1.0/24 to any port 2222
```

**Use for:**
- Office network access
- VPN subnet
- Trusted network range

---

### Rate Limiting

**Prevent brute force on SSH:**
```bash
# First delete existing rule
sudo ufw delete allow 2222/tcp

# Add rate limited rule
sudo ufw limit 2222/tcp
```

**What this does:**
- Allows 6 connections per 30 seconds per IP
- Blocks IP temporarily if exceeded
- Slows down brute force attacks

**Check status:**
```bash
sudo ufw status
```

**Should show:**
```
2222/tcp                   LIMIT       Anywhere
```

**"LIMIT" means rate limiting is active.**

---

### Application Profiles

**UFW has predefined profiles for common apps:**

**List available profiles:**
```bash
sudo ufw app list
```

**Might see:**
```
Available applications:
  OpenSSH
  WWW
  WWW Full
  WWW Secure
```

**Use profile instead of port number:**
```bash
# Instead of: ufw allow 22/tcp
sudo ufw allow OpenSSH

# Instead of: ufw allow 80/tcp and 443/tcp
sudo ufw allow "WWW Full"
```

**View profile details:**
```bash
sudo ufw app info "WWW Full"
```

**Output:**
```
Profile: WWW Full
Title: Web Server (HTTP,HTTPS)
Description: Web Server (HTTP,HTTPS)

Ports:
  80,443/tcp
```

---

## Managing Rules

### List All Rules with Numbers

```bash
sudo ufw status numbered
```

**Output:**
```
Status: active

     To                         Action      From
     --                         ------      ----
[ 1] 2222/tcp                   LIMIT IN    Anywhere
[ 2] 80/tcp                     ALLOW IN    Anywhere
[ 3] 443/tcp                    ALLOW IN    Anywhere
[ 4] 2222/tcp (v6)              LIMIT IN    Anywhere (v6)
[ 5] 80/tcp (v6)                ALLOW IN    Anywhere (v6)
[ 6] 443/tcp (v6)               ALLOW IN    Anywhere (v6)
```

---

### Delete a Rule

**Method 1: By number**
```bash
sudo ufw delete 3
```

**⚠️ Warning:** Numbers change after deletion!

---

**Method 2: By rule specification**
```bash
sudo ufw delete allow 80/tcp
```

**This deletes both IPv4 and IPv6 rules.**

---

### Disable/Enable Firewall

**Temporarily disable:**
```bash
sudo ufw disable
```

**Re-enable:**
```bash
sudo ufw enable
```

**Check if enabled:**
```bash
sudo ufw status | grep Status
```

---

### Reset Firewall

**Delete ALL rules and start over:**
```bash
sudo ufw reset
```

**⚠️ WARNING:** This removes all rules!

**You'll see:**
```
Resetting all rules to installed defaults. Proceed with operation (y|n)?
```

**After reset, must reconfigure:**
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2222/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

---

## Logging

### Enable Logging

**UFW logging is on by default.**

**Check logging status:**
```bash
sudo ufw status verbose | grep Logging
```

**Should show:**
```
Logging: on (low)
```

---

### Set Log Level

**Options:**
- `off` - Disable logging
- `low` - Log blocked packets
- `medium` - Log blocked + invalid
- `high` - Log medium + rate limit events
- `full` - Log everything (verbose)

**Set log level:**
```bash
sudo ufw logging medium
```

**For this course, use `low` or `medium`:**
```bash
sudo ufw logging low
```

---

### View Firewall Logs

**UFW logs to system log:**
```bash
sudo tail -f /var/log/ufw.log
```

**Example log entries:**
```
Nov 12 10:45:23 server kernel: [UFW BLOCK] IN=eth0 OUT= MAC=00:00:00:00:00:00 SRC=192.0.2.50 DST=192.0.2.100 LEN=60 TOS=0x00 PREC=0x00 TTL=52 ID=12345 DF PROTO=TCP SPT=54321 DPT=5432 WINDOW=29200 RES=0x00 SYN URGP=0
```

**Breaking it down:**
- `[UFW BLOCK]` - Packet was blocked
- `SRC=192.0.2.50` - Source IP (attacker)
- `DST=192.0.2.100` - Your server
- `PROTO=TCP` - TCP protocol
- `DPT=5432` - Destination port (PostgreSQL)
- `SYN` - New connection attempt

**This is an attack attempt being blocked!**

---

### Filter Logs

**See only blocked packets:**
```bash
sudo grep "UFW BLOCK" /var/log/ufw.log
```

**See attempts on specific port:**
```bash
sudo grep "DPT=5432" /var/log/ufw.log
```

**See attacks from specific IP:**
```bash
sudo grep "SRC=192.0.2.50" /var/log/ufw.log
```

**Count blocked attempts:**
```bash
sudo grep "UFW BLOCK" /var/log/ufw.log | wc -l
```

---

## Monitoring

### Check Active Connections

**See what's currently connected:**
```bash
sudo ss -tnp
```

**Output:**
```
State    Recv-Q   Send-Q   Local:Port      Peer:Port     Process
ESTAB    0        0        192.0.2.100:2222   192.0.2.50:54321   users:(("sshd"))
```

**Shows:**
- Your SSH connection
- Source and destination
- Process handling connection

---

### Watch Real-Time Connections

**Monitor connections in real-time:**
```bash
watch -n 1 'sudo ss -tnp'
```

**Press Ctrl+C to stop.**

---

### Check Listening Services

**What's listening on what ports:**
```bash
sudo ss -tlnp
```

**Compare with firewall rules:**
```bash
sudo ufw status
```

**Any listening service NOT in firewall rules is protected!**

---

## Firewall for Docker

### Docker and UFW Conflict

**⚠️ Important:** Docker can bypass UFW!

**The problem:**
- Docker modifies iptables directly
- Can expose containers even if UFW blocks port
- Security hole!

**Solution (covered in Chapter 12):**
- Use Docker's `internal` networks
- Route public traffic through reverse proxy
- Don't expose container ports directly

**For now:** Don't expose Docker ports to internet.

---

### Safe Docker Configuration

**In docker-compose.yml, avoid:**
```yaml
ports:
  - "5432:5432"  # DON'T DO THIS!
```

**This exposes PostgreSQL to internet, bypassing firewall!**

**Instead, use:**
```yaml
networks:
  - internal
```

**Only reverse proxy on `web` network.**

---

## Common Issues

### Problem: Locked Out After Enabling

**Symptoms:**
- Can't SSH to server
- Terminal 1 might still work

**Solution (using Terminal 1 if still connected):**
```bash
sudo ufw disable
sudo ufw allow 2222/tcp  # Your SSH port
sudo ufw enable
```

**If completely locked out:**
- Use DigitalOcean console
- Login via web console
- Disable firewall: `sudo ufw disable`
- Fix rules, re-enable

---

### Problem: Web Traffic Not Working

**Symptoms:**
- Can SSH fine
- Website not loading

**Check:**
```bash
sudo ufw status | grep -E "80|443"
```

**Should see both ports allowed.**

**If missing:**
```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

---

### Problem: Firewall Not Starting on Boot

**Check if enabled:**
```bash
sudo systemctl status ufw
```

**Should show:**
```
● ufw.service - Uncomplicated firewall
     Loaded: loaded (/lib/systemd/system/ufw.service; enabled)
     Active: active (exited)
```

**If not enabled:**
```bash
sudo systemctl enable ufw
sudo systemctl start ufw
```

---

### Problem: Rules Not Working

**Check rule syntax:**
```bash
sudo ufw status verbose
```

**Reload firewall:**
```bash
sudo ufw reload
```

**Or restart:**
```bash
sudo ufw disable
sudo ufw enable
```

---

## Best Practices

### Regular Maintenance

**Weekly tasks:**
```bash
# Review logs for attack patterns
sudo grep "UFW BLOCK" /var/log/ufw.log | tail -50

# Check active rules
sudo ufw status numbered

# Verify no unauthorized listening ports
sudo ss -tlnp
```

---

### Documentation

**Document your firewall rules:**

Create `/root/firewall-rules.txt`:
```bash
sudo nano /root/firewall-rules.txt
```

**Add:**
```
UFW Firewall Configuration
Server: mywebclass-prod
Date: 2025-11-12

Rules:
- 2222/tcp LIMIT - SSH (rate limited)
- 80/tcp ALLOW - HTTP (Let's Encrypt, redirects)
- 443/tcp ALLOW - HTTPS (main web traffic)

Default policies:
- Incoming: DENY
- Outgoing: ALLOW
- Routed: DENY

Notes:
- SSH rate limited to prevent brute force
- All other ports blocked by default
- Docker containers use internal networks only
```

---

### Security Checklist

**Before considering firewall "done":**

- ✅ SSH port allowed
- ✅ HTTP/HTTPS allowed
- ✅ Default policy is DENY incoming
- ✅ Rate limiting on SSH
- ✅ Logging enabled
- ✅ Tested new connections work
- ✅ Tested blocked ports are blocked
- ✅ Firewall enabled on boot
- ✅ Rules documented
- ✅ No unnecessary ports open

---

## Verification Checklist

**Before finishing this chapter:**

- ✅ UFW installed
- ✅ Default policies set (deny incoming, allow outgoing)
- ✅ SSH port allowed (CRITICAL!)
- ✅ HTTP (80) allowed
- ✅ HTTPS (443) allowed
- ✅ Firewall enabled and active
- ✅ Tested SSH connection works
- ✅ Tested blocked ports are blocked
- ✅ Logging enabled
- ✅ Firewall persists after reboot

---

## Practice Exercises

### Exercise 1: Review Configuration

```bash
# Check firewall status
sudo ufw status verbose

# List all rules with numbers
sudo ufw status numbered

# Check logging level
sudo ufw status verbose | grep Logging

# View recent logs
sudo tail -20 /var/log/ufw.log
```

---

### Exercise 2: Test Connectivity

```bash
# From your LOCAL computer:

# Test SSH
ssh -p 2222 deploy@YOUR_SERVER_IP

# Test HTTP (if web server running)
curl -I http://YOUR_SERVER_IP

# Test HTTPS (if web server running)
curl -I https://YOUR_SERVER_IP

# Port scan
nmap YOUR_SERVER_IP
```

---

### Exercise 3: Monitor Attacks

```bash
# On server:

# Watch for blocked packets
sudo tail -f /var/log/ufw.log | grep BLOCK

# Count blocks per source IP
sudo grep "UFW BLOCK" /var/log/ufw.log | grep -oP 'SRC=\K[^ ]+' | sort | uniq -c | sort -rn | head -10

# See most targeted ports
sudo grep "UFW BLOCK" /var/log/ufw.log | grep -oP 'DPT=\K[^ ]+' | sort | uniq -c | sort -rn | head -10
```

---

## Key Takeaways

**Remember:**

1. **Always allow SSH before enabling firewall**
   - Check twice
   - Keep connection open
   - Test before closing

2. **Default deny is most secure**
   - Block everything by default
   - Only allow what's needed
   - Review regularly

3. **Rate limiting prevents brute force**
   - Use on SSH port
   - Limits connection attempts
   - Blocks rapid attackers

4. **Logging shows attack patterns**
   - Review logs regularly
   - Identify threats
   - Adjust rules as needed

5. **Test thoroughly**
   - Test SSH connection
   - Test allowed ports
   - Test blocked ports
   - Port scan from outside

6. **Docker can bypass UFW**
   - Don't expose container ports
   - Use internal networks
   - Route through reverse proxy

---

## Next Steps

**You now have:**
- ✅ Active firewall
- ✅ Only necessary ports open
- ✅ Default deny policy
- ✅ Rate limiting on SSH
- ✅ Logging enabled

**In Chapter 10:**
- Install Fail2Ban
- Auto-ban repeated failed login attempts
- Configure jails for SSH and web services
- Monitor ban list
- Integrate with firewall

**Your server is much more secure now!**

---

## Quick Reference

### Essential Commands

```bash
# Enable/disable firewall
sudo ufw enable
sudo ufw disable

# Add rules
sudo ufw allow 80/tcp
sudo ufw limit 22/tcp
sudo ufw deny 5432/tcp

# View status
sudo ufw status
sudo ufw status verbose
sudo ufw status numbered

# Delete rule
sudo ufw delete 3
sudo ufw delete allow 80/tcp

# Logging
sudo ufw logging low
sudo tail -f /var/log/ufw.log

# Reload/reset
sudo ufw reload
sudo ufw reset
```

### Your Configuration

```bash
# Recreate your setup:
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw limit 2222/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

---

[← Previous: Chapter 8 - Firewall Fundamentals](08-firewall-fundamentals.md) | [Next: Chapter 10 - Fail2Ban →](10-fail2ban.md)
