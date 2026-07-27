# 06-jq JSON Processing, Mail Notifications, and rsyslog Log Forwarding

# Linux #Log Analysis #Text Processing #jq #JSON #mail #rsyslog #logger #Log Forwarding #Centralized Logs #Ops #SRE

---

## Recommended Path

01-Linux Basics and Server Ops/02-Logs and Text Processing/06-jq JSON Processing, Mail Notifications, and rsyslog Log Forwarding.md

---

## I. Document Overview

This article covers common scenarios in Linux operations involving **JSON log processing, mail notifications, and rsyslog log forwarding**.

Key topics include:

- Basic usage of jq
- JSON formatting
- Extracting fields with jq
- Outputting without quotes using jq
- Traversing arrays with jq
- Conditional filtering with jq
- Processing JSON lines in logs
- Common jq parameters
- Mail notifications
- Examples of mail notification scripts
- Basic understanding of rsyslog
- rsyslog UDP/TCP forwarding
- Configuring rsyslog server-side reception
- Configuring rsyslog client-side forwarding
- Using logger to send test logs
- Verifying log transmission with tcpdump
- Common troubleshooting approaches for rsyslog

This article is part of the Log and Text Processing series, focusing on:

```text
How to process JSON logs, how to send notification scripts, and how to forward system logs to remote servers.
```

The goals are:

- To be able to view and extract JSON fields using jq
- To handle common JSON logs effectively
- To send mail notifications through scripts
- To understand the differences between rsyslog UDP/TCP forwarding methods
- To configure rsyslog client-side log forwarding
- To set up rsyslog server-side remote log reception
- To use logger and tcpdump to verify the log forwarding process

---

## II. Core Commands Overview

The core commands involved in this article include:

```text
jq
→ For processing JSON files and logs
mail
→ For sending mail notifications
rsyslog
→ For system log collection and forwarding
logger
→ For writing test messages to system logs
systemctl
→ For managing the rsyslog service
ss
→ For checking 514 port monitoring
tcpdump
→ For capturing packets to verify log transmission or receipt
grep
→ For checking configuration files and filtering log content
```

In summary:

```text
jq is used for processing JSON data.
mail is used for sending notifications.
rsyslog is used for forwarding system logs.
logger is used for generating test logs.
tcpdump is used for verifying log packet transmission over the network.
```

---

## III. jq: Basics of JSON Processing

---

## Scenario 1: What isjq?

`jq` is a commonly used JSON command-line tool in Linux.

Common applications include:

```text
Formatting JSON data
Extracting JSON fields
Traversing JSON arrays
Conditionally filtering JSON content
Processing API response data
Working with JSON returned by Kubernetes/Docker/cloud platforms
Manipulating structured logs
```

In operations, it is often used for:

```text
Viewing API return values
Processing JSON-formatted logs
Filtering Pod/Node/Service information
Analyzing Docker inspect output
Interpreting cloud platform CLI results
```

---

## Scenario 2: Installing jq

For Ubuntu/Debian:

```bash
apt install -y jq
```

For RHEL/CentOS/Rocky/AlmaLinux:

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

## Scenario 3: Preparing a Sample JSON File

Sample file:

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

## IV. jq for JSON Formatting

---

## Scenario 4: Formatting JSON

### Command

```bash
jq . file.json
```

### Explanation

```text
.
→ Represents the entire current JSON object
```

Effect:

```text
Converts compressed, single-line JSON into a readable format
```

---

## Scenario 5: Formatting JSON via Pipeline

### Command

```bash
cat filejq -r '.items[] | select(.status != "Running") | [.name, .status] | @tsv' file.jsonjournalctl -u postfix -n 100
or:

journalctl -u sendmail -n 100

---

## Scenario 40: Recommended Production Notification Methods

Email is suitable for simple notifications, but in production, it is recommended to combine it with the following methods:

Email

WeChat Work / Lark / DingTalk Webhook

Monitoring and Alerting Systems

Prometheus Alertmanager

Log Platform Alerts

Ticket Systems

Reasons:

Email may experience delays.

Email messages might be intercepted.

Email alerts are easily overlooked.

Webhooks and alert systems are more suitable for real-time notifications.## Scenario 63: Testing in conjunction with logger

On the client side:

```bash
logger -t rsyslog-test "hello from client"
```

On the server side, capture packets:

```bash
tcpdump -i any -nn host client_IP and port 514
```

Check if logs have been saved on the server:

```bash
find /var/log/remote -type f -exec grep -H "hello from client" {} \;
```

## Chapter 19: Common Troubleshooting Approaches for rsyslog

## Scenario 64: Order of troubleshooting when rsyslog forwarding fails

Troubleshooting sequence:

```text
Check the client configuration.

→ Check the status of the client's rsyslog service.

→ Check the server configuration.

→ Check the status of the server's rsyslog service.

→ Verify that port 514 is being monitored.

→ Check the firewall and security groups.

