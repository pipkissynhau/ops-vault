# Interview Question 44: Notes on etcd Distributed Deployment (Three-Node Independent Cluster)

## Tags
#etcd #Distributed Storage #Raft #High Availability #Kubernetes #Control Plane #Interview Questions #Operations and Maintenance

## I. Interview Question

Question:  
Please explain how to deploy an etcd distributed cluster. Why are 3 nodes generally deployed? What parameters and risk points should be considered during deployment?

---

## II. What is etcd

etcd is a distributed key-value storage system commonly used to store cluster configuration data, service discovery information, and status data.  
In Kubernetes, etcd serves as the core data storage for the control plane, and the status of objects in the cluster is ultimately written to etcd.

It can be understood as follows:

- The “database” of K8s
- The “status storage center” of the control plane
- A distributed storage system that achieves consistency based on the Raft protocol

---

## III. Why Distributed Deployment?

The problems with a single-machine etcd are obvious:

- Single-point failure
- Data services become unavailable after a node crashes
- Unable to meet the high availability requirements of production environments

Therefore, in production environments, clusters with an odd number of nodes are generally deployed:

- 3 nodes
- 5 nodes
- Larger-scale deployments are less common

### Why 3 Nodes Are Common?

Because 3 nodes represent the most common balance between high availability and resource costs:

- A cluster with 3 nodes can tolerate the failure of 1 node.
- A cluster with 5 nodes can tolerate the failure of 2 nodes.
- The more nodes there are, the higher the cost of Raft replication and arbitration during data writes.

Conclusion:  
**In most production environments, a three-node etcd independent cluster is preferred.**

---

## IV. Core Principles of etcd Distributed Deployment

etcd uses the Raft protocol to ensure data consistency among multiple nodes.

It is sufficient to understand three types of roles in the cluster:

- Leader: The leader node, responsible for processing write requests and replicating logs.
- Follower: The follower nodes, which receive data synchronized from the leader.
- Candidate: Nodes in the election state, participating in the election when the leader fails.

### Writing Process

When a client writes data:

1. The request first reaches the leader node.
2. The leader node writes the changes to its local log.
3. The leader node replicates the log to the majority of nodes.
4. Only after the majority of nodes confirm it does the write consider successful.
5. The leader node then returns a success response to the client.

This is why etcd is not just “capable of storing data” but also emphasizes **strong consistency**.

---

## V. Example of Three-Node Deployment Topology

Assume there are three servers:

- etcd-1: 192.168.10.11
- etcd-2: 192.168.10.12
- etcd-3: 192.168.10.13

It is recommended to configure the hostnames in advance:

- etcd-1
- etcd-2
- etcd-3

It is also advisable to add hosts entries on all three machines for resolution:

    192.168.10.11 etcd-1
    192.168.10.12 etcd-2
    192.168.10.13 etcd-3

---

## VI. Preparations Before Deployment

## 1. Basic Requirements

The following needs to be confirmed in advance:

- The time on all three Linux machines is synchronized.
- The hostnames are fixed.
- The nodes are interconnected over the network.
- Firewalls allow access on ports 2379 and 2380.
- The disk performance should not be poor; it is recommended to use SSDs.
- A dedicated directory for etcd data should be allocated.

---

## 2. Port Information

Common ports used by etcd:

- 2379: Client access port
- 2380: Peer communication port between etcd nodes

It can be understood as follows:

- Port 2379 is used for access by kube-apiserver or etcdctl.
- Port 2380 is used for data synchronization among etcd cluster members.

---

## 3. Example of Directory Planning

It is recommended to use a unified directory structure:

    /opt/etcd/
    /etc/etcd/
    /var/lib/etcd/
    /etc/etcd/ssl/

Explanation:

- Program directory: /opt/etcd
- Configuration directory: /etc/etcd
- Data directory: /var/libThen check the health status:

    etcdctl \
      --endpoints=https://192.168.10.11:2379,https://192.168.10.12:2379,https://192.168.10.13:2379 \
      --cacert=/etc/etcd/ssl/ca.pem \
      --cert=/etc/etcd/ssl/client.pem \
      --key=/etc/etcd/ssl/client-key.pem \
      endpoint health

View the member list:

    etcdctl \
      --endpoints=https://192.168.10.11:2379,https://192.168.10.12:2379,https://192.168.10.13:2379 \
      --cacert=/etc/etcd/ssl/ca.pem \
      --cert=/etc/etcd/ssl/client.pem \
      --key=/etc/etcd/ssl/client-key.pem \
      member list

View detailed status:

    etcdctl \
      --endpoints=https://192.168.10.11:2379,https://192.168.10.12:2379,https://192.168.10.13:2379 \
      --cacert=/etc/etcd/ssl/ca.pem \
      --cert=/etc/etcd/ssl/client.pem \
      --key=/etc/etcd/ssl/client-key.pem \
      endpoint status -w table

Key points to check:

- Whether all three nodes are healthy
- Whether a leader has been successfully elected
- If the raft term is normal
- If the versions are consistent

---

## Section Fifteen: Common Issues and Troubleshooting Approaches

## 1. The cluster fails to start up

Common reasons:

- Incorrect configuration in initial-cluster
- Name mismatch with what is defined in initial-cluster
- Inability to communicate on port 2380 between nodes
- Mismatch in certificate SAN fields
- Existing old data in the data-dir directory
- Incorrect usage of initial-cluster-state

Troubleshooting steps:

    journalctl -u etcd -f
    systemctl status etcd

---

## 2. Certificate-related errors

Common symptoms:

- "x509 certificate signed by unknown authority"
- "Certificate is valid for xxx, not yyy"

Key areas to check:

- Whether the CA certificates are consistent
- Whether the certificates were issued by the correct CA
- Whether the SAN fields include the current access IP address or hostname
- Whether the peer and client certificate parameters match correctly

---

## 3. A single node can start up, but the cluster does not form

Common checks:

- Whether port 2380 is accessible between nodes
- Whether the peer URL points to a valid local address
- Whether the initial-cluster configurations on all three machines are exactly the same
- Whether firewalls or security groups are blocking communication

---

## 4. Unable to rejoin the cluster after rebuilding a node

Common issues:

- Old member information not deleted properly
- New node's data-dir directory not cleared
- Member information mismatching with the original cluster state

In such cases, simply restarting is insufficient; it is necessary to manage the members manually using the member list.

---

## Section Sixteen: Precautions for Production Environments

## 1. Use an odd number of nodes

Recommended configurations:

- 3 nodes
- 5 nodes

Using 2 or 4 nodes is not recommended for arbitration clusters.

---

## 2. Do not mix etcd with high-load services too closely

Reasons:

- etcd is sensitive to latency and disk performance
- High IO and CPU usage can affect write operations and election processes

---

## 3. Use stable disks, preferably SSDs

etcd requires strong consistency in storage, so disk performance and fsync capabilities are crucial.
Poor disk performance directly impacts cluster performance and stability.

---

## 4. Ensure proper time synchronization

Although etcd does not rely on perfectly synchronized times for data consistency, significant time drift can affect operational troubleshooting, certificate verification, and overall system stability.

---

## 5. Regularly perform snapshot backups

In production environments, it is essential to create regular backup snapshots of etcd data.

Example:

    etcdctl \
      --endpoints=https://192.168.10.11:2379 \
      --cacert=/etc/etcd/ssl/ca.pem \
      --cert=/etc/etcd/ssl/client.pem \
      --key=/etc/etcd/ssl/client-key.pem \
      snapshot## Chapter 23: Memory Version

Memory Mnemonic:

**Three nodes, odd number of devices; 2379 for clients, 2380 for inter-node communication; names must match, and clusters must be consistent; certificates first, then start up; perform health checks before proceeding with business integration.**