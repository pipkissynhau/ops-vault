# Ceph Security Reinforcement: CephX Permissions, Network Isolation, Dashboard, RGW HTTPS, and Key Governance

Recommended path: 05-Storage/01-Ceph/18-Ceph Security Reinforcement: CephX Permissions, Network Isolation, Dashboard, RGW HTTPS, and Key Governance.md

Tags: #Ceph #Secure. #CephX #GovernanceOfAuthority #Dashboard #RGW #HTTPS #NetworkIsolation #KeyManagement #SRE #AdvancedSre

---

## I. Document Explanation

This is the eighteenth article of the advanced SRE storage module for Ceph, focusing on methods for securing Ceph clusters.

Previously completed:

- Ceph cluster deployment
- OSD management
- Pool and PG
- CRUSH fault domain
- RBD block storage practice
- CephFS file storage practice
- RGW object storage practice
- Kubernetes RBD CSI
- Kubernetes CephFS CSI
- Ceph daily operations
- Ceph troubleshooting
- Ceph backup and recovery
- Ceph performance optimization

This article enters the security governance phase.

As a distributed storage system, Ceph's security risks mainly focus on:

    Is the cluster communication exposed?
    Are client permissions too broad?
    Is the admin key misused?
    Is the Dashboard exposed to the public internet?
    Does RGW use HTTPS?
    Is the S3 AccessKey leaked?
    Are Kubernetes Secrets too broad?
    Are MON / OSD / MGR ports accessed by irrelevant hosts?
    Is the cephadm SSH key secure?
    Do backup files contain sensitive keys?

This article covers:

- Understanding Ceph's security boundaries
- CephX authentication mechanism
- Designing minimal privilege users
- RBD user permission control
- CephFS user permission control
- RGW S3 user and key governance
- Dashboard HTTPS and account security
- RGW HTTPS unified entry point
- Network isolation and port access control
- cephadm SSH key security
- Kubernetes CSI Secret security
- Backup file security
- Auditing and daily checks
- Common security risks and remediation methods

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand Ceph's main security boundaries.
2. Understand the role of CephX.
3. Avoid abusing client.admin in business contexts.
4. Create minimal privilege users for RBD.
5. Create minimal privilege users for CephFS.
6. Create dedicated users for Kubernetes CSI.
7. Manage RGW S3 users, AccessKeys, and SecretKeys.
8. Configure Dashboard HTTPS and administrator accounts.
9. Expose RGW HTTPS through Nginx / LB as a unified entry point.
10. Restrict access to Ceph MON / OSD / MGR / RGW ports.
11. Protect cephadm SSH private keys and keyrings.
12. Identify which files and Secrets are sensitive.
13. Establish a Ceph security inspection checklist.
14. Establish key rotation and permission review processes.
15. Understand the Ceph security reinforcement baseline for production environments.

---

## III. Experiment and Production Environments

### 3.1 Ceph Cluster Nodes

This article continues using the Ceph module's experimental environment.

| IP | Hostname | Role |
|---|---|---|
| 10.0.0.31 | ceph-node01 | MON / MGR / OSD / RGW / MDS |
| 10.0.0.32 | ceph-node02 | MON / MGR / OSD / RGW / MDS |
| 10.0.0.33 | ceph-node03 | MON / MGR / OSD / RGW |
| 10.0.0.34 | ceph-node04 | OSD / Expansion / Fault Simulation (Optional) |
| 10.0.0.35 | ceph-client | RBD / CephFS / RGW Client Testing (Optional) |
| 10.0.0.36 | rgw-lb / nginx | RGW HTTPS Unified Entry Point (Optional) |

Main experimental system:

    Ubuntu Server 22.04.5 LTS

Supplementary system:

    Rocky Linux 9

Deployment method:

    cephadm

---

### 3.2 Kubernetes Cluster

If integrating with Kubernetes CSI:

| IP | Hostname | Role |
|---|---|---|
| 10.0.0.20 | k8s-master | Kubernetes Master |
| 10.0.0.21 | k8s-worker01 | Kubernetes Worker |
| 10.0.0.22 | k8s-worker02 | Kubernetes Worker |

Purpose:

    Use RBD CSI
    Use CephFS CSI
    Store Ceph user keys via Kubernetes Secrets

---

## IV. Understanding Ceph Security Boundaries

### 4.1 Ceph Security is Divided into Five Layers

Ceph security cannot be viewed in isolation; it should be understood in layers:

    First layer: Network boundary
      Control which hosts can access MON / OSD / MGR / RGW / Dashboard

    Second layer: Authentication boundary
      Use CephX to manage client and server authentication

    Third layer: Permission boundary
      Limit user access to Pools, CephFS, RBD, and RGW via caps

    Fourth layer: Entry boundary
      Security for Dashboard, RGW, Nginx, LB, Kubernetes CSI, etc.

    Fifth layer: Key boundary
      Keyring, S3 keys, Kubernetes Secrets, cephadm SSH keys, and backup file protection

---

### 4.2 Common Sensitive Information in Ceph

The following content is considered sensitive information: /think

| Type | Example |
|---|---|
| admin keyring | /etc/ceph/ceph.client.admin.keyring |
| Ceph User keyring | ceph.client.k8s-rbd.keyring |
| CephX key | ceph auth get-key client.xxx |
| RGW AccessKey | S3 access_key |
| RGW SecretKey | S3 secret_key |
| Kubernetes Secret | csi-rbd-secretI don't know.csi-cephfs-secret |
| cephadm SSH private key | /etc/ceph/cephadm-ssh-key |
| auth file in backup | ceph-auth-export.keyring |
| Dashboard administrator password | Dashboard admin user password |

