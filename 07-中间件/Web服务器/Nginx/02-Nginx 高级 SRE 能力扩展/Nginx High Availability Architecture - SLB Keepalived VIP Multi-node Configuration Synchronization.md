# 13-Nginx High Availability Architecture: SLB, Keepalived, VIP, Multi-Node, and Configuration Synchronization

#Nginx #HighAvailable #SLB #Keepalived #VIP #VRRP #LoadBalance #ConfigureSync #CertificateSynchronization #SRE #AccessLayerStructure

---

## Recommended Path

07-Middlewares/Web Server/Nginx/02-Nginx Advanced SRE Capabilities Expansion/13-Nginx High Availability Architecture: SLB, Keepalived, VIP, Multi-Node, and Configuration Synchronization.md

---

## I. Document Description

This document organizes the design and production operation methods for high availability architecture of Nginx access layer.

Previously, we have organized:

```text
Nginx Configure Foundation

Reverse Agent

upstream Load Balance

Production agent parameters

HTTPS Certificate

Real IP

JSON Log

Common failure screening

Performance capacity analysis

Flow and connection control

upstream High-level governance
```

This article further focuses on:

- Why single Nginx is a single point
- Common high availability architecture models for Nginx
- SLB / CLB / ELB + multiple Nginx instances
- Keepalived + VIP dual-node high availability
- VRRP basic concepts
- Master-slave mode
- Master-master / active-active mode
- Nginx health check
- Keepalived detecting Nginx availability
- VIP drift verification
- Differences between cloud and physical machine environments
- Multi-node configuration synchronization
- Multi-node certificate synchronization
- Multi-node log collection
- Multi-node change release process
- Common high availability architecture failures
- Production considerations

This article is the 13th in the series of Nginx Advanced SRE Capabilities Expansion.

This article's goals:

```text
I understand. Nginx Single-point risk

→ It makes a difference. SLB + Nginx and Keepalived + VIP Structure

→ I can read it. Keepalived Basic Configuration

→ Configure Nginx Health check script

→ Authentication VIP Floating

→ Understands the importance of multinode configuration synchronization and certificate synchronization

→ I can check. Nginx Common issues in high-availability structures

→ It forms a high-access-level design idea.
```

---

## II. Why Nginx Requires High Availability

If only one Nginx instance is used:

```text
User

→ Nginx Single Nodes

→ Backend Services
```

When this Nginx instance fails:

```text
Nginx Process hung up.

Server crashes

System restart

Cybercard anomaly.

Disk Full

Configure reload Failed

Certificate Error

Security team's wrong.

The machine room network is abnormal.
```

It may lead to:

```text
The entire service entrance is not available

All users cannot access

Backend service is normal but not external.

Increased range of operational failures

Recovery depends on manual repairs.
```

Therefore, in production, the Nginx entry layer should not have only a single instance.

One-sentence understanding:

```text
Just... Nginx It is an entry point for business, and it cannot exist as a single point for long.
```

---

## III. Common High Availability Architectures for Nginx

There are three common architecture types:

```text
Cloud Load Balance SLB / CLB / ELB + Multiple Nginx

Keepalived + VIP + Multiple Nginx

DNS / CDN / GSLB + Multiple entrances Nginx
```

---

## 1. SLB / CLB / ELB + Multiple Nginx Instances

Typical flow:

```text
User

→ Cloud Load Balance SLB / CLB / ELB

→ Nginx-1

→ Nginx-2

→ Nginx-3

→ Backend Services
```

Features:

```text
The cloud manufacturer has a load balance for the entrance.

Multiple Nginx As a back-end instance

SLB Periodic health checks Nginx

Unusual Nginx Automatic Node Removal

Internet portal by SLB Provision
```

Suitable for:

```text
Public cloud environment

Host load balance available

I hope to reduce the complexity of self-building. degrees

Multiple Nginx Horizontal extension

We need a public connection.
```

---

## 2. Keepalived + VIP + Nginx

Typical flow:

```text
User

→ VIP

→ Current Master Nginx

→ Backend Services
```

Nodes:

```text
Nginx-1 + Keepalived

Nginx-2 + Keepalived
```

The VIP will drift to one of the machines.

When Master fails:

```text
VIP From Nginx-1 Float To Nginx-2
```

Suitable for:

```text
Physical environment

Private cloud environment

No cloud load balance

Internal portals are highly accessible

Traditional IDC Environment

Self-build gateway entrance
```

