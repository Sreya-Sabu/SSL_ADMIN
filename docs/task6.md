# Task 6 — Database Security

**Induction 2026 | Database Setup and Security Configuration**

---

## Overview

This task involves installing and securing a MariaDB database server on the VM. The goals are to:

- Create a dedicated database with a minimal-privilege user
- Disable remote root login and restrict access to localhost only
- Set up automated daily backups stored securely

---

## Step 1 — Install MariaDB

### Update Packages and Install MariaDB

```bash
sudo apt update
sudo apt install mariadb-server -y
```

- `apt update` — Refreshes the local package index so the system fetches the latest available versions from configured repositories.
- `apt install mariadb-server` — Installs the MariaDB database server (a community-maintained fork of MySQL).
- `-y` — Automatically answers "yes" to any confirmation prompts during installation.

### Start and Enable the MariaDB Service

```bash
sudo systemctl start mariadb
sudo systemctl enable mariadb
```

- `systemctl start mariadb` — Starts the MariaDB service immediately so the database is ready to use.
- `systemctl enable mariadb` — Registers MariaDB to start automatically every time the system boots, so the database is always available after a reboot without manual intervention.

### Verify MariaDB Is Running

```bash
sudo systemctl status mariadb
```

- Displays the current state of the MariaDB service. You should see `Active: active (running)` in green. This also shows the process ID (PID), memory usage, and recent log lines.

---

## Step 2 — Run the Security Script

```bash
sudo mysql_secure_installation
```

- `mysql_secure_installation` — An interactive script bundled with MariaDB that walks through a series of security hardening steps. It is the recommended first step after any fresh installation.

During the script, the following options were applied:

| Prompt | Action | Reason |
|--------|--------|--------|
| Set root password | Yes | Prevents unauthenticated root access |
| Remove anonymous users | Yes | Anonymous users allow access without credentials |
| Disable remote root login | Yes | Root should never be reachable over the network |
| Remove test database | Yes | The default `test` DB is accessible to all users |
| Reload privilege tables | Yes | Applies all changes immediately |

---

## Step 3 — Create the Database

### Open the MariaDB Shell

```bash
sudo mysql
```

- `sudo mysql` — Opens an interactive MariaDB shell as the root database user. `sudo` is required because root login uses Unix socket authentication by default on fresh installs.

### Create the Database

```sql
CREATE DATABASE secure_onboarding;
```

- Creates a new database named `secure_onboarding`. Using a dedicated, named database rather than the default schema keeps data organized and makes it easier to assign targeted permissions to users.

### Verify the Database Was Created

```sql
SHOW DATABASES;
```

- Lists all databases on the server. `secure_onboarding` should appear in the output alongside system databases like `mysql` and `information_schema`.

---

## Step 4 — Create a Minimal Privilege User

### Create the User

```sql
CREATE USER 'secureuser'@'localhost' IDENTIFIED BY 'StrongPassword123';
```

- `CREATE USER` — Creates a new database account.
- `'secureuser'@'localhost'` — The username is `secureuser` and the `@'localhost'` clause restricts this account to connections from the local machine only. The same username from a remote host would be treated as a completely different (non-existent) account.
- `IDENTIFIED BY 'StrongPassword123'` — Sets the password for this account.

### Grant Only the Necessary Permissions

```sql
GRANT SELECT, INSERT, UPDATE, DELETE 
ON secure_onboarding.* 
TO 'secureuser'@'localhost';
```

- `GRANT SELECT, INSERT, UPDATE, DELETE` — Grants only the four core data-manipulation permissions. Administrative operations like `DROP`, `CREATE`, `ALTER`, or `GRANT` are deliberately excluded.
- `ON secure_onboarding.*` — Restricts these permissions to all tables within the `secure_onboarding` database only. The user has zero access to any other database on the server.
- `TO 'secureuser'@'localhost'` — Applies these grants specifically to the `secureuser` account on localhost.

> **This follows the Principle of Least Privilege** — every account should have exactly the permissions it needs to function, and nothing more. This limits the damage if the account's credentials are ever compromised.

### Apply the Privilege Changes

```sql
FLUSH PRIVILEGES;
```

- Forces MariaDB to reload the grant tables from disk into memory, ensuring all permission changes take effect immediately without needing to restart the service.

### Verify the User's Permissions

```sql
SHOW GRANTS FOR 'secureuser'@'localhost';
```

- Displays the exact permissions assigned to `secureuser`. The output should show only `SELECT, INSERT, UPDATE, DELETE` on `secure_onboarding.*` — nothing else.

---

## Step 5 — Restrict Database Access to Localhost

### Edit the MariaDB Server Configuration