Principles:

    These contents cannot be submitted to Git.
    Cannot be written to public notes.
    Cannot be transmitted in plaintext via chat tools.
    Cannot be placed in ordinary shared directories.
    Must restrict file permissions.
    Must have rotation and deprecation processes.

---

## 5. CephX Authentication Mechanism

### 5.1 What is CephX

CephX is Ceph's authentication mechanism.

It is used to control:

    Which clients can access the Ceph cluster.
    Which users can access which services.
    Which users can access which Pools.
    Which users can operate RBD / CephFS / RGW.

Common CephX user forms:

    client.admin
    client.k8s-rbd
    client.k8s-cephfs
    client.rbd-user
    client.cephfs-user

---

### 5.2 View CephX Authentication Configuration

View current authentication-related configuration:

    ceph config dump | grep auth

Common configurations:

    auth_cluster_required
    auth_service_required
    auth_client_required

Generally should be:

    cephx

If authentication is disabled, it is a high-risk configuration.

---

### 5.3 View All Authentication Users

    ceph auth ls

View specified user:

    ceph auth get client.admin

View user key:

    ceph auth get-key client.admin

High-risk warning:

    get-key outputs sensitive keys.
    Do not directly copy to public documentation.
    Do not post to group chats.
    Production environments should create dedicated users with minimal permissions.

---

## 6. Usage Boundaries of client.admin

### 6.1 What is client.admin

client.admin is the Ceph administrator user.

It has extremely high permissions, typically able to manage the entire Ceph cluster.

Suitable for:

- Cluster initialization
- Management node operations
- Emergency fault handling
- Cluster-level management commands

Not suitable for:

- Long-term use by business servers
- Use by Kubernetes CSI
- Mounting RBD by ordinary applications
- Mounting CephFS by ordinary applications
- Direct distribution to developers
- Writing to images, code repositories, or configuration centers

---

### 6.2 View admin Permissions

    ceph auth get client.admin

Typically you can see similar:

    caps mon = "allow *"
    caps mgr = "allow *"
    caps osd = "allow *"
    caps mds = "allow *"

This indicates it has extremely broad permissions.

---

### 6.3 Production Principles

In production, follow:

    Administrators use admin.
    Business uses dedicated users.
    Kubernetes CSI uses dedicated users.
    Create different users for RBD / CephFS / RGW separately.
    Different businesses use different keys.
    Keys can be disabled and rotated by business dimension after leakage.

---

## 7. RBD Minimal Permissions User

### 7.1 Create RBD Pool

Example Pool:

    rbd-secure-pool

Create:

    ceph osd pool create rbd-secure-pool 64
    ceph osd pool application enable rbd-secure-pool rbd
    ceph osd pool set rbd-secure-pool size 3
    ceph osd pool set rbd-secure-pool min_size 2
    rbd pool init rbd-secure-pool

---

### 7.2 Create RBD Dedicated User

Create user:

    ceph auth get-or-create client.rbd-secure-user \
      mon 'profile rbd' \
      mgr 'profile rbd pool=rbd-secure-pool' \
      osd 'profile rbd pool=rbd-secure-pool' \
      -o /etc/ceph/ceph.client.rbd-secure-user.keyring

View:

    ceph auth get client.rbd-secure-user

Expected:

    caps mon = "profile rbd"
    caps mgr = "profile rbd pool=rbd-secure-pool"
    caps osd = "profile rbd pool=rbd-secure-pool"

---

### 7.3 Verify RBD User Permissions

Create Image using this user:

    rbd --id rbd-secure-user -p rbd-secure-pool create secure-image --size 1G

View:

    rbd --id rbd-secure-user -p rbd-secure-pool ls

View Image:

    rbd --id rbd-secure-user info rbd-secure-pool/secure-image

---

### 7.4 Verify Cannot Access Other Pools

If other RBD Pools exist, for example:

    rbd-pool

Attempt to access:

    rbd --id rbd-secure-user -p rbd-pool ls

Expected:

    Permission denied or access failure

Note: /think

client.rbd-secure-user is restricted to rbd-secure-pool.
This is the value of minimal permissions.

---

### 7.5 Deleting Test Resources

    rbd --id rbd-secure-user rm rbd-secure-pool/secure-image

High-risk warning:

    rbd rm will delete the Image.
    Execute only after confirming it's a test Image.

---

## VIII. CephFS Minimal Permissions User

### 8.1 Prerequisites

CephFS already exists:

    cephfs

Check:

    ceph fs ls
    ceph fs status
    ceph mds stat

---

### 8.2 Creating CephFS Dedicated User

Create user:

    ceph auth get-or-create client.cephfs-secure-user \
      mon 'allow r' \
      mds 'allow rw fsname=cephfs' \
      osd 'allow rw tag cephfs data=cephfs' \
      -o /etc/ceph/ceph.client.cephfs-secure-user.keyring

Check:

    ceph auth get client.cephfs-secure-user

---

### 8.3 Extracting Secret

    ceph auth get-key client.cephfs-secure-user > /etc/ceph/cephfs-secure-user.secret

Set permissions:

    chmod 600 /etc/ceph/cephfs-secure-user.secret

---

### 8.4 Mounting CephFS with Dedicated User

Create mount directory:

    mkdir -p /mnt/cephfs-secure

