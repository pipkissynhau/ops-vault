# Ceph Security Reinforcement: CephX Permissions, Network Isolation, Dashboard, RGW HTTPS, and Key Management

Recommended Path: 05-Storage/01-Ceph/18-Ceph Security Reinforcement: CephX Permissions, Network Isolation, Dashboard, RGW HTTPS, and Key Management.md

Tags: #Ceph #SecurityReinforcement #CephX #PermissionManagement #Dashboard #RGW #HTTPS #NetworkIsolation #KeyManagement #SRE #AdvancedSRE

---

## I. Document Overview

This article is the eighteenth in the Ceph Advanced SRE storage series, focusing on methods to strengthen the security of Ceph clusters.

Previously covered topics include:

- Ceph cluster deployment
- OSD management
- Pool and PG configuration
- CRUSH fault domains
- RBD block storage practices
- CephFS file storage practices
- RGW object storage practices
- Kubernetes RBD CSI
- Kubernetes CephFS CSI
- Ceph daily operations and maintenance
- Ceph troubleshooting
- Ceph backup and recovery
- Ceph performance optimization

This article now moves onto the topic of security management.

As a distributed storage system, Ceph's main security risks include:

- Whether cluster communications are exposed.
- Whether client permissions are too broad.
- Whether the admin key is misused.
- Whether the Dashboard is accessible from the public internet.
- Whether RGW uses HTTPS.
- Whether S3 AccessKeys have been leaked.
- Whether Kubernetes Secrets are set with overly wide access levels.
- Whether OSD/MON/MGR ports are accessed by unrelated hosts.
- Whether the cephadm SSH key is secure.
- Whether backup files contain sensitive keys.

This article covers the following key areas:

- Understanding Ceph's security boundaries.
- The CephX authentication mechanism.
- Designing最小权限 users.
- Controlling RBD user permissions.
- Controlling CephFS user permissions.
- Managing RGW S3 users, AccessKeys, and SecretKeys.
- Configuring Dashboard HTTPS and administrator accounts.
- Providing a unified HTTPS entry point for RGW through Nginx/LB.
- Restricting access to Ceph MON/OSD/MGR/RGW ports.
- Protecting the cephadm SSH private key and keyring.
- Identifying sensitive files and Secrets.
- Establishing a Ceph security inspection checklist.
- Implementing key rotation and permission review processes.
- Understanding the baseline for securing Ceph in production environments.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand Ceph's main security boundaries.
2. Comprehend the role of CephX.
3. Avoid abusing the `client.admin` account in production scenarios.
4. Create minimum permission users for RBD.
5. Create minimum permission users for CephFS.
6. Establish dedicated users for Kubernetes CSI.
7. Manage RGW S3 users, AccessKeys, and SecretKeys.
8. Configure Dashboard HTTPS and administrator accounts.
9. Provide a unified HTTPS entry point for RGW through Nginx/LB.
10. Limit access to Ceph MON/OSD/MGR/RGW ports.
11. Protect the cephadm SSH private key and keyring.
12. Identify sensitive files and Secrets.
13. Develop a Ceph security inspection routine.
14. Establish key rotation and permission review processes.
15. Understand the baseline for securing Ceph in production environments.

---

## III. Experimental and Production Environments

### 3.1 Ceph Cluster Nodes

This article uses the same experimental environment as the Ceph module setup.

| IP | Host Name | Role |
|---|---|---|
| 10.0.0.31 | ceph-node01 | MON/MGR/OSD/RGW/MDS |
| 10.0.0.32 | ceph-node02 | MON/MGR/OSD/RGW/MDS |
| 10.0.0.33 | ceph-node03 | MON/MGR/OSD/RGW |
| 10.0.0.34 | ceph-node04 | OSD / Expansion/Fault Testing (optional) |
| 10.0.0.35 | ceph-client | RBD/CephFS/RGW Client Testing (optional) |
| 10.0.0.36 | rgw-lb/nginx | RGW HTTPS Unified Entry Point (optional) |

Primary Experimental System:

    Ubuntu Server 22.04.5 LTS

Additional System:

    Rocky Linux 9

Deployment Method:

    cephadm

---

### 3.2 Kubernetes Cluster

If using Kubernetes CSI:

| IP | Host Name | Role |
|---|---|---|
| 10.0.0.20 | k8s-master | Kubernetes Master |
| 10.0Example Pool:

    rbd-secure-pool

Creation:

    ceph osd pool create rbd-secure-pool 64
    ceph osd pool application enable rbd-secure-pool rbd
    ceph osd pool set rbd-secure-pool size 3
    ceph osd pool set rbd-secure-pool min_size 2
    rbd pool init rbd-secure-pool

---

### 7.2 Creating a Dedicated RBD User

