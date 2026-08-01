# 03-sed, find, xargs and Batch File Processing

#Linux #LogAnalysis #TextProcessing #sed #find #xargs #FileFind #BatchProcessing #Transport #Shell

---

## Recommended Path

01-Linux Foundation and Host Maintenance/02-Logs and Text Processing/03-sed, find, xargs and Batch File Processing.md

---

## One: Document Overview

This document organizes commonly used **sed, find, xargs and batch file processing** commands in Linux log analysis and text processing.

This article focuses on:

- sed to view specific line ranges
- sed text replacement
- sed to delete empty lines
- sed to delete matching lines
- sed in-place file modification
- find by file name
- find by file type
- find by file size
- find by file time
- find to perform batch operations
- xargs for batch input processing
- find + xargs combination
- batch file statistics
- batch log viewing
- batch compression, deletion, and file movement
- production environment batch operation precautions

This document is the third in the Log and Text Processing series, primarily solving:

```text
How to locate documents on condition and process large volumes of documents in bulk
```

The goal is:

- Be able to view and process text content with sed
→ Be able to find logs, large files, old files, and small files with find
→ Be able to batch process command input with xargs
→ Be able to combine find + grep/sed/xargs to process large logs
→ Be able to understand the risks of batch deletion, replacement, and file movement
→ Be able to safely execute batch file operations in production environments

---

## Two: Core Command Location

The core commands involved in this article are located as follows:

```text
sed
→ Line-by-line processing of text for scope, replacement, deletion, simple editing

find
→ Find files by filename, type, size, time, path

xargs
→ Change the previous command output to the last command parameter

grep
→ Cooperation find Search for file contents

tar / gzip
→ Cooperation find Do Log Archive

rm / mv / cp
→ Cooperation find Batch File Processing
```

One-sentence understanding:

```text
sed Processing of document contents

find Find File Objects

xargs Batch Reference

find + xargs Fits to process large volumes of files in bulk
```

---

## Three: Basic Concepts

---

## 1. What is sed

`sed` is a stream text processing tool.

Common uses:

```text
View specified line ranges

Replace text content

Delete the specified row

Remove empty lines

Print content on condition

Batch changes profile
```

Common features:

```text
Fit to line-by-line text

Fit to handle logs and profiles

Default does not modify the original file

Use -i Other Organiser
```

---

## 2. What is find

`find` is used to search for files in a directory.

It can search by the following conditions:

```text
Filename

File type

File Size

Modified

Visits

Permissions

Owner

Directory Depth

Path Mode
```

Common uses:

```text
Find Big Log

Find old logs

Find Temporary Files

Find empty files

Find certain suffix files

Find Recent Changes

Search for a large number of small files

Batch cleaning of expired files
```

---

## 3. What is xargs

`xargs` is used to convert standard input into command arguments.

Simple understanding:

```text
Previous command output many content

xargs Give these to the latter.
```

Example:

```bash
find /var/log -name "*.log" | xargs grep -i "error"
```

Meaning:

```text
find Find All .log Documentation

xargs Use these files as a path grep Parameters

grep Find in these files error
```

---

## 4. Difference between find -exec and xargs

### find -exec

```bash
find /var/log -name "*.log" -exec ls -lh {} \;
```

Features:

```text
find Do your own command.

{} Assemble Found Files

\; Means that every file executes an order once
```

### xargs

```bash
find /var/log -name "*.log" | xargs ls -lh
```

Features:

```text
xargs Merge multiple files and pass them to the command

Usually more efficient.

But be careful when there's room in the file name.
```

Safer writing:

```bash
find /var/log -name "*.log" -print0 | xargs -0 ls -lh
```

---

## Four: sed: View Specific Line Range

---

## Scenario 1: View Lines 100 to 120

### Command

```bash
sed -n '100,120p' app.log
```

### Explanation

```text
-n
→ Silent mode, output all lines without default

100,120
→ Specify No. 100 Present. 120 Okay.

p
→ Print
```

### Applicable Scenarios

```text
Based on grep -n Check the context when the wrong line number is found

View specified clips in large logs

I don't want to open the whole big file.

Positioning an anomaly log
```

---

## Scenario 2: View a Specific Line

### Command

```bash
sed -n '100p' app.log
```

### Explanation

Print only line 100.

---

## Scenario 3: View from a Specific Line to the End of the File

### Command

