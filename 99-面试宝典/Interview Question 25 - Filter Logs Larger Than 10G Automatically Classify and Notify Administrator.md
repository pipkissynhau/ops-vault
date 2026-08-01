---
tags: "[Operations and Maintenance, Logs, Shell, Python, Automation, Interview]"
---

# Interview Question 25: Filter Logs Larger Than 10G, Automatically Categorize and Notify Administrators

## Description
In operations, log files may grow rapidly, requiring regular monitoring, categorization, and alerts to prevent disk space exhaustion.  
Objective: **Automatically filter log files larger than 10G, move them to an archive directory, and notify administrators**.

---

## Approach

1. **Scan Log Directory**: Find files larger than 10G  
2. **Automatically Categorize**: Move to a specified archive directory, optionally organized by date or type  
3. **Notify Administrators**: Via email, DingTalk, or Slack  
4. **Optional**: Schedule execution using cron for periodic detection  

---

## Shell Example

```bash
#!/bin/bash

# Log Directory
LOG_DIR="/var/log/myapp"
ARCHIVE_DIR="/var/log/archive"
ADMIN_EMAIL="ops@example.com"

# Find More than 10G Documentation
find "$LOG_DIR" -type f -size +10G | while read file; do
    # Create Archive Directory
    mkdir -p "$ARCHIVE_DIR/$(date +%Y-%m-%d)"
    # Move File
    mv "$file" "$ARCHIVE_DIR/$(date +%Y-%m-%d)/"
    echo "Log $file Archived" >> /tmp/log_archive.log
done

# Send Mail Notification Administrator
if [ -f /tmp/log_archive.log ]; then
    mail -s "Large Log Archive Notifications" "$ADMIN_EMAIL" < /tmp/log_archive.log
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
SIZE_THRESHOLD = 10 * 1024 * 1024 * 1024  # 10G

archived_files = []

for root, dirs, files in os.walk(LOG_DIR):
    for f in files:
        file_path = os.path.join(root, f)
        if os.path.getsize(file_path) > SIZE_THRESHOLD:
            date_dir = os.path.join(ARCHIVE_DIR, datetime.now().strftime("%Y-%m-%d"))
            os.makedirs(date_dir, exist_ok=True)
            shutil.move(file_path, date_dir)
            archived_files.append(file_path)

# Send Notification Mail
if archived_files:
    body = "The following large log files have been archived:\n" + "\n".join(archived_files)
    msg = MIMEText(body)
    msg['Subject'] = 'Large Log Archive Notifications'
    msg['From'] = ADMIN_EMAIL
    msg['To'] = ADMIN_EMAIL

    with smtplib.SMTP('localhost') as s:
        s.send_message(msg)
```

---

## Key Points Summary

- Use `find` or `os.walk` to traverse the log directory  
- Check if file size exceeds the threshold (10G)  
- Automatically archive to directories organized by date  
- Use email or other notification mechanisms to inform administrators  
- Can be implemented via cron or system scheduling tools for automation  

---

## Interview Answer Example

> "I would first scan the log directory to find files larger than 10G. Once identified, I would move them to an archive directory organized by date and record the archival information.  
> Then, I would notify administrators via email or other notification mechanisms (e.g., DingTalk/Slack).  
> This can be implemented using Shell or Python. Shell would use `find` + `mv` + `mail`, while Python could use `os.walk` to traverse files and `shutil.move` to move them, with `smtplib` for sending emails.  
> Finally, I would schedule it via cron to automate the archival and alerting process."