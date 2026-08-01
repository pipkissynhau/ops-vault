# Interview Question 44: etcd Distributed Deployment Notes (Three-Node Independent Cluster)

## Tags
#etcd #DistributedStorage #Raft #HighAvailable #Kubernetes #ControlPlane #Interview #Transport

## I. Interview Question

Interview Question:  
Please explain how to deploy an etcd distributed cluster? Why is it generally deployed with 3 nodes? What parameters and risk points need to be considered during deployment?

---

## II. What is etcd

etcd is a distributed key-value storage system, commonly used to store cluster configuration data, service discovery information, and status data.  
In Kubernetes, etcd is the core data storage of the control plane, and the state of all objects in the cluster will eventually be written to etcd.

You can understand it as:

- K8s's "database"
- The "state storage center" of the control plane
- A distributed storage system implemented based on the Raft protocol

---

## III. Why Distributed Deployment is Needed

The problems of a single-machine etcd are obvious:

- Single point of failure
- Data service unavailable after node failure
- Unable to meet the high availability requirements of production environments

Therefore, production environments generally deploy as an odd-numbered node cluster:

- 3 nodes
- 5 nodes
- Larger clusters are rare

### Why 3 Nodes is Common

Because 3 nodes are the most common balance point between high availability and resource cost:

- 3 nodes can tolerate 1 node failure
- 5 nodes can tolerate 2 node failures
- The more nodes, the higher the Raft replication and arbitration cost during writing

Conclusion:

**Most production environments prioritize 3-node deployment for independent etcd clusters.**

---

## IV. Core Principles of etcd Distributed Deployment

etcd uses the Raft protocol to ensure data consistency among multiple nodes.

There are three types of roles in the cluster, which can be understood:

- Leader: The leader, responsible for handling write requests and replicating logs
- Follower: The follower, receiving data synchronization from the leader
- Candidate: Election state, participating in elections when the leader fails

### Write Process

When a client writes data:

1. The request first goes to the leader
2. The leader writes the change to its local log
3. The leader replicates the log to the majority of nodes
4. After the majority of nodes confirm, the write is considered successful
5. The leader then returns success to the client

This is why etcd is not just "able to store data," but more importantly, "strongly consistent."

---

## V. Three-Node Deployment Topology Example

Assume three servers:

- etcd-1: 192.168.10.11
- etcd-2: 192.168.10.12
- etcd-3: 192.168.10.13

It is recommended to configure the hostnames in advance:

- etcd-1
- etcd-2
- etcd-3

It is recommended to write hosts resolution on all three machines:

    192.168.10.11 etcd-1
    192.168.10.12 etcd-2
    192.168.10.13 etcd-3

---

## VI. Preparations Before Deployment

## 1. Basic Requirements

Need to confirm in advance:

- Three Linux hosts with synchronized time
- Fixed hostnames
- Network connectivity between nodes
- Firewall allows 2379 and 2380
- Disk performance should not be too poor, preferably using SSD
- etcd data directory should be planned separately

---

## 2. Port Explanation

Common etcd ports:

- 2379: Client access port
- 2380: Peer communication port between etcd nodes

You can understand it as:

- 2379 is for kube-apiserver or etcdctl access
- 2380 is for data synchronization between etcd cluster members

---

## 3. Directory Planning Example

Recommended unified planning:

    /opt/etcd/
    /etc/etcd/
    /var/lib/etcd/
    /etc/etcd/ssl/

Explanation:

- Program directory: /opt/etcd
- Configuration directory: /etc/etcd
- Data directory: /var/lib/etcd
- Certificate directory: /etc/etcd/ssl

---

## VII. Understanding Deployment Methods

etcd cluster startup has three common boot methods:

1. Static bootstrap
2. Discovery service bootstrap
3. DNS discovery

In enterprises, the most common, most controllable, and most suitable for interview answers is:

**Static bootstrap deployment.**

Because nodes, IPs, names, and initial member information are all pre-defined, it is the most intuitive and stable.

---

## VIII. Three-Node Static Bootstrap Deployment Approach

Install etcd binary on all three machines, and configure each node with:

