# Chapter 8: Firewall Fundamentals

**Understanding Firewalls and Network Security**

---

## Learning Objectives

By the end of this chapter, you'll be able to:
- ✅ Understand what a firewall does
- ✅ Explain ports and network traffic
- ✅ Understand UFW (Uncomplicated Firewall)
- ✅ Know which ports to allow/deny
- ✅ Understand firewall rules and policies
- ✅ Plan firewall configuration

**Time Required:** 20-30 minutes

**Note:** This chapter is theory. Chapter 9 covers hands-on configuration.

---

## What is a Firewall?

### Basic Concept

**Firewall = Security guard for your server**

**Real-world analogy:**
- **Server** = Building
- **Network traffic** = People trying to enter
- **Firewall** = Security guard at door
- **Firewall rules** = Guest list

**The guard checks:**
- Who's trying to enter? (IP address)
- Which door are they using? (port number)
- Are they allowed? (firewall rule)

**Action:**
- ✅ Allow = Let them in
- ❌ Deny = Turn them away
- 🔇 Drop = Ignore completely (no response)

---

### Why You Need a Firewall

**Without firewall:**
```
Internet → Your Server
Anyone can try to connect to ANY service on ANY port
```

**With firewall:**
```
Internet → Firewall → Your Server
Only approved connections get through
```

**What firewall prevents:**
- Unauthorized access to services
- Port scanning attacks
- Exploitation of unused services
- Direct attacks on vulnerable ports

**Example:**
- PostgreSQL runs on port 5432
- Without firewall: Anyone can try to connect
- With firewall: Only localhost can connect
- Attackers can't even see it's running!

---

## Understanding Ports

### What are Ports?

**Port = Virtual door on your server**

**Your server has 65,535 doors (ports):**
- Ports 0-1023: Well-known (system) ports
- Ports 1024-49151: Registered (application) ports
- Ports 49152-65535: Dynamic (temporary) ports

**Each service uses a specific port:**

| Service | Port | Purpose |
|---------|------|---------|
| SSH | 22 | Remote server access |
| HTTP | 80 | Web traffic (unencrypted) |
| HTTPS | 443 | Web traffic (encrypted) |
| FTP | 21 | File transfer |
| SMTP | 25 | Email sending |
| PostgreSQL | 5432 | Database |
| MySQL | 3306 | Database |
| Redis | 6379 | Cache |

---

### Port Analogy

**Think of your server as a building:**

**Port 80 (HTTP):**
- Front door for customers
- Open to public
- Anyone can enter

**Port 443 (HTTPS):**
- Main entrance with security
- Public, but encrypted
- Verified visitors

**Port 22 (SSH):**
- Staff entrance
- Key required
- Employees only

**Port 5432 (PostgreSQL):**
- Server room door
- Internal access only
- No external entry

---

### How Ports Work

**When traffic arrives at your server:**

1. **Packet arrives:** "I want to connect to port 80"
2. **Firewall checks:** "Is port 80 allowed?"
3. **If allowed:** Forwards to service on port 80
4. **If denied:** Blocks packet
5. **Service responds:** Sends data back through port 80

**Example connection:**
```
Your browser (192.0.2.50:54321) 
    → 
Your server (192.0.2.100:443)
```

**Breakdown:**
- Source: Your computer (random high port 54321)
- Destination: Your server port 443 (HTTPS)
- Firewall: Checks if port 443 is allowed
- Result: Connection allowed, website loads

---

## Common Ports Reference

### Web Services

**Port 80 - HTTP**
- Standard web traffic
- Not encrypted
- Redirects to HTTPS
- Must be open for Let's Encrypt

**Port 443 - HTTPS**
- Encrypted web traffic
- Modern standard
- Required for websites
- SSL/TLS certificates

---

### Remote Access

**Port 22 - SSH**
- Secure shell access
- Remote server management
- SFTP file transfers
- Should be restricted

**Port 3389 - RDP**
- Windows Remote Desktop
- Not used on Linux
- Often attacked

---

### Databases

**Port 5432 - PostgreSQL**
- Database connections
- Should be internal only
- Never expose to internet
- Use SSH tunnel if remote access needed

**Port 3306 - MySQL**
- MySQL/MariaDB database
- Same security as PostgreSQL
- Internal only

