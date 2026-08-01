# 05-Log Archiving, logrotate, and crontab Scheduled Tasks

#Linux #LogManagement #LogArchive #logrotate #crontab #tar #gzip #find #mail #Transport #SRE

---

## Recommended Path

01-Linux Foundation and Host Maintenance/02-Logs and Text Processing/05-Log Archiving, logrotate, and crontab Scheduled Tasks.md

---

## I. Document Explanation

This document organizes common **log archiving, log moving, log compression, log deletion, logrotate log rotation, and crontab scheduled tasks** scenarios in Linux operations.

This article focuses on:

- Why log archiving is needed
- Designing log retention periods
- Manually archiving logs
- Finding logs from 7 days ago
- Finding archived logs from 30 days ago
- Packaging logs with tar
- Compressing logs with gzip
- Moving old logs to an archive directory
- Deleting expired logs
- Safe deletion process
- logrotate basic configuration
- logrotate common parameters
- Differences between copytruncate and create
- postrotate scripts
- Manually testing logrotate
- crontab basic format
- Scheduling log archiving scripts
- cron `%` escape issues
- cron output redirection
- Email notification after archiving

This article is the 5th in the Logs and Text Processing series, mainly solving:

```text
How to get the log to be archived, compressed, rotated and cleaned on a periodic basis, avoiding unlimited growth of the log to fill disks
```

The goals are:

- Manually archive old logs
- Compress historical logs
- Move logs to an archive directory
- Safely delete expired logs
- Configure logrotate for automatic log rotation
- Use crontab to schedule log archiving tasks
- Output archive results to a log or send notifications
- Understand the risk boundaries of log cleanup in production

---

## II. Why Log Archiving and Rotation Are Needed

In production environments, if logs are not managed, common issues include:

```text
Log files grow indefinitely

/var/log Full system disk.

/data Log directory full of data discs

inode We're running out of little logs.

Application cannot continue writing logs

Service startup failed

Writing to database or intermediate failed

Docker The container log is bursting. /var/lib/docker

It's too big to open when you're checking.

Historic logs piled up unfailingly.
```

Therefore, we need to establish a log lifecycle:

```text
Writing in Real Time

→ Cut by day or size

→ History Log Compression

→ Archive to specified directory

→ Keep fixed cycle

→ Overdue cleanup

→ Important log forwards to log platform
```

One-sentence understanding:

```text
Logs can't just be checked, they have to be run.
```

---

## III. Overall Approach to Log Management

Recommended approach:

```text
Confirm log path first

→ Reconfirming log growth

→ Redetermination of the retention cycle

→ Reconfigure Rotation and Compression

→ Configure Archive Directory

→ Reconfigure timed cleanup

→ Last access to surveillance and alarm.
```

Common management methods:

```text
logrotate
→ Standard log rotation tool

crontab + find + tar/gzip
→ Time filing and cleaning

Apply your own log configuration
→ Cut by size, by sky

Docker daemon Log Limit
→ Prevent infinity of container logs

Log Platform
→ Centralized storage, retrieval, retention cycle management
```

---

## IV. Designing Log Retention Periods

---

## Scenario 1: Common Log Retention Strategies

Common strategies:

```text
Keep this machine 7 Skylight Log

Keep this machine 30 Day Compressed Archive

Important business logs forwarded to log platform

Security audit log maintained longer cycle

Debug logs keep shorter cycles

Keep error logs for longer periods
```

Example:

```text
Apply access.log
→ Here. 7-15 days

Apply error.log
→ Here. 30 days

Nginx access.log
→ Here. 7-15 days

Nginx error.log
→ Here. 30 days

Audit log
→ Retain as required by corporate compliance

Debug Log
→ Try not to open in production for long.
```

---

## Scenario 2: What to Consider When Designing Retention Periods

When designing retention periods, consider:

```text
Disk Capacity

Log growth rate

We need a fail-check.

Audit compliance requirements

Has the log platform been collected

Is there a backup?

Whether sensitive information is involved

Whether or not to affect operational performance
```

Do not simply:

```text
When you see a big log, just delete it.
```

Instead, first confirm:

```text
Is there a barrier?

Has it been archived?

Whether log platform is uploaded

Compliance requirements

Is there a confirmation from the business party?
```

---

## V. Finding Logs to Archive

---

## Scenario 3: Finding Logs from 7 Days Ago

### Command

```bash
find /var/log/myapp -type f -name "*.log" -mtime +7
```

### Explanation

```text
-mtime +7
→ Change earlier than 7 Day before
```

Suitable for finding historical logs to archive or compress.

---

## Scenario 4: Finding Logs from the Last 7 Days