```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

- Opens the main MariaDB server configuration file. The `50-server.cnf` file under `mariadb.conf.d/` is where per-server settings are defined. The numeric prefix `50` controls the order in which config files are loaded.

### Confirm the Bind Address

Inside the file, locate and verify:

```ini
bind-address = 127.0.0.1
```

- `bind-address` — Tells MariaDB which network interface to listen on for incoming connections.
- `127.0.0.1` — The IPv4 loopback address (localhost). By binding to this address only, MariaDB refuses connections from any external IP, including LAN machines and the internet. If this were set to `0.0.0.0`, the database would be reachable from any network interface.

### Restart MariaDB to Apply the Change

```bash
sudo systemctl restart mariadb
```

- Fully restarts MariaDB, forcing it to re-read the configuration file and rebind to `127.0.0.1`.

### Verify the Listening Address

```bash
sudo ss -tulnp | grep 3306
```

- `ss -tulnp` — Lists all listening sockets with their associated processes (see Task 5 for a full flag breakdown).
- `| grep 3306` — Filters for port `3306`, the default MariaDB port.
- The output should show `127.0.0.1:3306`, confirming the database is bound exclusively to localhost and not exposed to any external interface.

---

## Step 6 — Create the Automated Backup Script

### Create a Secure Backup Directory

```bash
sudo mkdir /secure_backups
sudo chmod 700 /secure_backups
```

- `mkdir /secure_backups` — Creates the directory where all database backup files will be stored.
- `chmod 700 /secure_backups` — Sets the directory permissions to `700`:
  - `7` (owner) — Read, write, and execute for root.
  - `0` (group) — No permissions for any group members.
  - `0` (others) — No permissions for any other users.
  
  This ensures only root can list, read, or write backup files — preventing other users from viewing or tampering with database dumps.

### Create the Backup Script

```bash
sudo nano /usr/local/bin/dbbackup.sh
```

- Opens a new file at `/usr/local/bin/dbbackup.sh` in the `nano` editor. `/usr/local/bin/` is the standard location for locally-added administrator scripts — it is included in the system `$PATH` so the script can be called by name from anywhere.

**Script content:**

```bash
#!/bin/bash

DATE=$(date +%F)

mysqldump secure_onboarding > /secure_backups/db_$DATE.sql

gzip /secure_backups/db_$DATE.sql
```

Line-by-line explanation:

- `#!/bin/bash` — The "shebang" line. Tells the OS to execute this script using the Bash shell.
- `DATE=$(date +%F)` — Runs the `date` command and captures its output into the variable `DATE`. The `+%F` format produces an ISO 8601 date string like `2025-07-14`, which makes backup files easy to sort and identify.
- `mysqldump secure_onboarding > /secure_backups/db_$DATE.sql` — `mysqldump` is a utility that exports the entire contents of a database to a plain-text SQL file. The `>` operator redirects this output to a file named `db_2025-07-14.sql` (using today's date). The resulting file contains all `CREATE TABLE` and `INSERT` statements needed to fully restore the database.
- `gzip /secure_backups/db_$DATE.sql` — Compresses the `.sql` file using gzip, producing `db_2025-07-14.sql.gz`. Compression reduces disk usage significantly for large databases and replaces the original uncompressed file.

### Make the Script Executable

```bash
sudo chmod +x /usr/local/bin/dbbackup.sh
```

- `chmod +x` — Adds execute permission to the script file. Without this, the OS would refuse to run it as a program and would return a "Permission denied" error.

### Test the Script Manually

```bash
sudo /usr/local/bin/dbbackup.sh
```

- Runs the backup script immediately to verify it works correctly before scheduling it via cron.

### Verify the Backup Was Created

```bash
sudo ls /secure_backups
```

- Lists the contents of the backup directory. You should see a `.sql.gz` file named with today's date, confirming the script ran successfully.

---

## Step 7 — Schedule Daily Automated Backups with Cron

### Open the Root Crontab

```bash
sudo crontab -e
```

- `crontab -e` — Opens the cron job configuration file for the current user (root, due to `sudo`) in the default text editor. Cron is the Unix job scheduler that runs commands at specified times.

### Add the Backup Schedule

```cron
0 2 * * * /usr/local/bin/dbbackup.sh
```

Cron uses five time fields followed by the command:

```
┌───── Minute     (0–59)
│ ┌─── Hour       (0–23)
│ │ ┌─ Day        (1–31)
│ │ │ ┌ Month     (1–12)
│ │ │ │ ┌ Weekday (0–7, 0 and 7 = Sunday)
│ │ │ │ │
0 2 * * * /usr/local/bin/dbbackup.sh
```

- `0 2 * * *` — At minute 0 of hour 2 (2:00 AM), every day of every month, every day of the week.
- `/usr/local/bin/dbbackup.sh` — The full path to the backup script to execute.

Running backups at 2 AM minimizes impact on database performance during peak usage hours.

### Verify the Cron Job Is Registered

```bash
sudo crontab -l
```

- Lists all cron jobs for the root user. The backup entry should appear exactly as added. This confirms the schedule is active.

---

## Security Measures Summary

| Category | Measure | Command / Setting |
|----------|---------|-------------------|
| Authentication | Root password set | `mysql_secure_installation` |
| User access | Anonymous users removed | `mysql_secure_installation` |
| Remote access | Remote root login disabled | `mysql_secure_installation` |
| Housekeeping | Test database removed | `mysql_secure_installation` |
| Least privilege | Dedicated user with minimal grants | `GRANT SELECT, INSERT, UPDATE, DELETE` |
| Network exposure | Database bound to localhost only | `bind-address = 127.0.0.1` |
| Backup security | Restricted backup directory | `chmod 700 /secure_backups` |
| Continuity | Automated daily compressed backups | Cron at 2 AM |

---

## Verification Commands

Use these commands to confirm all security measures are in place:

```bash
# List all databases
sudo mysql -e "SHOW DATABASES;"

# List all database users and their allowed hosts
sudo mysql -e "SELECT user, host FROM mysql.user;"

# Confirm MariaDB is only listening on localhost
sudo ss -tulnp | grep 3306

# List backup files
sudo ls /secure_backups

# Check scheduled cron jobs
sudo crontab -l
```

---

## Conclusion

MariaDB was successfully installed and hardened with the following outcomes:

- The `secure_onboarding` database was created and is accessible only to `secureuser` with minimal privileges.
- Remote root access was disabled and the database is bound exclusively to `127.0.0.1`, making it unreachable from any external machine.
- A compressed daily backup runs automatically at 2 AM, stored in a root-only directory at `/secure_backups`.

All required database security objectives were successfully achieved.