---

## 3. DNS / CDN / GSLB + Multiple Entry Nginx

Typical flow:

```text
User

→ DNS / CDN / GSLB

→ Multi-geographical entrances Nginx

→ Backend Services
```

Suitable for:

```text
Multi-geographical operations

Overflight rooms are highly usable

Accelerating global visits

Disaster preparedness switch

Multiple entrances to the public network
```

Note:

```text
DNS Toggle Cache and TTL Delay

CDN / GSLB Reliance on manufacturer capacity

Configuration and certificate consistency are more important.
```

---

## IV. Comparison of Different Architectures

| Architecture | Suitable Scenario | Advantages | Risks or Limitations |
|---|---|---|---|
| SLB + Multiple Nginx | Public Cloud | Simple, stable, mature health check | Relies on cloud vendor load balancer |
| Keepalived + VIP | Physical Machine / Private Cloud | Self-controlled, not dependent on cloud LB | Needs to handle VIP, network, brain split |
| DNS Multiple A Records | Simple Multi-Entry | Easy to implement | Slow failover, client cache uncontrollable |
| CDN / GSLB + Nginx | Public Internet, Multi-Region | Can do global scheduling and acceleration | Higher cost and configuration complexity |
| Single Nginx | Testing or Small Business | Simple | Obvious single point, unsuitable for core production |

---

## V. SLB + Multiple Nginx Architecture

---

## Scenario 1: Basic Architecture

```text
User

→ SLB Internet portal

→ Nginx-1:10.0.0.21

→ Nginx-2:10.0.0.22

→ Nginx-3:10.0.0.23

→ Backend Application
```

SLB Listener:

```text
80

443
```

Nginx Backend Port:

```text
80

443
```

---

## Scenario 2: SLB Health Check

SLB typically configures health checks:

```text
Check protocol:HTTP

Check port:80

Check path:/health

Health status code:200

Health check-ups:5s or 10s

Failure threshold:2 or 3 Minor
```

Nginx health check path configuration:

```nginx
server {
    listen 80;
    server_name example.com;

    location = /health {
        access_log off;
        return 200 "ok\n";
    }

    location / {
        proxy_pass http://app_backend;
    }
}
```

Verification:

```bash
curl -i http://127.0.0.1/health
```

Expected:

```text
HTTP/1.1 200 OK

ok
```

---

## Scenario 3: SLB Removing Abnormal Nginx Nodes

If a Nginx instance fails:

```text
Nginx Process Stop

Health screening /health Failed

SLB Mark this node unhealthy

SLB Other Organiser
```

Verify local Nginx health check:

```bash
curl -i http://127.0.0.1/health
```

Check Nginx status:

```bash
systemctl status nginx
```

Check port:

```bash
ss -lntp | grep ':80'
```

---

## Scenario 4: SLB + Nginx Architecture Considerations

Need to unify:

```text
All Nginx Node Configuration Consistent

All Nginx Node Certificate Same

All Nginx The node health check path is consistent.

All Nginx Nodal log collection is consistent

All Nginx Nodes real_ip Configure Same

All Nginx Node security policy is consistent.
```

Otherwise, it may cause:

```text
Some requests are normal, others are abnormal.

Some node certificates expired

Some node configuration not updated

Some white lists don't match.

Some nodes are real. IP Wrong.

Some node log fields are inconsistent
```

---

## VI. Keepalived + VIP Architecture Basics

---

## Scenario 5: What is Keepalived

Keepalived is commonly used for:

```text
Achieved VIP Floating

Achieving high availability

Check for service.

Lower priority or release in case of service anomalies VIP
```

Common protocols:

```text
VRRP
```

VRRP can be understood as:

```text
There's another machine competing for a virtual. IP

Priority high Master

Master Hold VIP

Backup Listen Master Status

Master After the anomaly. Backup Take over. VIP
```

---

## Scenario 6: Keepalived + Nginx Dual-Node Model

Nodes:

```text
Nginx-1:10.0.0.21

Nginx-2:10.0.0.22

VIP:10.0.0.100
```

Flow:

```text
User

→ VIP 10.0.0.100

→ Current Master Nodes

→ Nginx

→ Backend Services
```

Normal condition:

```text
Nginx-1 Hold VIP

Nginx-2 As Backup
```

Fault condition:

```text
Nginx-1 Fault

Nginx-2 Take over. VIP

User visits continue 10.0.0.100
```

---