- name
- data-dir
- listen-client-urls
- advertise-client-urls
- listen-peer-urls
- initial-advertise-peer-urls
- initial-cluster
- initial-cluster-state
- initial-cluster-token

The most critical ones are:

### 1. name
The current node name, must be consistent with the one defined in initial-cluster.

### 2. initial-cluster
Define all members of the cluster, for example:

    etcd-1=https://192.168.10.11:2380,etcd-2=https://192.168.10.12:2380,etcd-3=https://192.168.10.13:2380

### 3. initial-cluster-state
When deploying a new cluster, write:

    new

If it's an expansion or recovery scenario of an existing cluster, it's not arbitrary to always write new.

### 4. initial-cluster-token
Cluster identifier, used to distinguish different clusters, it's recommended to customize it, and not to share across multiple environments.

---

## IX. Certificate Preparation Approach

In production environments, it's recommended to enable TLS, at least including two types of communication encryption:

- Communication between client and etcd
- Peer communication between etcd nodes

Usually, you need to prepare:

- CA certificate
- Each node's service certificate and private key
- Optional client certificate

Certificates should include:

- Current node hostname
- Current node IP
- Possibly accessible VIP or domain name

Otherwise, certificate validation failures are likely to occur.

---

## X. systemd Management Deployment

In production environments, it's recommended to manage etcd processes with systemd instead of manually using nohup.

Benefits:

- Auto-start on boot
- Automatically restart if process fails
- Easier unified operations and maintenance
- Logs and service status are easier to troubleshoot

---

## XI. etcd-1 Node Configuration Example

The following is an example of systemd startup parameters for etcd-1:

    [Unit]
    Description=etcd
    Documentation=https://etcd.io
    After=network.target
    Wants=network-online.target

[Service]
Type=notify
ExecStart=/opt/etcd/etcd \
  --name=etcd-1 \
  --data-dir=/var/lib/etcd \
  --listen-client-urls=https://192.168.10.11:2379,https://127.0.0.1:2379 \
  --advertise-client-urls=https://192.168.10.11:2379 \
  --listen-peer-urls=https://192.168.10.11:2380 \
  --initial-advertise-peer-urls=https://192.168.10.11:2380 \
  --initial-cluster=etcd-1=https://192.168.10.11:2380,etcd-2=https://192.168.10.12:2380,etcd-3=https://192.168.10.13:2380 \
  --initial-cluster-token=etcd-cluster-01 \
  --initial-cluster-state=new \
  --client-cert-auth=true \
  --trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --cert-file=/etc/etcd/ssl/server.pem \
  --key-file=/etc/etcd/ssl/server-key.pem \
  --peer-client-cert-auth=true \
  --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-cert-file=/etc/etcd/ssl/server.pem \
  --peer-key-file=/etc/etcd/ssl/server-key.pem

Restart=always
RestartSec=10
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

---

## Twelve. How to Modify the Other Two Nodes

### Changes Required for etcd-2

Main modifications:

- --name=etcd-2
- Change the local client IP to 192.168.10.12
- Change the local peer IP to 192.168.10.12
- Replace the certificate with the corresponding one for this node

### Changes Required for etcd-3

Main modifications:

- --name=etcd-3
- Change the local client IP to 192.168.10.13
- Change the local peer IP to 192.168.10.13
- Replace the certificate with the corresponding one for this node

Note:

**The initial-cluster list of cluster members must be consistent across all three machines.**

---

## Thirteen. Startup Steps

After completing the configuration on all three machines, execute:

    systemctl daemon-reload
    systemctl enable etcd
    systemctl start etcd
    systemctl status etcd

If you're concerned about simultaneous startup making troubleshooting difficult, you can start them one by one, but the configuration must remain consistent.

---

## Fourteen. Cluster Health Check

After deployment, use etcdctl to check.

First, set the environment variable example:

    export ETCDCTL_API=3

Then check the health status:

    etcdctl \
      --endpoints=https://192.168.10.11:2379,https://192.168.10.12:2379,https://192.168.10.13:2379 \
      --cacert=/etc/etcd/ssl/ca.pem \
      --cert=/etc/etcd/ssl/client.pem \
      --key=/etc/etcd/ssl/client-key.pem \
      endpoint health