→ Send a test log using logger.

→ Use tcpdump to capture packets and confirm if they are sent and received.

→ Verify if logs have been successfully saved on the server.
```

## Scenario 65: Checking the client configuration

### Commands

```bash
grep -R "@" /etc/rsyslog.conf /etc/rsyslog.d/
```

View the configuration:

```bash
cat /etc/rsyslog.d/90-forward.conf
```

Check the service status:

```bash
systemctl status rsyslog
```

## Scenario 66: Checking server-side monitoring

### Commands

```bash
ss -tunlp | grep 514
```

Check the configuration:

```bash
grep -R "imudp\|imtcp\|514" /etc/rsyslog.conf /etc/rsyslog.d/
```

## Scenario 67: Checking the firewall

For firewalld:

```bash
firewall-cmd --list-ports
```

For iptables:

```bash
iptables -L INPUT -n -v
```

For ufw:

```bash
ufw status verbose
```

## Scenario 68: Viewing rsyslog's own logs

For systemd:

```bash
journalctl -u rsyslog -n 100
```

For real-time viewing:

```bash
journalctl -u rsyslog -f
```

## Scenario 69: Checking configuration syntax

Some systems support this:

```bash
rsyslogd -N1
```

Explanation:

```text
-N1
→ Checks the syntax of the rsyslog configuration.
```

After execution, judge whether there are any syntax issues based on the output.

## Chapter 20: Comprehensive Scenarios

## Scenario 70: Processing JSON returned by an API and extracting the status

### Commands

```bash
curl -s http://127.0.0.1:8080/health | jq -r '.status'
```

If the API returns:

```json
{"status":"ok","time":"2026-04-25T10:00:00+08:00"}
```

The output will be:

```text
ok
```

## Scenario 71: Using a script to determine the API status and notify via email

### Example

```bash
#!/bin/bash

URL="http://127.0.0.1:8080/health"
MAIL_TO="admin@example.com"

status=$(curl -s "$URL" | jq -r '.status // "unknown"')

if [ "$status" != "ok" ]; then
    echo "API health check failed. Current status: $status" | mail -s "API Health Check Failure" "$MAIL_TO"
fi
```

## Scenario 72: Counting errors in JSON logs and notifying

### Example

```bash
#!/bin/bash

LOG_FILE="/var/log/myapp/app-json.log"
MAIL_TO="admin@example.com"

error_count=$(cat "$LOG_FILE" | jq -r 'select(.level == "error") | .level' | wc -l)

if [ "$error_count" -gt 0 ]; then
    echo "Number of errors in the JSON logs: $error_count" | mail -s "JSON Log Error Statistics" "$MAIL_TO"
fi
```

## Scenario 73: Verifying the rsyslog forwarding chain

### On the client side:

```bash
logger -t rsyslog-test "test message from $(hostname)"
```

### On the client side, capture packets:

```bash
tcpdump -i any -nn host 10.0.0.10 and port 514
```

### On the server side, check monitoring:

$$
ss -tunlp | grep 514
$$

### On```bash
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

## Mail

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
echo "Log archiving completed" | mail -s "Archival Notification" admin@example.com
```

```bash
cat /tmp/result.txt | mail -s "Inspection Results" admin@example.com
```

```bash
df -h | mail -s "Disk Usage Inspection" admin@example.com
```

```bash
journalctl -p err -n 50 --no-pager | mail -s "System Error Logs" admin@example.com
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

## Viewing Rsyslog Configuration

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

## Rsyslog Client Forwarding

UDP Forwarding:

```bash
rsyslog *.* @10.0.0.10:514
```

TCP Forwarding:

```bash
rsyslog *.* @@10.0.0.10:514
```

Authentication Logs:

```bash
rsyslog authpriv.* @@10.0.0.10:514
```

Kernel Logs:

```bash
rsyslog kern.* @@10.0.0.10:514
```

---

## Rsyslog Server Reception

UDP Reception:

```bash
module(load="imudp")
input(type="imudp" port="514")
```

TCP Reception:

```bash
module LOAD="imtcp")
input(type="imtcp" port="514")
```

Checking Listening:

```bash
ss -tunlp | grep 514
```

---

## Logger Testing

```bash
logger "This is a Rsyslog test message"
```

```bash
logger -t myapp "This is a MyApp test message"
```

```bash
logger -p local0.info "local0 info test message"
```

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

## Firewalld Allowing Port 514

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

## Summary

This chapter focuses on:

```text
Using jq to process structured JSON data

Sending notifications via mail scripts

Implementing log forwarding with rsyslog

Generating test logs with logger

Verifying log transmission using tcpdump
```

Common usage of jq includes:

```text
Formatting JSON data

Extracting specific fields

Looping through arrays

Filtering based on conditions

Outputting in tsv/csv format

Combining with sort, uniq, and wc for statistics
```

For mail notifications:

```text
Executing scripts

Determining results

