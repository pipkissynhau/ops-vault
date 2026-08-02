---
tags: [Ops, Logs, Shell, Python, Automation, Interviews]
---

# Interview Question 25: Filter logs larger than 10GB, automatically categorize them, and notify the administrator

## Description
In Ops, log files can grow rapidly, requiring regular monitoring, categorization, and alerts to prevent disk space issues.  
Goal: **Automatically identify logs larger than 10GB, move them to an archive directory, and notify the administrator.**

---

## Approach

1. **Scan the log directory**: Locate files larger than 10GB.
2. **Automate categorization**: Move these files to a designated archive folder, which can be organized by date or type.
3. **Notify the administrator**: Send an email, message via DingTalk, or Slack.
4. **Optional**: Execute the process periodically using cron jobs.

---

## Shell Example

```bash
#!/bin/bash

# Log directory
LOG_DIR="/var/log/myapp"
ARCHIVE_DIR="/var/log/archive"
ADMIN_EMAIL="ops@example.com"

# Find files larger than 10GB
find "$LOG_DIR" -type f -size +10G | while read file; do
    # Create the archive folder
    mkdir -p "$ARCHIVE_DIR/$(date +%Y-%m-%d)"
    # Move the file to the archive
    mv "$file" "$ARCHIVE_DIR/$(date +%Y-%m-%d)/"
    echo "File $file has been archived" >> /tmp/log_archive.log
done

# Send an email notification
if [ -f /tmp/log_archive.log ]; then
    mail -s "Large Log Archiving Notification" "$ADMIN_EMAIL" < /tmp/log_archive.log
    rm /tmp/log_archive.log
fi
```

---

## Python Example

```python
#!/usr/bin/env python3
import os
import shutil
import smtplib
from email.mime.text import MIMEText
from datetime import datetime

LOG_DIR = "/var/log/myapp"
ARCHIVE_DIR = "/var/log/archive"
ADMIN_EMAIL = "ops@example.com"
SIZE_THRESHOLD = 10 * 1024 * 1024 * 1024  # 10GB

archived_files = []

for root, dirs, files in os.walk(LOG_DIR):
    for f in files:
        file_path = os.path.join(root, f)
        if os.path.getsize(file_path) > SIZE_THRESHOLD:
            date_dir = os.path.join(ARCHIVE_DIR, datetime.now().strftime("%Y-%m-%d"))
            os.makedirs(date_dir, exist_ok=True)
            shutil.move(file_path, date_dir)
            archived_files.append(file_path)

# Send a notification email
if archived_files:
    body = "The following large log files have been archived:\n" + "\n".join(archived_files)
    msg = MIMEText(body)
    msg['Subject'] = 'Large Log Archiving Notification'
    msg['From'] = ADMIN_EMAIL
    msg['To'] = ADMIN_EMAIL

    with smtplib.SMTP('localhost') as s:
        s.send_message(msg)
```

---

## Key Points Summary

- Use `find` or `os.walk` to traverse the log directory.
- Check if file sizes exceed the threshold (10GB).
- Automatically move files to an archive folder organized by date.
- Notify the administrator via email or other communication tools.
- Automation can be achieved using cron jobs or scheduled system tasks.

---

## Sample Interview Response

> "First, I would scan the log directory to identify files larger than 10GB. Once found, I would move them to an archive folder structured by date. Additionally, I would record the archiving details in a log file.
> Then, I would notify the administrator via email or other communication platforms like DingTalk/Slack.
> This process can be implemented using Shell scripts with `find`, `mv`, and `mail` commands, or Python with `os.walk` and `shutil.move`. For automation, cron jobs could be set up to execute this process regularly."