```bash
sed -n '100,$p' app.log
```

### Explanation

```text
$
→ Last line of the file
```

Meaning:

```text
From 100 Line printing to end of file
```

---

## Scenario 4: Combine with grep Line Number to View Context

### Step 1: Find Error Line Number

```bash
grep -n "error" app.log
```

Assume output:

```text
1200:error connecting database
```

### Step 2: View Nearby Logs

```bash
sed -n '1180,1220p' app.log
```

### Applicable Scenarios

```text
View the context where the error occurred

View the entire abnormal stack

View service startup failed
```

---

## Five: sed: Text Replacement

---

## Scenario 5: Replace First Matched Content per Line

### Command

```bash
sed 's/error/ERROR/' app.log
```

### Explanation

```text
s
→ substitute, Replace

error
→ Original content

ERROR
→ New content
```

Note:

```text
Default replaces only the first match per line

Default does not modify the original file, only output to screen
```

---

## Scenario 6: Replace All Matched Content per Line

### Command

```bash
sed 's/error/ERROR/g' app.log
```

### Explanation

```text
g
→ global, replace all matches in each line
```

---

## Scenario 7: Case-insensitive Replacement

### Command

```bash
sed 's/error/ERROR/Ig' app.log
```

### Explanation

```text
I
→ Ignore case

g
→ Global Replace
```

Can match:

```text
error

Error

ERROR
```

---

## Scenario 8: Replace and Output to New File

### Command

```bash
sed 's/old/new/g' app.conf > app.conf.new
```

### Explanation

This method does not modify the original file.

Suitable for generating new files first in production, then comparing confirmation.

Comparison:

```bash
diff -u app.conf app.conf.new
```

---

## Six: sed: In-place File Modification

---

## Scenario 9: In-place Replace File Content

### Command

```bash
sed -i 's/old/new/g' app.conf
```

### Explanation

```text
-i
→ Modified Document in Place
```

Production Note:

```text
sed -i Change files directly

Backup must be provided before execution

Do not directly execute production configurations without confirmation of rules
```

---

## Scenario 10: In-place Modification with Backup

### Method One: Manual Backup

```bash
cp -a app.conf app.conf.$(date +%F-%H%M%S).bak
```

Then execute:

```bash
sed -i 's/old/new/g' app.conf
```

---

### Method Two: sed Automatically Generate Backup

```bash
sed -i.bak 's/old/new/g' app.conf
```

After execution, it will generate:

```text
app.conf.bak
```

---

## Scenario 11: Replace Only Lines Containing a Specific Keyword

### Command

```bash
sed '/server_name/s/old.example.com/new.example.com/g' nginx.conf
```

### Explanation

Only perform replacement on lines containing `server_name`.

---

## Seven: sed: Delete Lines

---

## Scenario 12: Delete Empty Lines

### Command

```bash
sed '/^$/d' file
```

### Explanation

```text
^$
→ Empty Line

d
→ Delete
```

---

## Scenario 13: Delete Comment Lines

### Command

```bash
sed '/^#/d' file
```

### Applicable Scenarios

```text
View configuration file valid content

Remove the comments.

Check for effective configuration
```

---

## Scenario 14: Delete Empty Lines and Comment Lines

### Command

```bash
sed '/^#/d;/^$/d' file
```

### Common Uses

Viewing effective content of configuration files:

```bash
sed '/^#/d;/^$/d' /etc/ssh/sshd_config
```

---

## Scenario 15: Delete Lines Containing a Specific Keyword

### Command

```bash
sed '/debug/d' app.log
```

### Explanation

Delete lines containing `debug`.

Suitable for temporarily filtering noisy output.

---

## Scenario 16: Delete Specific Line Number

### Delete Line 10

```bash
sed '10d' file
```

### Delete Lines 10 to 20

```bash
sed '10,20d' file
```

---

## Eight: sed Production Precautions

---

## 1. Default Does Not Modify Original File

For example:

```bash
sed 's/old/new/g' app.conf
```

Only outputs the result to the screen, does not modify the original file.

---

## 2. Must Confirm Before Using -i

High-risk command:

```bash
sed -i 's/old/new/g' app.conf
```

Production recommendation:

```text
Not yet. -i Preview

Confirm that the results are correct.

Backup Original File

Reimplementation -i
```

Recommended process:

```bash
sed 's/old/new/g' app.conf | head
```