**Port 27017 - MongoDB**
- NoSQL database
- Internal only
- Often targeted by attackers

**Port 6379 - Redis**
- Cache/session store
- Internal only
- No authentication by default!

---

### Other Services

**Port 25 - SMTP**
- Email sending
- Often blocked by providers
- Use port 587 instead

**Port 53 - DNS**
- Domain name resolution
- Usually not needed on app servers
- Handled by system

**Port 123 - NTP**
- Time synchronization
- Usually allowed outbound only

---

## Firewall Policies

### Default Policies

**Two approaches:**

**1. Default Deny (Recommended)**
```
Block everything by default
Explicitly allow what's needed
```

**Benefits:**
- Most secure
- Must think about what to allow
- Unknown services blocked
- Industry standard

**2. Default Allow (Dangerous)**
```
Allow everything by default
Explicitly block what's dangerous
```

**Problems:**
- Must remember to block
- New services exposed automatically
- Easy to miss vulnerabilities
- Not recommended

**We'll use Default Deny!**

---

### Rule Priority

**Firewall rules are checked in order:**

```
1. Allow SSH from my IP only
2. Allow SSH from anywhere
3. Deny all
```

**Result:** Rule 1 matches first, rule 2 never used.

**Order matters!**
- More specific rules first
- General rules last
- Default policy at end

---

## UFW (Uncomplicated Firewall)

### What is UFW?

**UFW = Uncomplicated Firewall**

**Built on iptables:**
- iptables: Powerful but complex
- UFW: Simple interface to iptables
- Perfect for basic firewall needs
- Pre-installed on Ubuntu

**Philosophy:**
- Simple commands
- Sane defaults
- Easy to understand
- Hard to mess up

---

### UFW vs iptables

**iptables command:**
```bash
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -P INPUT DROP
```

**UFW equivalent:**
```bash
ufw allow 80
ufw allow 443
ufw enable
```

**Much simpler!**

---

### UFW Features

**Capabilities:**
- ✅ Allow/deny by port
- ✅ Allow/deny by IP address
- ✅ Application profiles
- ✅ IPv4 and IPv6 support
- ✅ Logging
- ✅ Rate limiting

**Limitations:**
- Not as flexible as iptables
- Basic traffic filtering only
- No advanced routing
- No packet inspection

**For web servers, UFW is perfect!**

---

## Planning Your Firewall

### What Ports to Open?

**Think about services you'll run:**

**Web server (Caddy, Nginx, Apache):**
- ✅ Port 80 (HTTP)
- ✅ Port 443 (HTTPS)

**SSH access:**
- ✅ Port 22 (or custom port like 2222)

**Database (PostgreSQL):**
- ❌ Port 5432 - Keep internal only!

**Docker containers:**
- ❌ Don't expose directly
- ✅ Route through reverse proxy (Caddy)

---

### Minimal Configuration

**For this course setup:**

```
Port 2222 - SSH (custom port from Chapter 7)
Port 80   - HTTP (redirects to HTTPS)
Port 443  - HTTPS (main web traffic)
```

**Everything else blocked!**