Creating the user:

    ceph auth get-or-create client.rbd-secure-user \
      mon 'profile rbd' \
      mgr 'profile rbd pool=rbd-secure-pool' \
      osd 'profile rbd pool=rbd-secure-pool' \
      -o /etc/ceph/ceph.client.rbd-secure-user.keyring

Checking:

    ceph auth get client.rbd-secure-user

Expected Output:

    caps mon = "profile rbd"
    caps mgr = "profile rbd pool=rbd-secure-pool"
    caps osd = "profile rbd pool=rbd-secure-pool"

---

### 7.3 Verifying RBD User Permissions

Using this user to create an Image:

    rbd --id rbd-secure-user -p rbd-secure-pool create secure-image --size 1G

Checking the Image:

    rbd --id rbd-secure-user -p rbd-secure-pool ls

Viewing Image Details:

    rbd --id rbd-secure-user info rbd-secure-pool/secure-image

---

### 7.4 Verifying Inability to Access Other Pools

If there are other RBD Pools, for example:

    rbd-pool

Attempting access:

    rbd --id rbd-secure-user -p rbd-pool ls

Expected Outcome:

    Insufficient permissions or access failure

Explanation:

    The client.rbd-secure-user user is restricted to the rbd-secure-pool.
    This demonstrates the value of minimal permissions.

---

### 7.5 Deleting Test Resources

    rbd --id rbd-secure-user rm rbd-secure-pool/secure-image

High-Risk Warning:

    The `rbd rm` command will delete the Image.
    Ensure it is a test Image before executing this command.

---

## Section Eight: CephFS Minimum Permission User

### 8.1 Prerequisites

A CephFS cluster already exists:

    cephfs

Checking:

    ceph fs ls
    ceph fs status
    ceph mds stat

---

### 8.2 Creating a Dedicated CephFS User

Creating the user:

    ceph auth get-or-create client.cephfs-secure-user \
      mon 'allow r' \
      mds 'allow rw fsname=cephfs' \
      osd 'allow rw tag cephfs data=cephfs' \
      -o /etc/ceph/ceph.client.cephfs-secure-user.keyring

Checking:

    ceph auth get client.cephfs-secure-user

---

### 8.3 Extracting the Secret Key

    ceph auth get-key client.cephfs-secure-user > /etc/ceph/cephfs-secure-user.secret

Setting permissions:

    chmod 600 /etc/ceph/cephfs-secure-user.secret

---

### 8.4 Mounting CephFS Using the Dedicated User

Creating a mount directory:

    mkdir -p /mnt/cephfs-secure

Mounting:

    mount -t ceph 10.0.0.31,10.0.0.32,10.0.0.33:/ /mnt/cephfs-secure \
      -o name=cephfs-secure-user,secretfile=/etc/ceph/cephfs-secure-user.secret,fs=cephfs

Verification:

    echo "cephfs secure user test" > /mnt/cephfs-secure/secure-test.txt
    cat /mnt/cephfs-secure/secure-test.txt

Unmounting:

    umount /mnt/cephfs-secure

---

### 8.5 Subdirectory Permission Control Philosophy

In production, it is not recommended that all services mount the CephFS root directory.

It is more advisable to use subdirectories such as:

    /apps/app01
    /apps/app02
    /data/team-a
    /data/team-b

And combine this with:

- CephX path restrictions, depending on the version and requirements
- Linux file permissions
- ACLs
- Kubernetes subdirectory isolation
- SubvolumeGroup/Subvolume management

The principle is to ensure that each service only has access to the directories it needs.

---

## Section Nine: Kubernetes CSI User Security

### 9.1 RBD CSI Users

ForDo not retain keys for users who do not use them for an extended period. Immediately disable or rotate them upon discovery of any leaks. Regularly check the ownership of users and buckets.

---

### 10.4 Setting Quotas for Users

User-level quotas:

    radosgw-admin quota set \
      --quota-scope=user \
      --uid=app-s3-user \
      --max-size=50G \
      --max-objects=100000

Enabling:

    radosgw-admin quota enable \
      --quota-scope=user \
      --uid=app-s3-user

Checking:

    radosgw-admin user info --uid=app-s3-user | jq '.user_quota'

---

### 10.5 Disabling Users

If a user's account is compromised or becomes obsolete:

    radosgw-admin user suspend --uid=app-s3-user

Reactivating:

    radosgw-admin user enable --uid=app-s3-user

Deleting a user:

    radosgw-admin user rm --uid=app-s3-user

Permanently deleting (high-risk):

    radosgw-admin user rm --uid=app-s3-user --purge-data

High-risk warning:

    --purge-data will delete all data associated with the user.
    In a production environment, confirm the ownership of buckets and objects before proceeding.

---

## Chapter Eleven: Unified RGW HTTPS Access

### 11.1 Security Boundaries

For experimental environments, you can use:

    http://10.0.0.31:7480