### Command

```bash
find /var/log/myapp -type f -name "*.log" -mtime -7
```

### Explanation

```text
-mtime -7
→ Recent 7 Modified within days
```

Suitable for confirming recently active logs.

---

## Scenario 5: Finding Archived Files from 30 Days Ago

### Command

```bash
find /data/log_archive -type f -mtime +30
```

### Applicable Scenario

```text
Clear Expiry Archive

Check if archive directories accumulate for long periods

Confirm archive retention cycle
```

---

## Scenario 6: Finding Logs Larger than 500M

### Command

```bash
find /var/log/myapp -type f -name "*.log" -size +500M -exec ls -lh {} \;
```

### Applicable Scenario

```text
Unusual growth log detected

Positioning disk to the big house.

Whether to rotate early.
```

---

## Scenario 7: Checking the Total Size of a Log Directory

### Command

```bash
du -sh /var/log/myapp
```

Check the next level:

```bash
du -h --max-depth=1 /var/log/myapp | sort -hr
```

### Applicable Scenario

```text
Judge the total occupancy of the log directory

Find out which subdirectories are largest

Provides a basis for archiving policy
```

---

## VI. Manually Archiving Logs

---

## Scenario 8: Archiving Logs from 7 Days Ago with One Command

### Command

```bash
find /var/log/myapp -type f -name "*.log" -mtime +7 -print0 | tar --null -czf /tmp/myapp-log-$(date +%F).tar.gz --files-from=-
```

### Approach

```text
find
→ Find 7 It's from the sky. .log Documentation

-print0
→ Use NULL Character separator filenames, special characters such as compatible spaces

tar --null
→ Press NULL Separate Read File List

-czf
→ Create gzip Compressors

--files-from=-
→ Read File List From Standard Inputs
```

### Applicable Scenario

```text
Batch Archive Old Log

Avoids errors when filenames are available

Manual temporary archive

Delete the original log after archiving
```

---

## Scenario 9: Generate File List Before Archiving

### Step 1: Generate File List

```bash
find /var/log/myapp -type f -name "*.log" -mtime +7 > /tmp/myapp-old-log-list.txt
```

### Step 2: View the List

```bash
cat /tmp/myapp-old-log-list.txt
```

### Step 3: Archive

```bash
tar czf /tmp/myapp-log-$(date +%F).tar.gz -T /tmp/myapp-old-log-list.txt
```

### Step 4: Confirm the Archive

```bash
ls -lh /tmp/myapp-log-$(date +%F).tar.gz
```

### Explanation

```text
-T
→ Read the package from the file list
```

Advantages of this method:

```text
You can manually confirm the file list first.

Careful operation appropriate to the production environment

Enables file lists to be kept as queuing evidence
```

---

## Scenario 10: Archive to a Specified Directory

### Create Archive Directory

```bash
mkdir -p /data/log_archive
```

### Archive

```bash
find /var/log/myapp -type f -name "*.log" -mtime +7 -print0 | tar --null -czf /data/log_archive/myapp-log-$(date +%F).tar.gz --files-from=-
```

### View Results

```bash
ls -lh /data/log_archive/
```

---

## Scenario 11: Archive by Date Directory

### Create Date Directory

```bash
mkdir -p /data/log_archive/$(date +%F)
```

### Move Old Logs

```bash
find /var/log/myapp -type f -name "*.log" -mtime +7 -exec mv {} /data/log_archive/$(date +%F)/ \;
```

### View Results

```bash
ls -lh /data/log_archive/$(date +%F)/
```

---

## VII. Compressing Historical Logs

---

## Scenario 12: Compress Logs from 7 Days Ago

### Command

```bash
find /var/log/myapp -type f -name "*.log" -mtime +7 -exec gzip {} \;
```

### Explanation

After execution:

```text
app.log
→ app.log.gz
```

### Applicable Scenario

```text
History log still to be kept

But I want to save disk space.

It doesn't have to be direct. grep Original Log
```

---

## Scenario 13: Preview Before Compression

### Command

```bash
find /var/log/myapp -type f -name "*.log" -mtime +7
```

Confirm accuracy before execution:

```bash
find /var/log/myapp -type f -name "*.log" -mtime +7 -exec gzip {} \;
```

---

## Scenario 14: View Compressed Logs

### View Without Decompression

```bash
zcat /var/log/myapp/app.log.gz | head
```

### Search Compressed Logs

```bash
zgrep -i "error" /var/log/myapp/app.log.gz
```

### Count Errors in Compressed Logs

```bash
zgrep -i "error" /var/log/myapp/app.log.gz | wc -l
```

---

## VIII. Moving Old Logs