Check the member list:

    etcdctl \
      --endpoints=https://192.168.10.11:2379,https://192.168.10.12:2379,https://192.168.10.13:2379 \
      --cacert=/etc/etcd/ssl/ca.pem \
      --cert=/etc/etcd/ssl/client.pem \
      --key=/etc/etcd/ssl/client-key.pem \
      member list

Check detailed status:

    etcdctl \
      --endpoints=https://192.168.10.11:2379,https://192.168.10.12:2379,https://192.168.10.13:2379 \
      --cacert=/etc/etcd/ssl/ca.pem \
      --cert=/etc/etcd/ssl/client.pem \
      --key=/etc/etcd/ssl/client-key.pem \
      endpoint status -w table

Focus on:

- Whether all three nodes are healthy
- Whether a leader has been elected successfully
- Whether the raft term is normal
- Whether the version is consistent

---

## Fifteen. Common Issues and Troubleshooting

## 1. Cluster Fails to Start

Common causes:

- initial-cluster is incorrectly configured
- name is inconsistent with the definition in initial-cluster
- Communication failure on port 2380 between nodes
- SAN mismatch in certificates
- Old data exists in data-dir
- Incorrect use of initial-cluster-state

Troubleshooting steps:

    journalctl -u etcd -f
    systemctl status etcd

---

## 2. Certificate Errors

Common manifestations:

- x509 certificate signed by unknown authority
- certificate is valid for xxx, not yyy

Troubleshooting focus:

- Whether CA is consistent
- Whether the certificate is issued by the correct CA
- Whether SAN includes the current access IP/host name
- Whether peer certificate and client certificate parameters are correctly matched

---

## 3. Single Node Can Start, but Cluster Not Formed

Typically check:

- Whether port 2380 is reachable between nodes
- Whether peer URL uses the actual reachable address of this node
- Whether initial-cluster is completely consistent across all three machines
- Whether firewall or security group is blocking

---

## 4. Node Cannot Rejoin After Rebuilding

This type of issue is often due to:

- Old member not cleaned up completely
- New node's data-dir not cleared
- Member information inconsistent with the original cluster view

In such cases, you cannot simply "restart and try again"—you need to perform member management based on the member list.

---

## Sixteen. Production Environment Notes

## 1. Use Odd Number of Nodes

Recommended:

- 3 nodes
- 5 nodes

Not recommended to use 2 nodes or 4 nodes as arbitration clusters.

---

## 2. Do Not Overload etcd with High-Load Business

Reasons:

- etcd is sensitive to latency and disk performance
- High IO and CPU contention will affect write stability and election

---

## 3. Disk Must Be Stable, Prefer SSD

etcd is a strongly consistent storage system that is sensitive to fsync and disk latency.  
Poor disk performance will directly impact cluster performance and stability.

---

## 4. Time Synchronization Must Be Done Properly

Although etcd does not rely on "exactly synchronized time" for data consistency, severe time drift can affect operations troubleshooting, certificate validation, and overall system stability.

---

## 5. Regular Snapshots Must Be Taken

Snapshot backups are essential in production environments.

Example:

    etcdctl \
      --endpoints=https://192.168.10.11:2379 \
      --cacert=/etc/etcd/ssl/ca.pem \
      --cert=/etc/etcd/ssl/client.pem \
      --key=/etc/etcd/ssl/client-key.pem \
      snapshot save /backup/etcd-snapshot.db

This step is especially important before recovery and upgrades.

---

## 6. Monitoring Must Be Implemented

It is recommended to monitor at least:

- Leader change frequency
- Disk fsync latency
- WAL write latency
- Backend commit latency
- Database size
- Network round-trip latency
- Node health status

---

## Seventeen. Relationship with Kubernetes

If this is an interview note for a Kubernetes scenario, you can add:

In Kubernetes production environments, etcd has two common configurations:

1. Stacked etcd  
   etcd and control plane components are deployed on the same batch of master nodes

