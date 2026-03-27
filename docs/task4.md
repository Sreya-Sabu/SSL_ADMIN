# Task 4 – User and Permission Management

A complete guide covering user creation, privilege assignment, home directory security, ACL-based audit access, disk quotas, and automated backups.

---

## Step 1 — Create Users

```bash
sudo adduser exam_1
sudo adduser exam_2
sudo adduser exam_3
sudo adduser examadmin
sudo adduser examaudit
```

Creates five users: `exam_1`, `exam_2`, `exam_3` as regular users, `examadmin` as the privileged admin, and `examaudit` as the read-only auditor. You'll be prompted to set a password for each.

---

## Step 2 — Grant examadmin Root Privileges

```bash
sudo usermod -aG sudo examadmin
```

Adds `examadmin` to the `sudo` group, giving it root-level access.

**Verify it works:**

```bash
su - examadmin
sudo whoami
# Expected output: root
exit
```

---

## Step 3 — Secure Home Directory Permissions

```bash
sudo chmod 700 /home/exam_1
sudo chmod 700 /home/exam_2
sudo chmod 700 /home/exam_3
```

Sets each exam user's home directory to `700`:
- **Owner** → full read, write, execute access
- **Group / Others** → no access at all

**Check permissions:**

```bash
ls -ld /home/exam_1
# Expected: drwx------ 2 exam_1 exam_1 ...
```

---

## Step 4 — Give examaudit Read-Only Access via ACL

Standard Linux permissions only support owner/group/others. Since `examaudit` needs access to all exam home directories *without* grouping those users together, we use **ACL (Access Control Lists)**.

**Install ACL tools:**

```bash
sudo apt install acl
```

**Grant read + enter (`rx`) access to examaudit:**

```bash
sudo setfacl -m u:examaudit:rx /home/exam_1
sudo setfacl -m u:examaudit:rx /home/exam_2
sudo setfacl -m u:examaudit:rx /home/exam_3
```

- `setfacl` → set Access Control List
- `-m` → modify existing ACL
- `u:examaudit:rx` → grant user `examaudit` read + execute (enter) rights

**Verify ACL is set:**

```bash
getfacl /home/exam_1
# Should include: user:examaudit:r-x
```

**Test examaudit access:**

```bash
su - examaudit
cd /home/exam_1
ls                # Should work ✓
touch test.txt    # Should fail — no write access ✓
exit
```

---

## Step 5 — Set Up Disk Quotas

Disk quotas prevent any single user from filling up the disk (e.g., from runaway scripts or log files).

**Install quota tools:**

```bash
sudo apt install quota quotatool
```

**Enable quota in fstab:**

```bash
sudo nano /etc/fstab
```

Add `usrquota` to your root filesystem mount options, for example:

```
UUID=... / ext4 defaults,usrquota 0 1
```

**Remount the filesystem to apply changes:**

```bash
sudo mount -o remount /
```

**Create the quota database:**

```bash
sudo quotacheck -cum /
```

**Enable quotas:**

```bash
sudo quotaon /
```

> **Note:** If you see `Quota format not supported in kernel`, install the extra kernel modules:
> ```bash
> sudo apt install linux-modules-extra-$(uname -r)
> ```
> Then reboot and retry `sudo quotaon /`.

**Set limits per user (opens a text editor):**

```bash
sudo edquota exam_1
```

Example values to set inside the editor:

```
Filesystem  blocks  soft    hard    inodes  soft  hard
/dev/root   0       500000  550000  0       0     0
```

- `soft` = warning threshold (~500 MB)
- `hard` = absolute maximum (~550 MB)

Repeat for `exam_2` and `exam_3`.

---

## Step 6 — Backup Script

### Create a Secure Backup Directory

```bash
sudo mkdir /backups
sudo chown examadmin:examadmin /backups
sudo chmod 700 /backups
```

Only `examadmin` can access `/backups`. All others are denied.

### Create the Script

```bash
sudo nano /usr/local/bin/exam_backup.sh
```

Paste the following:

```bash
#!/bin/bash

DATE=$(date +%Y-%m-%d)

tar -czf /backups/exam_backup_$DATE.tar.gz /home/exam_*

echo "Backup completed"
```

**Line-by-line explanation:**

| Line | What it does |
|------|--------------|
| `#!/bin/bash` | Tells the system to run this script using bash |
| `DATE=$(date +%Y-%m-%d)` | Captures today's date (e.g., `2026-03-27`) so each backup has a unique filename |
| `tar -czf ...` | Creates a compressed `.tar.gz` archive of all `exam_*` home directories |
| `/home/exam_*` | Wildcard that matches `exam_1`, `exam_2`, `exam_3` home folders |
| `echo "Backup completed"` | Prints a confirmation message when done |

### Set Permissions and Ownership

```bash
sudo chmod 700 /usr/local/bin/exam_backup.sh
sudo chown examadmin:examadmin /usr/local/bin/exam_backup.sh
```

Only `examadmin` can read or execute the script.

### Test the Script

```bash
su - examadmin
/usr/local/bin/exam_backup.sh
ls /backups
# Expected: exam_backup_2026-03-27.tar.gz
exit
```

### Schedule Daily Backups with Cron

```bash
su - examadmin
crontab -e
```

Add this line:

```
0 2 * * * /usr/local/bin/exam_backup.sh
```

| Field | Value | Meaning |
|-------|-------|---------|
| Minute | `0` | At the 0th minute |
| Hour | `2` | At 2 AM |
| Day | `*` | Every day |
| Month | `*` | Every month |
| Weekday | `*` | Any day of the week |

The script will now run automatically every night at **2:00 AM**.

---

## Summary

| Task | What Was Done |
|------|---------------|
| User creation | Created 5 users with `adduser` |
| Admin privileges | `examadmin` added to `sudo` group |
| Home dir security | `chmod 700` applied to exam user homes |
| Audit access | ACL used to give `examaudit` read-only access without breaking isolation |
| Disk quotas | Quota limits set to prevent disk abuse |
| Backup script | Compressed daily backup of exam homes, restricted to `examadmin` |
| Cron job | Backup runs automatically at 2 AM daily |