## VII. Installing Keepalived

---

## Scenario 7: Ubuntu / Debian Installation

```bash
apt update
```

```bash
apt install -y keepalived
```

Check version:

```bash
keepalived --version
```

---

## Scenario 8: RHEL / CentOS / Rocky / AlmaLinux Installation

```bash
yum install -y keepalived
```

Or:

```bash
dnf install -y keepalived
```

Check version:

```bash
keepalived --version
```

---

## Scenario 9: Check Service Status

```bash
systemctl status keepalived
```

Start:

```bash
systemctl start keepalived
```

Enable on boot:

```bash
systemctl enable keepalived
```

---

## VIII. Keepalived Master-Slave Configuration Example

---

## Scenario 10: Master Node Configuration

Node information:

```text
Master:10.0.0.21

Backup:10.0.0.22

VIP:10.0.0.100

Nic:eth0
```

Configuration file:

```bash
vi /etc/keepalived/keepalived.conf
```

Master configuration:

```conf
global_defs {
    router_id nginx_master
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass nginx_ha
    }

    virtual_ipaddress {
        10.0.0.100/24
    }
}
```

---

## Scenario 11: Backup Node Configuration

Backup configuration:

```conf
global_defs {
    router_id nginx_backup
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 51
    priority 90
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass nginx_ha
    }

    virtual_ipaddress {
        10.0.0.100/24
    }
}
```

---

## Scenario 12: Key Parameter Explanation

```text
router_id
→ Current Node Identification

state
→ Initial state,MASTER or BACKUP

interface
→ VIP A bound card.

virtual_router_id
→ VRRP Group IDThe same set must be the same.

priority
→ The greater the value, the higher the priority. Master

advert_int
→ VRRP Notification interval

auth_type / auth_pass
→ Master authentication information

virtual_ipaddress
→ To drift. VIP
```

Note:

```text
Group I Keepalived Yes. virtual_router_id It has to be consistent.

Different business or different VIP Yes. virtual_router_id No conflict.

auth_pass The master must be consistent.

interface You have to write a real card name.
```

Check network interface name:

```bash
ip addr
```

---

## IX. Start Keepalived and Verify VIP

---

## Scenario 13: Start Service

Both nodes execute:

```bash
systemctl enable keepalived
```

```bash
systemctl start keepalived
```

Check status:

```bash
systemctl status keepalived
```

---

## Scenario 14: Check if VIP is Bound

On Master node:

```bash
ip addr show eth0
```

Or:

```bash
ip a | grep 10.0.0.100
```

Normally, VIP should appear on the Master node.

On Backup node:

```bash
ip a | grep 10.0.0.100
```

Normally, Backup node should not hold VIP.

---

## Scenario 15: Access Nginx via VIP

```bash
curl -I http://10.0.0.100
```

If a domain is configured:

```bash
curl -I -H "Host: example.com" http://10.0.0.100
```

---

## Scenario 16: View Keepalived Logs

Ubuntu / systemd:

```bash
journalctl -u keepalived -n 100
```

Real-time viewing:

```bash
journalctl -u keepalived -f
```

If the system writes to messages:

```bash
tail -f /var/log/messages
```

---

## Ten. Keepalived Detects Nginx Health

Simply detecting host availability is insufficient.

You also need to detect:

```text
Nginx Existence of the process

Nginx Port listening

Nginx Is the health check path normal?
```

Otherwise, it may result in:

```text
Keepalived Normal

VIP Still Master

But... Nginx It's dead.

User access VIP Failed
```

---

## Scenario 17: Write Nginx Detection Script

Script path:

```bash
vi /etc/keepalived/check_nginx.sh
```

Script content:

```bash
#!/bin/bash

curl -fsS --max-time 2 http://127.0.0.1/health >/dev/null 2>&1

if [ $? -eq 0 ]; then
    exit 0
else
    exit 1
fi
```

Permissions:

```bash
chmod +x /etc/keepalived/check_nginx.sh
```

Test the script:

```bash
/etc/keepalived/check_nginx.sh
```

Check return code:

```bash
echo $?
```

Return:

```text
0
→ Health

Not 0
→ Not healthy.
```

---

## Scenario 18: Keepalived Integrates Health Check Script

Master configuration example:

```conf
global_defs {
    router_id nginx_master
}

vrrp_script check_nginx {
    script "/etc/keepalived/check_nginx.sh"
    interval 2
    timeout 2
    fall 2
    rise 2
    weight -30
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass nginx_ha
    }

    virtual_ipaddress {
        10.0.0.100/24
    }

    track_script {
        check_nginx
    }
}
```