**This allows:**
- Web traffic to your sites
- SSH access for management
- Certificate renewal (Let's Encrypt)

**This blocks:**
- Database access from internet
- All other ports
- Unknown services

---

### Security Principles

**Principle 1: Least Privilege**
- Only open what's necessary
- Close everything else
- Review regularly

**Principle 2: Defense in Depth**
- Firewall is one layer
- Also use: SSH hardening, Fail2Ban, strong passwords
- Multiple security layers

**Principle 3: Fail Secure**
- If unsure, block it
- Can always open later
- Easier to open than close

**Principle 4: Monitor and Log**
- Track what's blocked
- Review logs
- Detect attack patterns

---

## Traffic Types

### Inbound Traffic

**Coming TO your server:**
- User visiting website (port 443)
- You connecting via SSH (port 2222)
- Attacker port scanning (blocked!)

**This is what firewall mainly controls.**

**Default policy:** Deny all inbound
**Exceptions:** Ports we explicitly allow

---

### Outbound Traffic

**Going FROM your server:**
- Server downloading updates
- Sending email
- Connecting to APIs
- DNS lookups

**Usually allowed by default.**

**Why allow outbound?**
- Server needs to download packages
- Can't function isolated
- Can restrict specific ports if needed

---

### Stateful Firewall

**UFW is stateful:**

**What this means:**
```
1. You SSH to server (outbound from your computer)
2. Server responds (inbound to your computer)
3. Firewall remembers this is part of same connection
4. Response is automatically allowed
```

**Benefits:**
- Don't need separate rules for responses
- More secure
- Easier to configure

**Tracks connection state:**
- NEW: First packet of new connection
- ESTABLISHED: Part of existing connection
- RELATED: Related to existing connection

---

## Attack Scenarios

### Without Firewall

**Attacker scans your server:**
```bash
nmap YOUR_SERVER_IP
```

**Results:**
```
Port 22   - SSH (open)
Port 80   - HTTP (open)
Port 443  - HTTPS (open)
Port 5432 - PostgreSQL (open) ← DANGER!
Port 6379 - Redis (open) ← DANGER!
```

**Attacker sees vulnerabilities:**
- Can try to access database
- Can try Redis exploits
- Knows what services are running
- Can target specific versions

---

### With Firewall

**Attacker scans your server:**
```bash
nmap YOUR_SERVER_IP
```

**Results:**
```
Port 22   - Filtered (blocked)
Port 80   - Open
Port 443  - Open
Port 2222 - Open (custom SSH port)
Port 5432 - Filtered (blocked)
Port 6379 - Filtered (blocked)
```

**Attacker sees:**
- Only web ports open
- Can't access database
- Can't find SSH (on non-standard port)
- Most services invisible

**"Filtered" means firewall is blocking.**

---

## Firewall Policies Explained

### Allow Policy

**Rule:** `ufw allow 443`

**Meaning:**
- Accept incoming connections on port 443
- From any IP address
- Both IPv4 and IPv6
- TCP and UDP (both protocols)

**Use for:**
- Public web services
- Services accessible to everyone

---

### Deny Policy

**Rule:** `ufw deny 5432`

**Meaning:**
- Reject incoming connections on port 5432
- Sends rejection packet back
- Attacker knows port exists but is closed

**Use for:**
- Explicitly blocking services
- Testing configurations

**Note:** Deny tells attacker port exists!

---

### Reject vs Drop

**Deny (Reject):**
```
Attacker: "Can I connect to port 5432?"
Server: "No, connection refused."
```

**Drop:**
```
Attacker: "Can I connect to port 5432?"
Server: [silence]
```

**Drop is more secure:**
- No response = attacker doesn't know if port exists
- Slows down port scans
- Default for unmatched traffic

**UFW uses drop by default for unmatched traffic.**

---

## Common Mistakes

### Mistake 1: Forgetting SSH Port

**Scenario:**
```bash
ufw enable
# Oops! Didn't allow SSH port first
```

**Result:**
- Locked out of server
- Can't SSH in
- Need console access to fix

**Always allow SSH before enabling firewall!**

---

### Mistake 2: Opening Database Ports

**Bad configuration:**
```bash
ufw allow 5432  # PostgreSQL to internet!
```

**Why this is dangerous:**
- Database exposed to internet
- Brute force attacks possible
- One weak password = compromised data

**Correct approach:**
- Don't open database port
- Access via SSH tunnel if needed
- Use localhost connections only

---

### Mistake 3: Default Allow Policy

**Dangerous:**
```bash
ufw default allow incoming
```

**Problems:**
- Everything open by default
- New services automatically exposed
- Must remember to block vulnerabilities

**Always use:**
```bash
ufw default deny incoming
ufw default allow outgoing
```

---

### Mistake 4: Not Testing

**Problem scenario:**
```bash
ufw allow 80
ufw allow 443
ufw enable
# Forgot to allow SSH on custom port 2222!
```

**Result:** Locked out

**Solution:**
- Keep existing SSH connection open
- Test in new window
- Don't close original until verified

---

## IPv4 vs IPv6

### Understanding Both

**IPv4:**
- Old standard: 192.0.2.100
- 4 billion addresses
- Nearly exhausted
- Everyone knows this format

**IPv6:**
- New standard: 2001:db8::1
- 340 undecillion addresses
- Future proof
- Adoption growing

**UFW handles both automatically!**

---

### Firewall Implications

**UFW rules apply to both by default:**
```bash
ufw allow 443
```

**Allows:**
- IPv4 connections to port 443
- IPv6 connections to port 443

**If you want IPv4 only:**
```bash
ufw allow from any to any port 443 proto tcp # IPv4
```

**Usually, allowing both is fine.**

---

## Rate Limiting

### What is Rate Limiting?

**Prevents brute force attacks:**

**Without rate limiting:**
```
Attacker tries 1000 password attempts per second
Eventually might guess correct password
```

**With rate limiting:**
```
Attacker tries 6+ connections in 30 seconds
UFW blocks their IP temporarily
Brute force becomes impractical
```

**UFW rate limiting:**
```bash
ufw limit 22
```

**Limits to 6 connections per 30 seconds per IP.**

---

### When to Use Rate Limiting

**Good for:**
- ✅ SSH ports (limit brute force)
- ✅ Admin panels
- ✅ Login endpoints

**Don't use for:**
- ❌ Web traffic (port 80/443)
- ❌ APIs with legitimate high traffic
- ❌ Services behind load balancers

**Why not for web traffic?**
- Legitimate users might make many requests
- Would block real traffic
- Use Fail2Ban instead (Chapter 10)

---

## Logging

### Firewall Logs

**UFW can log blocked traffic:**

**Log levels:**
- `off` - No logging
- `low` - Blocked packets only
- `medium` - Low + invalid packets
- `high` - Medium + rate limiting
- `full` - Everything (lots of data!)

**For this course:** We'll use `low`

**Log location:** `/var/log/ufw.log`

**Example log entry:**
```
Nov 12 10:30:45 server kernel: [UFW BLOCK] IN=eth0 OUT= SRC=192.0.2.50 DST=192.0.2.100 PROTO=TCP DPT=5432
```

**Translation:**
- Blocked incoming connection
- From 192.0.2.50
- To our server port 5432 (PostgreSQL)
- This is an attack attempt!

---

## Verification Tools

### Testing Your Firewall

**From outside:**
```bash
# Port scanning (on YOUR computer, not server)
nmap YOUR_SERVER_IP

# Test specific port
nc -zv YOUR_SERVER_IP 443

# Test SSH
ssh -p 2222 deploy@YOUR_SERVER_IP
```

**On server:**
```bash
# Check UFW status
sudo ufw status verbose

# Check listening ports
sudo ss -tlnp

# Check active connections
sudo ss -tnp
```

---

## Next Chapter Preview

**In Chapter 9, you'll:**
- Install UFW (if not already installed)
- Configure firewall rules
- Allow SSH, HTTP, HTTPS
- Enable the firewall
- Test configuration
- Review logs

**Critical steps:**
- Allow SSH port BEFORE enabling
- Test in separate terminal
- Keep existing connection open
- Don't lock yourself out!

---

## Key Takeaways

**Remember:**

1. **Firewall = Security guard**
   - Controls what traffic gets in
   - Default deny is most secure
   - Only open necessary ports

2. **Ports = Virtual doors**
   - Each service uses specific port
   - Well-known ports: 0-1023
   - Common: 22 (SSH), 80 (HTTP), 443 (HTTPS)

3. **Internal services should stay internal**
   - Never expose databases to internet
   - Use SSH tunnels for remote access
   - Reverse proxy for web services

4. **UFW makes firewall easy**
   - Simple commands
   - Sane defaults
   - Built on iptables

5. **Plan before implementing**
   - Know what ports you need
   - Test before enabling
   - Keep SSH connection open

6. **Rate limiting prevents brute force**
   - Use on SSH port
   - Blocks rapid connection attempts
   - Slows down attackers

---

## Quick Reference

### Common Ports

```
22    - SSH (or custom like 2222)
80    - HTTP
443   - HTTPS
3306  - MySQL
5432  - PostgreSQL
6379  - Redis
```

### UFW Concepts (Preview for Chapter 9)

```bash
# Will use in next chapter:
ufw allow 22
ufw deny 5432
ufw limit 22
ufw enable
ufw status
```

### Security Principles

- Default deny incoming
- Default allow outgoing
- Least privilege
- Defense in depth
- Monitor and log

---

[← Previous: Chapter 7 - SSH Hardening](07-ssh-hardening.md) | [Next: Chapter 9 - Firewall Configuration →](09-firewall-configuration.md)