```bash
cp -a app.conf app.conf.$(date +%F-%H%M%S).bak
```

```bash
sed -i 's/old/new/g' app.conf
```

```bash
diff -u app.conf.$(date +%F-%H%M%S).bak app.conf
```

---

## 3. Pay Attention to Separator When Replacing Paths

If the replacement content contains `/`, for example, path:

```text
/var/log/app
```

You can use other separators.

Example:

```bash
sed 's#/var/log/app#/data/logs/app#g' app.conf
```

This is clearer than the following:

```bash
sed 's/\/var\/log\/app/\/data\/logs\/app/g' app.conf
```

---

## 9. find: Search by Name

---

## Scenario 17: Search by Filename

### Command

```bash
find /var/log -name "app.log"
```

### Description

Search for a file named `app.log` in the directory `/var/log`.

---

## Scenario 18: Search Log Files with Wildcards

### Command

```bash
find /var/log -name "*.log"
```

Search for compressed logs:

```bash
find /var/log -name "*.gz"
```

Search for access logs:

```bash
find /var/log -name "*access*"
```

---

## Scenario 19: Case-Insensitive Search

### Command

```bash
find /var/log -iname "*.log"
```

### Description

```text
-iname
→ Ignore case matching filenames
```

Can match:

```text
app.log

APP.LOG

App.Log
```

---

## 10. find: Search by Type

---

## Scenario 20: Search for Regular Files

### Command

```bash
find /var/log -type f
```

### Description

```text
-type f
→ Normal
```

---

## Scenario 21: Search for Directories

### Command

```bash
find /var/log -type d
```

### Description

```text
-type d
→ Contents
```

---

## Scenario 22: Search for Symbolic Links

### Command

```bash
find /var/log -type l
```

### Description

```text
-type l
→ Soft Link
```

---

## 11. find: Search by Size

---

## Scenario 23: Search for Files Larger than 100M

### Command

```bash
find /var/log -type f -size +100M
```

### Description

```text
-size +100M
→ Greater than 100M
```

---

## Scenario 24: Search for Files Larger than 1G

### Command

```bash
find / -type f -size +1G 2>/dev/null
```

### Description

```text
2>/dev/null
→ Ignore error output such as inadequate permissions
```

---

## Scenario 25: Search and Display Detailed Size

### Command

```bash
find / -type f -size +1G -exec ls -lh {} \; 2>/dev/null
```

### Description

```text
-exec
→ Execute command for found files

{}
→ find Found Files

\;
→ exec End Tag
```

---

## Scenario 26: Search for Files Smaller than 1K

### Command

```bash
find /tmp -type f -size -1k
```

### Applicable Scenarios

```text
Search for a large number of small files

Check. inode High usage

Temporary file for location anomaly
```

---

## 12. find: Search by Time

---

## Scenario 27: Search for Files Modified 7 Days Ago

### Command

```bash
find /var/log -type f -mtime +7
```

### Description

```text
-mtime +7
→ Change earlier than 7 Day before
```

Suitable for searching old logs.

---

## Scenario 28: Search for Files Modified in the Last 7 Days

### Command

```bash
find /var/log -type f -mtime -7
```

### Description

```text
-mtime -7
→ Recent 7 Modified within days
```

---

## Scenario 29: Search for Logs from 30 Days Ago

### Command

```bash
find /var/log -type f -name "*.log" -mtime +30
```

---

## Scenario 30: Search for Files Modified in the Last 1 Hour

### Command

```bash
find /var/log -type f -mmin -60
```

### Description

```text
-mmin -60
→ Recent 60 Modified in minutes.
```

---

## Scenario 31: Search for Files Modified 1 Hour Ago

### Command

```bash
find /var/log -type f -mmin +60
```

---

## 13. find: Limit Directory Depth

---

## Scenario 32: Limit Search to One Level Below Current Directory

### Command

```bash
find /var/log -maxdepth 1 -type f
```

### Description

```text
-maxdepth 1
→ Find only the current directory level
```

---

## Scenario 33: Set Minimum Depth

### Command

```bash
find /var/log -mindepth 2 -type f
```

### Description

```text
-mindepth 2
→ Search from the second floor.
```

---

## 14. find: Search by Permissions and User

---

## Scenario 34: Search for Files Owned by a Specific User

### Command

```bash
find /data -user appuser
```

---