For production external access, it is necessary to use:

    https://s3.example.com

Recommended architecture:

    Client
      |
      | HTTPS 443
      v
    Nginx / HAProxy / LB
      |
      | HTTP 7480, within a trusted internal network
      v
    RGW backend instances

Explanation:

    For internal trusted networks, HTTP can be used to reduce complexity and TLS overhead.
    For public or untrusted networks, HTTPS must be used.
    If corporate compliance requires encryption for internal links as well, consider evaluating TLS at the RGW backend or full-link TLS.

---

### 11.2 Nginx HTTPS Example

Configuration directory:

    mkdir -p /etc/nginx/ssl

Place official certificates:

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

Check configuration:

    nginx -t

Reload configuration:

    systemctl reload nginx

---

### 11.3 Verifying the HTTPS Access

Configure DNS or hosts on the client side:

    10.0.0.36 s3.example.com

Test access:

    curl -I https://s3.example.com

Using AWS CLI:

    export RGW_ENDPOINT="https://s3.example.com"

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 ls

---

### 11.4 Security Inspection of HTTPS Access

Inspection items:

| Item | Requirement |
|---|---|
| Certificate | Use a formal CA or an enterprise-approved CA |
| Private key permissions | Only root should have read access |
| Port 80 | Should redirect to HTTPS |
| Port 443 | Must be open to external access |
| RGW port 7480 | Access should be limited to internal networks or the entry layer |
| Host headers || Dashboard | 8443, varies by configuration |
| Prometheus mgr module | 9283, varies by configuration |
| RGW | 7480, varies by configuration |
| SSH | 22 |

---

### 13.2 Access Control Principles

| Source of Access | Allowed Access |
|---|---|
| Between Ceph Nodes | MON / OSD / MGR / SSH |
| Kubernetes Nodes | MON / OSD |
| RBD Clients | MON / OSD |
| CephFS Clients | MON / OSD / MDS-related communications |
| RGW Entrance Layer | RGW backend ports |
| Ordinary Office Network | Should not directly access OSD / MON |
| Public Network | Should not directly access MON / OSD / Dashboard / RGW HTTP |

---

### 13.3 UFW Examples

If using ufw on Ubuntu, refer to the following examples.

To allow Ceph nodes to access MON:

    ufw allow from 10.0.0.0/24 to any port 3300 proto tcp
    ufw allow from 10.0.0.0/24 to any port 6789 proto tcp

To allow OSD ports:

    ufw allow from 10.0.0.0/24 to any port 6800:7300 proto tcp

To restrict RGW backend access to only the internal network:

    ufw allow from 10.0.0.0/24 to any port 7480 proto tcp

To allow the Dashboard to be accessed only by the operations network segment:

    ufw allow from 10.0.0.0/24 to any port 8443 proto tcp

To check the current rules:

    ufw status numbered

---

### 13.4 firewalld Examples

For Rocky Linux 9, use firewalld instead.

To allow MON access:

    firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.0.0.0/24" port protocol="tcp" port="3300" accept'
    firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.0.0.0/24" port protocol="tcp" port="6789" accept'

To allow OSD access:

    firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.0.0.0/24" port protocol="tcp" port="6800-7300" accept'

To reload the rules:

    firewall-cmd --reload

To list all current rules:

    firewall-cmd --list-all

---

### 13.5 Production Network Recommendations

For production environments, it is recommended to:

- Not expose the Ceph cluster network directly to the public internet.
- Only allow trusted clients to access MON / OSD ports.
- Place the Dashboard in the operations network segment or behind a VPN.
- Expose only the HTTPS port for RGW externally.
- Not expose the RGW HTTP port directly to the public internet.
- Clearly permit Kubernetes nodes to access Ceph ports.
- Keep records of security groups, firewalls, and network ACL settings.
- Verify PVC mounting, RBD, CephFS, and RGW access before making any changes to the firewall configuration.

---

## Chapter Fourteen: Security of cephadm SSH Keys

### 14.1 Role of cephadm SSH Keys

cephadm manages cluster nodes via SSH.

Common files involved include:

    /etc/ceph/ceph.pub
    /etc/ceph/cephadm-ssh-key

Where:

    ceph.pub is the public key
    cephadm-ssh-key is the private key

The private key must be kept secure at all times.

---

### 14.2 Checking File Permissions

Use the following commands to check file permissions:

    ls -l /etc/ceph/ceph.pub
    ls -l /etc/ceph/cephadm-ssh-key

It is recommended to set the permissions as follows:

    chmod 600 /etc/ceph/cephadm-ssh-key
    chmod 644 /etc/ceph/ceph.pub

---

### 14.3 Risk Assessment

If the cephadm private key gets leaked, an attacker could use it to log in to Ceph nodes.

Potential risks include:

- Gaining control over Ceph nodes
- Modifying services
- Deleting data
- Stealing the keyring
- Potential for lateral movement within the network

Production recommendations include:

- Restricting access to the private key.
- Encrypting the backup of the private key.
- Not submitting it to Git.
- Not sharing it via regular file transfer methods.
- Ensuring that former employees- Whether to use strong passwords
- Whether there are redundant users
- Whether access sources are restricted

---

### 18.4 RGW Inspection

    ceph orch ps --daemon_type rgw
    radosgw-admin user list
    radosgw-admin bucket list

Inspection:

- Whether there are obsolete users
- Whether users have not set quotas
- Whether AccessKeys have not been rotated for a long time
- Whether the RGW HTTP port is exposed to the public network
- Whether HTTPS is used externally

---

### 18.5 Kubernetes Secret Inspection

    kubectl get secret -n ceph-csi
    kubectl get sa -n ceph-csi
    kubectl get role,rolebinding -n ceph-csi
    kubectl get clusterrole,clusterrolebinding | grep ceph

Inspection:

- Whether the Secret is in the correct Namespace
- Whether ordinary users can read the ceph-csi Secret
- Whether CSI permissions are too broad
- Whether StorageClasses have been arbitrarily modified

---

### 18.6 Network Port Inspection

On Ceph nodes:

    ss -lntp

Inspection:

- MON 3300 / 6789
- OSD 6800-7300
- Dashboard 8443
- RGW 7480
- Prometheus 9283
- SSH 22

Confirmation:

- Which ports are listening on 0.0.0.0.
- Which ports require firewall restrictions.
- Which ports should not be exposed to the public network.

---

## Chapter Nineteen: Examples of Security Remediation

### 19.1 Discovery that a Business is Using client.admin

Risk:

- The business has cluster administrator privileges.

Remediation:

- Create a dedicated user for the business.
- Limit access to specific Pools or CephFS.
- Modify the business configuration to use the new user.
- Verify that the business can read and write normally.
- Remove the admin keyring from the business server.
- Check for any risks of admin keys being leaked.
- Rotate the admin key if necessary, with careful planning.

---

### 19.2 Discovery that RGW HTTP is Exposed to the Public Network

Risk:

- AccessKeys, signed requests, and object data may be exposed through insecure links.

Remediation:

- Add an Nginx / LB HTTPS layer in front.
- Allow only entry-level access to the RGW backend ports.
- Expose only port 443 to the public network.
- Use official certificates.
- Verify HTTPS access for S3 clients.
- Check if old HTTP entries are still accessible.
- Notify the business to switch endpoints.

---

### 19.3 Discovery that Kubernetes Users Can Read ceph-csi Secrets

Risk:

- Users may obtain Ceph keys and bypass Kubernetes to directly access Ceph.

Remediation:

- Check Kubernetes RBAC settings.
- Limit ordinary users from reading the ceph-csi Namespace.
- Restrict get/list/watch secret permissions.
- Set minimum permissions for business Namespaces.
- Rotate Ceph CSI user keys if necessary.
- Re-create Secrets and restart CSI components accordingly.

---

### 19.4 Discovery that Backup Directory Keyring Permissions are Too Broad

Risk:

- Any local user can read Ceph keys.

Remediation:

    chmod -R go-rwx /backup/ceph/auth
    chmod -R go-rwx /backup/ceph/config
    chmod 600 /backup/ceph/auth/*/*.keyring

Inspection:

    find /backup/ceph -type f -name "*.keyring" -exec ls -l {} \;

---

## Chapter Twenty: Key Rotation Best Practices

### 20.1 Why Key Rotation is Needed

Key rotation helps reduce the risk of leaks.

Common triggers for rotation:

- Staff turnover
- Suspected key leakage
- Business decommissioning
- Regular security requirements
- Accidental Git commits
- Excessive sharing scope
- Changes in permission models

---

### 20.2 CephX User Key Rotation

To view a user:

    ceph auth get client.rbd-secure-user

The process of generating new keys needs careful planning.

Recommended production procedure:

1. Create a new user or key.
2. Switch the business configuration to the new user.
3. Verify that the business can read and write normally.
4. Monitor for a period of time.
5. Disable or delete the old user.
6. Clean up the old keyring.
7. Update documentation and CMDB records.

It is not recommended to delete the old key before verifying that the business configuration has been switched.

---

### 20.3 RGW S3 Key Rotation

Recommended production procedure:

1. Add a new set of keys for the user.
2. Switch the business configuration to the new keys.
3. Verify```bash
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
ls -l /etc/ceph/cephADM-ssh-key /etc/ceph/ceph.pub 2>/dev/null || true
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

执行：

    chmod +x ceph-security-check.sh
    ./ceph-security-check.sh
```

**Instructions:**

This script performs only read-only checks. It does not automatically modify permissions, delete users, or rotate keys.
```