Mount:

    mount -t ceph 10.0.0.31,10.0.0.32,10.0.0.33:/ /mnt/cephfs-secure \
      -o name=cephfs-secure-user,secretfile=/etc/ceph/cephfs-secure-user.secret,fs=cephfs

Verify:

    echo "cephfs secure user test" > /mnt/cephfs-secure/secure-test.txt
    cat /mnt/cephfs-secure/secure-test.txt

Unmount:

    umount /mnt/cephfs-secure

---

### 8.5 Subdirectory Permission Control Concept

In production, it's not recommended to mount the CephFS root directory for all business applications.

Recommended instead:

    /apps/app01
    /apps/app02
    /data/team-a
    /data/team-b

Combined with:

- CephX path restrictions (version-dependent)
- Linux file permissions
- ACL
- Kubernetes subdirectory isolation
- SubvolumeGroup/Subvolume management

Principle:

    Business only sees the directories it needs.
    Business cannot directly access the entire CephFS root directory.

---

## IX. Kubernetes CSI User Security

### 9.1 RBD CSI User

RBD CSI should not use client.admin.

Recommended user:

    client.k8s-rbd

Creation method:

    ceph auth get-or-create client.k8s-rbd \
      mon 'profile rbd' \
      mgr 'profile rbd pool=k8s-rbd' \
      osd 'profile rbd pool=k8s-rbd' \
      -o /etc/ceph/ceph.client.k8s-rbd.keyring

Check:

    ceph auth get client.k8s-rbd

---

### 9.2 CephFS CSI User

Recommended user:

    client.k8s-cephfs

Creation method:

    ceph auth get-or-create client.k8s-cephfs \
      mon 'allow r' \
      mgr 'allow rw' \
      mds 'allow rw fsname=cephfs' \
      osd 'allow rw tag cephfs data=cephfs' \
      -o /etc/ceph/ceph.client.k8s-cephfs.keyring

Check:

    ceph auth get client.k8s-cephfs

Note:

    Different Ceph/Ceph CSI versions may have slightly different requirements for mgr/mds/osd caps.
    In production, verify in a test environment first, then gradually converge permissions.

---

### 9.3 Kubernetes Secret Security

Check Secret:

    kubectl get secret -n ceph-csi

Viewing Secret content will expose base64-encoded key:

    kubectl get secret -n ceph-csi csi-rbd-secret -o yaml

Note:

    Base64 is not encryption.
    Any user who can read Secret may decode the Ceph key.

---

### 9.4 Kubernetes RBAC Recommendations

In production, restrict:

- Who can view ceph-csi Namespace
- Who can get/list/watch Secret
- Who can modify StorageClass
- Who can delete PVC/PV
- Who can modify CSI Deployment/DaemonSet

Recommendations:

    Ordinary business users cannot access ceph-csi Namespace.
    Ordinary business users cannot read CSI Secret.
    StorageClass modifications must follow change process.
    PVC deletion must have data confirmation process.

---

## X. RGW S3 User and Key Governance

### 10.1 Creating RGW User

Create user:

    radosgw-admin user create \
      --uid="app-s3-user" \
      --display-name="Application S3 User"

Output includes:

    access_key
    secret_key

Save to secure location: /think

radosgw-admin user info --uid="app-s3-user" > /root/app-s3-user.json

Setting permissions:

    chmod 600 /root/app-s3-user.json

---

### 10.2 Viewing User Information

    radosgw-admin user info --uid="app-s3-user"

Key focus areas:

- keys
- user_quota
- bucket_quota
- suspended
- caps

---

### 10.3 AccessKey / SecretKey Security Principles

S3 Keys are equivalent to object storage access credentials.

Production requirements:

    Do not write to code repositories.
    Do not write to public documentation.
    Do not send plaintext via chat tools.
    Do not share multiple services using the same key set.
    Do not retain keys for long-unused users.
    Disable or rotate keys immediately after leakage.
    Regularly audit user and bucket ownership.

---

### 10.4 Setting User Quotas

User-level quotas:

    radosgw-admin quota set \
      --quota-scope=user \
      --uid=app-s3-user \
      --max-size=50G \
      --max-objects=100000

Enable:

    radosgw-admin quota enable \
      --quota-scope=user \
      --uid=app-s3-user

View:

    radosgw-admin user info --uid=app-s3-user | jq '.user_quota'

---

### 10.5 Disabling Users

If a user leaks or is abandoned:

    radosgw-admin user suspend --uid=app-s3-user

Restore:

    radosgw-admin user enable --uid=app-s3-user

Delete user:

    radosgw-admin user rm --uid=app-s3-user

High-risk deletion:

    radosgw-admin user rm --uid=app-s3-user --purge-data

High-risk warning:

    --purge-data will delete data associated with this user.
    Execute only after confirming bucket and object ownership in production environments.

---

## ElevenI don't know.RGW HTTPS Unified Entry Point

### 11.1 Security Boundary

Experimental environments can use:

    http://10.0.0.31:7480

Production external access must use:

    https://s3.example.com

Recommended architecture:

    Client
      |
      | HTTPS 443
      v
    Nginx / HAProxy / LB
      |
      | HTTP 7480, internal trusted network
      v
    RGW backend instance

Notes:

    HTTP can be used on internal trusted networks to reduce complexity and TLS overhead.
    HTTPS must be used for public or untrusted networks.
    If enterprise compliance requires encryption for internal links, evaluate RGW backend TLS or full-chain TLS.

---

### 11.2 Nginx HTTPS Example

Configuration directory:

    mkdir -p /etc/nginx/ssl

