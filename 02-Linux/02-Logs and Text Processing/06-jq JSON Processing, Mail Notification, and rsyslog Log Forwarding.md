# 06-jq JSON Processing, Mail Notification, and rsyslog Log Forwarding

#Linux #LogAnalysis #TextProcessing #jq #JSON #mail #rsyslog #logger #LogForward #FocusLog #Transport #SRE

---

## Recommended Path

01-Linux Foundation and Host Maintenance/02-Logs and Text Processing/06-jq JSON Processing, Mail Notification, and rsyslog Log Forwarding.md

---

## One: Document Overview

This document organizes common **JSON log processing, email notification, and rsyslog log forwarding** scenarios in Linux operations.

Key focuses of this document include:

- jq basic usage
- JSON formatting
- jq field extraction
- jq output without quotes
- jq array traversal
- jq conditional filtering
- jq processing JSON lines in logs
- jq common parameters
- mail notification
- mail notification script example
- rsyslog basic understanding
- rsyslog UDP / TCP forwarding
- rsyslog server receiving configuration
- rsyslog client forwarding configuration
- logger sending test logs
- tcpdump packet capture verification
- rsyslog common troubleshooting approaches

This document is the 6th in the Logs and Text Processing series, primarily solving:

```text
How? JSON Logs, how to send script notifications, how to forward system logs to remote log servers
```

The goal is:

Be able to view and extract JSON fields with jq

→ Be able to process common JSON logs

→ Be able to send email notifications in scripts

→ Be able to understand the difference between rsyslog UDP / TCP forwarding

→ Be able to configure rsyslog client log forwarding

→ Be able to configure rsyslog server to receive remote logs

→ Be able to use logger and tcpdump to verify log forwarding chain

---

## Two: Core Command Location

The core commands involved in this document are located as follows:

```text
jq
→ Processing JSON Documentation JSON Log

mail
→ Organisation

rsyslog
→ System log collection and forwarding

logger
→ Write test messages to system logs

systemctl
→ Management rsyslog Services

ss
→ Inspection 514 Port listening

tcpdump
→ Capture package to verify whether the log was sent out or arrived

grep
→ Check configuration and filter log contents
```

One-sentence understanding:

```text
jq Responsible for JSON

mail Sending notification

rsyslog Responsible for transmitting logs

logger Make test logs.

tcpdump Check if there's a log package on the network level.
```

---

## Three: jq - JSON Processing Basics

---

## Scenario 1: What is jq

`jq` is a commonly used JSON command-line processor in Linux.

Common uses:

```text
Formatting JSON

Extract JSON Fields

Through JSON Array

Filter conditionally JSON

Process interface returns result

Processing Kubernetes / Docker / Cloud platform returned JSON

Process structured logs
```

Common operation scenarios:

```text
View interface return values

Processing API Return Result

Processing JSON Format Log

Filter Pod / Node / Service Information

Analysis Docker inspect Output

Analysis of cloud platforms CLI Return Result
```

---

## Scenario 2: Installing jq

Ubuntu / Debian:

```bash
apt install -y jq
```

RHEL / CentOS / Rocky / AlmaLinux:

```bash
yum install -y jq
```

Or:

```bash
dnf install -y jq
```

Verification:

```bash
jq --version
```

---

## Scenario 3: Preparing an Example JSON File

Example file:

```bash
vi file.json
```

Content:

```json
{
  "name": "myapp",
  "status": "Running",
  "replicas": 3,
  "namespace": "prod",
  "items": [
    {
      "name": "pod-1",
      "status": "Running",
      "ip": "10.0.0.11"
    },
    {
      "name": "pod-2",
      "status": "Pending",
      "ip": "10.0.0.12"
    },
    {
      "name": "pod-3",
      "status": "Running",
      "ip": "10.0.0.13"
    }
  ]
}
```

---

## Four: jq JSON Formatting

---

## Scenario 4: JSON Formatting

### Command

```bash
jq . file.json
```

### Description

```text
.
→ For the current full JSON Object
```

Function:

```text
Compress it into a line. JSON Format into readable format
```

---

## Scenario 5: Formatting JSON via Pipe

### Command

```bash
cat file.json | jq .
```

### Description

This syntax is suitable for processing JSON output from a previous command.

Example:

