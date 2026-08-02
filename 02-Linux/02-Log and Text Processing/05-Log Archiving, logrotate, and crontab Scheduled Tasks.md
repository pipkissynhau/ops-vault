# 05-Log Archiving, logrotate, and crontab Scheduled Tasks

# Linux # Log Management # Log Archiving # logrotate # crontab # tar # gzip # find # mail # Operations and Maintenance # SRE

---

## Recommended Path

01-Linux Basics and Host Operations and Maintenance/02-Logs and Text Processing/05-Log Archiving, logrotate, and crontab Scheduled Tasks.md

---

## I. Document Overview

This article outlines common scenarios in Linux operations and maintenance regarding **log archiving, log movement, log compression, log deletion, log rotation using logrotate, and crontab scheduled tasks**.

Key points include:

- Why log archiving is necessary
- Designing the log retention period
- Manually archiving logs
- Searching for logs from 7 days ago
- Finding archived logs from 30 days ago
- Packaging logs with tar
- Compressing logs using gzip
- Moving old logs to the archive directory
- Deleting expired logs
- Safe deletion procedures
- Basic configuration of logrotate
- Common parameters for logrotate
- Differences between copytruncate and create
- The postrotate script
- Manually testing logrotate
- Basic format of crontab
- Scheduling log archiving tasks with crontab
- Escaping `%` characters in cron
- Redirectioning cron output
- Sending email notifications after archiving

This article is part of the logs and text processing series, focusing on:

```text
How to archive, compress, rotate, and clean up logs periodically to prevent them from consuming all disk space.
```

The goals are:

- To be able to manually archive old logs
- To compress historical logs
- To move logs to the archive directory
- To securely delete expired logs
- To configure logrotate for automatic log rotation
- To use crontab to schedule log archiving tasks
- To output the archived results to logs or send notifications
- To understand the risk boundaries of log cleanup in production environments

---

## II. Why Log Archiving and Rotation Are Needed

In a production environment, if logs are not managed properly, common issues include:

```text
Unlimited growth of log files
/The /var/log directory filling up the system disk
/The /data log directory occupying the data disk
/A large number of small logs consuming inode space
Applications being unable to write logs
Service failures upon startup
Failures in database or middleware writes
/Docker container logs overflowing the /var/lib/docker directory
difficulty in opening large log files during troubleshooting
/Historical logs accumulating chaotically, making it hard to locate relevant information quickly
```

Therefore, it is essential to establish a log lifecycle:

```text
Real-time writing
→ Cutting logs daily or based on size
→ Compressing historical logs
→ Archiving them in a designated directory
→ Retaining them for a fixed period
→ Deleting expired logs
→ Forwarding important logs to a log platform
```

In short:

```text
Logs should not only be monitored but also managed effectively.
```

---

## III. General Approach to Log Management

The recommended approach is:

```text
First, determine the log storage location
→ Then, assess the rate at which logs are growing
→ Next, decide on the retention period
→ Configure log rotation and compression
→ Set up the archive directory
→ Arrange scheduled log cleanup
→ Finally, integrate monitoring and alerts
```

Common management methods include:

```text
logrotate
→ A standard tool for log rotation
crontab + find + tar/gzip
→ Scheduled archiving and cleaning
Application-specific log configuration
→ Cutting logs based on size or date
Docker daemon log limits
→ Preventing container log overflow
Log platforms
→ Centralized storage, retrieval, and retention period management
```

---

## IV. Designing the Log Retention Period

---

## Scenario 1: Common Log Retention Strategies

Common strategies include:

```text
Retaining detailed logs on the local machine for 7 days
Keeping compressed archived logs locally for 30 days
Forwarding important business logs to a log platform
Retaining security audit logs for an extended period
Keeping debugging logs for a shorter duration
Storing error logs for a longer time
```

Examples:

```text
Application access.log
→ 7–15 days on the local machine
Application error.log
→ 30 days on the local machine
Nginx access.log
→ 7–15 days on the local machine
Nginx error.log
→ 30 days on the local machine
Audit logs
→ Retained according to company compliance requirements
Debugging logs
→ Should not be kept open in production for an extended period
```

---

## Scenario 2: Factors to Consider When Designing the Retention Period

When designing the retention period, consider:

```text
Disk capacity### View without decompression

```bash
zcat /var/log/myapp/app.log.gz | head
```

### Search in compressed logs

```bash
zgrep -i "error" /var/log/myapp/app.log.gz
```

### Count errors in compressed logs

```bash
zgrep -i "error" /var/log/myapp/app.log.gz | wc -l
```

---

## Section 8: Moving Old Logs

---

## Scenario 15: Move logs from 7 days ago to the archive directory

### Create a directory

```bash
mkdir -p /data/log_archive
```

### Move logs

```bash
find /var/log/myapp -type f -name "*.log" -mtime +7 -exec mv {} /data/log_archive/ \;
```

### Check the archive directory

```bash
ls -lh /data/log_archive/
```

---

## Scenario 16: Preview logs before moving

### Command

```bash
find /var/log/myapp -type f -name "*.log" -mtime +7
```

Confirm everything is correct before proceeding:

```bash
find /var/log/myapp -type f -name "*.log" -mtime +7 -exec mv {} /data/log_archive/ \;
```

---

## Scenario 17: Move logs to a directory named after the date

### Command

```bash
mkdir -p /data/log_archive/$(date +%F)
```

```bash
find /var/log/myapp -type f -name "*.log" -mtime +7 -exec mv {} /data/log-archive/$(date +%F)/ \;
```

---

## Section 9: Deleting Expired Logs

---

## Scenario 18: Delete archived files from 30 days ago

### Preview first

```bash
find /data/log_archive -type f -mtime +30
```

### Delete then

```bash
find /data/log_archive -type f -mtime +30 -delete
```

---

## Scenario 19: Delete gzipped logs from 30 days ago

### Preview first

```bash
find /var/log/myapp -type f -name "*.gz" -mtime +30
```

### Delete then

```bash
find /var/log/myapp -type f -name "*.gz" -mtime +30 -delete
```

---

## Scenario 20: Delete empty logs in a specified directory

### Preview first

```bash
find /var/log/myapp -type f -name "*.log" -empty
```

### Delete then

```bash
find /var/log/myapp -type f -name "*.log" -empty -delete
```

---

## Scenario 21: Using a file list to delete logs

### Step 1: Generate a deletion list

```bash
find /data/log_archive -type f -mtime +30 > /tmp/delete-log-list.txt
```

### Step 2: Confirm the list

```bash
cat /tmp/delete-log-list.txt
```

### Step 3: Count the number of files

```bash
wc -l /tmp/delete-log-list.txt
```

### Step 4: Delete the files

```bash
cat /tmp/delete-log-list.txt | xargs rm -f
```

### Note

If file names may contain spaces, it is not recommended to use `xargs rm -f` directly.

A safer approach:

```bash
find /data/log_archive -type f -mtime +30 -print0 | xargs -0 rm -f
```

---

## Section 10: Backup Process Before Deletion

---

## Scenario 22: Pack the logs before deleting them

### Step 1: Generate a file list

```bash
find /var/log/myapp -type f -name "*.log" -mtime +30 > /tmp/delete-log-list.txt
```

### Step 2: Confirm the file list

```bash
cat /tmp/delete-log-list.txt
```

### Step 3: Create a backup archive

```bash
tar czf /data/log_archive/delete-before-clean-$(date +%F).tar.gz -T /tmp/delete-log-list.txt
```

### Step 4: Verify the existence of the archive

```bash
ls -lh /data/log-archive/delete-before-clean-$(date +%F).tar.gz
```

### Step 5: Delete the files listed in the list

```bash
cat /tmp/delete-log-list.txt | xargs rm -f
```

### Explanation

This process is suitable for:

```text
When it's uncertain whether the logs will still be needed later.
When an archive needs to be kept first.
When cleaning up in a production environment requires caution.
When evidence of operations needs to be retained.
```

---

## Section 11: Basics of logrotate

---

##The service is unable to reopen the log file.

For simple applications, it is straightforward to write logs to a fixed file.

However, this makes it inconvenient to restart the service.

---

## Scenario 36: Execute a script after log rotation

### Example

```logrotate
postrotate
    systemctl reload nginx > /dev/null 2>&1 || true