Place formal certificates:

    /etc/nginx/ssl/s3.example.com.pem
    /etc/nginx/ssl/s3.example.com.key

Configuration:

    cat > /etc/nginx/conf.d/ceph-rgw-https.conf <<'EOF'
    upstream ceph_rgw_backend {
        server 10.0.0.31:7480 max_fails=3 fail_timeout=10s;
        server 10.0.0.32:7480 max_fails=3 fail_timeout=10s;
        server 10.0.0.33:7480 max_fails=3 fail_timeout=10s;
    }

    server {
        listen 80;
        server_name s3.example.com;

        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name s3.example.com;

        ssl_certificate     /etc/nginx/ssl/s3.example.com.pem;
        ssl_certificate_key /etc/nginx/ssl/s3.example.com.key;

        client_max_body_size 0;

        proxy_request_buffering off;
        proxy_buffering off;

        location / {
            proxy_pass http://ceph_rgw_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto https;
        }
    }
    EOF

Check:

    nginx -t

Reload:

    systemctl reload nginx

---

### 11.3 Verifying HTTPS Entry

Client configure DNS or hosts:

    10.0.0.36 s3.example.com

Test:

    curl -I https://s3.example.com

AWS CLI:

    export RGW_ENDPOINT="https://s3.example.com"

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 ls

---

### 11.4 HTTPS Entry Security Check

Check items: /think

| Check Item | Requirement |
|---|---|
| Certificate | Use formal CA or enterprise-trusted CA |
| Private Key Permissions | Only root-readable |
| Port 80 | Redirect to HTTPS |
| Port 443 | Open to the public |
| RGW 7480 | Accessible only via internal network or entry layer |
| Host Header | Should not be incorrectly rewritten |
| Upload Size | client_max_body_size 0 or business-defined limit |
| Logs | access/error log must be auditable |

---

## Twelve. Dashboard Security Reinforcement

### 12.1 View Dashboard Status

    ceph mgr services

Possible output:

    dashboard: https://ceph-node01:8443/

Check module:

    ceph mgr module ls | grep dashboard

---

### 12.2 Enable Dashboard

    ceph mgr module enable dashboard

Check service:

    ceph mgr services

---

### 12.3 Set Dashboard HTTPS Certificate

Production should use formal certificate or enterprise internal CA certificate.

Example:

    ceph dashboard set-ssl-certificate -i /path/to/dashboard.pem
    ceph dashboard set-ssl-certificate-key -i /path/to/dashboard.key

Restart MGR:

    ceph orch ps --daemon_type mgr
    ceph orch daemon restart mgr.<mgr-daemon-name>

Actual MGR name is determined by ceph orch ps output.

---

### 12.4 Create Dashboard Administrator User

Create password file:

    echo 'StrongPasswordHere' > /root/dashboard-pass.txt
    chmod 600 /root/dashboard-pass.txt

Create user:

    ceph dashboard ac-user-create admin -i /root/dashboard-pass.txt administrator

Delete password file:

    shred -u /root/dashboard-pass.txt

Note:

    Password should not be directly written in command history.
    Using file is more secure.
    Production should use strong password and rotate regularly.

---

### 12.5 Dashboard Access Control

Production recommendations:

    Dashboard should not be exposed to public internet.
    Only allow access from operations network segment.
    Can add VPN / bastion host / unified authentication in front.
    Administrator account should be minimized.
    Remove accounts of departed personnel promptly.
    Rotate password regularly.
    Enable HTTPS.

---

### 12.6 Dashboard User Management

Commands to view users can refer to Dashboard management commands:

    ceph dashboard ac-user-show admin

If need to delete user:

    ceph dashboard ac-user-delete <username>

Change password:

    echo 'NewStrongPasswordHere' > /root/dashboard-new-pass.txt
    chmod 600 /root/dashboard-new-pass.txt

    ceph dashboard ac-user-set-password <username> -i /root/dashboard-new-pass.txt

    shred -u /root/dashboard-new-pass.txt

---

## Thirteen. Network Isolation and Port Control

### 13.1 Ceph Common Ports

| Service | Port |
|---|---|
| MON v2 | 3300 |
| MON v1 | 6789 |
| OSD | 6800-7300 |
| Dashboard | 8443, depends on configuration |
| Prometheus mgr module | 9283, depends on configuration |
| RGW | 7480, depends on configuration |
| SSH | 22 |

---

### 13.2 Access Control Principles

| Access Source | Allowed Access |
|---|---|
| Between Ceph Nodes | MON / OSD / MGR / SSH |
| Kubernetes Nodes | MON / OSD |
| RBD Client | MON / OSD |
| CephFS Client | MON / OSD / MDS related communication |
| RGW Entry Layer | RGW backend ports |
| Ordinary Office Network | Should not directly access OSD / MON |
| Public Internet | Should not directly access MON / OSD / Dashboard / RGW HTTP |

---

### 13.3 UFW Example

Ubuntu can reference the following if using ufw.

Allow Ceph nodes to access MON:

    ufw allow from 10.0.0.0/24 to any port 3300 proto tcp
    ufw allow from 10.0.0.0/24 to any port 6789 proto tcp

Allow OSD ports:

    ufw allow from 10.0.0.0/24 to any port 6800:7300 proto tcp

Allow RGW backend accessible only via internal network:

    ufw allow from 10.0.0.0/24 to any port 7480 proto tcp

Allow Dashboard accessible only via operations network segment:

    ufw allow from 10.0.0.0/24 to any port 8443 proto tcp

Check status:

    ufw status numbered

---