Backup configuration is similar, but:

```conf
state BACKUP
priority 90
```

---

## Scenario 19: Health Check Parameter Explanation

```text
script
→ Check script path

interval
→ Do it every second

timeout
→ Script timeout

fall
→ How many times do you think you failed?

rise
→ How many times in a row did they think they'd recovered?

weight
→ Adjust priority when check failed
```

Example:

```text
Master priority = 100

weight = -30

When check failed, priority becomes 70

Backup priority = 90

At this point, Backup I'll take over. VIP
```

---

## Eleven. VIP Drift Test

---

## Scenario 20: Stop Nginx on Master

Execute on Master node:

```bash
systemctl stop nginx
```

Observe Keepalived logs:

```bash
journalctl -u keepalived -f
```

Check VIP on Backup node:

```bash
ip a | grep 10.0.0.100
```

Verify access:

```bash
curl -I http://10.0.0.100
```

---

## Scenario 21: Restore Nginx on Master

```bash
systemctl start nginx
```

Check health status:

```bash
curl -i http://127.0.0.1/health
```

Observe if VIP is switched back:

```bash
ip a | grep 10.0.0.100
```

Note:

```text
Whether or not to automatically switch depends Keepalived Priority and capture configuration

Whether automatic retracting is allowed in production requires careful design
```

---

## Scenario 22: Stop Keepalived on Master

```bash
systemctl stop keepalived
```

Observe if Backup takes over VIP:

```bash
ip a | grep 10.0.0.100
```

Restore:

```bash
systemctl start keepalived
```

---

## Twelve. Preemption and Non-Preemption

---

## Scenario 23: What is Preemption

By default, if the original Master recovers and has higher priority, it may reclaim VIP.

This is called:

```text
Take it.
```

Advantages:

```text
Back to the original structure when the main node is restored
```

Risks:

```text
VIP Drifting round and round

Connection Interrupted

The failure nodes have just returned to instability and re-routing.

Short-term business shaking
```

---

## Scenario 24: Non-Preemption nopreempt

You can configure:

```conf
nopreempt
```

Example:

```conf
vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    nopreempt

    authentication {
        auth_type PASS
        auth_pass nginx_ha
    }

    virtual_ipaddress {
        10.0.0.100/24
    }
}
```

Explanation:

```text
nopreempt To avoid recovery of high-priority nodes VIP
```

Production recommendation:

```text
Core production portals can be considered for non-jacking.

After avoidance of breakdown VIP Frequent Switches

It's best if you have a manual confirmation.
```

---

## Thirteen. Keepalived Brain Split Issue

---

## Scenario 25: What is Brain Split

Brain split refers to:

```text
They both think they are. Master

Both of them. VIP
```

Risks:

```text
ARP Conflict

Flows are drifting abnormally.

Request random calls to different nodes

The connection is unstable.

Log Chaos

Business visits unusual
```

---

## Scenario 26: Common Causes of Brain Split

Common causes:

```text
In between. VRRP Communications are blocked.

Firewall interdiction VRRP

Switches or cloud networks do not support grouping

virtual_router_id Conflict

auth_pass Inconsistencies

interface Wrong match.

Network vibrating badly.

Security strategy prevents protocol. 112
```

VRRP protocol number:

```text
112
```

---

## Scenario 27: Brain Split Troubleshooting Commands

Check if both nodes hold VIP:

```bash
ip a | grep 10.0.0.100
```

Check Keepalived logs:

```bash
journalctl -u keepalived -n 100
```

Capture VRRP packets:

```bash
tcpdump -i eth0 vrrp -nn
```

If tcpdump doesn't support vrrp filtering, you can use:

```bash
tcpdump -i eth0 proto 112 -nn
```

Check firewall:

```bash
iptables -L -n
```

Or:

```bash
firewall-cmd --list-all
```

---

## Fourteen. Limitations of Keepalived in Cloud Environments

---

## Scenario 28: Cloud Environments May Not Support VIP Drift

In public clouds, Keepalived + VIP may be restricted.

Reason:

```text
The cloud factory network is closed.

Do Not Support Any ARP Radio

Custom is not allowed VIP Floating

Security unit or route unrecognized VIP

Organisation VRRP Not Available

Virtual Web Match ARP Limited
```