```bash
curl -s http://127.0.0.1:8080/api/status | jq .
```

---

## Scenario 6: Compact JSON Output

### Command

```bash
jq -c . file.json
```

### Description

```text
-c
→ compact♪ Squeeze the output ♪ JSON
```

Applicable scenarios:

```text
Log Output

Script Processing

One in a row. JSON scene of object
```

---

## Scenario 7: Output Sorted by Key

### Command

```bash
jq -S . file.json
```

### Description

```text
-S
→ sort keysPress key Sort
```

Suitable for:

```text
Comparison JSON Contents

View configuration differences

Make the output more stable.
```

---

## Five: jq Field Extraction

---

## Scenario 8: Extracting Top-Level Fields

### Command

```bash
jq '.name' file.json
```

Output may be:

```json
"myapp"
```

### Description

Default output is JSON string, so double quotes are included.

---

## Scenario 9: Extracting Fields and Removing Quotes

### Command

```bash
jq -r '.name' file.json
```

Output:

```text
myapp
```

### Description

```text
-r
→ raw outputOriginal output, no JSON String quote
```

This is one of the most commonly used parameters in operation scripts.

---

## Scenario 10: Extracting Multiple Fields

### Command

```bash
jq '.name, .status, .replicas' file.json
```

Without quotes:

```bash
jq -r '.name, .status, .replicas' file.json
```

---

## Scenario 11: Concatenating Multiple Fields

### Command

```bash
jq -r '.name + " " + .status' file.json
```

Output:

```text
myapp Running
```

---

## Scenario 12: Output as Tab-Separated

### Command

```bash
jq -r '[.name, .status, .replicas] | @tsv' file.json
```

Output:

```text
myapp   Running 3
```

Suitable for:

```text
Output table in script

Follow-up. awk / sort Processing

Generate simple reports
```

---

## Six: jq Processing Nested Fields

---

## Scenario 13: Extracting Nested Fields

If the JSON structure is:

```json
{
  "metadata": {
    "name": "myapp",
    "namespace": "prod"
  }
}
```

Extract:

```bash
jq -r '.metadata.name' file.json
```

Extract namespace:

```bash
jq -r '.metadata.namespace' file.json
```

---

## Scenario 14: Avoiding Errors When Fields Do Not Exist

### Command

```bash
jq -r '.metadata.name // "unknown"' file.json
```

### Description

```text
//
→ Default value
```

Meaning:

```text
If .metadata.name Does not exist or is nullJust output. unknown
```

Suitable for avoiding exceptions caused by empty values in scripts.

---

## Seven: jq Traversing Arrays

---

## Scenario 15: Traversing Array Elements

### Command

```bash
jq '.items[]' file.json
```

### Description

```text
.items[]
→ Through items Each element of an array
```

---

## Scenario 16: Extracting name from Each Object in Array

### Command

```bash
jq -r '.items[].name' file.json
```

Output:

```text
pod-1
pod-2
pod-3
```

---

## Scenario 17: Extracting Multiple Fields from Array

### Command

```bash
jq -r '.items[] | [.name, .status, .ip] | @tsv' file.json
```

Output similar to:

```text
pod-1   Running 10.0.0.11
pod-2   Pending 10.0.0.12
pod-3   Running 10.0.0.13
```

---

## Eight: jq Conditional Filtering

---

## Scenario 18: Filtering Objects with status = Running

### Command

```bash
jq '.items[] | select(.status == "Running")' file.json
```

---

## Scenario 19: Output Only name of Running Objects

### Command

```bash
jq -r '.items[] | select(.status == "Running") | .name' file.json
```

Output:

```text
pod-1
pod-3
```

---

## Scenario 20: Filtering Objects That Are Not Running

### Command

```bash
jq -r '.items[] | select(.status != "Running") | [.name, .status] | @tsv' file.json
```

---

## Scenario 21: Filtering by Field Containing Keywords

### Command

```bash
jq -r '.items[] | select(.name | contains("pod")) | .name' file.json
```

---

## Scenario 22: Filtering Numeric Fields

If the JSON contains:

```json
{
  "name": "api",
  "request_time": 1.35
}
```

Filter logs with duration > 1 second:

```bash
jq 'select(.request_time > 1)' access.json
```

---

## Nine: jq Processing JSON Logs

---

