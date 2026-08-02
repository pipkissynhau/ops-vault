# 07-Firewalls, DNS, Logging, and Download Assistance Commands

# Linux # Operations # Troubleshooting # Firewalls # DNS # System Logs # journalctl # firewalld # iptables # ufw # nslookup # dig # wget # curl # scp # rsync

---

## Recommended Path

01-Linux Basics and Server Operations/01-Server Troubleshooting/07-Firewalls, DNS, Logging, and Download Assistance Commands.md

---

## I. Document Overview

This article compiles common **firewall, DNS, system logging, and download/file transfer assistance commands** used in Linux server troubleshooting.

Key topics include:

- Firewalld troubleshooting
- iptables troubleshooting
- ufw troubleshooting
- Opening firewall ports
- Viewing firewall rules
- DNS resolution troubleshooting
- `/etc/resolv.conf`
- `nslookup`
- `dig`
- `getent hosts`
- System log viewing
- `/var/log/messages`
- `/var/log/syslog`
- `journalctl`
- `dmesg`
- File downloading
- `wget`
- `curl`
- File transfer
- `scp`
- `rsync`
- Compression and decompression
- Temporary troubleshooting aids

Objectives:

- Determine if a port is blocked by the local firewall.
- Identify DNS resolution issues.
- View system and service logs.
- Download troubleshooting tools or installation packages.
- Transfer logs, configurations, and backup files between servers.
- Perform basic compression, decompression, and temporary troubleshooting tasks.

---

## II. This Article's Focus

Previous articles have covered:

```text
01
→ General Server Troubleshooting and System Resource Inspection

02
→ CPU, Memory, and Load Monitoring

03
→ Disk Space, Inodes, and Disk I/O Analysis

04
→ Disk Partitioning, Mounting, and LVM Expansion

05
→ Process and systemd Service Investigation

06
→ Network Connectivity, Ports, Routing, and Traffic Analysis
```

This article adds supplementary capabilities for server troubleshooting:

```text
Firewalls
→ Why is a port listening but not accessible externally?

DNS
→ Why can't a domain name be resolved or resolves to the wrong IP?

Logging
→ Why are services failing, system errors occurring, or kernel messages appearing?

Downloading and Transfer
→ How to quickly retrieve tools, upload logs, and synchronize files.

Compression and Decompression
→ How to package logs, back up configurations, and transfer troubleshooting materials.
```

---

## III. General Firewall Troubleshooting Approach

When encountering issues such as:

```text
A service port is listening locally.
Local curl requests work fine.
External access is blocked.
Ping may succeed, but the port is unreachable.
NC connections time out.
Some machines in the same network segment can access it, while others cannot,
```

Consider factors like firewalls, security groups, ACLs, and network policies.

Common local firewall tools include:

```text
firewalld
iptables
ufw
```

Recommended troubleshooting order:

```text
Confirm if the service is listening on the port.
→ Verify if the listening address is correct.
→ Check the local firewall settings.
→ Review cloud security groups/network ACLs.
→ Conduct NC/curl tests from the remote end.
→ Use tcpdump to confirm whether requests reach the server.
```

---

## IV. Firewalld Troubleshooting

---

## Scenario 1: Checking Firewalld Status

### Command

```bash
systemctl status firewalld
```

or:

```bash
firewall-cmd --state
```

### Possible Results

```text
running
→ Firewalld is currently running.

not running
→ Firewalld is not running.
```

---

## Scenario 2: Checking the Current Default Zone

### Command

```bash
firewall-cmd --get-default-zone
```

### Explanation

Firewalld manages rules using zones.

Common zones include:

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

## Scenario 3: Checking Currently Active Zones

### Command

```bash
firewall-cmd --get-active-zones
```

### Purpose

To see which network interfaces are associated with each zone.

---

## Scenario 4: Viewing Rules for a Specific Zone

### Command

```bash
firewall-cmd --list-all
```

