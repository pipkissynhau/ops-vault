# 07-Firewall, DNS, Logs, and Download Assistance Commands

#Linux #Transport #TheBarrier. #Firewall #DNS #SystemLog #journalctl #firewalld #iptables #ufw #nslookup #dig #wget #curl #scp #rsync

---

## Recommended Path

01-Linux Foundation and Host Maintenance/01-Host Troubleshooting/07-Firewall, DNS, Logs, and Download Assistance Commands.md

---

## I. Document Description

This document organizes common **firewall, DNS, system logs, download, and file transfer assistance commands** in Linux host troubleshooting.

Key focuses include:

- firewalld troubleshooting
- iptables troubleshooting
- ufw troubleshooting
- Firewall port forwarding
- Firewall rule inspection
- DNS resolution troubleshooting
- `/etc/resolv.conf`
- `nslookup`
- `dig`
- `getent hosts`
- System log inspection
- `/var/log/messages`
- `/var/log/syslog`
- `journalctl`
- `dmesg`
- File download
- `wget`
- `curl`
- File transfer
- `scp`
- `rsync`
- Compression and decompression
- Temporary troubleshooting assistance commands

Goals:

- Determine if a port is blocked by the local firewall
- Determine if DNS resolution is abnormal
- Inspect system logs and service logs
- Download troubleshooting tools or installation packages
- Transfer logs, configurations, and backup files between hosts
- Perform basic compression, decompression, and temporary troubleshooting operations

---

## II. Document Scope

Previous articles have already organized:

```text
01
→ Host-disable overview and system resource inventory

02
→ CPUMemory and load checking

03
→ Disk space,inode With Disk IO Check.

04
→ Disk Partition, Mounting and LVM Expansion

05
→ Processes and systemd Service clearance

06
→ Network connectivity, port, route and traffic mapping
```

This article mainly supplements auxiliary capabilities in host troubleshooting:

```text
Firewall
→ Why is the port wired, but no external access.

DNS
→ Why can't you parse or parse to errors? IP

Log
→ Why service failure, system anomaly, kernel error.

Download and Transfer
→ How to quickly pull tools, upload logs, sync files

Compression decompression
→ How to pack logs, backup configurations, and transport material for detoxification
```

---

## III. Firewall Troubleshooting Overview

When encountering:

```text
Service port monitored

Here. curl Normal

No external visits

ping It might work, but the port doesn't.

nc Connection timed out

Part of the network is accessible, part of it is inaccessible
```

Consider factors such as firewall, security group, ACL, and network policies.

Common local firewall tools on hosts include:

```text
firewalld
iptables
ufw
```

Recommended troubleshooting order:

```text
Confirm whether or not the service is listening mouth

→ Confirms whether the listening address is correct.

→ View the machine firewall

→ Check cloud security. / Network ACL

→ End nc / curl Test

→ Here. tcpdump Grab the bag and confirm the request.
```

---

## IV. firewalld Troubleshooting

---

## Scenario 1: Check firewalld Status

### Command

```bash
systemctl status firewalld
```

Or:

```bash
firewall-cmd --state
```

### Possible Results

```text
running
→ firewalld Running

not running
→ firewalld Not Run
```

---

## Scenario 2: Check Current Default Zone

### Command

```bash
firewall-cmd --get-default-zone
```

### Note

firewalld uses zones to manage rules.

Common zones:

```text
public
trusted
internal
external
dmz
work
home
block
drop
```

---

## Scenario 3: Check Current Active Zone

### Command

```bash
firewall-cmd --get-active-zones
```

### Purpose

Check which network interfaces are bound to which zones.

---

## Scenario 4: Check Current Zone Rules

### Command

```bash
firewall-cmd --list-all
```

Check a specific zone:

```bash
firewall-cmd --zone=public --list-all
```

### Key Focus

```text
services
→ Released services

ports
→ Released Port

interfaces
→ Current zone A bound card.

sources
→ Source Address Rules
```

---

## Scenario 5: Check Allowed Ports

### Command

```bash
firewall-cmd --list-ports
```

Check a specific zone:

```bash
firewall-cmd --zone=public --list-ports
```

---

## Scenario 6: Temporarily Allow Port

### Command

Allow TCP 8080:

```bash
firewall-cmd --add-port=8080/tcp
```

Allow UDP 53:

```bash
firewall-cmd --add-port=53/udp
```

### Note

Omitting `--permanent` makes it temporary. It may expire after firewalld reload or restart.

---

## Scenario 7: Permanently Allow Port

### Command

```bash
firewall-cmd --permanent --add-port=8080/tcp
```

Reload:

```bash
firewall-cmd --reload
```

Verify:

```bash
firewall-cmd --list-ports
```

---

## Scenario 8: Remove Port Allowance

### Temporarily Remove