## Scenario 23: Logs with One JSON Line per Line

Assume the log file `app-json.log` contains:

```json
{"time":"2026-04-25T10:00:01+08:00","level":"info","service":"api","message":"request ok","status":200,"cost":0.12}
{"time":"2026-04-25T10:00:02+08:00","level":"error","service":"api","message":"db timeout","status":500,"cost":2.31}
{"time":"2026-04-25T10:00:03+08:00","level":"warn","service":"api","message":"slow request","status":200,"cost":1.52}
```

---

## Scenario 24: Formatting to View JSON Logs

### Command

```bash
cat app-json.log | jq .
```

---

## Scenario 25: Filtering Error-Level Logs

### Command

```bash
cat app-json.log | jq 'select(.level == "error")'
```

Only output message:

```bash
cat app-json.log | jq -r 'select(.level == "error") | .message'
```

---

## Scenario 26: Filtering 5xx Logs

### Command

```bash
cat app-json.log | jq 'select(.status >= 500)'
```

Output core fields:

```bash
cat app-json.log | jq -r 'select(.status >= 500) | [.time, .service, .status, .message] | @tsv'
```

---

## Scenario 27: Filtering Slow Requests

### Command

```bash
cat app-json.log | jq 'select(.cost > 1)'
```

Output time, status code, duration, and message:

```bash
cat app-json.log | jq -r 'select(.cost > 1) | [.time, .status, .cost, .message] | @tsv'
```

---

## Scenario 28: Statistics on Status Code Distribution in JSON Logs

### Command

```bash
cat app-json.log | jq -r '.status' | sort | uniq -c | sort -nr
```

---

## Scenario 29: Counting Error Occurrences

### Command

```bash
cat app-json.log | jq -r 'select(.level == "error") | .level' | wc -l
```

---

## Ten. Common jq Parameters and Expressions

---

## 1. Common jq Parameters

```text
-r
→ Original output without JSON String quote

-c
→ Close output, one line. JSON

-S
→ Press key Sort Output

-e
→ Set exit code based on filter result, suitable for script judgement
```

---

## 2. Common jq Expressions

```text
.
→ Current Total JSON

.name
→ Extract name Fields

.items[]
→ Through items Array

select(...)
→ Condition filter

// "default"
→ Set Default

@tsv
→ Output As tab Separate

@csv
→ Output As CSV Format
```

---

## 3. Common jq Commands

```bash
jq . file.json
```

```bash
jq -r '.name' file.json
```

```bash
jq -c . file.json
```

```bash
jq -S . file.json
```

```bash
jq '.items[]' file.json
```

```bash
jq -r '.items[].name' file.json
```

```bash
jq '.items[] | select(.status == "Running")' file.json
```

```bash
jq -r '.items[] | select(.status == "Running") | .name' file.json
```

```bash
jq -r '[.name, .status] | @tsv' file.json
```

---

## Eleven. mail: Email Notification

---

## Scenario 30: What is mail

`mail` is commonly used to send simple notifications in scripts.

Suitable scenarios:

```text
Log archive completion notification

Backup Completion Notification

We're gonna have to report the inspection results.

Disk space alert.

Synchronising "%s"

Time job results communicated
```

Note:

```text
mail The successful delivery depends on whether the service is equipped or not SMTP Forward.
```

---

## Scenario 31: Installing the mail Command

Ubuntu / Debian:

```bash
apt install -y mailutils
```

RHEL / CentOS / Rocky / AlmaLinux:

```bash
yum install -y mailx
```

Or:

```bash
dnf install -y mailx
```

---

## Scenario 32: Sending Simple Emails

### Command

```bash
echo "Log archive completed" | mail -s "Archive Notifications" admin@example.com
```

### Explanation

```text
-s
→ Mail Theme
```

---

## Scenario 33: Sending File Contents

### Command

```bash
cat /tmp/result.txt | mail -s "Inspection results" admin@example.com
```

Suitable for:

```text
Send inspection reports.

Send Backup Results

Sending misstatic results
```

---

## Scenario 34: Sending Command Output

### Command

```bash
df -h | mail -s "Disk Usage Inspection" admin@example.com
```

Sending system error logs:

```bash
journalctl -p err -n 50 --no-pager | mail -s "System Error Log" admin@example.com
```

---