To view rules for a specific zone:

```bash
firewall-cmd --zone=public --list-all
```

### Key Areas to Focus On

```text
services
→ Allowed services.
ports
→ Allowed ports.
interfaces
→ Network interfaces bound to the zone.
sources
→ Source address rules.
```

---

## Scenario 5: Listing Allowed Ports

### Command

```bash
firewall-cmd --list-ports
```

To view rules for a specific zone:

```bash
firewall-cmd --Do not execute it casually in a production environment.
```

---

## Scenario 18: The Relationship between iptables and Docker

Docker maintains iptables rules.

Common chains:

```text
DOCKER
DOCKER-USER
DOCKER-ISOLATION-STAGE-1
DOCKER-ISOLATION-STAGE-2
```

Viewing:

```bash
iptables -L DOCKER-USER -n -v
```

```bash
iptables -t nat -L DOCKER -n -v
```

Explanation:

```text
Docker port mapping relies on iptables NAT rules.

Clearing iptables may cause Docker network issues.
```

---

## VI. Ufw Troubleshooting

---

## Scenario 19: Checking the Ufw Status

### Command

```bash
ufw status
```

For detailed viewing:

```bash
ufw status verbose
```

Viewing with numbered results:

```bash
ufw status numbered
```

### Explanation

This is common on Ubuntu systems.

---

## Scenario 20: Enabling and Disabling ufw

### Enabling

```bash
ufw enable
```

### Disabling

```bash
ufw disable
```

Note in production:

```text
In a remote SSH environment, you must allow the SSH port before enabling ufw.
Otherwise, you may lock yourself out of the machine.
```

---

## Scenario 21: Allowing Ports

### Command

Allowing port 8080:

```bash
ufw allow 8080/tcp
```

Allowing HTTP:

```bash
ufw allow 80/tcp
```

Allowing HTTPS:

```bash
ufw allow 443/tcp
```

Allowing SSH:

```bash
ufw allow 22/tcp
```

---

## Scenario 22: Denying Ports

### Command

```bash
ufw deny 8080/tcp
```

---

## Scenario 23: Deleting Rules

First, view the numbered results:

```bash
ufw status numbered
```

Delete a specific numbered rule:

```bash
ufw delete rule number
```

Example:

```bash
ufw delete 3
```

---

## VII. Typical Firewall Troubleshooting Scenarios

---

## Scenario 24: The Port Is Listening, but External Access is Blocked

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
ss monitors 127.0.0.1:8080
→ External access is blocked; the service only listens on this machine.

ss monitors 0.0.0.0:8080, but the firewall does not allow it
→ The port needs to be allowed.

tcpdump does not show any requests
→ It may be intercepted by an upstream firewall or security group.

tcpdump shows requests but there is no response
→ Check the local firewall, service logs, and application status.
```

---

## Scenario 25: Temporarily Disabling the Firewall for Verification

### firewalld

```bash
systemctl stop firewalld
```

### ufw

```bash
ufw disable
```

### Explanation

This should only be used as a temporary verification method.

Disabling the firewall for extended periods is not recommended in a production environment.

After verification, restore it:

```bash
systemctl start firewalld
```

or:

```bash
ufw enable
```

---

## VIII. General DNS Troubleshooting Approach

Common DNS issues include:

```text
The IP address can be reached via ping, but the domain name cannot be resolved.

curl fails to resolve the domain name, but succeeds with the IP address.

Applications experience timeout when connecting to domain names.

The wrong IP address is resolved for the domain name.

Different machines give different resolution results.

Domain names cannot be resolved within containers.

DNS is abnormal within Kubernetes Pods.

Accessing internal domain names fails.
```

Troubleshooting order:

```text
Confirm if the IP address is reachable.

→ Check /etc/resolv.conf.

→ Use nslookup or dig for queries.

→ Use getent hosts to check the system's resolution results.

→ Test with specific DNS servers.