```bash
firewall-cmd --remove-port=8080/tcp
```

### Permanently Remove

```bash
firewall-cmd --permanent --remove-port=8080/tcp
```

Reload:

```bash
firewall-cmd --reload
```

---

## Scenario 9: Allow Service

### Command

Allow HTTP:

```bash
firewall-cmd --permanent --add-service=http
```

Allow HTTPS:

```bash
firewall-cmd --permanent --add-service=https
```

Reload:

```bash
firewall-cmd --reload
```

Check:

```bash
firewall-cmd --list-services
```

---

## Scenario 10: firewalld Common Troubleshooting Flow

### Command

```bash
ss -tunlp | grep 8080
```

```bash
firewall-cmd --state
```

```bash
firewall-cmd --get-active-zones
```

```bash
firewall-cmd --list-all
```

```bash
firewall-cmd --list-ports
```

```bash
tcpdump -i any -nn port 8080
```

### Judgment

```text
ss I can see it.
→ The server is listening.

firewalld No Port Released
→ Could be intercepted by the firewall.

tcpdump I can't see the request.
→ Request may not be on-board. Continue to check the security team, route, upper firewall.

tcpdump I saw the request but didn't respond.
→ Keep checking the machine services, firewalls, logs.
```

---

## V. iptables Troubleshooting

---

## Scenario 11: Check filter Table Rules

### Command

```bash
iptables -L -n -v
```

### Parameter Notes

```text
-L
→ List rules

-n
→ Number shows that hostname and port are not parsed First Name

-v
→ Show detailed count
```

Key focus:

```text
INPUT
OUTPUT
FORWARD
DROP
REJECT
ACCEPT
```

---

## Scenario 12: Check nat Table Rules

### Command

```bash
iptables -t nat -L -n -v
```

### Purpose

Check NAT rules.

Common scenarios:

```text
Docker Port Map

SNAT / MASQUERADE

DNAT Forward

Gateway Forward
```

---

## Scenario 13: Check iptables Original Rules

### Command

```bash
iptables -S
```

Check NAT table:

```bash
iptables -t nat -S
```

### Purpose

Display rules in a format suitable for copying, analysis, and saving.

---

## Scenario 14: Check INPUT Chain

### Command

```bash
iptables -L INPUT -n -v
```

### Purpose

Troubleshoot whether incoming traffic is allowed.

If INPUT default policy is DROP:

```text
Chain INPUT (policy DROP)
```

Then explicit allow rules must exist.

---

## Scenario 15: Temporarily Allow Port

### Command

Allow TCP 8080:

```bash
iptables -I INPUT -p tcp --dport 8080 -j ACCEPT
```

Allow specific source to access 8080:

```bash
iptables -I INPUT -p tcp -s 10.0.0.5 --dport 8080 -j ACCEPT
```

### Note

This method is typically temporary rules, which may be lost after reboot, depending on whether the system saves iptables rules.

---

## Scenario 16: Delete iptables Rules

First check rule numbers:

```bash
iptables -L INPUT -n --line-numbers
```

Delete specific numbered rule:

```bash
iptables -D INPUT Rule number
```

Example:

```bash
iptables -D INPUT 3
```

---

## Scenario 17: Do Not Arbitrarily Clear iptables

High-risk commands:

```bash
iptables -F
```

```bash
iptables -t nat -F
```

Risks:

```text
Could be. Docker Port map anomaly

Could be. Kubernetes Node network anomaly

Possible clearance of firewall rules

Possible exposure or disruption of operations
```

Do not execute in production environments.

---

## Scenario 18: Relationship Between iptables and Docker

Docker maintains iptables rules.

Common chains:

```text
DOCKER
DOCKER-USER
DOCKER-ISOLATION-STAGE-1
DOCKER-ISOLATION-STAGE-2
```

Check:

```bash
iptables -L DOCKER-USER -n -v
```

```bash
iptables -t nat -L DOCKER -n -v
```

Note:

```text
Docker PortMap Dependence iptables NAT Rule

Clear iptables Could be. Docker Network anomaly
```

---

## VI. ufw Troubleshooting

---

## Scenario 19: Check ufw Status

### Command

```bash
ufw status
```

Detailed check:

```bash
ufw status verbose
```

Check with numbering:

```bash
ufw status numbered
```

### Note

Common on Ubuntu systems.

---

## Scenario 20: Enable and Disable ufw

### Enable

```bash
ufw enable
```

### Disable

```bash
ufw disable
```

Production Notes:

```text
Remote SSH Enabled in Environment ufw Before, we must let go. SSH Port
Or you could lock yourself in the machine. Outside.
```

---

## Scenario 21: Allow Port

### Commands

Allow 8080:

```bash
ufw allow 8080/tcp
```

Allow HTTP:

```bash
ufw allow 80/tcp
```