### 13.4 firewalld Example

Rocky Linux 9 can use firewalld.

Allow MON:

    firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.0.0.0/24" port protocol="tcp" port="3300" accept'
    firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.0.0.0/24" port protocol="tcp" port="6789" accept'

Allow OSD:

firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.0.0.0/24" port protocol="tcp" port="6800-7300" accept'

Reload:

    firewall-cmd --reload

Check:

    firewall-cmd --list-all

---

### 13.5 Production Network Recommendations

Production environment recommendations:

- Ceph cluster network should not be directly exposed to the public internet.
- MON / OSD ports should only allow access from trusted clients.
- Dashboard should be placed in the operations network segment or behind a VPN.
- RGW should only expose HTTPS as a unified entry point.
- Backend RGW HTTP ports should not be directly exposed to the public internet.
- Kubernetes node access to Ceph ports should be explicitly allowed.
- Security groups, firewalls, and network ACLs should have documented records.
- Verify PVC mounting, RBD, CephFS, and RGW access before and after firewall changes.

---

## FourteenI don't know.cephadm SSH Key Security

### 14.1 cephadm SSH Key Purpose

cephadm manages cluster nodes via SSH.

Common files:

    /etc/ceph/ceph.pub
    /etc/ceph/cephadm-ssh-key

Where:

    ceph.pub is the public key
    cephadm-ssh-key is the private key

The private key must be strictly protected.

---

### 14.2 Check File Permissions

    ls -l /etc/ceph/ceph.pub
    ls -l /etc/ceph/cephadm-ssh-key

Recommendations:

    chmod 600 /etc/ceph/cephadm-ssh-key
    chmod 644 /etc/ceph/ceph.pub

---

### 14.3 Risk Description

If the cephadm private key is leaked, attackers may log into Ceph nodes using this key.

Risks include:

- Controlling Ceph nodes
- Modifying services
- Deleting data
- Stealing keyrings
- Lateral movement

Production recommendations:

    Restrict private key access permissions.
    Encrypt backups when storing.
    Do not commit to Git.
    Do not share via regular files.
    Clean up SSH access for departing personnel.
    Regularly audit authorized_keys.

---

## FifteenI don't know.Backup File Security

### 15.1 Common Sensitive Files in Backups