## Scenario 35: Search for Files Owned by a Specific Group

### Command

```bash
find /data -group appgroup
```

---

## Scenario 36: Search for Files with Specific Permissions

### Command

```bash
find /data -type f -perm 777
```

### Production Note

Files with permission 777 require special attention.

Potential issues:

```text
Too wide.

Security risks

Error Configuration

Application write permissions abnormal
```

---

## Scenario 37: Search for SUID Files

### Command

```bash
find / -perm -4000 -type f 2>/dev/null
```

### Description

SUID files have special permissions and should be monitored for any suspicious files.

---

## 15. find: Search for Empty Files and Directories

---

## Scenario 38: Search for Empty Files

### Command

```bash
find /var/log -type f -empty
```

---

## Scenario 39: Search for Empty Directories

### Command

```bash
find /data -type d -empty
```

---

## Scenario 40: Delete Empty Files

### Command

```bash
find /var/log -type f -empty -delete
```

### Production Note

`-delete` is a high-risk operation.

Before execution, it's recommended to preview:

```bash
find /var/log -type f -empty
```

Confirm accuracy before proceeding with deletion.

---

## 16. find: Execute Commands

---

## Scenario 41: Execute ls on Found Files

### Command

```bash
find /var/log -type f -name "*.log" -exec ls -lh {} \;
```

---

## Scenario 42: Search and grep Content

### Command

```bash
find /var/log -type f -name "*.log" -exec grep -i "error" {} \;
```

### Show File Names

```bash
find /var/log -type f -name "*.log" -exec grep -H -i "error" {} \;
```

---

## Scenario 43: Search and Count Lines in Files

### Command

```bash
find /var/log -type f -name "*.log" -exec wc -l {} \;
```

---

## Scenario 44: Search and Compress Old Logs

### Command

```bash
find /var/log/myapp -type f -name "*.log" -mtime +7 -exec gzip {} \;
```

### Description

Compress `.log` files from 7 days ago into `.gz`.

Production Note:

```text
Confirm that the application did not continue with these logs

Confirm the log rotation mechanism

Make sure the compression doesn't affect the check.

Make sure you have enough disk space.
```

---

## 17. xargs: Basic Usage

---

## Scenario 45: xargs Basic Example

### Command

```bash
echo "file1 file2 file3" | xargs ls -lh
```

### Description

`xargs` will convert input into parameters for `ls -lh`.

---

## Scenario 46: find + xargs View Files

### Command

```bash
find /var/log -name "*.log" | xargs ls -lh
```

### Description

Pass files found by find to `ls -lh`.

---

## Scenario 47: find + xargs grep

### Command

```bash
find /var/log -name "*.log" | xargs grep -i "error"
```

Show file names:

```bash
find /var/log -name "*.log" | xargs grep -H -i "error"
```

---

## Scenario 48: xargs Specify Number of Items per Batch

### Command

```bash
find /var/log -name "*.log" | xargs -n 10 ls -lh
```

### Description

```text
-n 10
→ Every time. 10 A command to the back of a parameter
```

---

## 18. xargs: Safe Handling of Special Filenames

---

## Scenario 49: Problem with Filenames Containing Spaces

Normal writing:

```bash
find /data -name "*.log" | xargs ls -lh
```

If filenames contain spaces, processing may fail.

Safer writing:

```bash
find /data -name "*.log" -print0 | xargs -0 ls -lh
```

### Description

```text
-print0
→ find Output by NULL Separate

xargs -0
→ Press NULL Separate Read
```

---

## Scenario 50: Safe grep for Special Filenames

### Command

```bash
find /data -name "*.log" -print0 | xargs -0 grep -H -i "error"
```

---

## 19. find + xargs Common Combinations

---

## Scenario 51: Batch Count Log Lines

### Command

```bash
find /var/log/myapp -type f -name "*.log" -print0 | xargs -0 wc -l
```

---

## Scenario 52: Batch Search for error

### Command

```bash
find /var/log/myapp -type f -name "*.log" -print0 | xargs -0 grep -H -i "error"
```

---

## Scenario 53: Batch Search for Multiple Keywords

### Command

```bash
find /var/log/myapp -type f -name "*.log" -print0 | xargs -0 grep -H -Ei "error|failed|timeout|exception"
```

---

## Scenario 54: Batch View error in Recent 7 Days Logs

### Command