---

## Scenario 15: Move Logs from 7 Days Ago to Archive Directory

### Create Directory

```bash
mkdir -p /data/log_archive
```

### Move Logs

```bash
find /var/log/myapp -type f -name "*.log" -mtime +7 -exec mv {} /data/log_archive/ \;
```

### View Archive Directory

```bash
ls -lh /data/log_archive/
```

---

## Scenario 16: Preview Before Moving

### Command

```bash
find /var/log/myapp -type f -name "*.log" -mtime +7
```

Confirm accuracy before execution:

```bash
find /var/log/myapp -type f -name "*.log" -mtime +7 -exec mv {} /data/log_archive/ \;
```

---

## Scenario 17: Move to Date Directory

### Command

```bash
mkdir -p /data/log_archive/$(date +%F)
```

```bash
find /var/log/myapp -type f -name "*.log" -mtime +7 -exec mv {} /data/log_archive/$(date +%F)/ \;
```

---

## IX. Deleting Expired Logs

---

## Scenario 18: Delete Archived Files from 30 Days Ago

### Preview First

```bash
find /data/log_archive -type f -mtime +30
```

### Delete

```bash
find /data/log_archive -type f -mtime +30 -delete
```

---

## Scenario 19: Delete 30-Day-Old gz Compressed Logs

### Preview First

```bash
find /var/log/myapp -type f -name "*.gz" -mtime +30
```

### Delete

```bash
find /var/log/myapp -type f -name "*.gz" -mtime +30 -delete
```

---

## Scenario 20: Delete Empty Logs in a Specified Directory

### Preview First

```bash
find /var/log/myapp -type f -name "*.log" -empty
```

### Delete

```bash
find /var/log/myapp -type f -name "*.log" -empty -delete
```

---

## Scenario 21: Delete Using a File List

### Step 1: Generate Deletion List

```bash
find /data/log_archive -type f -mtime +30 > /tmp/delete-log-list.txt
```

### Step 2: Confirm the List

```bash
cat /tmp/delete-log-list.txt
```

### Step 3: Count Items

```bash
wc -l /tmp/delete-log-list.txt
```

### Step 4: Delete

```bash
cat /tmp/delete-log-list.txt | xargs rm -f
```

### Note

If filenames may contain spaces, avoid using ordinary `xargs rm -f` directly.

A safer approach:

```bash
find /data/log_archive -type f -mtime +30 -print0 | xargs -0 rm -f
```

---

## X. Pre-Deletion Backup Process /think

---

## Scenario 22: Delete After Packaging

### Step 1: Generate File List

```bash
find /var/log/myapp -type f -name "*.log" -mtime +30 > /tmp/delete-log-list.txt
```

### Step 2: Confirm File List

```bash
cat /tmp/delete-log-list.txt
```

### Step 3: Package Backup

```bash
tar czf /data/log_archive/delete-before-clean-$(date +%F).tar.gz -T /tmp/delete-log-list.txt
```

### Step 4: Confirm Compressed File Exists

```bash
ls -lh /data/log_archive/delete-before-clean-$(date +%F).tar.gz
```

### Step 5: Delete Files in List

```bash
cat /tmp/delete-log-list.txt | xargs rm -f
```

### Notes

This process is suitable for:

```text
I'm not sure if the log will be used again.

We need to leave an archive.

Prudent clean-up of production environment

Need to keep operational evidence
```

---

## Section 11: logrotate Basics

---

## Scenario 23: What is logrotate

`logrotate` is a commonly used log rotation tool in Linux.

It can achieve:

```text
Turn around.

Round by week

Rotation by month

Rotation by size

Keep fixed copies

Compress old logs

Delete Expiry Log

Script after rotation

Create a new log file

Empty original file after copying
```

Suitable for managing:

```text
Nginx Log

Apply Log

System service log

Custom Script Log

Middle Log
```

---

## Scenario 24: logrotate Configuration Paths

Common paths:

```text
/etc/logrotate.conf

/etc/logrotate.d/
```

Check main configuration:

```bash
cat /etc/logrotate.conf
```

Check extension configurations:

```bash
ls -lh /etc/logrotate.d/
```

Check a specific configuration:

```bash
cat /etc/logrotate.d/nginx
```

---

## Scenario 25: logrotate Configuration Example

### Configuration File

```bash
vi /etc/logrotate.d/myapp
```

### Example Configuration

```logrotate
/var/log/myapp/*.log {
    daily
    rotate 7
    compress
    missingok
    notifempty
    copytruncate
}
```

### Meaning Explanation

