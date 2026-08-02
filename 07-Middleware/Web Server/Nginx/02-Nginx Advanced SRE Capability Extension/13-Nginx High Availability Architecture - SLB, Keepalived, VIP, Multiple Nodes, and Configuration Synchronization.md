# 13-Nginx High Availability Architecture: SLB, Keepalived, VIP, Multiple Nodes, and Configuration Synchronization

#Nginx #High Availability #SLB #Keepalived #VIP #VRRP #Load Balancing #Configuration Synchronization #Certificate Synchronization #SRE #Access Layer Architecture

---

## Recommended Reading Path

07-Middleware/Web Server/Nginx/02-Nginx Advanced SRE Capability Expansion/13-Nginx High Availability Architecture: SLB, Keepalived, VIP, Multiple Nodes, and Configuration Synchronization.md

---

## I. Document Overview

This document outlines the design and operational methods for high availability of Nginx in the access layer.

Previous sections have covered:

```text
Nginx Configuration Basics

Reverse Proxy

upstream Load Balancing

Production Proxy Parameters

HTTPS Certificates

Real IP Addresses

JSON Logs

Common Fault Troubleshooting

Performance Capacity Analysis

Throttling and Connection Control

upstream Advanced Management
```

This chapter delves further into:

- Why a single Nginx server represents a single point of failure
- Common high availability architectures for Nginx
- SLB/CLB/ELB combined with multiple Nginx servers
- Keepalived + VIP for dual-machine high availability
- Basic concepts of VRRP
- Master-slave mode
- Active-active/multi-active modes
- Nginx health checks
- How Keepalived monitors Nginx status
- VIP drift verification
- Differences between cloud and physical environments
- Configuration synchronization across multiple nodes
- Certificate synchronization across multiple nodes
- Log collection on multiple nodes
- Change deployment processes for multiple nodes
- Common faults in high availability architectures
- Production considerations

This document is the 13th installment in the Nginx Advanced SRE Capability Expansion series.

Objectives of this chapter:

```text
To understand the risks associated with using a single Nginx server

→ To distinguish between SLB + Nginx and Keepalived + VIP architectures

→ To comprehend the basic configuration of Keepalived

→ To learn how to configure Nginx health check scripts

→ To verify VIP drift detection

→ To recognize the importance of configuration and certificate synchronization across multiple nodes

→ To be able to troubleshoot common issues in high availability architectures for Nginx

→ To develop a solid understanding of access layer high availability design principles
```

---

## II. Why Nginx Requires High Availability

If there is only one Nginx server:

```text
Users

→ Single Nginx server

→ Backend services
```

Any issue with this server can lead to:

```text
Nginx process crashes

Server downtime

System restart

Network card errors

Full disk space

Configuration reload failures

Certificate issues

Security group misconfigurations

Data center network problems
```

As a result:

```text
The entire service becomes unavailable

All users are unable to access the system

Backend services are running but not accessible from outside

The scope of business disruption increases

Recovery requires manual intervention
```

Therefore, in production environments, Nginx should never be used as a single server.

In simple terms:

```text
Anytime Nginx serves as the primary entry point for services, it must not function as a single point of failure.
```

---

## III. Common Nginx High Availability Architectures

There are three main types of architectures:

```text
Cloud load balancing SLB/CLB/ELB combined with multiple Nginx servers

Keepalived + VIP combined with multiple Nginx servers

DNS/CDN/GSLB combined with multiple Nginx servers at multiple entry points
```

---

## 1. SLB/CLB/ELB Combined with Multiple Nginx Servers

Typical setup:

```text
Users

→ Cloud load balancing SLB/CLB/ELB

→ Nginx-1

→ Nginx-2

→ Nginx-3

→ Backend services
```

Advantages:

```text
Cloud providers handle the high availability at the entry point

Multiple Nginx servers act as backend instances

SLB performs regular health checks on Nginx servers

Abnormal nodes are automatically removed from the load balancing

The public network entrance is managed by SLB
```

Suitable for:

```text
Public cloud environments

Where managed load balancing services are available

When reducing the complexity of self-built high availability solutions

For scenarios requiring horizontal scaling of multiple Nginx servers

When a unified public network entrance is needed
```

---

## 2. Keepalived + VIP Combined with Multiple Nginx Servers

Typical setup:

```text
Users

→ VIP address

→ Current Master Nginx server

→ Backend services
```

Two nodes are involved:

```text
Nginx-1 + Keepalived

NAutostart at boot:

```bash
systemctl enable keepalived
```

---

## Section 8: Keepalived Master-Backup Configuration Examples

---

## Scenario 10: Master Node Configuration

Node Information:

```text
Master: 10.0.0.21

Backup: 10.0.0.22

VIP: 10.0.0.100

Network Interface: eth0
```

Configuration File:

```bash
vi /etc/keepalived/keepalived.conf
```

Master Configuration:

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

Backup Configuration:

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

## Scenario 12: Explanation of Key Parameters

```text
router_id
→ Current node identifier

state
→ Initial state, MASTER or BACKUP

interface
→ Network interface bound to the VIP

virtual-router_id
→ VRRP group ID; must be consistent across master and backup nodes

priority
→ Priority level; higher values indicate a greater chance of becoming the Master

advert_int
→ VRRP advertisement interval

auth_type / auth_pass
→ Authentication details for master and backup nodes

virtual_ipaddress
→ VIP address to be migrated
```

Notes:

```text
The virtual-router_id must be identical across all Keepalived instances in the same group.

Virtual-router IDs should not conflict for different services or VIP addresses.

The auth_pass values must match on both master and backup nodes.

The interface field must specify the actual network interface name.
```

To determine the network interface name:

```bash
ip addr
```

---

## Section 9: Starting Keepalived and Verifying the VIP

---

## Scenario 13: Starting the Service

Both nodes should execute the following commands:

```bash
systemctl enable keepalived
```

```bash
systemctl start keepalived
```

To check the service status:

```bash
systemctl status keepalived
```

---

## Scenario 14: Checking if the VIP is Assigned

On the Master Node:

```bash
ip addr show eth0
```

Or:

```bash
ip a | grep 10.0.0.100
```

Under normal circumstances, the VIP should be listed on the Master Node.

On the Backup Node:

```bash
ip a | grep 10.0.0.100
```

In this case, the Backup Node should not possess the VIP address.

---

## Scenario 15: Accessing Nginx via the VIP

```bash
curl -I http://10.0.0.100
```

If a domain name is configured:

```bash
curl -I -H "Host: example.com" http://10.0.0.100
```

---

## Scenario 16: Checking Keepalived Logs

For Ubuntu / systemd systems:

```bash
journalctl -u keepalived -n 100
```

To view logs in real-time:

```bash
journalctl -u keepalived -f
```

If the system writes messages to the standard error log, you can also check:

```bash
tail -f /var/log/messages
```

---

## Section 10: Keepalived Monitoring Nginx Availability

Simply checking whether the host is running is not sufficient.

It is also necessary to verify the following:

```text
Whether the Nginx process is running

Whether the Nginx port is listening

Whether the health check paths are functioning correctly
```

Otherwise, potential issues may arise, such as:

```text
Keepalived is working normally,

the VIP remains assigned to the Master node,

but Nginx has crashed,

resulting in failed access attempts.
```

---

## Scenario 17: Creating an Nginx Monitoring Script

Script Path:

```bash
vi /etc/keepalived/check_nginx.sh
```

Script Content:

```bash
#!/bin/bash

```ini
auth_type PASS
auth_pass nginx_ha
}

virtual_ipaddress {
    10.0.0.100/24
}
}
```

Note:

`nopreempt` is used to prevent high-priority nodes from immediately reclaiming the VIP after recovery.

Production Recommendations:

For core production scenarios, non-preempt mode can be considered.

Avoid frequent VIP switches after fault recovery.

It is best to execute the failback process only after manual confirmation.

---

## Section 13: Keepalived Brain Split Issues

---

## Scenario 25: What is Brain Split

Brain split occurs when:

```text
Both nodes consider themselves the Master.
Both nodes possess the VIP.
```

Risks include:

```text
ARP conflicts.
Abnormal traffic fluctuations.
Requests randomly directed to different nodes.
Unstable connections.
Confused logs.
Abnormal service access.
```

---

## Scenario 26: Common Causes of Brain Split

Common reasons include:

```text
Blockage in VRRP communication between master and replica.
Firewalls blocking VRRP packets.
Switches or cloud networks not supporting multicast.
Conflicts in `virtualRouter_id`.
Inconsistent `auth_pass` values.
Incorrect `interface` settings.
Severe network jitter.
Security policies preventing protocol 112.
```

VRRP protocol number:

```text
112
```

---

## Scenario 27: Commands for Troubleshooting Brain Split

To check if both nodes hold the VIP:

```bash
ip a | grep 10.0.0.100
```

To view Keepalived logs:

```bash
journalctl -u keepalived -n 100
```

To capture VRRP packets:

```bash
tcpdump -i eth0 vrrp -nn
```

If `tcpdump` does not support VRRP filtering, use:

```bash
tcpdump -i eth0 proto 112 -nn
```

To check the firewall settings:

```bash
iptables -L -n
```

or:

```bash
firewall-cmd --list-all
```

---

## Section 14: Limitations of Keepalived in Cloud Environments

---

## Scenario 28: Cloud environments may not support VIP drift

In public clouds, using Keepalived with VIPs might be restricted due to the following reasons:

```text
Cloud providers do not open their layer 2 networks.
They do not support arbitrary ARP broadcasts.
Custom VIP drift is not allowed.
Security groups or routing mechanisms may not recognize VIPs.
Multicast VRRP is not available.
Virtual networks have restrictions on ARP traffic.
```

Therefore, in cloud environments, it is recommended to use:

```text
Cloud Load Balancers such as SLB/CLB/ELB.
Cloud providers' high-availability VIP solutions.
Elastic Network Card drift mechanisms.
Private IP address binding and switching techniques.
Cloud APIs for automatic switching.
```

---

## Scenario 29: Recommended Architecture in Clouds

Recommended approach:

```text
Public network users → Cloud Load Balancers such as SLB/CLB/ELB → Multiple Nginx servers → Backend services.
```

Advantages include:

```text
Cloud LBs offer built-in high availability features.
Their health check mechanisms are mature.
Abnormal nodes are automatically removed from the load balancing process.
The public network access is stable.
There is no need to manage ARP, VRRP, or VIP settings manually.
```

---

## Section 15: Synchronization of Configuration on Multiple Nginx Nodes

---

## Scenario 30: Why Configuration Synchronization is Important

If multiple Nginx nodes have inconsistent configurations, the following issues may occur:

```text
Some requests are processed correctly.
Some requests result in a 404 error.
Some requests receive a 502 error.
Some nodes use incorrect certificates.
Some nodes have different rate-limiting policies.
Some nodes have different allowlists.
Some nodes use different log formats.
Some nodes resolve real IP addresses differently.
```

Such issues are difficult to debug because users may experience unpredictable behavior, such as:

```text
Sometimes the service works fine, and sometimes it doesn’t.
```

The root cause is usually inconsistent configurations across Nginx nodes.

---

## Scenario 31: Methods for Configuration Synchronization

Common methods include:

```text
Manual synchronization.
rsync synchronization.
Git-based management with pull operations.
Batch deployment using Ansible.
CI/CD pipelines for automating configuration updates.
Centralized configuration management systems.
Mirroring of configuration files across nodes.
```

For production use, it is recommended to:

```text
Store configurations in Git.
Have changes reviewed before deployment.
Automatically execute `nginx -t` commands after changes are applied.
Deploy configurations batch-wise across all nodes.
Ensure quick rollback capabilities in case of errors.
```

---

## Scenario 3## Scenario 42: Node-based Statistics for 5xx Errors

If the JSON log contains the hostname:

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status >= 500) | .hostname' | sort | uniq -c | sort -nr
```

In the log platform, aggregation and analysis can be performed based on:

```text
hostname