## Scenario 35: Sending Notifications Based on Script Results

### Example

```bash
#!/bin/bash

LOG_FILE="/var/log/archive_log.log"
MAIL_TO="admin@example.com"

if grep -qi "archive success" "$LOG_FILE"; then
    echo "Log archive successful, see $LOG_FILE" | mail -s "Log archive successfully" "$MAIL_TO"
else
    echo "Log archive may fail, please check $LOG_FILE" | mail -s "Log archive abnormal" "$MAIL_TO"
fi
```

---

## Scenario 36: Sending Emails When Disk Usage Exceeds Threshold

### Example Script

```bash
#!/bin/bash

THRESHOLD=85
MAIL_TO="admin@example.com"

df -hP | awk 'NR>1 {print $5,$6}' | while read usage mountpoint; do
    percent=${usage%\%}

    if [ "$percent" -ge "$THRESHOLD" ]; then
        echo "Mount Point $mountpoint Usage ${percent}%above threshold ${THRESHOLD}%" \
        | mail -s "Disk Space Alert:$mountpoint" "$MAIL_TO"
    fi
done
```

### Explanation

```text
df -hP
→ Use POSIX Output format for script resolution

${usage%\%}
→ Remove the percentage number.
```

---

## Twelve. mail Troubleshooting

---

## Scenario 37: Common Reasons mail Fails to Send

Common reasons:

```text
mail Command not installed

This machine is not configured MTA

SMTP Forward Unconfigured

Cloud Server 25 Port restricted

Address error

Mail intercepted by spam strategy.

Here. hostname / Sender Address Bad

Firewall or security team restrictions
```

---

## Scenario 38: Checking the mail Command

### Command

```bash
which mail
```

```bash
mail -V
```

Some systems do not support `-V`, you can use:

```bash
which mail
```

---

## Scenario 39: Viewing Email Logs

RHEL / CentOS / Rocky / AlmaLinux:

```bash
tail -n 100 /var/log/maillog
```

Ubuntu / Debian:

```bash
tail -n 100 /var/log/mail.log
```

systemd:

```bash
journalctl -u postfix -n 100
```

Or:

```bash
journalctl -u sendmail -n 100
```

---

## Scenario 40: Recommended Production Notification Methods

Email is suitable for simple notifications, but in production environments it's recommended to combine with:

```text
Mail

Corporate Wisdom / Flying Book. / Nail. Webhook

Surveillance and alarm system

Prometheus Alertmanager

Log Platform Alert

Worksheet system
```

Reason:

```text
Mail may be delayed

Mail could be intercepted.

Mail alarms are easily ignored.

Webhook It's better for real-time notification.
```

---

## Thirteen. rsyslog: Basic Log Forwarding

---

## Scenario 41: What is rsyslog

`rsyslog` is a common system log service in Linux.

It can be used for:

```text
Collect the log of the system

Writing /var/log/messages or /var/log/syslog

Receive remote host log

Forward this log to the remote log server

Press facility / level Classification processing log
```

Common uses:

```text
System Log Focus

Security log centralization

Multiple host log collections

Send Host Log to Log Server

For subsequent access ELK / Loki / SIEM Prepare for basics.
```

---

## Scenario 42: Common rsyslog Configuration Paths

Main configuration:

```text
/etc/rsyslog.conf
```

Extended configuration directory:

```text
/etc/rsyslog.d/
```

View main configuration:

```bash
grep -v '^#' /etc/rsyslog.conf | grep -v '^$'
```

View extended directory:

```bash
ls -lh /etc/rsyslog.d/
```

Search for forwarding configurations:

```bash
grep -R "@" /etc/rsyslog.conf /etc/rsyslog.d/
```

---

## Scenario 43: Checking rsyslog Service Status

### Command

```bash
systemctl status rsyslog
```

Start:

```bash
systemctl start rsyslog
```

Restart:

```bash
systemctl restart rsyslog
```

Set to start on boot:

```bash
systemctl enable rsyslog
```

---

## Fourteen. rsyslog Client Log Forwarding

---

## Scenario 44: UDP Forwarding All Logs

### Configuration

```rsyslog
*.* @10.0.0.10:514
```

### Explanation

```text
@ 
→ UDP Forward

10.0.0.10
→ Log Server IP

514
→ syslog Default Port

*.*
→ All facilityAll level
```