```bash
find /var/log/myapp -type f -name "*.log" -mtime -7 -print0 | xargs -0 grep -H -i "error"
```

---

## Scenario 55: Batch Search Large Logs and Sort

### Command

```bash
find /var/log -type f -size +100M -exec ls -lh {} \; | sort -k5 -hr
```

---

## Scenario 56: Batch Count File Numbers

### Command

```bash
find /var/log/myapp -type f | wc -l
```

---

## Scenario 57: Count Files in Each Directory

### Command

```bash
for dir in /var/*; do echo "$dir"; find "$dir" -xdev -type f 2>/dev/null | wc -l; done
```

### Applicable Scenarios

```text
inode High usage

We suspect a lot of small files.

An abnormal number of directories to locate.
```

---

## Twenty, Batch Delete Files

---

## Scenario 58: Delete Log Files Older Than 30 Days

### Preview First

```bash
find /var/log/myapp -type f -name "*.log" -mtime +30
```

### Delete Now

```bash
find /var/log/myapp -type f -name "*.log" -mtime +30 -delete
```

---

## Scenario 59: Use rm to Delete

### Command

```bash
find /var/log/myapp -type f -name "*.log" -mtime +30 -exec rm -f {} \;
```

---

## Scenario 60: Use xargs to Delete

### Command

```bash
find /var/log/myapp -type f -name "*.log" -mtime +30 -print0 | xargs -0 rm -f
```

---

## Scenario 61: Batch Delete Empty Files

### Preview First

```bash
find /tmp -type f -empty
```

### Delete Now

```bash
find /tmp -type f -empty -delete
```

---

## Twenty-one, Batch Move and Copy Files

---

## Scenario 62: Batch Move Old Logs

### Create Archive Directory

```bash
mkdir -p /backup/old-logs
```

### Move Logs Older Than 30 Days

```bash
find /var/log/myapp -type f -name "*.log" -mtime +30 -exec mv {} /backup/old-logs/ \;
```

---

## Scenario 63: Batch Copy Files

### Command

```bash
find /var/log/myapp -type f -name "*.log" -mtime -1 -exec cp {} /tmp/recent-logs/ \;
```

### Production Note

Create target directory before copying:

```bash
mkdir -p /tmp/recent-logs
```

---

## Twenty-two, Batch Compress and Archive

---

## Scenario 64: Batch Compress Old Logs

### Command

```bash
find /var/log/myapp -type f -name "*.log" -mtime +7 -exec gzip {} \;
```

---

## Scenario 65: Find Old Logs and Archive

### Command

```bash
find /var/log/myapp -type f -name "*.log" -mtime +7 > /tmp/old-log-list.txt
```

View list:

```bash
cat /tmp/old-log-list.txt
```

Archive:

```bash
tar czf /backup/old-logs-$(date +%F).tar.gz -T /tmp/old-log-list.txt
```

### Notes

```text
-T
→ Read the package from the file list
```

---

## Scenario 66: Archive Then Delete

### Step 1: Generate File List

```bash
find /var/log/myapp -type f -name "*.log" -mtime +30 > /tmp/delete-log-list.txt
```

### Step 2: Confirm List

```bash
cat /tmp/delete-log-list.txt
```

### Step 3: Archive Backup

```bash
tar czf /backup/delete-logs-backup-$(date +%F).tar.gz -T /tmp/delete-log-list.txt
```

### Step 4: Confirm Archive

```bash
ls -lh /backup/delete-logs-backup-$(date +%F).tar.gz
```

### Step 5: Delete Files in List

```bash
cat /tmp/delete-log-list.txt | xargs rm -f
```

If paths may contain spaces, recommend redesigning the process with safer methods, don't directly use ordinary `xargs rm`.

---

## Twenty-three, Batch Replace File Contents

---

## Scenario 67: Batch Replace Configuration Content

### First Find Files Containing Old Content

```bash
grep -R "old.example.com" /etc/nginx
```

### Batch Replace

```bash
find /etc/nginx -type f -name "*.conf" -exec sed -i.bak 's/old.example.com/new.example.com/g' {} \;
```

### Verify

```bash
grep -R "new.example.com" /etc/nginx
```

Check backup files:

```bash
find /etc/nginx -type f -name "*.bak"
```

---

## Scenario 68: Preview Before Batch Replace

### Command

```bash
find /etc/nginx -type f -name "*.conf" -exec grep -H "old.example.com" {} \;
```