```text
/var/log/myapp/*.log
→ Under this path .log Entry into force of documents

daily
→ Rotation every day

rotate 7
→ Reservations 7 Grandpa.

compress
→ Compress old logs

missingok
→ The log doesn't exist and it's not wrong.

notifempty
→ Air logs don't rotate

copytruncate
→ Empty original file after copying log
```

---

## Section 12: Common logrotate Parameters

---

## Scenario 26: Rotate by Time

### Daily Rotation

```logrotate
daily
```

### Weekly Rotation

```logrotate
weekly
```

### Monthly Rotation

```logrotate
monthly
```

---

## Scenario 27: Retain by Count

### Example

```logrotate
rotate 7
```

### Notes

```text
Reservations 7 History
```

---

## Scenario 28: Rotate by Size

### Example

```logrotate
size 100M
```

### Notes

```text
Logo Achieved 100M Clock
```

Common writing format:

```logrotate
size 500M
```

```logrotate
size 1G
```

---

## Scenario 29: Compress Old Logs

### Example

```logrotate
compress
```

### Notes

```text
The old log will be compressed after the rotation.
```

---

## Scenario 30: Compress After One Rotation

### Example

```logrotate
delaycompress
```

### Notes

```text
Usually with compress Use Together

This means that the latest round of rotating files will not be compressed and the next round will be compressed.
```

Suitable for:

```text
Some programs may continue to read the log from the wheel for a short time
```

---

## Scenario 31: Do Not Error If Log Does Not Exist

### Example

```logrotate
missingok
```

### Notes

```text
Log file without error
```

---

## Scenario 32: Do Not Rotate Empty Files

### Example

```logrotate
notifempty
```

### Notes

```text
Logs are empty without rotation
```

---

## Scenario 33: Create New Log File

### Example

```logrotate
create 0644 appuser appgroup
```

### Notes

```text
Rotate and create a new log file

Permission: 0644

Subject appuser

Group as appgroup
```

---

## Scenario 34: Clear Original File After Copy

### Example

```logrotate
copytruncate
```

### Notes

```text
Copy the current log as a rotation file first

Empty original log file again
```

Suitable for:

```text
Apply does not automatically reopen log files

Can't restart or reload Apply

Apply always holding the log handle
```

Risks:

```text
Very few logs may be lost between copying and clearing

High Write Log scene requires caution
```

---

## Scenario 35: How to Choose Between create and copytruncate

### create is Suitable For

```text
Service support reload

Service receives a signal to reopen log file

NginxSome standard service log scenarios
```

### copytruncate is Suitable For

```text
Service not supported reload

Service cannot reopen log file

Simple application to write a fixed log file

Unable to restart service
```

Comparison:

```text
create
→ More regular, but need to apply restart logs

copytruncate
→ Simple and direct, but very high, with a small risk of losing logs.
```

---

## Scenario 36: Execute Script After Rotation

### Example

```logrotate
postrotate
    systemctl reload nginx > /dev/null 2>&1 || true
endscript
```

### Notes

```text
postrotate
→ Script after rotation

endscript
→ End of script
```

Suitable for:

```text
Jean. Nginx reload Reopen log file after

Notification service reload log handles

Perform custom cleanup actions
```

---

## Section 13: Nginx logrotate Example

---

## Scenario 37: Nginx Log Rotation Configuration Example

### Configuration File

```bash
vi /etc/logrotate.d/nginx
```

### Example Configuration

```logrotate
/var/log/nginx/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    create 0640 www-data adm
    sharedscripts
    postrotate
        systemctl reload nginx > /dev/null 2>&1 || true
    endscript
}
```

### Notes

```text
daily
→ Rotation every day

rotate 14
→ Reservations 14 Grandpa.

compress
→ Compress old logs

delaycompress
→ Delay compression of the latest round of old logs

create
→ Create a new log file

sharedscripts
→ Multiple log files executed only once postrotate Script

postrotate
→ Rotation reload nginx
```

### Notes

Nginx user may differ across systems:

```text
www-data

nginx

www

root
```

Confirm before execution:

```bash
ps -ef | grep nginx | grep -v grep
```

Or check configuration:

```bash
grep -n "user" /etc/nginx/nginx.conf
```

---

## Section 14: Testing logrotate

---

## Scenario 38: Debug logrotate Configuration

### Command

```bash
logrotate -d /etc/logrotate.d/myapp
```

### Notes

```text
-d
→ debug Debug Mode

Show only what will be executed

No real rotation log.
```

Production recommendation:

```text
Use the new configuration first -d Test
```

---

## Scenario 39: Force Execute logrotate

### Command

```bash
logrotate -f /etc/logrotate.d/myapp
```

### Notes

```text
-f
→ force, enforce rotation
```