---

## Scenario 45: TCP Forwarding All Logs

### Configuration

```rsyslog
*.* @@10.0.0.10:514
```

### Explanation

```text
@@
→ TCP Forward
```

TCP vs UDP:

```text
Better reliability.

I can feel the connection.

Fit to more important log forwards

But it costs a little more.
```

UDP vs TCP:

```text
Simple

Low cost.

But it could be lost.

No guarantee of reliable delivery.
```

In production, it's recommended to use TCP for important logs.

---

## Scenario 46: Forwarding Only Authentication Logs

### Configuration

```rsyslog
authpriv.* @@10.0.0.10:514
```

Suitable for:

```text
Centralized collection SSH Login

sudo

Authentication Failed

Security audit-related logs
```

---

## Scenario 47: Forwarding Only Kernel Logs

### Configuration

```rsyslog
kern.* @@10.0.0.10:514
```

Suitable for:

```text
Collect kernel error

Disk Error

Cybercard Error

OOM

File system anomaly
```

---

## Scenario 48: Client Configuration File Example

### Creating Configuration

```bash
vi /etc/rsyslog.d/90-forward.conf
```

### TCP Forwarding Example

```rsyslog
*.* @@10.0.0.10:514
```

### Restarting Service

```bash
systemctl restart rsyslog
```

### Checking Status

```bash
systemctl status rsyslog
```

---

## Fifteen. rsyslog Server Receiving Logs

---

## Scenario 49: Enabling UDP Reception on Server

### Configuration File

```bash
vi /etc/rsyslog.d/10-server.conf
```

### Content

```rsyslog
module(load="imudp")
input(type="imudp" port="514")
```

### Restart

```bash
systemctl restart rsyslog
```

---

## Scenario 50: Enabling TCP Reception on Server

### Configuration File

```bash
vi /etc/rsyslog.d/10-server.conf
```

### Content

```rsyslog
module(load="imtcp")
input(type="imtcp" port="514")
```

### Restart

```bash
systemctl restart rsyslog
```

---

## Scenario 51: Enabling Both TCP and UDP

### Configuration

```rsyslog
module(load="imudp")
input(type="imudp" port="514")

module(load="imtcp")
input(type="imtcp" port="514")
```

---

## Scenario 52: Checking 514 Port Listening

### Command

```bash
ss -tunlp | grep 514
```

If only UDP is enabled, you might see:

```text
udp   UNCONN 0 0 0.0.0.0:514
```

If TCP is enabled, you might see:

```text
tcp   LISTEN 0 0 0.0.0.0:514
```

---

## Scenario 53: Firewall Allowing Port 514

firewalld allowing TCP 514:

```bash
firewall-cmd --permanent --add-port=514/tcp
```

firewalld allowing UDP 514:

```bash
firewall-cmd --permanent --add-port=514/udp
```

Reload:

```bash
firewall-cmd --reload
```

Check:

```bash
firewall-cmd --list-ports
```

---

## Sixteen. rsyslog Storing Logs by Host

---

## Scenario 54: Why Store Logs by Host

If multiple machines forward logs to the same server, it's recommended to store logs by hostname or IP address.

Benefits:

```text
Easy to distinguish source hosts

It's easy to sort one machine log.

Enable filing and retention cycle management

Avoid all logs mixed in one file Medium
```

---

## Scenario 55: Storing Remote Logs by Hostname

### Server Configuration

```bash
vi /etc/rsyslog.d/20-remote-template.conf
```

### Example

```rsyslog
$template RemoteLogs,"/var/log/remote/%HOSTNAME%/%PROGRAMNAME%.log"
*.* ?RemoteLogs
& stop
```

### Creating Directory

```bash
mkdir -p /var/log/remote
```

### Restarting Service

```bash
systemctl restart rsyslog
```

---

## Scenario 56: Checking if Remote Logs are Written

### Command

```bash
find /var/log/remote -type f | head
```

Viewing latest logs:

```bash
tail -f /var/log/remote/*/*.log
```

---

## 17. logger Test Logs

---

## Scenario 57: What is logger

`logger` Can write a test message to the system log.

Suitable for verifying:

```text
Here. rsyslog Is it normal?

Whether client can send logs

Whether logs are received by service providers

Does the log drop as expected?
```