endscript
```

### Explanation

```text
postrotate
→ Executes a script after log rotation

endscript
→ Marks the end of the script execution process
```

This is suitable for:

```text
Reopening the log file after Nginx reloads

Notifying the service to reload the log handler

Performing custom cleanup tasks
```

---

## Chapter 13: Examples of Nginx log rotation

---

## Scenario 37: Example configuration for Nginx log rotation

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

### Explanation

```text
daily
→ Logs are rotated daily

rotate 14
→ Keeps 14 copies of the logs

compress
→ Compresses old logs

delaycompress
→ Delays the compression of the most recent round of old logs

create
→ Creates new log files

sharedscripts
→ The postrotate script is executed only once for multiple log files

postrotate
→ Reloads Nginx after log rotation
```

### Note

The Nginx user may vary on different systems:

```text
www-data

nginx

www

root
```

Before execution, confirm:

```bash
ps -ef | grep nginx | grep -v grep
```

Or check the configuration file:

```bash
grep -n "user" /etc/nginx/nginx.conf
```

---

## Chapter 14: Testing log rotation

---

## Scenario 38: Debugging log rotation configuration

### Command

```bash
logrotate -d /etc/logrotate.d/myapp
```

### Explanation

```text
-d
→ Debug mode for testing purposes

Only displays the commands that will be executed

Does not actually rotate logs
```

Production tip:

```text
Test new configurations using -d first
```

---

## Scenario 39: Forcing log rotation to execute

### Command

```bash
logrotate -f /etc/logrotate.d/myapp
```

### Explanation

```text
-f
→ Forces the execution of log rotation
```

This is suitable for:

```text
Verifying whether the configuration actually takes effect

Manually triggering log rotation

Checking if the application continues to write new logs after rotation
```

---

## Scenario 40: Viewing the status file of log rotation

### Common Path

```text
/var/lib/logrotate/status
```

To view:

```bash
cat /var/lib/logrotate/status
```

To filter for a specific log:

```bash
grep "myapp" /var/lib/logrotate/status
```

### Explanation

logrotate keeps track of the last time it rotated logs.

---

## Scenario 41: Viewing the timing mechanism of log rotation

On different systems, this may be done through cron or systemd timers.

To view cron tasks:

```bash
ls -lh /etc/cron.daily/
```

```bash
ls -lh /etc/cron.daily/logrotate
```

To view systemd timers:

```bash
systemctl list-timers | grep logrotate
```

```bash
systemctl status logrotate.timer
```

---

## Chapter 15: Basics of crontab

---

## Scenario 42: Viewing the current user's scheduled tasks

### Command

```bash
crontab -l
```

---

## Scenario 43: Editing the current user's scheduled tasks

### Command

```bash
crontab -e
```

---

## Scenario 44: Deleting all scheduled tasks for the current user

### Command

```bash
crontab -r
```

### Production Note

```text
Using crontab -r will delete all cron tasks for the current user

Use it with caution in a production environment
```

It is recommended to back up your tasks before deletion:

```bash
crontab -l > /tmp/crontab-$(date +%F-%H%M%S).bak
```

---

## Scenario 45: Cron time format

### Format

```cron
minute hour day month week command
```

Field Explanation:

```text
The 1st column
→ Minutes, rangingif [ $? -ne 0 ]; then
    echo "$(date '+%F %T') Archiving failed"
    exit 1
fi

echo "$(date '+%F %T') Archive created: $ARCHIVE_FILE"