Suitable for:

```text
Test if the configuration is actually effective

Manual Trigger Log Cut

Validate whether a new log continues to be written for a rotational application
```

---

## Scenario 40: View logrotate Status File

### Common Paths

```text
/var/lib/logrotate/status
```

View:

```bash
cat /var/lib/logrotate/status
```

Filter a specific log:

```bash
grep "myapp" /var/lib/logrotate/status
```

### Notes

logrotate records the last rotation time.

---

## Scenario 41: View logrotate Scheduling Mechanism

Different systems may execute via cron or systemd timer.

Check cron:

```bash
ls -lh /etc/cron.daily/
```

```bash
ls -lh /etc/cron.daily/logrotate
```

Check systemd timer:

```bash
systemctl list-timers | grep logrotate
```

```bash
systemctl status logrotate.timer
```

---

## Section 15: crontab Basics

---

## Scenario 42: View Current User's Scheduled Tasks

### Command

```bash
crontab -l
```

---

## Scenario 43: Edit Current User's Scheduled Tasks

### Command

```bash
crontab -e
```

---

## Scenario 44: Delete All Current User's Scheduled Tasks

### Command

```bash
crontab -r
```

### Production Note

```text
crontab -r Other Organiser cron Tasks

Good working environment
```

Recommend backing up before deletion:

```bash
crontab -l > /tmp/crontab-$(date +%F-%H%M%S).bak
```

---

## Scenario 45: cron Time Format

### Format

```cron
min Time Day Month Week Command
```

Field explanations:

```text
I don't think so. 1 Columns
→ Minutes,0-59

I don't think so. 2 Columns
→ Hours.0-23

I don't think so. 3 Columns
→ Date1-31

I don't think so. 4 Columns
→ The month.1-12

I don't think so. 5 Columns
→ I don't know.0-7I don't know.0 and 7 It usually means Sunday.

I don't think so. 6 Columns
→ Command to execute
```

---

## Scenario 46: Common cron Examples

Run at 1 AM daily:

```cron
0 1 * * * /bin/bash /opt/scripts/archive_log.sh
```

Run every 5 minutes:

```cron
*/5 * * * * /bin/bash /opt/scripts/check_log.sh
```

Run at 10th minute every hour:

```cron
10 * * * * /bin/bash /opt/scripts/check_space.sh
```

Run at 2:30 AM daily:

```cron
30 2 * * * /bin/bash /opt/scripts/cleanup_log.sh
```

Run at 3 AM on Sunday mornings:

```cron
0 3 * * 0 /bin/bash /opt/scripts/weekly_archive.sh
```

---

## Section 16: crontab Scheduled Log Archiving

---

## Scenario 47: Schedule Archiving Logs from 7 Days Ago

### crontab Example

```cron
0 2 * * * find /var/log/myapp -type f -name "*.log" -mtime +7 -print0 | tar --null -czf /data/log_archive/myapp-$(date +\%F).tar.gz --files-from=-
```

### Notes

```text
Every morning. 2 Do something!

Find 7 Sky Log

Pack as tar.gz

Save To /data/log_archive
```

### Key Notes

```text
cron Medium % Need to write \%
```

Otherwise `%` may be specially handled by cron, causing command anomalies.

---

## Scenario 48: Schedule Delete Archives Older Than 30 Days

### crontab Example

```cron
30 2 * * * find /data/log_archive -type f -mtime +30 -delete
```

### Notes

```text
Every morning. 2:30 Implementation

Delete /data/log_archive Down 30 Daybook
```

Production recommendation:

```text
Manually verify the deletion of the task find Conditions

Confirm directory range correct

Don't be straight. /var/log or / Root Path to Wide Remove
```

---

## Scenario 49: Send Email Notification After Scheduled Archiving

### crontab Example

```cron
0 3 * * * /bin/bash /opt/scripts/archive_log.sh && echo "Log archive completed" | mail -s "Archive Notifications" admin@example.com
```

### Notes

```text
Send mail after script execution is successful

&& Indicates that the last command was successfully executed
```

---

## Scenario 50: cron Output Redirection

### Example /think

```cron
0 1 * * * /bin/bash /opt/scripts/archive_log.sh >> /var/log/archive_log.log 2>&1
```

### Meaning

```text
>>
→ Add Standard Output to Log File

2>&1
→ Redirect the error output to the standard output.
```

Function:

```text
It's easy to check. cron Reason for Mission Failure

Keep Script Execution Record

Avoid cron Output lost
```

---

## 17. Log Archiving Script Example

---

## Scenario 51: Basic Archiving Script

### File Path

```bash
vi /opt/scripts/archive_myapp_log.sh
```