Therefore, in cloud environments, prioritize:

```text
Cloud Load Balance SLB / CLB / ELB

The cloud factory is very useful. VIP Products

Flexnet card drift

Private IP Tie Switch

Clouds API Autotoggle
```

---

## Scenario 29: Recommended Architecture in the Cloud

Recommended:

```text
Internet users

→ Cloud Load Balance SLB / CLB / ELB

→ Multiple Nginx

→ Backend Services
```

Advantages:

```text
Clouds LB Self-capable

Health check matures

Automatic removal of abnormal nodes

The public entrance is stable.

You don't have to deal with it yourself. ARP / VRRP / VIP
```

---

## Fifteen. Configuration Synchronization for Multi-Node Nginx

---

## Scenario 30: Why Configuration Synchronization is Important

If multiple Nginx instances have inconsistent configurations, it may lead to:

```text
Partial request normal.

Partial requests 404

Partial requests 502

Partial Certificate Error

Part of the node limit policy is different

Part of the white list is different.

Some nodes have different log formats

Some nodes are real IP Parsing Different
```

This issue is very hard to troubleshoot because users may feel:

```text
Sometimes good, sometimes bad.
```

The root cause may be:

```text
Different. Nginx Node configuration inconsistent
```

---

## Scenario 31: Configuration Synchronization Methods

Common methods:

```text
Manual Sync

rsync Sync

Git Management + Pull

Ansible Batch Release

CI/CD Release Nginx Configure

Configure Center Out

Mirrorization Nginx Configure
```

Production recommendation:

```text
Configure Entry Git

Change review

Auto Execute nginx -t

Batch to Multinodes

Support rapid rollback
```

---

## Scenario 32: rsync Configuration Synchronization Example

Sync from management node to Nginx node:

```bash
rsync -avz /etc/nginx/ root@10.0.0.21:/etc/nginx/
```

```bash
rsync -avz /etc/nginx/ root@10.0.0.22:/etc/nginx/
```

Note:

```text
rsync Make sure you do not overwrite the wrong file

Note private key privileges when synchronizing certificates

We'll need every one of them. nginx -t

Do not sync temporary backup file overlay production configuration
```

---

## Scenario 33: Batch Check Nginx Configuration

```bash
for host in 10.0.0.21 10.0.0.22; do
    echo "===== $host ====="
    ssh root@$host "nginx -t"
done
```

---

## Scenario 34: Batch Reload Nginx

```bash
for host in 10.0.0.21 10.0.0.22; do
    echo "===== $host ====="
    ssh root@$host "systemctl reload nginx"
done
```

---

## Scenario 35: Batch View Configuration Summary

```bash
for host in 10.0.0.21 10.0.0.22; do
    echo "===== $host ====="
    ssh root@$host "nginx -T 2>/dev/null | grep -n 'server_name example.com' -A 20"
done
```

---

## Sixteen. Certificate Synchronization for Multi-Node

---

## Scenario 36: Why Certificate Synchronization is Important

If multiple Nginx instances have inconsistent certificates, it may result in:

```text
Partial Request Certificate Normal

Partially requested certificate expiry

Partially requested certificate domain name mismatch

Partial node certificate chain incomplete

Some users occasionally HTTPS Wrong.
```

If there's an SLB in front routing to different Nginx nodes, user experience will be very unstable.

---

## Scenario 37: Batch Check Certificate Expiry

```bash
for host in 10.0.0.21 10.0.0.22; do
    echo "===== $host ====="
    ssh root@$host "openssl x509 -in /etc/nginx/certs/example.com/fullchain.pem -noout -dates"
done
```

---

## Scenario 38: Batch Check Certificate SAN

```bash
for host in 10.0.0.21 10.0.0.22; do
    echo "===== $host ====="
    ssh root@$host "openssl x509 -in /etc/nginx/certs/example.com/fullchain.pem -noout -text | grep -A 2 'Subject Alternative Name'"
done
```

---

## Scenario 39: Batch Synchronize Certificates

```bash
for host in 10.0.0.21 10.0.0.22; do
    echo "===== sync cert to $host ====="
    rsync -avz /etc/nginx/certs/example.com/ root@$host:/etc/nginx/certs/example.com/
done
```

Set permissions:

```bash
for host in 10.0.0.21 10.0.0.22; do
    echo "===== chmod cert on $host ====="
    ssh root@$host "chmod 644 /etc/nginx/certs/example.com/fullchain.pem && chmod 600 /etc/nginx/certs/example.com/privkey.pem"
done
```