---

## Scenario 58: Send Test Logs

### Command

```bash
logger "this is a rsyslog test message"
```

View local logs:

RHEL / CentOS / Rocky / AlmaLinux:

```bash
tail -n 50 /var/log/messages
```

Ubuntu / Debian:

```bash
tail -n 50 /var/log/syslog
```

---

## Scenario 59: Send Logs with Tag

### Command

```bash
logger -t myapp "this is a myapp test message"
```

### Notes

```text
-t
→ Assign tag, usually displayed as a program name
```

Suitable for verifying logs categorized by `%PROGRAMNAME%`.

---

## Scenario 60: Specify Facility and Level

### Command

```bash
logger -p local0.info "local0 info test message"
```

```bash
logger -p authpriv.notice "authpriv notice test message"
```

### Notes

```text
-p
→ Assign facility.level
```

---

## 18. rsyslog Log Forwarding Verification

---

## Scenario 61: Client Packet Capture to Verify Log Transmission

### UDP

```bash
tcpdump -i any -nn udp port 514
```

### TCP

```bash
tcpdump -i any -nn tcp port 514
```

### Protocol-Independent

```bash
tcpdump -i any -nn port 514
```

---

## Scenario 62: Server Packet Capture to Verify Log Reception

### Command

```bash
tcpdump -i any -nn host ClientIP and port 514
```

Example:

```bash
tcpdump -i any -nn host 10.0.0.20 and port 514
```

---

## Scenario 63: Combined logger Test

Client execution:

```bash
logger -t rsyslog-test "hello from client"
```

Server packet capture:

```bash
tcpdump -i any -nn host ClientIP and port 514
```

Server log check:

```bash
find /var/log/remote -type f -exec grep -H "hello from client" {} \;
```

---

## 19. rsyslog Common Troubleshooting

---

## Scenario 64: rsyslog Forwarding Failure Troubleshooting Order

Troubleshooting order:

```text
Check Client Configuration

→ Check Client rsyslog Status

→ Check service-end configuration

→ Check service end rsyslog Status

→ Inspection 514 Port listening

→ Check firewall and security team.

→ logger Send Test Log

→ tcpdump Grab the bag and confirm if it's sent and arrived.

→ See if service has been successfully closed
```

---

## Scenario 65: Check Client Configuration

### Command

```bash
grep -R "@" /etc/rsyslog.conf /etc/rsyslog.d/
```

View configuration:

```bash
cat /etc/rsyslog.d/90-forward.conf
```

Check service:

```bash
systemctl status rsyslog
```

---

## Scenario 66: Check Server Listening

### Command

```bash
ss -tunlp | grep 514
```

Check configuration:

```bash
grep -R "imudp\|imtcp\|514" /etc/rsyslog.conf /etc/rsyslog.d/
```

---

## Scenario 67: Check Firewall

firewalld:

```bash
firewall-cmd --list-ports
```

iptables:

```bash
iptables -L INPUT -n -v
```

ufw:

```bash
ufw status verbose
```

---

## Scenario 68: View rsyslog Own Logs

systemd:

```bash
journalctl -u rsyslog -n 100
```

Real-time view:

```bash
journalctl -u rsyslog -f
```

---

## Scenario 69: Configuration Syntax Check

Some systems support:

```bash
rsyslogd -N1
```

Notes:

```text
-N1
→ Inspection rsyslog Configure Syntax:
```

Run and judge configuration syntax issues based on output.

---

## 20. Comprehensive Scenarios

---

## Scenario 70: Process Interface Return JSON and Extract Status

### Command

```bash
curl -s http://127.0.0.1:8080/health | jq -r '.status'
```

If interface returns:

```json
{"status":"ok","time":"2026-04-25T10:00:00+08:00"}
```

Output:

```text
ok
```

---

## Scenario 71: Script to Judge Interface Status and Email Notification

### Example

```bash
#!/bin/bash

URL="http://127.0.0.1:8080/health"
MAIL_TO="admin@example.com"

status=$(curl -s "$URL" | jq -r '.status // "unknown"')

if [ "$status" != "ok" ]; then
    echo "Interface health check abnormal. Current status:$status" | mail -s "Interface health check abnormal." "$MAIL_TO"
fi
```

---