2. External etcd  
   etcd is deployed as a separate cluster, accessed remotely by kube-apiserver

Generally:

- Small-scale environments commonly use stacked etcd
- External etcd is chosen when emphasizing independence and high availability governance

---

## Eighteen. Interview Answer Template

If the interviewer asks: "How to deploy an etcd distributed cluster?"

You can answer:

etcd production environments typically use an odd number of nodes, most commonly three nodes, because three nodes achieve a balance between cost and high availability, and can tolerate one node failure. Deployment typically uses static bootstrap deployment. You need to plan each node's hostname, IP, 2379 and 2380 ports, data directory, and TLS certificates in advance. Each node must configure its own name, listen-client-urls, advertise-client-urls, listen-peer-urls, initial-advertise-peer-urls, and the initial-cluster member list must be consistent across all three machines. It is recommended to enable bidirectional TLS for client and peer in production environments, and manage the etcd service via systemd. After deployment, use etcdctl to check endpoint health, member list, and endpoint status to confirm three-node health and successful leader election. In production environments, pay special attention to disk performance, time synchronization, snapshot backups, and monitoring alerts.

---

## Nineteen. Commonly Follow-up Questions in Interviews

### 1. Why 3 nodes instead of 2?
Because etcd relies on majority arbitration, 2 nodes can only tolerate 0 node failures, which has no practical high availability meaning.

### 2. Why is the number of nodes recommended to be odd?
Because the arbitration mechanism is based on majority, odd-numbered nodes are more resource-efficient.

### 3. What's the difference between 2379 and 2380?
2379 is the client access port, 2380 is the port for peer-to-peer data synchronization.

### 4. When is initial-cluster-state=new used?
It is used when creating a new cluster for the first time; it is not always set to new in all scenarios.

### 5. How to confirm the cluster is healthy after deployment?
Use etcdctl to check endpoint health, member list, and endpoint status.

### 6. What is etcd most afraid of?
Common risks include high disk latency, network jitter, certificate issues, accidental data directory operations, and inconsistent member configuration.

---

## Twenty. Common Mistakes

### Mistake 1: name and initial-cluster are inconsistent
For example, if the local configuration is:

    --name=etcd01

But initial-cluster is written as:

    etcd-1=https://192.168.10.11:2380

This will cause member recognition issues.

---

### Mistake 2: Peer address written as 127.0.0.1
Peer communication must use the real address that other nodes can access, not just the local loopback address.

---

### Mistake 3: Old data directory not cleaned
If a previous cluster was started incorrectly, the data-dir already contains old metadata. Restarting directly with changed configurations can cause various strange issues.

---

### Mistake 4: Certificate SAN lacks IP or hostname
This is a very common pitfall in production environments.

---

### Mistake 5: Treating etcd as a regular middleware for arbitrary restarts
etcd is a core component of the Kubernetes control plane. Restarting and upgrading it requires considering arbitration and backups, not just stopping it arbitrarily.

---

## Twenty-one. Summary

The core of etcd distributed deployment is not "just running three processes," but the following points:

1. Clear node planning
2. Consistent static bootstrap parameters
3. Clear separation of client and peer addresses
4. Correct TLS certificates
5. Port accessibility
6. Independent data directory
7. Use etcdctl to verify health
8. Backup, monitoring, and recovery plans

One-sentence summary:

**etcd distributed deployment essentially builds a highly available cluster capable of majority arbitration and strong consistency storage, based on proper node planning, certificate system, and static bootstrap parameters.**

---

## Twenty-two. Reference Links

- etcd Clustering Guide  
  https://etcd.io/docs/v3.5/op-guide/clustering/

- etcd Transport Security Model  
  https://etcd.io/docs/v3.6/op-guide/security/

- etcd Configuration Options  
  https://etcd.io/docs/

- etcd Upgrade Guide  
  https://etcd.io/docs/v3.6/upgrades/upgrade_3_6/

## Twenty-three. Memorization Version

Memory mnemonic:

**Three nodes, odd number of nodes; 2379 for clients, 2380 for node-to-node; name matches, cluster must be consistent; first certificates, then startup; first health check, then business access.**