Check and reload:

```bash
for host in 10.0.0.21 10.0.0.22; do
    echo "===== reload nginx on $host ====="
    ssh root@$host "nginx -t && systemctl reload nginx"
done
```

---

## Seventeen. Log Collection for Multi-Node

---

## Scenario 40: Why Log Collection Should Be Unified

Multiple Nginx nodes must have unified log collection, otherwise:

```text
I can only see some requests.

Unable to measure global QPS

Unable to measure global 5xx

Could not close temporary folder: %s

It's impossible to judge if a node is abnormal.

The failure check is incomplete.
```

It's recommended to have all Nginx instances output to:

```text
Same log_format

Same Field Name

Same Time Format

Same log path rule

Additional Node Identification hostname
```

---

## Scenario 41: Add hostname to JSON Logs

You can add:

```nginx
"hostname":"$hostname"
```

Example:

```nginx
log_format access_json escape=json
    '{'
    '"time":"$time_iso8601",'
    '"hostname":"$hostname",'
    '"remote_addr":"$remote_addr",'
    '"host":"$host",'
    '"method":"$request_method",'
    '"uri":"$uri",'
    '"status":$status,'
    '"request_time":"$request_time",'
    '"upstream_addr":"$upstream_addr",'
    '"upstream_response_time":"$upstream_response_time"'
    '}';
```

Purpose:

```text
The log platform allows a distinction to be made between which one is requested Nginx Nodes
```

---

## Scenario 42: Aggregate 5xx by Node

If JSON logs include hostname:

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status >= 500) | .hostname' | sort | uniq -c | sort -nr
```

You can aggregate analysis by:

```text
hostname

status

uri

upstream_addr
```

---

## Eighteen. Multi-Node Change Deployment Process

---

## Scenario 43: Recommended Change Process

Multi-node Nginx change recommendations:

```text
Configure Entry Git

→ Change review

→ Test Environment nginx -t

→ Select One Nginx Greyscale Release

→ nginx -t

→ reload

→ curl Authentication

→ Observation access.log / error.log

→ Confirm no anomaly.

→ Batch release of remaining nodes

→ Check configuration version for all nodes

→ Record changes
```

---

## Scenario 44: Gray-scale One Nginx First

Assuming two nodes:

```text
Nginx-1:10.0.0.21

Nginx-2:10.0.0.22
```

First deploy on Nginx-1:

```bash
ssh root@10.0.0.21 "nginx -t && systemctl reload nginx"
```

Specify IP verification:

```bash
curl -I --resolve example.com:80:10.0.0.21 http://example.com
```

HTTPS verification:

```bash
curl -I --resolve example.com:443:10.0.0.21 https://example.com
```

Observe logs:

```bash
ssh root@10.0.0.21 "tail -n 100 /var/log/nginx/error.log"
```

---

## Scenario 45: Batch Deploy Remaining Nodes

```bash
for host in 10.0.0.22 10.0.0.23; do
    echo "===== deploy $host ====="
    rsync -avz /etc/nginx/ root@$host:/etc/nginx/
    ssh root@$host "nginx -t && systemctl reload nginx"
done
```

---

## Scenario 46: Batch Verify All Nodes

```bash
for host in 10.0.0.21 10.0.0.22 10.0.0.23; do
    echo "===== check $host ====="
    curl -I --resolve example.com:80:$host http://example.com
done
```

HTTPS: /think

```bash
for host in 10.0.0.21 10.0.0.22 10.0.0.23; do
    echo "===== check $host ====="
    curl -I --resolve example.com:443:$host https://example.com
done
```

---

## 19. Common Nginx High Availability Faults

---

## Scenario 47: VIP Not Bound

Troubleshooting:

```bash
systemctl status keepalived
```

```bash
journalctl -u keepalived -n 100
```

```bash
ip a | grep 10.0.0.100
```

Common Causes:

```text
Keepalived Not started.

Profile Syntax Error

interface Wrong

VIP Network error

priority Configure Abnormal

VRRP Communications anomaly

Script failure leads to lower priority
```

---

## Scenario 48: Access Failure After VIP Drift

Troubleshooting:

```bash
ip a | grep 10.0.0.100
```

```bash
curl -I http://127.0.0.1/health
```

```bash
curl -I http://10.0.0.100/health
```

```bash
ss -lntp | grep ':80'
```

Common Causes:

```text
Backup Nodes Nginx Not started.