## Scenario 72: Count JSON Log Abnormalities and Notify

### Example

```bash
#!/bin/bash

LOG_FILE="/var/log/myapp/app-json.log"
MAIL_TO="admin@example.com"

error_count=$(cat "$LOG_FILE" | jq -r 'select(.level == "error") | .level' | wc -l)

if [ "$error_count" -gt 0 ]; then
    echo "Current JSON Logging error Number:$error_count" | mail -s "JSON Log Error Statistics" "$MAIL_TO"
fi
```

---

## Scenario 73: Verify rsyslog Forwarding Chain

### Client

```bash
logger -t rsyslog-test "test message from $(hostname)"
```

### Client Packet Capture

```bash
tcpdump -i any -nn host 10.0.0.10 and port 514
```

### Server Check Listening

```bash
ss -tunlp | grep 514
```

### Server Check Logs

```bash
grep -R "rsyslog-test" /var/log/remote/
```

---

## 21. Production Notes

---

## 1. Confirm JSON Log Format Before Using jq

If the log is not standard JSON, `jq` will report errors.

For example, plain text logs:

```text
2026-04-25 ERROR db timeout
```

Cannot directly use jq.

Logs suitable for jq are typically:

```json
{"time":"2026-04-25T10:00:00+08:00","level":"error","message":"db timeout"}
```

---

## 2. jq -r is More Suitable for Scripts

Scripts usually need unquoted strings.

Recommended:

```bash
jq -r '.name' file.json
```

Instead of:

```bash
jq '.name' file.json
```

---

## 3. mail May Not Be Default Available

Before using mail, confirm:

```text
mail Whether or not to install the command

Can I send an email?

SMTP or MTA Configure

Whether the cloud server is restricted 25 Port

Is the mail intercepted?
```

---

## 4. Recommend Using TCP for Important Logs in rsyslog

UDP is simple but may lose logs.

Recommended for important logs:

```rsyslog
*.* @@10.0.0.10:514
```

That is, TCP forwarding.

---

## 5. rsyslog Server Must Allow Firewall

Even if the server is configured to receive, it doesn't mean the client can send.

Also check:

```text
514 Port listening

Is the firewall clear?

Security team clear.

Client-to-service network availability
```

---

## 6. Log Forwarding Success Does Not Equal Log Platform Availability

rsyslog forwarding is only part of the chain.

Also confirm:

```text
Could not close temporary folder: %s

Whether or not the log collector is collected

Whether the log platform is indexed

Whether the field is correct

Time field correct
```

---

## 7. Do Not Forward All Logs Indiscriminately Long-Term

`*.*` is simple but may bring:

```text
It's too big.

Increased network traffic

Log Server Disk Pressure

Index cost increases

Discovery of sensitive information
```

In production, select based on needs:

```text
facility

level

Log Source

Log Type

Retention cycle

De-sensitivity strategy
```

---

## 22. Common Commands Summary in This Article

---

## jq Installation and Basics

```bash
apt install -y jq
```

```bash
yum install -y jq
```

```bash
dnf install -y jq
```

```bash
jq --version
```

```bash
jq . file.json
```

```bash
cat file.json | jq .
```

```bash
jq -c . file.json
```

```bash
jq -S . file.json
```

---

## jq Field Extraction

```bash
jq '.name' file.json
```

```bash
jq -r '.name' file.json
```

```bash
jq '.name, .status, .replicas' file.json
```

```bash
jq -r '.name + " " + .status' file.json
```

```bash
jq -r '[.name, .status, .replicas] | @tsv' file.json
```

```bash
jq -r '.metadata.name' file.json
```

```bash
jq -r '.metadata.name // "unknown"' file.json
```

---

## jq Array Handling

```bash
jq '.items[]' file.json
```

```bash
jq -r '.items[].name' file.json
```

```bash
jq -r '.items[] | [.name, .status, .ip] | @tsv' file.json
```

```bash
jq '.items[] | select(.status == "Running")' file.json
```

§

## jq JSON Logs

```bash
cat app-json.log | jq .
```

```bash
cat app-json.log | jq 'select(.level == "error")'
```

```bash
cat app-json.log | jq -r 'select(.level == "error") | .message'
```

```bash
cat app-json.log | jq 'select(.status >= 500)'
```