→ Compare resolution results across different machines.

→ Verify DNS cache, DNS services, and network settings.
```

---

## IX. /etc/resolv.conf

---

## Scenario 26: Checking DNS Configuration

### Command

```bash
cat /etc/resolv.conf
```

### Common Contents

```text
nameserver 114.114.114.Machine B:

```bash
dig +short domain name
```

Specify the same DNS:

```bash
dig @10.0.0.2 domain name
```

View local configuration:

```bash
cat /etc/resolv.conf
```

```bash
cat /etc/hosts
```

### Possible Reasons

```text
Using different DNS servers

Inconsistent /etc/hosts files

DNS cache not refreshed

Different internal network DNS views

Load balancing DNS returns different IPs

CDN intelligent resolution
```

---

## Scenario 42: DNS Errors within Containers

### Troubleshooting Commands

Enter the container:

```bash
docker exec -it containerID /bin/sh
```

View DNS settings:

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

View on the host machine:

```bash
cat /etc/resolv.conf
```

Docker DNS configuration:

```bash
cat /etc/docker/daemon.json
```

Configuration may be needed:

```json
{
  "dns": ["114.114.114.114", "8.8.8.8"]
}
```

Restart Docker:

```bash
systemctl restart docker
```

Note:

```text
After changing Docker DNS settings, you usually need to recreate the container for the changes to take effect.
```

---

## Section 14: System Log Troubleshooting

---

## Scenario 43: Common System Log Paths

Log paths vary across different distributions.

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

Systemd systems commonly use:

```text
journalctl
```

---

## Scenario 44: Viewing messages

### Commands

```bash
tail -n 100 /var/log/messages
```

Real-time viewing:

```bash
tail -f /var/log/messages
```

Search for errors:

```bash
grep -i error /var/logmessages
```

```bash
grep -i fail /var/log/messages
```

---

## Scenario 45: Viewing syslog

### Commands

```bash
tail -n 100 /var/log/syslog
```

Real-time viewing:

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

## Scenario 46: Viewing authentication logs

### For RHEL systems:

```bash
tail -n 100 /var/log/secure
```

### For Ubuntu / Debian:

```bash
tail -n 100 /var/log/auth.log
```

Suitable for troubleshooting:

```text
SSH login failures

sudo permission issues

User authentication failures

Traces of brute force attacks

PAM authentication problems
```

---

## Scenario 47: Viewing cron logs

### For RHEL systems:

```bash
tail -n 100 /var/log/cron
```

### For Ubuntu / Debian:

Cron logs are typically found in syslog on Ubuntu:

```bash
grep CRON /var/log/syslog
```

Suitable for troubleshooting:

```text
Whether scheduled tasks are being executed

If execution times match expectations

If tasks encounter errors

If any environment variables are missing
```

---

## Section 15: journalctl Log Troubleshooting

---

## Scenario 48: Viewing system logs

### Command

```bash
journalctl
```

View the last 100 lines:

```bash
journalctl -n 100
```

Real-time viewing:

```bash
journalctl -f
```

---

## Scenario 49: Viewing logs since startup

### Command

```bash
journalctl -b
```

View the last 100 lines:

```bash
journalctl -b -n 100
```

---

## Scenario 50: Viewing logs from the previous startup

### Command

```bash
journalctl -b -1
```

Suitable for troubleshooting:

```text
What happened before the system last restarted

Why a service failed to start previously

If there were any kernel errors before an abnormal restart
```

---

## Scenario 51: Viewing logs for a specific service

### Command

```bash
journalctl -u service_name
```

Example:

```bash
journalctl -u nginx
```

View the last 100 lines:

```bash
journalctl -u nginx -nscp root@10.0.0.10:/var/log/messages /tmp/
firewall-cmd --list-services
→ Download tools or installation packages

scp / rsync
→ Transfer logs and configuration files

tar / zip
→ Package troubleshooting materials
```