Backup Node Configuration Incomplete

Backup Node firewall not released

VIP Floating but ARP Cache not updated

Nginx Not listening to port

Health check path is abnormal.
```

---

## Scenario 49: Both Machines Have VIP

This typically indicates a split-brain risk.

Troubleshooting:

```bash
ip a | grep 10.0.0.100
```

```bash
tcpdump -i eth0 proto 112 -nn
```

```bash
journalctl -u keepalived -n 100
```

Check:

```text
virtual_router_id Consistency

auth_pass Consistency

interface Correct?

Is the firewall clear? VRRP

Network support VRRP

Is there a single one? VRID Other Keepalived
```

---

## Scenario 50: Partial Request Abnormalities After SLB

Common Causes:

```text
Some station. Nginx Configuration Inconsistent

Some station. Nginx Certificate Expiry

Some station. Nginx Backend's not working.

Some station. Nginx real_ip Configure Error

Some station. Nginx Log format is different

Some station. Nginx Nope. reload

SLB Inaccuracy of health check-ups
```

Troubleshoot Each Node:

```bash
for host in 10.0.0.21 10.0.0.22 10.0.0.23; do
    echo "===== $host ====="
    curl -I --resolve example.com:80:$host http://example.com
done
```

Check Nginx Status on Each Machine:

```bash
for host in 10.0.0.21 10.0.0.22 10.0.0.23; do
    echo "===== $host ====="
    ssh root@$host "systemctl status nginx --no-pager | head -n 20"
done
```

---

## Scenario 51: Nginx Not Receiving Traffic

Common Causes:

```text
SLB Health check failure

Node by SLB Remove

Nginx Local /health Failed

Security team's blocking. SLB Present. Nginx

Nginx No listening port

Weight 0

Node is not here. SLB Back-end pool
```

Troubleshooting:

```bash
curl -i http://10.0.0.21/health
```

```bash
ss -lntp | grep ':80'
```

```bash
tail -n 100 /var/log/nginx/access.log
```

---

## 20. Production Notes

---

## 1. Prioritize Cloud Load Balancer in Cloud Environments

In public clouds, it's typically preferred to use:

```text
SLB

CLB

ELB

ALB

NLB
```

Rather than forcibly building Keepalived + VIP.

Reason:

```text
The cloud network doesn't necessarily support the second layer. VIP

Cloud load balance self-contained high

Health check-ups and nodes removal are more mature

The public entrance is more stable.
```

---

## 2. Keepalived is More Suitable for Physical Machines and Private Clouds

Suitable for:

```text
Traditional IDC

Naked Metal Server

Private clouds

Intranet VIP

Self-build gateway entrance
```

But pay special attention to:

```text
VRRP Communications

Brain fracture.

ARP

Firewall

Cybercard

VIP Floating Authentication
```

---

## 3. High Availability is Not Just About Having VIP

True high availability also includes:

```text
Multi-node configuration consistent

Certificates Same

Log Complete

Security alert.

Health screening

Autoextract

Change can roll back

Fault Exercise

Capacity redundancy
```

---

## 4. Multi-Node Configurations Must Be Version Controlled

Avoid manually logging into each machine to modify configurations long-term.

Recommended:

```text
Git Manage Configuration

Ansible / CI/CD Release

Auto nginx -t

One greyscale.

Batch reload

Failed to automatically stop publication

Keep Rollback Version
```

---

## 5. Certificate Updates Must Cover All Nodes

After certificate updates, check:

```text
Each fullchain Consistency

Each privkey Matches

Each nginx -t Passed

Does every one of them... reload

Online access to obtain new certificates
```

---

## 6. High Availability Requires Regular Drills

At least drill:

```text
Stop Nginx

Stop Keepalived

Close Master Nodes

VIP Floating

SLB Remove Nodes

Certificate Update Rollback

Configure Release Rollback

Single node failure

Inconsistencies in multinode configuration
```

---

## 7. Don't Make Health Checks Too Simple

Merely:

```text
return 200 "ok"
```

Only proves Nginx is alive.

If you want to prove business chain availability, you also need to consider:

```text
Nginx Is it normal to the back end?

Is the back end healthy?

Is the core dependence normal?