Allow HTTPS:

```bash
ufw allow 443/tcp
```

Allow SSH:

```bash
ufw allow 22/tcp
```

---

## Scenario 22: Deny Port

### Commands

```bash
ufw deny 8080/tcp
```

---

## Scenario 23: Delete Rule

First check the ID:

```bash
ufw status numbered
```

Delete specified ID:

```bash
ufw delete Rule number
```

Example:

```bash
ufw delete 3
```

---

## Seven. Firewall Typical Troubleshooting Scenarios

---

## Scenario 24: Port is listening, but external access is not working

### Troubleshooting Commands

```bash
ss -tunlp | grep 8080
```

```bash
firewall-cmd --list-all
```

```bash
iptables -L INPUT -n -v
```

```bash
ufw status verbose
```

```bash
tcpdump -i any -nn port 8080
```

### Judgment

```text
ss Listen 127.0.0.1:8080
→ External access is impossible. Service is only listening.

ss Listen 0.0.0.0:8080But the firewall was not released
→ Require port release

tcpdump I can't see the request.
→ Could be intercepted by an upstream firewall or a security unit.

tcpdump I saw the request but didn't respond.
→ Check machine firewalls, service logs, application status
```

---

## Scenario 25: Temporarily disable firewall for verification

### firewalld

```bash
systemctl stop firewalld
```

### ufw

```bash
ufw disable
```

### Notes

This can only be used as a temporary verification method.

It is not recommended to disable firewall in production environments long-term.

After verification, restore:

```bash
systemctl start firewalld
```

Or:

```bash
ufw enable
```

---

## Eight. DNS Troubleshooting General Approach

Common DNS issues:

```text
ping IP All right.ping Domainname not valid

curl Domainname failed, but curl IP Success

Apply connection domain name timeout

Domain Name Parsing Error IP

The results of different machine resolution are different.

Can not parse domain name inside container

Kubernetes Pod Internal DNS Unusual

Failed to access inline domain name
```

Troubleshooting order:

```text
Confirm. IP Is it possible?

→ View /etc/resolv.conf

→ nslookup Question

→ dig Question

→ getent hosts Query system resolution results

→ Assign DNS Server Test

→ Compare the results of different machines

→ Inspection DNS Cache,DNS Services and network strategies
```

---

## Nine. /etc/resolv.conf

---

## Scenario 26: Check DNS Configuration

### Commands

```bash
cat /etc/resolv.conf
```

### Common Content

```text
nameserver 114.114.114.114
nameserver 8.8.8.8
search localdomain
options timeout:2 attempts:3
```

Pay special attention to:

```text
nameserver Correct?

DNS Server availability

Was it... NetworkManager / systemd-resolved Management

Inside the container resolv.conf Correct?
```

---

## Scenario 27: Test DNS Server Reachability

### Commands

```bash
ping -c 4 DNSServersIP
```

Example:

```bash
ping -c 4 114.114.114.114
```

Test DNS port:

```bash
nc -zv -u 114.114.114.114 53
```

Notes:

```text
DNS Common UDP 53Or it could be used. TCP 53
```

Test TCP 53:

```bash
nc -zv 114.114.114.114 53
```

---

## Ten. nslookup

---

## Scenario 28: Resolve Domain Name

### Commands

```bash
nslookup www.baidu.com
```

### Purpose

Check domain name resolution results.

---

## Scenario 29: Specify DNS Server for Resolution

### Commands

```bash
nslookup www.baidu.com 114.114.114.114
```

```bash
nslookup www.baidu.com 8.8.8.8
```

### Purpose

Compare DNS server resolution results.

Suitable for determining:

```text
Here. DNS Is the configuration abnormal?

Intranet DNS Is it unusual?

Internet DNS Is it normal?

Different. DNS Whether to return different IP
```

---

## Scenario 30: Resolve Internal Network Domain

### Commands

```bash
nslookup Inner domain name IntranetDNSServersIP
```

Example:

```bash
nslookup gitlab.example.local 10.0.0.2
```

If public DNS cannot resolve internal network domain, this is normal.

Internal network domains should use internal DNS.

---

## Eleven. dig

---

## Scenario 31: Use dig to Query Domain

### Commands

```bash
dig www.baidu.com
```

### Purpose

`dig` output is more detailed than `nslookup`.

Pay special attention to:

```text
ANSWER SECTION
SERVER
Query time
status
```

---

## Scenario 32: Show Simplified Resolution Results

### Commands

```bash
dig +short www.baidu.com
```

### Purpose

Only display resolved IP or CNAME.

---

## Scenario 33: Specify DNS Server

### Commands

```bash
dig @114.114.114.114 www.baidu.com
```

```bash
dig @8.8.8.8 www.baidu.com
```

Internal DNS Example:

```bash
dig @10.0.0.2 gitlab.example.local
```

---

## Scenario 34: Check A Record

### Commands

```bash
dig A www.baidu.com
```

---

## Scenario 35: Check CNAME Record

### Commands

```bash
dig CNAME www.example.com
```

---

## Scenario 36: Check MX Record

### Commands

```bash
dig MX example.com
```

---

## Scenario 37: Check TXT Record

### Commands

```bash
dig TXT example.com
```

---

## Twelve. getent hosts

---

## Scenario 38: Check System Actual Resolution Results

### Commands

```bash
getent hosts www.baidu.com
```

### Purpose

`getent hosts` follows the system's name resolution chain.

It references:

```text
/etc/nsswitch.conf

/etc/hosts

DNS
```

Suitable for determining:

```text
What happens when the system actually solves the domain?
```

---

## Scenario 39: /etc/hosts Affects Resolution

Check hosts:

```bash
cat /etc/hosts
```

If `/etc/hosts` contains:

```text
10.0.0.10 app.example.com
```

The system may prioritize resolving to this IP instead of the DNS result.

When troubleshooting, note:

```bash
getent hosts app.example.com
```

```bash
cat /etc/hosts
```

---

## Thirteen. DNS Typical Troubleshooting Scenarios

---

## Scenario 40: IP is reachable, but domain is not

### Troubleshooting Commands

```bash
ping -c 4 8.8.8.8
```

```bash
cat /etc/resolv.conf
```

```bash
nslookup www.baidu.com
```

```bash
dig +short www.baidu.com
```

```bash
getent hosts www.baidu.com
```

### Possible Causes

```text
DNS Server not available

/etc/resolv.conf Configure Error

DNS The server itself is abnormal.

Domain name does not exist

Internal domain names are used on the public network. DNS

/etc/hosts Overwrite Parsing

Firewall blocked. UDP/TCP 53
```

---

## Scenario 41: Inconsistent Resolution Results Between Machines

### Troubleshooting Commands

Machine A:

```bash
dig +short Domain Name
```

Machine B:

```bash
dig +short Domain Name
```

Specify the same DNS:

```bash
dig @10.0.0.2 Domain Name
```

Check local configuration:

```bash
cat /etc/resolv.conf
```

```bash
cat /etc/hosts
```

### Possible Causes

```text
It's different. DNS Servers

/etc/hosts Inconsistencies

DNS Cache not refreshed

Intranet DNS Different View

Load Balance DNS Return Different IP

CDN Smart Parsing
```

---

## Scenario 42: DNS Abnormalities in Containers

### Troubleshooting Commands

Enter container:

```bash
docker exec -it ContainersID /bin/sh
```

Check DNS:

```bash
cat /etc/resolv.conf
```

Test resolution:

```bash
nslookup www.baidu.com
```

Or:

```bash
getent hosts www.baidu.com
```

Check host:

```bash
cat /etc/resolv.conf
```

Docker DNS Configuration:

```bash
cat /etc/docker/daemon.json
```

May need to configure:

```json
{
  "dns": ["114.114.114.114", "8.8.8.8"]
}
```

Restart Docker:

```bash
systemctl restart docker
```

Notes:

```text
Docker DNS After the configuration changes, the container is normally re-created to be fully effective.
```

---

## Fourteen. System Log Troubleshooting

---

## Scenario 43: Common System Log Paths

Log paths vary by distribution.

Common for RHEL / CentOS / Rocky / AlmaLinux:

```text
/var/log/messages
/var/log/secure
/var/log/cron
/var/log/dmesg
```

Common for Ubuntu / Debian:

```text
/var/log/syslog
/var/log/auth.log
/var/log/kern.log
```

Common for systemd systems:

```text
journalctl
```

---

## Scenario 44: Check messages

### Commands

```bash
tail -n 100 /var/log/messages
```

Real-time check:

```bash
tail -f /var/log/messages
```

Search for errors:

```bash
grep -i error /var/log/messages
```

```bash
grep -i fail /var/log/messages
```

---

## Scenario 45: Check syslog

### Commands

```bash
tail -n 100 /var/log/syslog
```

Real-time check:

```bash
tail -f /var/log/syslog
```

Search for errors:

```bash
grep -i error /var/log/syslog
```

```bash
grep -i fail /var/log/syslog
```

---

## Scenario 46: Check Authentication Logs

### RHEL-based

```bash
tail -n 100 /var/log/secure
```

### Ubuntu / Debian

```bash
tail -n 100 /var/log/auth.log
```

Suitable for troubleshooting:

```text
SSH Login Failed

sudo Questions of competence

User authentication failed

Violence cracks traces.

PAM Accreditation issues
```

---

## Scenario 47: View cron logs

### RHEL-based systems

```bash
tail -n 100 /var/log/cron
```

### Ubuntu / Debian