Backup directories may contain:

    /backup/ceph/auth/ceph-auth-export.keyring
    /backup/ceph/config/ceph.client.admin.keyring
    /backup/ceph/config/cephadm-ssh-key
    /backup/ceph/rgw/*user*.json
    /backup/ceph/orch/service-spec.yaml
    Kubernetes Secret YAML

All of these need protection.

---

### 15.2 Set Permissions

Example:

    chmod -R go-rwx /backup/ceph/auth
    chmod -R go-rwx /backup/ceph/config
    chmod -R go-rwx /backup/ceph/rgw

Check:

    ls -ld /backup/ceph/auth
    ls -ld /backup/ceph/config
    ls -ld /backup/ceph/rgw

---

### 15.3 Backup Security Principles

Production recommendations:

- Encrypt backup storage.
- Limit login access to backup servers.
- Restrict permissions for backup directories.
- Use SSH / TLS for backup transmission.
- Regularly check backup file permissions.
- Do not store keyrings in plaintext in knowledge bases.
- Do not directly commit Secret YAML to Git.
- Use test keys or desensitized configurations during recovery drills.

---

## SixteenI don't know.Cluster Communication Encryption Strategy

### 16.1 Is Internal Communication Mandatory to Encrypt

Ceph internal cluster communication is typically deployed in a trusted internal network.

Security policies can be divided into:

| Scenario | Policy |
|---|---|
| Experimental environment | Prioritize internal network isolation |
| Ordinary internal production | Network isolation + CephX + minimal permissions |
| High compliance production | Evaluate msgr2 secure mode / full-chain encryption |
| Across untrusted networks | Must encrypt, do not expose Ceph internal ports |

---

### 16.2 Messenger v2 and secure mode

Ceph new versions support Messenger v2 and can provide stronger transmission security capabilities.

Check MON address:

    ceph mon dump

Typically, you can see v2 addresses:

    [v2:10.0.0.31:3300/0,v1:10.0.0.31:6789/0]

Whether to enable encryption mode depends on:

- Ceph version
- Client compatibility
- Performance impact
- Compliance requirements
- Test verification

Production reminder:

    Do not directly modify cluster communication mode without understanding the impact.
    Must validate RBD, CephFS, RGW, CSI, and client compatibility in a test environment before enabling.
    Must record original configuration and rollback methods before making changes.

---

## SeventeenI don't know.Operating System Security Baseline

### 17.1 Basic Security Items

Ceph nodes should meet:

- Only open necessary ports
- Prohibit weak passwords
- Limit SSH sources
- Control root login policies according to enterprise standards
- Install security patches
- Time synchronization is normal
- Reasonable log retention
- System disk capacity monitoring
- Do not install unrelated services
- Do not run non-essential business on Ceph nodes

---

### 17.2 SSH Security Recommendations

Check SSH configuration:

    grep -E 'PermitRootLogin|PasswordAuthentication|PubkeyAuthentication' /etc/ssh/sshd_config

In production, combine with enterprise standards:

    Limit password login
    Use key-based login
    Limit source IP
    Log in via a bastion host
    Record audit logs

Must confirm that changes will not lock you out before modifying SSH.

---

### 17.3 System Patches

Ubuntu:

    apt update
    apt list --upgradable

Rocky Linux:

    dnf check-update

Production reminder:

    Upgrading system packages on Ceph nodes requires a maintenance window.
    Kernel upgrades require a reboot.
    Assess impact on OSD / MON / MDS / RGW before rebooting.
    Do not reboot multiple critical nodes simultaneously.

---

## EighteenI don't know.Security Audit Checklist

### 18.1 CephX Check

Check authentication configuration:

    ceph config dump | grep auth

Check users: /think

# ceph auth ls

Key Checks:

- Are there unknown client users?
- Do business users have excessive permissions?
- Are many users using allow *?
- Is Kubernetes CSI using admin?
- Are obsolete users not deleted?

---

### 18.2 keyring File Check

    find /etc/ceph -type f -name "*.keyring" -exec ls -l {} \;

Check:

- Are permissions too permissive?
- Are there unused keyring files?
- Can regular users read them?

---

### 18.3 Dashboard Check

    ceph mgr services

Check:

- Is HTTPS enabled?
- Is it exposed to the public internet?
- Are strong passwords used?
- Are there extra users?
- Is access source restricted?

---

### 18.4 RGW Check

    ceph orch ps --daemon_type rgw
    radosgw-admin user list
    radosgw-admin bucket list

Check:

- Are there obsolete users?
- Are users without quota settings?
- Are AccessKeys not rotated for a long time?
- Is RGW HTTP port exposed to the public internet?
- Is HTTPS used for public access?

---

### 18.5 Kubernetes Secret Check

    kubectl get secret -n ceph-csi
    kubectl get sa -n ceph-csi
    kubectl get role,rolebinding -n ceph-csi
    kubectl get clusterrole,clusterrolebinding | grep ceph

Check:

- Are Secrets in the correct Namespace?
- Can regular users read ceph-csi Secrets?
- Is CSI permission too broad?
- Is StorageClass arbitrarily modified?

---

### 18.6 Network Port Check

On Ceph nodes:

    ss -lntp

Check:

- MON 3300 / 6789
- OSD 6800-7300
- Dashboard 8443
- RGW 7480
- Prometheus 9283
- SSH 22

Confirm:

    Which ports are listening on 0.0.0.0.
    Which ports need firewall restrictions.
    Which ports should not be exposed to the public internet.

---

## NineteenI don't know.Security Remediation Examples

### 19.1 Discovery of Business Using client.admin

Risk:

    Business has cluster administrator permissions.

Remediation:

    1. Create a dedicated business user.
    2. Restrict access to specific Pool or CephFS.
    3. Modify business configuration to use the new user.
    4. Verify business read/write functionality.
    5. Remove admin keyring from business servers.
    6. Check for admin key leakage risks.
    7. Rotate admin key if necessary, with careful planning.

---

### 19.2 Discovery of RGW HTTP Exposed to Public Internet

Risk:

    AccessKey, signed requests, and object data may be exposed over insecure links.

Remediation:

    1. Add Nginx / LB HTTPS in front.
    2. Allow RGW backend ports only for entry-level access.
    3. Expose only 443 to the public internet.
    4. Use formal certificates.
    5. Verify S3 client HTTPS access.
    6. Check if old HTTP endpoints are still accessible.
    7. Notify business to switch endpoint.

---

### 19.3 Discovery of Kubernetes Users Can Read ceph-csi Secret

Risk:

    Users may obtain Ceph key, bypassing Kubernetes to directly access Ceph.

Remediation:

    1. Check Kubernetes RBAC.
    2. Restrict regular users from reading ceph-csi Namespace.
    3. Restrict get/list/watch secret permissions.
    4. Set minimal permissions for business Namespace.
    5. Rotate Ceph CSI user key if necessary.
    6. Rebuild Secret and restart CSI components.

---

### 19.4 Discovery of Backup Directory with Wide keyring Permissions

Risk:

    Any local user can read Ceph key.

Remediation:

    chmod -R go-rwx /backup/ceph/auth
    chmod -R go-rwx /backup/ceph/config
    chmod 600 /backup/ceph/auth/*/*.keyring

Check:

    find /backup/ceph -type f -name "*.keyring" -exec ls -l {} \;

---

## TwentyI don't know.Key Rotation Strategy

### 20.1 Why Key Rotation is Needed

Key rotation is used to reduce leakage risks.

Common triggers:

- Employee departure
- Suspected key leakage
- Business decommissioning
- Regular security requirements
- Accidental Git submission
- Excessive sharing scope
- Permission model adjustments

---

### 20.2 CephX User Key Rotation

Check user:

    ceph auth get client.rbd-secure-user

Recreating key requires careful planning.

Production recommended process:

    1. Create new user or key.
    2. Switch business configuration to new user.
    3. Verify business read/write functionality.
    4. Observe for a period.
    5. Disable or delete old user.
    6. Clean up old keyring.
    7. Update documentation and CMDB.

Do not delete old key before verifying business configuration.

---

### 20.3 RGW S3 Key Rotation

Production recommended process:

    1. Add a new key for the user.
    2. Switch business configuration to new key.
    3. Verify upload/download.
    4. Observe business stability.
    5. Delete old key.
    6. Update key management records.

Check user:

    radosgw-admin user info --uid=app-s3-user

Create new key:

    radosgw-admin key create --uid=app-s3-user --key-type=s3

Delete old key:

    radosgw-admin key rm --uid=app-s3-user --key-type=s3 --access-key=<old-access-key>

High-risk warning:

    Must confirm business has switched before deleting old key.
    Otherwise, it will cause business access to object storage failure.

---