Do you need a deeper health check?
```

But health checks shouldn't be too heavy either, to avoid stressing the backend.

---

## 21. Summary of Common Commands

---

## Nginx Status

```bash
systemctl status nginx
```

```bash
nginx -t
```

```bash
nginx -T
```

```bash
ss -lntp | grep ':80'
```

```bash
ss -lntp | grep ':443'
```

```bash
curl -i http://127.0.0.1/health
```

---

## Keepalived Management

```bash
systemctl status keepalived
```

```bash
systemctl start keepalived
```

```bash
systemctl stop keepalived
```

```bash
systemctl restart keepalived
```

```bash
systemctl enable keepalived
```

```bash
journalctl -u keepalived -n 100
```

```bash
journalctl -u keepalived -f
```

---

## VIP Check

```bash
ip addr
```

```bash
ip a | grep 10.0.0.100
```

```bash
ip addr show eth0
```

```bash
curl -I http://10.0.0.100
```

```bash
curl -I -H "Host: example.com" http://10.0.0.100
```

---

## VRRP Packet Capture

```bash
tcpdump -i eth0 vrrp -nn
```

```bash
tcpdump -i eth0 proto 112 -nn
```

---

## Health Check Script

```bash
vi /etc/keepalived/check_nginx.sh
```

```bash
chmod +x /etc/keepalived/check_nginx.sh
```

```bash
/etc/keepalived/check_nginx.sh
```

```bash
echo $?
```

---

## Configuration Synchronization

```bash
rsync -avz /etc/nginx/ root@10.0.0.21:/etc/nginx/
```

```bash
rsync -avz /etc/nginx/ root@10.0.0.22:/etc/nginx/
```

```bash
for host in 10.0.0.21 10.0.0.22; do
    echo "===== $host ====="
    ssh root@$host "nginx -t"
done
```

```bash
for host in 10.0.0.21 10.0.0.22; do
    echo "===== $host ====="
    ssh root@$host "systemctl reload nginx"
done
```

---

## Certificate Check

```bash
for host in 10.0.0.21 10.0.0.22; do
    echo "===== $host ====="
    ssh root@$host "openssl x509 -in /etc/nginx/certs/example.com/fullchain.pem -noout -dates"
done
```

```bash
for host in 10.0.0.21 10.0.0.22; do
    echo "===== $host ====="
    ssh root@$host "openssl x509 -in /etc/nginx/certs/example.com/fullchain.pem -noout -text | grep -A 2 'Subject Alternative Name'"
done
```

---

## Node-Specific Verification

```bash
curl -I --resolve example.com:80:10.0.0.21 http://example.com
```

```bash
curl -I --resolve example.com:443:10.0.0.21 https://example.com
```

```bash
for host in 10.0.0.21 10.0.0.22 10.0.0.23; do
    echo "===== check $host ====="
    curl -I --resolve example.com:80:$host http://example.com
done
```

```bash
for host in 10.0.0.21 10.0.0.22 10.0.0.23; do
    echo "===== check $host ====="
    curl -I --resolve example.com:443:$host https://example.com
done
```

---

## Log Troubleshooting

```bash
tail -n 100 /var/log/nginx/access.log
```

```bash
tail -n 100 /var/log/nginx/error.log
```

```bash
journalctl -u nginx -n 100
```

```bash
journalctl -u keepalived -n 100
```

---

## 22. One-Sentence Summary

The core of Nginx high availability is not "installing another Nginx", but:

```text
High access available

Nodal health check

Auto-elimination of malfunctions

VIP or SLB Switch

Configure Same

Certificates Same

Log Complete

Change controllable

Faultable exercise.
```

Common Architecture Choices:

```text
A cloud.
→ Priority SLB / CLB / ELB + Multiple Nginx

Physical machines / Private clouds
→ Available Keepalived + VIP + Multiple Nginx

Multi-geographical network
→ CDN / GSLB / DNS + Multiple entrances Nginx
```

Keepalived Core Capabilities:

```text
VRRP

VIP Floating

priority Priority

track_script Health screening

Master / Backup Switch
```

Key Items for Basic VIP Configuration:

```text
interface

virtual_router_id

priority

auth_pass

virtual_ipaddress

track_script
```

Production Recommendations:

```text
It's a cloud load balance.

Keepalived Better for physics and private clouds.

Not just testing the machine to survive. Nginx Health

To prevent brain fractures.

Multinom configuration must enter Git

Could not close temporary folder: %s

Logs must be collected centrally

Change greyscale one, then batch release

High availability requires regular exercises

Single Nginx Not suitable for long-term core production entry
```