### Script Content

```bash
#!/bin/bash

set -u

LOG_DIR="/var/log/myapp"
ARCHIVE_DIR="/data/log_archive/myapp"
RETENTION_DAYS=30
ARCHIVE_DAYS=7
TODAY="$(date +%F)"
ARCHIVE_FILE="${ARCHIVE_DIR}/myapp-log-${TODAY}.tar.gz"
LIST_FILE="/tmp/myapp-log-list-${TODAY}.txt"

mkdir -p "$ARCHIVE_DIR"

find "$LOG_DIR" -type f -name "*.log" -mtime +"$ARCHIVE_DAYS" > "$LIST_FILE"

if [ ! -s "$LIST_FILE" ]; then
    echo "$(date '+%F %T') no logs need archive"
    exit 0
fi

tar czf "$ARCHIVE_FILE" -T "$LIST_FILE"

if [ $? -ne 0 ]; then
    echo "$(date '+%F %T') archive failed"
    exit 1
fi

echo "$(date '+%F %T') archive created: $ARCHIVE_FILE"

find "$ARCHIVE_DIR" -type f -name "*.tar.gz" -mtime +"$RETENTION_DAYS" -print
```

### Permissions

```bash
chmod +x /opt/scripts/archive_myapp_log.sh
```

### Manual Execution

```bash
/bin/bash /opt/scripts/archive_myapp_log.sh
```

### Explanation

This script only does:

```text
Find old logs

Generate List

Pack up the archive.

Print 30 File Archived Day Before
```

It does not automatically delete source logs to reduce risk.

---

## Scenario 52: Archiving Script with Deletion Action

### Explanation

In production environments, if you want to delete source logs after successful archiving, you must ensure:

```text
Archive successful

Compressor package exists

Compact package size normal

File list confirmed

Application does not continue writing these files

Retention cycle meets requirements
```

### Example Script

```bash
#!/bin/bash

set -u

LOG_DIR="/var/log/myapp"
ARCHIVE_DIR="/data/log_archive/myapp"
ARCHIVE_DAYS=7
TODAY="$(date +%F)"
ARCHIVE_FILE="${ARCHIVE_DIR}/myapp-log-${TODAY}.tar.gz"
LIST_FILE="/tmp/myapp-log-list-${TODAY}.txt"

mkdir -p "$ARCHIVE_DIR"

find "$LOG_DIR" -type f -name "*.log" -mtime +"$ARCHIVE_DAYS" > "$LIST_FILE"

if [ ! -s "$LIST_FILE" ]; then
    echo "$(date '+%F %T') no logs need archive"
    exit 0
fi

tar czf "$ARCHIVE_FILE" -T "$LIST_FILE"

if [ $? -ne 0 ]; then
    echo "$(date '+%F %T') archive failed"
    exit 1
fi

if [ ! -s "$ARCHIVE_FILE" ]; then
    echo "$(date '+%F %T') archive file is empty or missing"
    exit 1
fi

echo "$(date '+%F %T') archive success: $ARCHIVE_FILE"

cat "$LIST_FILE" | xargs rm -f

echo "$(date '+%F %T') source logs removed"
```

### Production Note

If the path may contain spaces, ordinary `xargs rm -f` is not reliable enough.

A safer solution is to use:

```bash
find "$LOG_DIR" -type f -name "*.log" -mtime +"$ARCHIVE_DAYS" -print0 | xargs -0 rm -f
```

But still need to preview before deletion.

---

## 18. mail Email Notification

---

## Scenario 53: Send Email After Archiving

### Command

```bash
echo "Log archive completed" | mail -s "Archive Notifications" admin@example.com
```

### Explanation

```text
-s
→ Mail Theme
```

Suitable for:

```text
Archive Notifications

We're gonna have to report the inspection results.

Notification of backup results

Notification of cleanup results
```

---

## Scenario 54: Send File Content

### Command

```bash
cat /tmp/result.txt | mail -s "Inspection results" admin@example.com
```

---

## Scenario 55: Send Notification in Script

### Example

```bash
if [ $? -eq 0 ]; then
    echo "Log archive successfully:$ARCHIVE_FILE" | mail -s "Log archive successfully" admin@example.com
else
    echo "Log archive failed. Check server" | mail -s "Log archive failed" admin@example.com
fi
```

### Note

Email sending depends on:

```text
Here. mail Commands Available

Here. MTA Available

Or configured SMTP Forward

Target mailbox accepted
```

---

## 19. cron Troubleshooting

---

## Scenario 56: Check cron Service Status

Service names may vary across systems.

Common:

```bash
systemctl status cron
```

Or:

```bash
systemctl status crond
```

---

## Scenario 57: Check Current User's cron

```bash
crontab -l
```

---

## Scenario 58: Check System-level cron Directory

```bash
ls -lh /etc/cron.d/
```

```bash
ls -lh /etc/cron.daily/
```

```bash
ls -lh /etc/cron.hourly/
```

```bash
ls -lh /etc/cron.weekly/
```

```bash
ls -lh /etc/cron.monthly/
```

---

## Scenario 59: Check cron Logs

RHEL / CentOS / Rocky / AlmaLinux:

```bash
tail -n 100 /var/log/cron
```

Ubuntu / Debian:

```bash
grep CRON /var/log/syslog | tail -n 100
```

systemd logs:

```bash
journalctl -u cron -n 100
```

Or:

```bash
journalctl -u crond -n 100
```

---

## Scenario 60: Common Reasons cron Tasks Fail to Execute

Common reasons:

```text
cron Service not running

Script is not executed.

Script path error

Command does not write an absolute path

Environmental variables are missing

Script Dependence PATH Command in

cron Medium % No conversion.

Insufficient user permissions

Destination directory does not exist

The output is not redirected, error information is not visible.

Script Windows Line change leads to failure of execution
```

Troubleshooting commands:

```bash
systemctl status cron
```

```bash
systemctl status crond
```

```bash
crontab -l
```

```bash
ls -lh /opt/scripts/archive_log.sh
```

```bash
/bin/bash /opt/scripts/archive_log.sh
```

```bash
grep CRON /var/log/syslog | tail -n 100
```

---

## 20. Production Notes

---

## 1. Must Preview Before Deleting Logs

High risk:

```bash
find /data/log_archive -type f -mtime +30 -delete
```

Must run:

```bash
find /data/log_archive -type f -mtime +30
```

Confirm the scope is correct before deletion.

---

## 2. Do Not Perform Large-scale Deletion on Root Directory

High risk:

```bash
find / -name "*.log" -mtime +30 -delete
```

Risks:

```text
It's too wide.

Possible system log error

Possible deletion of business logs

Possible Mount Point

Possible deletion of files still in use

It's hard to check and recover.
```

Should limit the directory:

```bash
find /var/log/myapp -type f -name "*.log" -mtime +30
```

---

## 3. Do Not mv or gzip Logs Being Written

Risks:

```text
Apply to continue writing old file handles

New log no longer appears on the expected path

Archived files are still occupied by process

Log loss or check difficulties
```

Handling method:

```text
Use Priority logrotate

Service support reload use create + postrotate

Service not supported reopen Use the log carefully copytruncate
```

---

## 4. Test logrotate Configuration Before Use

After new configuration, first execute:

```bash
logrotate -d /etc/logrotate.d/myapp
```

Confirm everything is correct before considering:

```bash
logrotate -f /etc/logrotate.d/myapp
```

---

## 5. Use Absolute Paths in cron

Not recommended:

```cron
0 1 * * * bash archive_log.sh
```

Recommended:

```cron
0 1 * * * /bin/bash /opt/scripts/archive_log.sh >> /var/log/archive_log.log 2>&1
```

Reason:

```text
cron Environmental variables are rare

Current Directory Unexpected

PATH and interactive shell Different.
```

---

## 6. Escape % in cron's date

Incorrect example:

```cron
0 2 * * * echo $(date +%F)
```

Recommended format:

```cron
0 2 * * * echo $(date +\%F)
```

---

## 7. Cleaning Tasks Should Have Logs

It is recommended to add output to all scheduled cleaning tasks:

```cron
0 2 * * * /bin/bash /opt/scripts/archive_log.sh >> /var/log/archive_log.log 2>&1
```

Otherwise, it's hard to troubleshoot when tasks fail.

---

## 8. Log Archiving Directory Should Also Be Monitored

The archiving directory may also fill up disk space.

Need to monitor:

```text
/data/log_archive Size

Number of Archived Files

Continued growth in archiving

Expiry clearance effective

Use of disk in archive directory
```

---

## 21. Common Commands Summary in This Section

---

## Find Logs

```bash
find /var/log/myapp -type f -name "*.log" -mtime +7
```

```bash
find /var/log/myapp -type f -name "*.log" -mtime -7
```

```bash
find /data/log_archive -type f -mtime +30
```

```bash
find /var/log/myapp -type f -name "*.log" -size +500M -exec ls -lh {} \;
```

```bash
du -sh /var/log/myapp
```

```bash
du -h --max-depth=1 /var/log/myapp | sort -hr
```

---

## Manual Archiving