status

uri

upstream_addr
```

---

## Section 18: Multi-node Change Deployment Process

---

## Scenario 43: Recommended Change Deployment Process for Multi-node Nginx

Recommended steps for changing multiple Nginx nodes:

```text
Add configuration to Git

→ Conduct change review

→ Test in the staging environment using `nginx -t`

→ Deploy the changes in a graybox environment on one node

→ Verify again using `nginx -t` and reload the configuration

→ Use `curl` to check the functionality

→ Monitor `access.log` and `error.log` for any issues

→ Ensure everything is working correctly before deploying to all nodes

→ Verify the configuration versions on all nodes

→ Record all changes made
```

---

## Scenario 44: Deploy Changes First on One Nginx Node in a Graybox Environment

Suppose you have two Nginx servers:

```text
Nginx-1: 10.0.0.21

Nginx-2: 10.0.0.22
```

First, deploy the changes on Nginx-1:

```bash
ssh root@10.0.0.21 "nginx -t && systemctl reload nginx"
```

Verify the functionality using a specified IP address:

```bash
curl -I --resolve example.com:80:10.0.0.21 http://example.com
```

Check HTTPS connectivity:

```bash
curl -I --resolve example.com:443:10.0.0.21 https://example.com
```

Monitor the logs:

```bash
ssh root@10.0.0.21 "tail -n 100 /var/log/nginx/error.log"
```

---

## Scenario 45: Deploy Changes Batchwise to the Remaining Nodes

```bash
for host in 10.0.0.22 10.0.0.23; do
    echo "===== Deploying on $host ====="
    rsync -avz /etc/nginx/ root@$host:/etc/nginx/
    ssh root@$host "nginx -t && systemctl reload nginx"
done
```

---

## Scenario 46: Verify Functionality Batchwise on All Nodes

```bash
for host in 10.0.0.21 10.0.0.22 10.0.0.23; do
    echo "===== Checking $host ====="
    curl -I --resolve example.com:80:$host http://example.com
done
```

For HTTPS:

```bash
for host in 10.0.0.21 10.0.0.22 10.0.0.23; do
    echo "===== Checking $host ====="
    curl -I --resolve example.com:443:$host https://example.com
done
```

---

## Section 19: Common Faults in Nginx High Availability

---

## Scenario 47: The VIP Is Not Bound

To troubleshoot this issue:

```bash
systemctl status keepalived
```

```bash
journalctl -u keepalived -n 100
```

```bash
ip a | grep 10.0.0.100
```

Common causes include:

```text
Keepalived is not running

Syntax errors in the configuration file

Incorrect interface specified

Incorrect VIP IP range

Abnormal priority settings

VRRP communication failures

Script execution failures leading to reduced priority
```

---

## Scenario 48: Inaccessibility After the VIP Drifts

To diagnose this issue:

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

Common reasons include:

```text
The backup Nginx server is not running

Incomplete configuration on the backup node

Firewall rules blocking access to the backup node

The VIP has drifted, but the ARP cache has not been updated

Nginx is not listening on the correct port

Abnormal health check paths
```

---

## Scenario```bash
systemctl enable keepalived
```

```bash
journalctl -u keepalived -n 100
```

```bash
journalctl -u keepalived -f
```

---

## VIP Verification

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

## Health Check Scripts

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

## Certificate Verification

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

## Specified Node Verification

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

## Twenty-Two, One Sentence Summary

The core of Nginx high availability does not lie in "installing multiple Nginx instances," but rather in:

```text
High availability at the entry point

Node health checks

Automatic failure removal

VIP or SLB switching

Consistent configuration

Unified certificates

Comprehensive logging

Controllable changes

Regular fault drills
```

Common architectural choices include:

```text
Public clouds
→ Prioritize SLB/CLB/ELB + multiple Nginx instances

Physical servers / Private clouds
→ Use Keepalived + VIP + multiple Nginx instances

Multi-region public networks
→ CDN/GSLB/DNS + multiple entry-point Nginx servers
```

Key capabilities of Keepalived include:

```text