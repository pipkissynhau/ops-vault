# 03-sed, find, xargs and Batch File Processing

# Linux # Log Analysis # Text Processing # sed # find # xargs # File Searching # Batch Processing # Operations and Maintenance # Shell

---

## Recommended Path

01-Linux Basics and Server Operations and Maintenance/02-Logs and Text Processing/03-sed, find, xargs and Batch File Processing.md

---

## I. Document Introduction

This document compiles commonly used commands such as **sed, find, xargs, and batch file processing** in Linux log analysis and text processing.

Key points include:

- Using sed to view specified line ranges
- Replacing text with sed
- Deleting empty lines with sed
- Removing matching lines with sed
- Modifying files in-place with sed
- Using find to search for files by name, type, size, time, or path
- Performing batch operations with find
- Handling input in batches with xargs
- Combining find and xargs
- Conducting bulk file statistics
- Viewing logs in batches
- Compressing, deleting, and moving files in batches
- Precautions for batch operations in production environments

This article is the 03rd in the log and text processing series, focusing on:

```text
How to search for files based on conditions and perform batch processing on a large number of files
```

The goals are:

- Being able to view and process text content using sed
- Being able to use find to locate logs, large files, old files, or small files
- Being able to use xargs to handle command inputs in batches
- Being able to combine find, grep, sed, and xargs to process large amounts of logs
- Understanding the risks associated with bulk deletion, replacement, and movement of files
- Safely executing batch file operations in production environments

---

## II. Core Command Positioning

The core commands discussed in this article include:

```text
sed
→ Processes text line by line, suitable for viewing ranges, replacing content, deleting lines, and simple editing
find
→ Searches for files based on name, type, size, time, path, etc.
xargs
→ Converts the output of a previous command into parameters for the next command
grep
→ Used in conjunction with find to search for file contents
tar / gzip
→ Used with find to archive logs
rm / mv / cp
→ Used with find to perform batch file processing
```

In simple terms:

```text
sed is used to process file content
find is used to locate files
xargs is used to pass parameters in batches
The combination of find and xargs is ideal for handling large numbers of files
```

---

## III. Basic Concepts

---

## 1. What is sed?

`sed` is a stream-based text processing tool.

Common uses:

```text
Viewing specified line ranges
Replacing text content
Deleting specified lines
Removing empty lines
Printing content based on conditions
Batch-modifying configuration files
```

Key features:

```text
Suitable for processing text line by line
Ideal for working with logs and configuration files
Does not modify the original file by default
The -i option is required to modify files in-place
```

---

## 2. What is find?

`find` is used to search for files within directories.

It can be used to search based on:

```text
File name
File type
File size
Modification time
Access time
Permissions
Owner
Directory depth
Path pattern
```

Common uses:

```text
Locating large logs
Finding old logs
Identifying temporary files
Removing empty files
Searching for files with specific extensions
Locating recently modified files
Processing a large number of small files
Batch-removing expired files
```

---

## 3. What is xargs?

`xargs` is used to convert standard input into parameters for subsequent commands.

Simply put:

```text
If the previous command generates multiple outputs, xargs can process these outputs as arguments for the next command
```

Example:

```bash
find /var/log -name "*.log" | xargs grep -i "error"
```

Meaning:

```text
find locates all .log files
xargs passes these file paths as arguments to grep
grep searches these files for the word "error"
```

---

## 4. Differences between find -exec and xargs

### find -exec

```bash
find /var/log -name "*.log" -exec ls -lh {} \;
```

Features:

```text
find executes the command itself
{} represents the found file
\; indicates that the command is executed once for each file found
```

### xargs

```bash
find /var/log -name "*.log" | xargs ls -lh
```

Features:

```text
xargs combines multiple files into a single input for the command
Usually more```bash
sed 's/old/new/g' app.conf | head
```## Chapter 18: xargs: Safely Handling Special File Names

---

## Scenario 49: Issues with File Names Containing Spaces

Normal approach:

```bash
find /data -name "*.log" | xargs ls -lh
```

If the file name contains spaces, it may cause processing errors.

Safer approach:

```bash
find /data -name "*.log" -print0 | xargs -0 ls -lh
```

### Explanation

```text
-print0
→ Causes find to output files separated by NULL characters.