```bash
find /var/log/myapp -type f -name "*.log" -mtime +7 -print0 | tar --null -czf /tmp/myapp-log-$(date +%F).tar.gz --files-from=-
```

```bash
find /var/log/myapp -type f -name "*.log" -mtime +7 > /tmp/myapp-old-log-list.txt
```

```bash
tar czf /tmp/myapp-log-$(date +%F).tar.gz -T /tmp/myapp-old-log-list.txt
```

```bash
ls -lh /tmp/myapp-log-$(date +%F).tar.gz
```

---

## Compress Logs

```bash
find /var/log/myapp -type f -name "*.log" -mtime +7 -exec gzip {} \;
```

```bash
zcat /var/log/myapp/app.log.gz | head
```

```bash
zgrep -i "error" /var/log/myapp/app.log.gz
```

```bash
zgrep -i "error" /var/log/myapp/app.log.gz | wc -l
```

---

## Move Logs

```bash
mkdir -p /data/log_archive
```

```bash
find /var/log/myapp -type f -name "*.log" -mtime +7 -exec mv {} /data/log_archive/ \;
```

```bash
mkdir -p /data/log_archive/$(date +%F)
```

```bash
find /var/log/myapp -type f -name "*.log" -mtime +7 -exec mv {} /data/log_archive/$(date +%F)/ \;
```

---

## Delete Expired Logs

```bash
find /data/log_archive -type f -mtime +30
```

```bash
find /data/log_archive -type f -mtime +30 -delete
```

```bash
find /var/log/myapp -type f -name "*.gz" -mtime +30
```

```bash
find /var/log/myapp -type f -name "*.gz" -mtime +30 -delete
```

```bash
find /data/log_archive -type f -mtime +30 -print0 | xargs -0 rm -f
```

---

## logrotate

```bash
cat /etc/logrotate.conf
```

```bash
ls -lh /etc/logrotate.d/
```

```bash
cat /etc/logrotate.d/nginx
```

```bash
vi /etc/logrotate.d/myapp
```

```bash
logrotate -d /etc/logrotate.d/myapp
```

```bash
logrotate -f /etc/logrotate.d/myapp
```

```bash
cat /var/lib/logrotate/status
```

```bash
grep "myapp" /var/lib/logrotate/status
```

---

## crontab

```bash
crontab -l
```

```bash
crontab -e
```

```bash
crontab -r
```

```bash
crontab -l > /tmp/crontab-$(date +%F-%H%M%S).bak
```

---

## cron Service and Logs

```bash
systemctl status cron
```

```bash
systemctl status crond
```

```bash
tail -n 100 /var/log/cron
```

```bash
grep CRON /var/log/syslog | tail -n 100
```

```bash
journalctl -u cron -n 100
```

```bash
journalctl -u crond -n 100
```

---

## Scheduled Task Examples

```cron
0 1 * * * /bin/bash /opt/scripts/archive_log.sh
```

```cron
*/5 * * * * /bin/bash /opt/scripts/check_log.sh
```

```cron
10 * * * * /bin/bash /opt/scripts/check_space.sh
```

```cron
0 2 * * * find /var/log/myapp -type f -name "*.log" -mtime +7 -print0 | tar --null -czf /data/log_archive/myapp-$(date +\%F).tar.gz --files-from=-
```

```cron
30 2 * * * find /data/log_archive -type f -mtime +30 -delete
```

```cron
0 1 * * * /bin/bash /opt/scripts/archive_log.sh >> /var/log/archive_log.log 2>&1
```

---

## mail Notification

```bash
echo "Log archive completed" | mail -s "Archive Notifications" admin@example.com
```

```bash
cat /tmp/result.txt | mail -s "Inspection results" admin@example.com
```

---

## 22. One-Sentence Summary

The core of log archiving and rotation is:

```text
Cut by cycle

→ Compress history logs

→ Keep fixed cycle

→ Clear Expiry Archive

→ Keep the logs on disks.
```

Manual archiving chain:

```text
find Find old logs

→ tar / gzip Compression

→ Move to Archive Directory

→ Validate compression package

→ Clear old archive by retention cycle
```

logrotate is suitable for:

```text
Standard log rotation

Cut by day or size

Keep fixed copies

Compress old logs

Rotation reload Services
```

crontab is suitable for:

```text
Time Archive

Timed Cleanup

Timed Inspection

Regularly send notifications
```

Production recommendations:

```text
You must preview before deleting

Do not delete in the root directory

Don't just write in the log. mv or gzip

Priority logrotate Manage long-run service logs

cron Medium date Yes. % To write \%

Time jobs must output logs

logrotate Configure First Use -d Test

The archive catalogue itself monitors disk usage.
```