Confirm and execute:

```bash
find /etc/nginx -type f -name "*.conf" -exec sed -i.bak 's/old.example.com/new.example.com/g' {} \;
```

---

## Twenty-four, Production Batch Operation Notes

---

## 1. Preview Before Deletion

High-risk commands:

```bash
find /var/log/myapp -type f -name "*.log" -mtime +30 -delete
```

Must run before execution:

```bash
find /var/log/myapp -type f -name "*.log" -mtime +30
```

Confirm output matches expectations completely before deletion.

---

## 2. Backup Before Batch sed -i

High-risk commands:

```bash
sed -i 's/old/new/g' file
```

Batch high-risk commands:

```bash
find /etc/nginx -type f -name "*.conf" -exec sed -i 's/old/new/g' {} \;
```

Recommended use:

```bash
sed -i.bak 's/old/new/g' file
```

Batch use:

```bash
find /etc/nginx -type f -name "*.conf" -exec sed -i.bak 's/old/new/g' {} \;
```

---

## 3. Don't Batch Delete Arbitrarily in Root Directory

High-risk:

```bash
find / -name "*.log" -delete
```

Risks:

```text
It's too wide.

Possible system log error

Possible deletion of business logs

Possible Mount Point

Possible impact on running services
```

Should restrict directory:

```bash
find /var/log/myapp -type f -name "*.log" -mtime +30
```

---

## 4. Note find Across Filesystems

When troubleshooting file counts, use:

```bash
find /var -xdev -type f
```

Notes:

```text
-xdev
→ Do Not Cross File System
```

Suitable to avoid scanning other mount points.

---

## 5. Use print0 + xargs -0 for File Names with Spaces

Recommended:

```bash
find /data -type f -name "*.log" -print0 | xargs -0 grep -H "error"
```

Not recommended:

```bash
find /data -type f -name "*.log" | xargs grep "error"
```

---

## 6. Test Small Scope Before Batch Operations

Recommended order:

```text
Find first

→ Again. head Look at the previous ones.

→ Small directory test

→ Backup

→ Then officially.

→ Final Authentication
```

Example:

```bash
find /var/log/myapp -type f -name "*.log" -mtime +30 | head
```

---

## Twenty-five, Common Commands Summary in This Section

---

## sed View Line Range

```bash
sed -n '100,120p' app.log
```

```bash
sed -n '100p' app.log
```

```bash
sed -n '100,$p' app.log
```

---

## sed Replace

```bash
sed 's/error/ERROR/' app.log
```

```bash
sed 's/error/ERROR/g' app.log
```

```bash
sed 's/error/ERROR/Ig' app.log
```

```bash
sed 's/old/new/g' app.conf > app.conf.new
```

```bash
sed -i 's/old/new/g' app.conf
```

```bash
sed -i.bak 's/old/new/g' app.conf
```

```bash
sed 's#/var/log/app#/data/logs/app#g' app.conf
```

---

## sed Delete

```bash
sed '/^$/d' file
```

```bash
sed '/^#/d' file
```

```bash
sed '/^#/d;/^$/d' file
```

```bash
sed '/debug/d' app.log
```

```bash
sed '10d' file
```

```bash
sed '10,20d' file
```

---

## find by Name

```bash
find /var/log -name "app.log"
```

```bash
find /var/log -name "*.log"
```

```bash
find /var/log -name "*.gz"
```

```bash
find /var/log -name "*access*"
```

```bash
find /var/log -iname "*.log"
```

---

## find by Type

```bash
find /var/log -type f
```

```bash
find /var/log -type d
```

```bash
find /var/log -type l
```

---

## find by Size

```bash
find /var/log -type f -size +100M
```

```bash
find / -type f -size +1G 2>/dev/null
```

```bash
find / -type f -size +1G -exec ls -lh {} \; 2>/dev/null
```

```bash
find /tmp -type f -size -1k
```

---

## find by Time

```bash
find /var/log -type f -mtime +7
```

```bash
find /var/log -type f -mtime -7
```

```bash
find /var/log -type f -name "*.log" -mtime +30
```

```bash
find /var/log -type f -mmin -60
```

```bash
find /var/log -type f -mmin +60
```

---

## find Limit Depth

```bash
find /var/log -maxdepth 1 -type f
```

```bash
find /var/log -mindepth 2 -type f
```

---