xargs -0
→ Reads input files separated by NULL characters.
```

---

## Scenario 50: Safe Use of grep for Special File Names

### Command

```bash
find /data -name "*.log" -print0 | xargs -0 grep -H -i "error"
```

---

## Chapter 19: Common Combinations of find and xargs

---

## Scenario 51: Batch Counting the Number of Log Lines

### Command

```bash
find /var/log/myapp -type f -name "*.log" -print0 | xargs -0 wc -l
```

---

## Scenario 52: Batch Searching for "error"

### Command

```bash
find /var/log/myapp -type f -name "*.log" -print0 | xargs -0 grep -H -i "error"
```

---

## Scenario 53: Batch Searching for Multiple Keywords

### Command

```bash
find /var/log/myapp -type f -name "*.log" -print0 | xargs -0 grep -H -Ei "error|failed|timeout|exception"
```

---

## Scenario 54: Batch Viewing "error" in Logs from the Last 7 Days

### Command

```bash
find /var/log/myapp -type f -name "*.log" -mtime -7 -print0 | xargs -0 grep -H -i "error"
```

---

## Scenario 55: Batch Searching for Large Logs and Sorting Them

### Command

```bash
find /var/log -type f -size +100M -exec ls -lh {} \; | sort -k5 -hr
```

---

## Scenario 56: Batch Counting the Number of Files

### Command

```bash
find /var/log/myapp -type f | wc -l
```

---

## Scenario 57: Counting the Number of Files in Each Directory

### Command

```bash
for dir in /var/*; do echo "$dir"; find "$dir" -xdev -type f 2>/dev/null | wc -l; done
```

### Application Scenarios

```text
High inode usage rate.

Suspected presence of a large number of small files.

Identifying which directory has an abnormal number of files.
```

---

## Chapter 20: Batch File Deletion

---

## Scenario 58: Deleting Logs from 30 Days Ago

### Preview First

```bash
find /var/log/myapp -type f -name "*.log" -mtime +30
```

### Delete Then

```bash
find /var/log/myapp -type f -name "*.log" -mtime +30 -delete
```

---

## Scenario 59: Using rm for Deletion

### Command

```bash
find /var/log/myapp -type f -name "*.log" -mtime +30 -exec rm -f {} \;
```

---

## Scenario 60: Using xargs for Deletion

### Command

```bash
find /var/log/myapp -type f -name "*.log" -mtime +30 -print0 | xargs -0 rm -f
```

---

## Scenario 61: Batch Deleting Empty Files

### Preview First

```bash
find /tmp -type f -empty
```

### Delete Then

```bash
find /tmp -type f -empty -delete
```

---

## Chapter 21: Batch Moving and Copying Files

---

## Scenario 62: Batch Moving Old Logs

### Create an Archive Directory

```bash
mkdir -p /backup/old-logs
```

### Move Logs from 30 Days Ago

```bash
find /var/log/myapp -type f -name "*.log" -mtime +30 -exec mv {} /backup/old-logs/ \;
```

---

## Scenario 63: Batch Copying Files

### Command

```bash
find /var/log/myapp -type f -name "*.log" -mtime -1 -exec cp {} /tmp/recent-logs/ \;
```

### Production Note

Create the target directory before copying:

```bash
mkdir -p /tmp/recent-logs
```

---

## Chapter 22: Batch CompressionMay affect services that are currently running.```bash
find /var/log/myapp -type f -name "*.log" -mtime +7 -exec gzip {} \;
```

```bash
find /var/log/myapp -type f -name "*.log" -mtime +7 > /tmp/old-log-list.txt
```

```bash
tar czf /backup/old-logs-$(date +%F).tar.gz -T /tmp/old-log-list.txt
```

---

## Batch Replacement

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

## Summary in One Sentence

The core roles of `sed`, `find`, and `xargs` are as follows:

```text
sed
→ Processes file content

find
→ Searches for files based on specified criteria

xargs
→ Sends the search results to multiple commands in batches
```

Common uses of `sed` include:

```text
Viewing specific lines within a file

Replacing text

Removing empty lines

Eliminating comment lines

Modifying configuration files in place
```

Common uses of `find` include:

```text
Searching for files by name, type, size, time, or permissions

Identifying empty files

Executing commands on multiple files simultaneously
```

Common uses of `xargs` include:

```text
Using `find` to locate numerous files and then passing them to `grep`, `ls`, `rm`, `wc`, etc. in batches
```

Production tips include:

```text
Always preview the results before making any deletions

Create backups when using `sed -i`

Test batch operations on a small set of files first

Limit the number of directories that `find` searches across

Avoid performing bulk deletions in the root directory

When file names contain spaces, use `-print0` and `xargs -0` to ensure proper handling

After making bulk configuration changes, always compare the results using `diff`, check for syntax errors, and verify that services are functioning correctly
```