Cron logs on Ubuntu are typically in syslog:

```bash
grep CRON /var/log/syslog
```

Suitable for troubleshooting:

```text
Timed tasks performed

Expected execution time

Whether the job is wrong or not

Whether the environment variable is missing
```

---

## Section 15: journalctl log troubleshooting

---

## Scenario 48: View system logs

### Commands

```bash
journalctl
```

View last 100 lines:

```bash
journalctl -n 100
```

Real-time viewing:

```bash
journalctl -f
```

---

## Scenario 49: View logs since last boot

### Commands

```bash
journalctl -b
```

View last 100 lines:

```bash
journalctl -b -n 100
```

---

## Scenario 50: View logs from previous boot

### Commands

```bash
journalctl -b -1
```

Suitable for troubleshooting:

```text
What happened before the system was restarted?

Why service failed last time

Was there a kernel error before the machine was restarted?
```

---

## Scenario 51: View logs for specific service

### Commands

```bash
journalctl -u Service Name
```

Example:

```bash
journalctl -u nginx
```

Last 100 lines:

```bash
journalctl -u nginx -n 100
```

Real-time viewing:

```bash
journalctl -u nginx -f
```

---

## Scenario 52: View logs by time

### Commands

View today's logs:

```bash
journalctl --since today
```

View last 1 hour:

```bash
journalctl --since "1 hour ago"
```

View specific time range:

```bash
journalctl --since "2026-04-25 10:00:00" --until "2026-04-25 11:00:00"
```

View specific service time range:

```bash
journalctl -u nginx --since "2026-04-25 10:00:00" --until "2026-04-25 11:00:00"
```

---

## Scenario 53: View logs by priority

### Commands

View error and more severe logs:

```bash
journalctl -p err
```

View warning and more severe:

```bash
journalctl -p warning
```

View errors since last boot:

```bash
journalctl -b -p err
```

---

## Scenario 54: Output without pagination

### Commands

```bash
journalctl --no-pager
```

Specific service:

```bash
journalctl -u nginx --no-pager
```

---

## Section 16: dmesg kernel logs

---

## Scenario 55: View kernel logs

### Commands

```bash
dmesg
```

View recent logs:

```bash
dmesg | tail
```

With readable timestamps:

```bash
dmesg -T | tail -n 100
```

---

## Scenario 56: Filter kernel errors

### Commands

```bash
dmesg -T | grep -i error
```

```bash
dmesg -T | grep -i fail
```

```bash
dmesg -T | grep -i oom
```

```bash
dmesg -T | grep -i kill
```

```bash
dmesg -T | grep -i xfs
```

```bash
dmesg -T | grep -i ext4
```

```bash
dmesg -T | grep -i nvme
```

```bash
dmesg -T | grep -i scsi
```

Suitable for troubleshooting:

```text
OOM

Disk Error

Filesystem Error

Driver anomaly

kernel error.

Cybercard anomaly.

System Unusual Reboot Thread
```

---

## Section 17: wget file download

---

## Scenario 57: Download files

### Commands

```bash
wget URL
```

Example:

```bash
wget http://example.com/file.tar.gz
```

---

## Scenario 58: Specify output filename

### Commands

```bash
wget -O filename.tar.gz URL
```

---

## Scenario 59: Resume interrupted downloads

### Commands

```bash
wget -c URL
```

### Function

Suitable for resuming large file downloads after interruption.

---

## Scenario 60: Background download

### Commands

```bash
wget -b URL
```

View download logs:

```bash
tail -f wget-log
```

---

## Section 18: curl download and API testing

---

## Scenario 61: Download files

### Commands

```bash
curl -O URL
```

Specify filename:

```bash
curl -o filename.tar.gz URL
```

---

## Scenario 62: Follow redirects for download

### Commands

```bash
curl -L -O URL
```

Specify filename:

```bash
curl -L -o filename.tar.gz URL
```

---

## Scenario 63: View HTTP response headers

### Commands

```bash
curl -I URL
```

---

## Scenario 64: Detailed request process

### Commands

```bash
curl -v URL
```

Ignore SSL certificate for HTTPS:

```bash
curl -vk https://example.com
```

---

## Scenario 65: Set timeout

### Commands

```bash
curl -m 5 URL
```

---

## Section 19: scp file transfer

---

## Scenario 66: Copy files from local to remote

### Commands

```bash
scp Local File Username@FarIP:/Destination Directory/
```

Example:

```bash
scp app.log root@10.0.0.10:/tmp/
```

---

## Scenario 67: Copy files from remote to local

### Commands

```bash
scp Username@FarIP:/Remote File Local Directory/
```

Example:

```bash
scp root@10.0.0.10:/var/log/messages /tmp/
```

---

## Scenario 68: Copy directories

### Commands