## find by Permissions and User

```bash
find /data -user appuser
```

```bash
find /data -group appgroup
```

```bash
find /data -type f -perm 777
```

```bash
find / -perm -4000 -type f 2>/dev/null
```

---

## find Empty Files and Directories

```bash
find /var/log -type f -empty
```

```bash
find /data -type d -empty
```

```bash
find /var/log -type f -empty -delete
```

---

## find -exec

```bash
find /var/log -type f -name "*.log" -exec ls -lh {} \;
```

```bash
find /var/log -type f -name "*.log" -exec grep -H -i "error" {} \;
```

```bash
find /var/log -type f -name "*.log" -exec wc -l {} \;
```

```bash
find /var/log/myapp -type f -name "*.log" -mtime +7 -exec gzip {} \;
```

---

## xargs

```bash
echo "file1 file2 file3" | xargs ls -lh
```

```bash
find /var/log -name "*.log" | xargs ls -lh
```

```bash
find /var/log -name "*.log" | xargs grep -H -i "error"
```

```bash
find /var/log -name "*.log" | xargs -n 10 ls -lh
```

```bash
find /data -name "*.log" -print0 | xargs -0 ls -lh
```

```bash
find /data -name "*.log" -print0 | xargs -0 grep -H -i "error"
```

---

## find + xargs Combination

```bash
find /var/log/myapp -type f -name "*.log" -print0 | xargs -0 wc -l
```

```bash
find /var/log/myapp -type f -name "*.log" -print0 | xargs -0 grep -H -i "error"
```

```bash
find /var/log/myapp -type f -name "*.log" -print0 | xargs -0 grep -H -Ei "error|failed|timeout|exception"
```

```bash
find /var/log/myapp -type f -name "*.log" -mtime -7 -print0 | xargs -0 grep -H -i "error"
```

---

## Batch Delete

```bash
find /var/log/myapp -type f -name "*.log" -mtime +30
```

```bash
find /var/log/myapp -type f -name "*.log" -mtime +30 -delete
```

```bash
find /var/log/myapp -type f -name "*.log" -mtime +30 -exec rm -f {} \;
```

```bash
find /var/log/myapp -type f -name "*.log" -mtime +30 -print0 | xargs -0 rm -f
```

---

## Batch Move and Copy

```bash
mkdir -p /backup/old-logs
```

```bash
find /var/log/myapp -type f -name "*.log" -mtime +30 -exec mv {} /backup/old-logs/ \;
```

```bash
mkdir -p /tmp/recent-logs
```

```bash
find /var/log/myapp -type f -name "*.log" -mtime -1 -exec cp {} /tmp/recent-logs/ \;
```

---

## Batch Compress and Archive

```bash
find /var/log/myapp -type f -name "*.log" -mtime +7 -exec gzip {} \;
```

```bash
find /var/log/myapp -type f -name "*.log" -mtime +7 > /tmp/old-log-list.txt
```

```bash
tar czf /backup/old-logs-$(date +%F).tar.gz -T /tmp/old-log-list.txt
```

---

## Batch Replace

```bash
grep -R "old.example.com" /etc/nginx
```

```bash
find /etc/nginx -type f -name "*.conf" -exec grep -H "old.example.com" {} \;
```

```bash
find /etc/nginx -type f -name "*.conf" -exec sed -i.bak 's/old.example.com/new.example.com/g' {} \;
```

```bash
grep -R "new.example.com" /etc/nginx
```

---

## Twenty-six, One-sentence Summary

# The core division of labor for sed, find, and xargs is:

```text
sed
→ Processing of document contents

find
→ Find File Objects

xargs
→ Pass the search to subsequent commands
```

## Common use cases for sed:

```text
View specified line ranges

Replace Text

Remove empty lines

Delete Comment Line

Modify Profile In situ
```

## Common use cases for find:

```text
Find by Name

Find by Type

Find by Size

Find by Time

Find with Permission

Find empty files

Batch execution orders
```

## Common use cases for xargs:

```text
find Found a lot of files.

xargs Hand over the batch. grep / ls / rm / wc Processing
```

## Production recommendations:

```text
Preview before deleting

sed -i Backup First

Batch operation first small scale test

As limited as possible find Search Directory

Do not remove in random bulk in root directory

File names may be used when spaces are available -print0 and xargs -0

Batch modified configuration must diffChecking grammar and validation services
```