## Twenty-oneI don't know.Production Security Baseline

### 21.1 Account and Permissions

Production recommendation:

- Prohibit business use of client.admin.
- RBD, CephFS, RGW, CSI should use dedicated users respectively.
- Limit user permissions to specified Pool / FS / Bucket.
- Regularly clean up unused users.
- Regularly review allow * permissions.
- Rotate important keys regularly.

---

### 21.2 Network and Entry Points

Production recommendations:

- Do not expose Ceph internal ports to the public internet.
- MON / OSD should only allow access from trusted nodes.
- Dashboard should only allow access from the operations network segment.
- RGW must use HTTPS for external access.
- Backend RGW HTTP ports should only allow access from the entry layer.
- Kubernetes node access to Ceph ports must be explicitly allowed.
- Firewall and security group rules should be incorporated into change management.

---

### 21.3 Dashboard

Production recommendations:

- Enable HTTPS.
- Use official certificates or enterprise CA.
- Use strong passwords.
- Minimize administrator accounts.
- Restrict access sources.
- RemoveSeparations personnel accounts promptly.
- Dashboard should not be directly exposed to the public internet.

---

### 21.4 RGW

Production recommendations:

- Use a unified HTTPS endpoint externally.
- Do not share AccessKey.
- Set quotas for users and Buckets.
- Rotate AccessKey regularly.
- Suspend or delete obsolete users promptly.
- Incorporate Bucket lifecycle and access policies into governance.
- Monitor 4xx / 5xx / latency / capacity.

---

### 21.5 Kubernetes CSI

Production recommendations:

- CSI should use a dedicated Ceph user.
- Secrets should be placed in the ceph-csi Namespace.
- Ordinary users should not read CSI Secrets.
- StorageClass modifications should follow change procedures.
- Confirm data retention policies before deleting PVCs.
- Use fixed version images and internal repositories.
- Do not arbitrarily modify containerd configurations for image pulls.

---

### 21.6 Backup and Keys

Production recommendations:

- Encrypt and save auth export.
- Strictly protect cephadm SSH private keys.
- Minimize backup directory permissions.
- Do not commit backup files to Git.
- Use desensitized or test keys for recovery drills.
- Have a key rotation process in case of key leaks.

---

## Twenty-two, Common Security Risks and Handling

### 22.1 Risk One: Business servers have admin key

Check:

    find /etc/ceph -type f -name "*admin*"

Handling:

    Create dedicated business users.
    Switch business configurations.
    Remove admin key from business servers.
    Check for leakage risks.

---

### 22.2 Risk Two: RGW HTTP is open to the public

Check:

    curl -I http://InternetIP:7480

Handling:

    Disable public access.
    Configure HTTPS unified entry point.
    Firewall restricts 7480 sources.
    Notify business to switch endpoint.

---

### 22.3 Risk Three: Secret is readable by ordinary users

Check:

    kubectl auth can-i get secret -n ceph-csi --as=<user>

Handling:

    Adjust RBAC.
    Limit ceph-csi Namespace.
    Rotate Secret if necessary.

---

### 22.4 Risk Four: Backup keyring has overly broad permissions

Check:

    find /backup/ceph -type f \( -name "*.keyring" -o -name "*secret*" \) -exec ls -l {} \;

Handling:

    chmod 600 <file>
    chmod -R go-rwx <directory>

---

### 22.5 Risk Five: Obsolete RGW users not cleaned up

Check:

    radosgw-admin user list

Handling:

    Confirm user ownership.
    Suspend if no business usage.
    Delete after observation.
    Confirm Bucket and object ownership before deletion.

---

## Twenty-three, Security Inspection Script Example

Recommended path:

    05-Storage/01-Ceph/scripts/ceph-security-check.sh

Script content:

    #!/usr/bin/env bash

    set -euo pipefail

    echo "===== Ceph Security Check ====="
    echo

    echo "===== 1. Auth Config ====="
    ceph config dump | grep auth || true
    echo

    echo "===== 2. Ceph Users ====="
    ceph auth ls
    echo

    echo "===== 3. Keyring Files ====="
    find /etc/ceph -type f -name "*.keyring" -exec ls -l {} \; || true
    echo

    echo "===== 4. Cephadm SSH Key ====="
    ls -l /etc/ceph/cephadm-ssh-key /etc/ceph/ceph.pub 2>/dev/null || true
    echo

    echo "===== 5. Dashboard Services ====="
    ceph mgr services || true
    echo

    echo "===== 6. RGW Services ====="
    ceph orch ps --daemon_type rgw || true
    echo

    echo "===== 7. Listening Ports ====="
    ss -lntp || true
    echo

    echo "===== 8. Backup Sensitive Files ====="
    find /backup/ceph -type f \( -name "*.keyring" -o -name "*secret*" -o -name "*auth*" \) -exec ls -l {} \; 2>/dev/null || true
    echo

    echo "===== Security Check Finished ====="

Execution:

    chmod +x ceph-security-check.sh
    ./ceph-security-check.sh

Notes:

    This script only performs read-only checks.
    Does not automatically modify permissions.
    Does not automatically delete users.
    Does not automatically rotate keys.

---

## Twenty-four, Production Change Considerations

The following operations must go through a change process:

- Delete CephX user
- Modify CephX user caps
- Rotate Kubernetes CSI key
- Rotate RGW AccessKey
- Modify Dashboard certificate
- Modify RGW HTTPS entry
- Modify firewall rules
- Modify Ceph internal communication mode
- Delete old keyring
- Delete RGW user or Bucket
- Modify StorageClass Secret
- Change cephadm SSH key