```bash
scp -r Local Directory Username@FarIP:/Destination Directory/
```

Example:

```bash
scp -r /tmp/logs root@10.0.0.10:/tmp/
```

---

## Scenario 69: Specify SSH port

### Commands

```bash
scp -P 2222 Documentation Username@FarIP:/Destination Directory/
```

Note:

```text
scp Specify port using uppercase -P
ssh Specify port to use lowercase -p
```

---

## Section 20: rsync file synchronization

---

## Scenario 70: Sync directory to remote

### Commands

```bash
rsync -avz Local Directory/ Username@FarIP:/Destination Directory/
```

Example:

```bash
rsync -avz /data/logs/ root@10.0.0.10:/backup/logs/
```

### Parameter explanation

```text
-a
→ Archive mode, retention rights, time, etc.

-v
→ Show detailed process

-z
→ Compression on Transfer
```

---

## Scenario 71: Sync from remote to local

### Commands

```bash
rsync -avz Username@FarIP:/Remote Directory/ Local Directory/
```

Example:

```bash
rsync -avz root@10.0.0.10:/backup/logs/ /data/logs/
```

---

## Scenario 72: Show synchronization progress

### Commands

```bash
rsync -avz --progress Local Directory/ Username@FarIP:/Destination Directory/
```

---

## Scenario 73: Delete extra files on target

### Commands

```bash
rsync -avz --delete Local Directory/ Username@FarIP:/Destination Directory/
```

### Note

```text
--delete Can not delete folder: %s: No such folder
Good working environment
```

---

## Scenario 74: Specify SSH port

### Commands

```bash
rsync -avz -e "ssh -p 2222" Local Directory/ Username@FarIP:/Destination Directory/
```

---

## Section 21: Compression and decompression helper commands

---

## Scenario 75: Package directory as tar.gz

### Commands

```bash
tar czf logs.tar.gz /var/log/nginx
```

---

## Scenario 76: Decompress tar.gz

### Commands

```bash
tar xzf logs.tar.gz
```

Extract to specific directory:

```bash
tar xzf logs.tar.gz -C /tmp/
```

---

## Scenario 77: View contents of compressed file

### Commands

```bash
tar tzf logs.tar.gz
```

---

## Scenario 78: Compress as zip

### Commands

```bash
zip -r logs.zip /var/log/nginx
```

---

## Scenario 79: Decompress zip

### Commands

```bash
unzip logs.zip
```

Extract to specific directory:

```bash
unzip logs.zip -d /tmp/logs
```

---

## Section 22: Temporary troubleshooting helper commands

---

## Scenario 80: View current directory

### Commands

```bash
pwd
```

---

## Scenario 81: View file list

### Commands

```bash
ls -lah
```

---

## Scenario 82: View command path

### Commands

```bash
which Command
```

Example:

```bash
which nginx
```

---

## Scenario 83: View command help

### Commands

```bash
Command --help
```

Example:

```bash
curl --help
```

View manual:

```bash
man Command
```

Example:

```bash
man systemctl
```

---

## Scenario 84: View history commands

### Commands

```bash
history
```

Search history commands:

```bash
history | grep docker
```

---

## Scenario 85: View current user

### Commands

```bash
whoami
```

View user ID and groups:

```bash
id
```

---

## Scenario 86: View logged-in users

### Commands /think

```bash
who
```

```bash
w
```

---

## Scenario 87: Check System Time

### Command

```bash
date
```

Check timezone:

```bash
timedatectl
```

---

## Twenty-Three, Common Comprehensive Troubleshooting Scenarios

---

## Scenario 88: Service Port Listening is Normal, but External Access Times Out

### Troubleshooting Commands

```bash
ss -tunlp | grep Port
```

```bash
firewall-cmd --list-all
```

```bash
iptables -L INPUT -n -v
```

```bash
ufw status verbose
```

```bash
tcpdump -i any -nn port Port
```

### Judgment Direction

```text
Service listening 127.0.0.1
→ External access is not possible

Firewall not released
→ Release Port

tcpdump I can't see the request.
→ Security team, route, upper firewall

tcpdump There's no response to the request.
→ Check machine services, firewalls, application logs
```

---

## Scenario 89: Domain Access Fails, but IP Can Be Accessed

### Troubleshooting Commands

```bash
cat /etc/resolv.conf
```

```bash
nslookup Domain Name
```

```bash
dig +short Domain Name
```

```bash
getent hosts Domain Name
```

```bash
cat /etc/hosts
```

### Judgment Direction

```text
DNS Unattainable.

DNS Configure Error

/etc/hosts Overwrite

Internal domain names are used on the public network. DNS

Domain Name Resolution Error IP
```

---

## Scenario 90: Service Startup Failed, Need to Check Logs

### Troubleshooting Commands

```bash
systemctl status Service Name
```