```bash
cat app-json.log | jq -r 'select(.status >= 500) | [.time, .service, .status, .message] | @tsv'
```

```bash
cat app-json.log | jq 'select(.cost > 1)'
```

```bash
cat app-json.log | jq -r '.status' | sort | uniq -c | sort -nr
```

---

## mail

```bash
apt install -y mailutils
```

```bash
yum install -y mailx
```

```bash
dnf install -y mailx
```

```bash
echo "Log archive completed" | mail -s "Archive Notifications" admin@example.com
```

```bash
cat /tmp/result.txt | mail -s "Inspection results" admin@example.com
```

```bash
df -h | mail -s "Disk Usage Inspection" admin@example.com
```

```bash
journalctl -p err -n 50 --no-pager | mail -s "System Error Log" admin@example.com
```

```bash
which mail
```

```bash
tail -n 100 /var/log/maillog
```

```bash
tail -n 100 /var/log/mail.log
```

---

## rsyslog Configuration View

```bash
grep -v '^#' /etc/rsyslog.conf | grep -v '^$'
```

```bash
ls -lh /etc/rsyslog.d/
```

```bash
grep -R "@" /etc/rsyslog.conf /etc/rsyslog.d/
```

```bash
systemctl status rsyslog
```

```bash
systemctl restart rsyslog
```

```bash
systemctl enable rsyslog
```

```bash
rsyslogd -N1
```

---

## rsyslog Client Forwarding

UDP Forwarding:

```rsyslog
*.* @10.0.0.10:514
```

TCP Forwarding:

```rsyslog
*.* @@10.0.0.10:514
```

Authentication Logs:

```rsyslog
authpriv.* @@10.0.0.10:514
```

Kernel Logs:

```rsyslog
kern.* @@10.0.0.10:514
```

---

## rsyslog Server Receivers

UDP Reception:

```rsyslog
module(load="imudp")
input(type="imudp" port="514")
```

TCP Reception:

```rsyslog
module(load="imtcp")
input(type="imtcp" port="514")
```

Check Listening:

```bash
ss -tunlp | grep 514
```

---

## logger Test

```bash
logger "this is a rsyslog test message"
```

```bash
logger -t myapp "this is a myapp test message"
```

§§code_274§§

```bash
logger -p authpriv.notice "authpriv notice test message"
```

---

## tcpdump Verification

```bash
tcpdump -i any -nn udp port 514
```

```bash
tcpdump -i any -nn tcp port 514
```

```bash
tcpdump -i any -nn port 514
```

```bash
tcpdump -i any -nn host 10.0.0.20 and port 514
```

---

## firewalld Allow 514

```bash
firewall-cmd --permanent --add-port=514/tcp
```

```bash
firewall-cmd --permanent --add-port=514/udp
```

```bash
firewall-cmd --reload
```

```bash
firewall-cmd --list-ports
```

---

## Twenty-Three, One-Sentence Summary

This article's core is:

```text
jq Process structured JSON

mail Send script notifications

rsyslog Realize Log Forward

logger Generate Test Log

tcpdump Validate log links
```

jq Common Chains:

```text
Formatting JSON

→ Extract Fields

→ Through arrays

→ Condition filter

→ Output As tsv/csv

→ Combined sort / uniq / wc Statistics.
```

mail Common Chains:

```text
Script Execution

→ Findings

→ Generate notification content

→ mail Send Mail

→ See if mail log confirmed successful
```

rsyslog Forwarding Chains:

```text
Client Configuration Forward

→ Service-based open receiver

→ Let go! 514 Port

→ logger Send Test Log

→ tcpdump Catch bag confirmed.

→ Service-end check log descent
```

UDP and TCP Differences:

```text
@ 
→ UDP, simple but possibly lost the log

@@
→ TCPIt's more reliable. It's more recommended in the production log.
```

Production Recommendations:

```text
JSON Logs must confirm the correct format and use it again. jq

Prefer extraction fields in scripts jq -r

mail Notification of reliance on local mail environment for production of more recommended access to the alarm system or Webhook

rsyslog The service must not only configure the reception, but also check the port, firewall and security. Group

After the log is forwarded, confirm whether the service has been successfully closed

Do not forward all logs without discrimination for long periods, taking into account log volume, cost, dissensitisation and retention cycles
```