Before making changes, you must confirm:

    Which business will be affected?
    Is there a rollback plan?
    Is there a maintenance window?
    Is there a backup?
    Is there a verification command?
    Has the business been notified?
    Is there someone for review?

---

## 25. Advanced SRE Methodology

### 25.1 Security is not the final patch, but daily governance

Ceph security should not wait until problems occur.

You should consider security in the following stages:

    Deployment planning stage
    Pool design stage
    RBD / CephFS / RGW user creation stage
    Kubernetes CSI integration stage
    Dashboard enablement stage
    RGW exposure stage
    Backup recovery stage
    Daily inspection stage

---

### 25.2 Least privilege is the core principle

Incorrect approach:

    All business uses client.admin.
    All systems share a group of S3 keys.
    All Kubernetes clusters share a Ceph user.
    All secrets are readable by everyone.

Correct approach:

    One business, one user.
    One purpose, one user.
    Permissions limited to Pool / FS / Bucket.
    Disable unused users promptly.
    After leakage, partial rotation can be done without affecting the whole system.

---

### 25.3 Network isolation is more fundamental than simple encryption

Encryption is important, but network isolation is more fundamental.

Ceph internal ports should not be exposed to irrelevant networks.

Principles:

    Expose as little as possible.
    If exposure is necessary, restrict sources.
    External entry must use HTTPS.
    Management entry must use the operation network segment, VPN, or bastion host.

---

### 25.4 Key leakage must have a contingency plan

In production, you must know:

    Which systems use this key?
    How to generate a new key?
    How to switch business?
    How to verify business normalcy?
    How to delete old keys?
    How to determine if it has been abused?

A key without a rotation plan is a long-term risk.

---

## 26. Interview Answering Strategy

If asked in an interview:

    How to secure a Ceph cluster?

You can answer:

    I will strengthen Ceph security from five aspects: network, authentication, permissions, entry, and keys.
    At the network level, Ceph's MON, OSD, MGR, Dashboard, etc., internal ports should not be exposed to the public internet and should only allow Ceph nodes, Kubernetes nodes, or specified clients to access. RGW should be exposed through Nginx, HAProxy, or LB with a unified HTTPS entry, and the backend RGW HTTP port should only allow access from the entry layer.
    At the authentication level, Ceph should enable CephX and not disable auth. Business-side should not use client.admin, but instead create dedicated users for RBD, CephFS, and Kubernetes CSI, and limit permissions to specific Pools or CephFS.
    At the permission level, RBD users should only access specific RBD Pools; CephFS users should only access specific fs; Kubernetes RBD CSI and CephFS CSI should use dedicated CephX users; ordinary Kubernetes users should not read secrets in the ceph-csi Namespace.
    At the entry level, Dashboard should enable HTTPS, strong passwords, and restrict access to the operation network segment; RGW external access must use HTTPS and manage AccessKey, SecretKey, Bucket quotas, and user lifecycle.
    At the key level, keyring, S3 keys, Kubernetes Secrets, cephadm SSH private keys, and auth backups are all sensitive information that should not be submitted to Git, should not be transmitted in plaintext, backups should have restricted permissions, and key rotation and obsolete user cleanup processes should be established.

---

## 27. Summary of This Article

This article mainly organizes Ceph security reinforcement methods:

1. Ceph security needs to be governed from five layers: network, authentication, permissions, entry, and keys.
2. CephX is Ceph's authentication mechanism and should not be disabled in production.
3. client.admin is suitable only for management and not for long-term business use.
4. RBD should create dedicated users restricted to specific Pools.
5. CephFS should create dedicated users restricted to specific file systems.
6. Kubernetes CSI should use dedicated CephX users and not use admin.
7. Kubernetes Secrets are not encrypted storage, and RBAC must restrict read permissions.
8. RGW AccessKey / SecretKey are equivalent to access credentials and must be managed securely.
9. RGW external access should use HTTPS unified entry.
10. Dashboard should enable HTTPS, strong passwords, and restrict access sources.
11. Ceph MON / OSD / MGR ports should not be exposed to the public internet.
12. cephadm SSH private keys must be strictly protected.
13. auth export, keyring, Secret YAML are all sensitive backup content.
14. Security inspections should regularly check users, permissions, ports, Secrets, and backup permissions.
15. Advanced SRE should establish mechanisms for least privilege, network isolation, key rotation, and security review.

---

## 28. Reference Documents

Ceph user management:

    https://docs.ceph.com/en/latest/rados/operations/user-management/

CephX configuration reference:

    https://docs.ceph.com/en/latest/rados/configuration/auth-config-ref/

Ceph Dashboard documentation:

    https://docs.ceph.com/en/latest/mgr/dashboard/

Cephadm operations documentation:

    https://docs.ceph.com/en/latest/cephadm/operations/

Ceph RGW management documentation:

    https://docs.ceph.com/en/latest/radosgw/admin/

Ceph RGW Frontends:

    https://docs.ceph.com/en/latest/radosgw/frontends/

Ceph RGW S3 user management:

    https://docs.ceph.com/en/latest/radosgw/adminops/

CephFS client permissions:

    https://docs.ceph.com/en/latest/cephfs/client-auth/

Ceph CSI project:

    https://github.com/ceph/ceph-csi

Kubernetes Secret documentation:

    https://kubernetes.io/docs/concepts/configuration/secret/

Kubernetes RBAC documentation:

    https://kubernetes.io/docs/reference/access-authn-authz/rbac/