find "$ARCHIVE_DIR" -type f -name "*.tar.gz" -mtime +"$RETENTION_days" -print
```

### Permission

```bash
chmod +x /opt/scripts/archive_myapp_log.sh
```

### Execute Manually

```bash
/bin/bash /opt/scripts/archive_myapp_log.sh
```

### Description

This script performs the following tasks:

- Searches for old logs.
- Generates a list of files to be archived.
- Packs and archives these files.
- Prints out the archive files created 30 days ago.

It does not automatically delete the source logs to reduce potential risks.

---

## Scenario 52: Archive Script with Deletion Action

### Description

In a production environment, if you want to delete the source logs after successful archiving, you must ensure that:

- The archiving is completed successfully.
- The archive package exists.
- The size of the archive package is normal.
- The list of files is correct.
- The application will no longer write to these files.
- The retention period is met.

### Sample Script

```bash
#!/bin/bash

set -u

LOG_DIR="/var/log/myapp"
ARCHIVE_DIR="/data/log_archive/myapp"
ARCHIVE_days=7
TODAY="$(date +%F)"
ARCHIVE_FILE="${ARCHIVE_DIR}/myapp-log-${TODAY}.tar.gz"
LIST_FILE "/tmp/myapp-log-list-${TODAY}.txt"

mkdir -p "$ARCHIVE_DIR"

find "$LOG_DIR" -type f -name "*.log" -mtime +"$ARCHIVE_days" > "$LIST_FILE"

if [ ! -s "$LIST_FILE" ]; then
    echo "$(date '+%F %T') No logs need to be archived"
    exit 0
fi

tar czf "$ARCHIVE_FILE" -T "$LIST_FILE"

if [ $? -ne 0 ]; then
    echo "$(date '+%F %T') Archiving failed"
    exit 1
fi

if [ ! -s "$ARCHIVE_FILE" ]; then
    echo "$(date '+%F %T') Archive file is empty or missing"
    exit 1
fi

echo "$(date '+%F %T') Archiving successful: $ARCHIVE_FILE"

cat "$LIST_FILE" | xargs rm -f

echo "$(date '+%F %T') Source logs removed"
```

### Production Notes

If the path may contain spaces, using `xargs rm -f` directly is not safe. A safer approach is:

```bash
find "$LOG_DIR" -type f -name "*.log" -mtime +"$ARCHIVE_days" -print0 | xargs -0 rm -f
```

However, it's still recommended to preview the files before deleting them.

---

## Section 18: Mail Notifications

---

## Scenario 53: Send an Email After Archiving is Completed

### Command

```bash
echo "Log archiving completed" | mail -s "Archiving Notification" admin@example.com
```

### Description

The `-s` option is used to specify the subject of the email.

This is suitable for sending notifications such as:

- Archiving results
- Inspection outcomes
- Backup completion
- Cleanup messages

---

## Scenario 54: Send the Contents of a File via Email

### Command

```bash
cat /tmp/result.txt | mail -s "Inspection Results" admin@example.com
```

---

## Scenario 55: Send Notifications Within a Script

### Example

```bash
if [ $? -eq 0 ]; then
    echo "Log archiving successful: $ARCHIVE_FILE" | mail -s "Log Archiving Successful" admin@example.com
else
    echo "Log archiving failed. Please check the server." | mail -s "Log Archiving Failed" admin@example.com
fi
```

### Note

Sending emails relies on:

- The availability of the local mail command.
- The functionality of the local MTA.
- Or a configured SMTP relay.
- The target email address being able to receive messages.

---

## Section 19: Cron Monitoring

---

## Scenario 56: Check the Status of the Cron Service

The service names may vary depending on the system:

- Commonly used commands:
  ```bash
  systemctl status cron
  ```
  or:
  ```bash
  systemctl status crond
  ```

---

## Scenario 57: View the Current User's Cron Schedule

```bash
crontab -l
```

---

## Scenario 58: View System-Level Cron Directories

```bash
ls -lh /etc/cron.d/
```
```bash
lsDisk usage of the archive directory