```bash
journalctl -u Service Name -n 100
```

```bash
journalctl -u Service Name -f
```

```bash
dmesg -T | tail -n 100
```

If it's traditional logs:

```bash
tail -n 100 /var/log/messages
```

Or:

```bash
tail -n 100 /var/log/syslog
```

---

## Scenario 91: Need to Package Logs for Others to Analyze

### Command

Package log directory:

```bash
tar czf nginx-logs-$(date +%F).tar.gz /var/log/nginx
```

Transfer to remote:

```bash
scp nginx-logs-$(date +%F).tar.gz root@10.0.0.10:/tmp/
```

Or synchronize directory:

```bash
rsync -avz --progress /var/log/nginx/ root@10.0.0.10:/tmp/nginx-logs/
```

---

## Twenty-Four, Production Notes

---

## 1. Do Not Arbitrarily Close Firewall

Temporary firewall closure is only for verification.

After verification, it should be restored and necessary ports should be precisely allowed.

---

## 2. Do Not Arbitrarily Clear iptables

Especially for machines running Docker, Kubernetes, NAT, or gateway services.

Clearing iptables may cause network interruption.

---

## 3. DNS Troubleshooting Should Distinguish System Resolution and Tool Resolution

`dig` and `nslookup` mainly directly check DNS.

`getent hosts` is closer to the system's actual resolution chain.

If `/etc/hosts` has records, system resolution may differ from DNS results.

---

## 4. Log Troubleshooting Should Pay Attention to Time Range

When troubleshooting, try to clearly define:

```text
Time of problem

Impact services

Corresponding log files

Has the system been restarted?

Whether service has been restarted
```

Suggested combined with:

```bash
journalctl --since "Time" --until "Time"
```

---

## 5. Pay Attention to Desensitization Before Transmitting Logs

Logs may contain:

```text
Username

Password

token

AccessKey

Cookie

IP Address

Business sensitive data
```

Before transmission, confirm whether desensitization is needed.

---

## 6. rsync --delete Should Be Used with Caution

`--delete` will delete extra files on the target end.

Before using in production, confirm that source directory and target directory are not reversed.

---

## Twenty-Five, Summary of Common Commands in This Chapter

---

## firewalld

```bash
systemctl status firewalld
```

```bash
firewall-cmd --state
```

```bash
firewall-cmd --get-default-zone
```

```bash
firewall-cmd --get-active-zones
```

```bash
firewall-cmd --list-all
```

```bash
firewall-cmd --zone=public --list-all
```

```bash
firewall-cmd --list-ports
```

```bash
firewall-cmd --add-port=8080/tcp
```

```bash
firewall-cmd --permanent --add-port=8080/tcp
```

```bash
firewall-cmd --reload
```

```bash
firewall-cmd --permanent --remove-port=8080/tcp
```

```bash
firewall-cmd --permanent --add-service=http
```

```bash
firewall-cmd --permanent --add-service=https
```

```bash
firewall-cmd --list-services
```

---

## iptables

```bash
iptables -L -n -v
```

```bash
iptables -t nat -L -n -v
```

```bash
iptables -S
```

```bash
iptables -t nat -S
```

```bash
iptables -L INPUT -n -v
```

```bash
iptables -L INPUT -n --line-numbers
```

```bash
iptables -I INPUT -p tcp --dport 8080 -j ACCEPT
```

```bash
iptables -I INPUT -p tcp -s 10.0.0.5 --dport 8080 -j ACCEPT
```

```bash
iptables -D INPUT Rule number
```

```bash
iptables -L DOCKER-USER -n -v
```

```bash
iptables -t nat -L DOCKER -n -v
```

---

## ufw

```bash
ufw status
```

```bash
ufw status verbose
```

```bash
ufw status numbered
```

```bash
ufw enable
```

```bash
ufw disable
```

```bash
ufw allow 8080/tcp
```

```bash
ufw allow 22/tcp
```

```bash
ufw deny 8080/tcp
```

```bash
ufw delete Rule number
```

---

## DNS

```bash
cat /etc/resolv.conf
```

```bash
cat /etc/hosts
```

```bash
nslookup www.baidu.com
```

```bash
nslookup www.baidu.com 114.114.114.114
```

```bash
dig www.baidu.com
```

```bash
dig +short www.baidu.com
```

```bash
dig @114.114.114.114 www.baidu.com
```

```bash
dig A www.baidu.com
```

```bash
dig CNAME www.example.com
```

```bash
dig MX example.com
```

```bash
dig TXT example.com
```

```bash
getent hosts www.baidu.com
```

---

## System Logs

```bash
tail -n 100 /var/log/messages
```

```bash
tail -f /var/log/messages
```

```bash
tail -n 100 /var/log/syslog
```

```bash
tail -f /var/log/syslog
```

```bash
tail -n 100 /var/log/secure
```

```bash
tail -n 100 /var/log/auth.log
```

```bash
grep CRON /var/log/syslog
```

---

## journalctl

```bash
journalctl
```

```bash
journalctl -n 100
```

```bash
journalctl -f
```

```bash
journalctl -b
```

```bash
journalctl -b -1
```

```bash
journalctl -u Service Name
```

```bash
journalctl -u Service Name -n 100
```

```bash
journalctl -u Service Name -f
```

```bash
journalctl --since today
```

```bash
journalctl --since "1 hour ago"
```

```bash
journalctl --since "2026-04-25 10:00:00" --until "2026-04-25 11:00:00"
```

```bash
journalctl -p err
```

```bash
journalctl -b -p err
```

```bash
journalctl --no-pager
```

---

## dmesg

```bash
dmesg
```

```bash
dmesg | tail
```

```bash
dmesg -T | tail -n 100
```

```bash
dmesg -T | grep -i error
```

```bash
dmesg -T | grep -i fail
```

```bash
dmesg -T | grep -i oom
```

```bash
dmesg -T | grep -i kill
```

```bash
dmesg -T | grep -i xfs
```

```bash
dmesg -T | grep -i ext4
```

```bash
dmesg -T | grep -i nvme
```

```bash
dmesg -T | grep -i scsi
```

---

## wget

```bash
wget URL
```

```bash
wget -O filename.tar.gz URL
```

```bash
wget -c URL
```

```bash
wget -b URL
```

```bash
tail -f wget-log
```

---

## curl

```bash
curl -O URL
```

```bash
curl -o filename.tar.gz URL
```

```bash
curl -L -O URL
```

```bash
curl -L -o filename.tar.gz URL
```

```bash
curl -I URL
```

```bash
curl -v URL
```

```bash
curl -vk https://example.com
```

```bash
curl -m 5 URL
```

---

## scp

```bash
scp Local File Username@FarIP:/Destination Directory/
```

```bash
scp Username@FarIP:/Remote File Local Directory/
```

```bash
scp -r Local Directory Username@FarIP:/Destination Directory/
```

```bash
scp -P 2222 Documentation Username@FarIP:/Destination Directory/
```

---

## rsync

```bash
rsync -avz Local Directory/ Username@FarIP:/Destination Directory/
```

```bash
rsync -avz Username@FarIP:/Remote Directory/ Local Directory/
```

```bash
rsync -avz --progress Local Directory/ Username@FarIP:/Destination Directory/
```

```bash
rsync -avz --delete Local Directory/ Username@FarIP:/Destination Directory/
```

```bash
rsync -avz -e "ssh -p 2222" Local Directory/ Username@FarIP:/Destination Directory/
```

---

## Compression and Decompression

```bash
tar czf logs.tar.gz /var/log/nginx
```

```bash
tar xzf logs.tar.gz
```

```bash
tar xzf logs.tar.gz -C /tmp/
```

```bash
tar tzf logs.tar.gz
```

```bash
zip -r logs.zip /var/log/nginx
```

```bash
unzip logs.zip
```

```bash
unzip logs.zip -d /tmp/logs
```

---

## Temporary Assistance

```bash
pwd
```

```bash
ls -lah
```

```bash
which nginx
```

```bash
curl --help
```

```bash
man systemctl
```

```bash
history
```

```bash
history | grep docker
```

```bash
whoami
```

```bash
id
```

```bash
who
```

```bash
w
```

```bash
date
```

```bash
timedatectl
```

---

## Twenty-Six: One-Sentence Summary

The core of Chapter 07 is:

```text
The firewall decides if the port will come in.

DNS To determine if the domain name can be deciphered

The log decides if there's evidence of a malfunction.

Downloads and transfers determine whether the material can flow quickly.
```

Firewall troubleshooting chain:

```text
ss Watch port listening.

→ firewalld / iptables / ufw Look at the rules.

→ tcpdump See if the request arrives.

→ Combining the security team,ACLWe'll continue to judge.
```

DNS troubleshooting chain:

§§code_383§

Log troubleshooting chain:

```text
systemctl status

→ journalctl -u Service Name

→ /var/log/messages or /var/log/syslog

→ dmesg Look at the kernel.

→ Filter logs by problem time
```

File transfer and auxiliary chain:

```text
wget / curl
→ Download tool or install package

scp / rsync
→ Transfer log and configuration

tar / zip
→ Pack up the material.
```

Production recommendations:

```text
Don't close the firewall for long.
Don't leave it empty. iptables
Don't just do it. ping Watch the network
DNS Look at it at the same time. resolv.confI don't know.hostsI don't know.digI don't know.getent
Check the logs for time frames.
Pay attention to de-sensitization before the transfer log.
rsync --delete It must be used with caution.
```