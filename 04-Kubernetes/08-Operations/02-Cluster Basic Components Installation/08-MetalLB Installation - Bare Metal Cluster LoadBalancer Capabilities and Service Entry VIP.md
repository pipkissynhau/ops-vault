# 08-MetalLB Installation: Bare-Metal Cluster LoadBalancer Capabilities and Business Entry VIP

Recommended Path:

    04-Kubernetes/08-Operations/02-Cluster Base Component Installation/08-MetalLB Installation: Bare-Metal Cluster LoadBalancer Capabilities and Business Entry VIP.md

Tags:

    #Kubernetes
    #MetalLB
    #LoadBalancer
    #NakedMachineCluster
    #Layer2
    #IPAddressPool
    #L2Advertisement
    #ServiceExposure.
    #ClusterBasicComponents

---

## I. Document Description

This document records the installation, configuration, and verification methods for MetalLB in a bare-metal Kubernetes cluster.

MetalLB's purpose is:

    To provide Service type: LoadBalancer capability for bare-metal Kubernetes clusters

In cloud vendor-managed Kubernetes, after creating a LoadBalancer type Service, the cloud platform typically automatically creates SLB/ELB.

However, in kubeadm self-built clusters, bare-metal clusters, and private environments, there is no cloud vendor LoadBalancer capability by default.

At this time, MetalLB can assign an internal network VIP to LoadBalancer Service, for example:

    10.0.0.100
    10.0.0.101
    10.0.0.102

This document uses:

    MetalLB
    Helm
    Layer2 mode
    IPAddressPool
    L2Advertisement
    LoadBalancer Service
    nginx test service
    ingress-nginx LoadBalancer exposure verification

This document's objectives:

    1. Install MetalLB
    2. Configure kube-proxy strictARP
    3. Create IPAddressPool
    4. Create L2Advertisement
    5. Create LoadBalancer Service
    6. Verify EXTERNAL-IP allocation
    7. Verify business access
    8. Understand the relationship between MetalLB and Ingress/Gateway API
    9. Master common troubleshooting methods

Execution node:

    k8s-master-01

Prerequisites:

    1. Kubernetes cluster has been deployed
    2. Node status is Ready
    3. CNI is normal
    4. CoreDNS is normal
    5. kube-proxy is normal
    6. Helm is installed
    7. kubectl can access the cluster normally

---

## II. What Problem Does MetalLB Solve

### 2.1 Without MetalLB

In a bare-metal Kubernetes cluster, if creating:

    type: LoadBalancer

You will typically see:

    EXTERNAL-IP: <pending>

Example:

    NAME        TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)
    nginx-lb    LoadBalancer   10.96.10.100    <pending>     80:31234/TCP

Reason:

    kubeadm self-built clusters lack cloud vendor LoadBalancer controllers.
    Kubernetes itself does not create external load balancer IPs out of thin air.

---

### 2.2 With MetalLB

After installing MetalLB and configuring the address pool, creating a LoadBalancer Service will allocate an IP from the address pool.

Example:

    NAME        TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)
    nginx-lb    LoadBalancer   10.96.10.100    10.0.0.100     80:31234/TCP

Then you can directly access:

    curl http://10.0.0.100

---

## III. Relationship Between MetalLB and Ingress/Gateway API

MetalLB, Ingress, and Gateway API are not mutually exclusive.

Their relationship is as follows:

    MetalLB
        Responsible for allocating external IP for LoadBalancer Service

    ingress-nginx
        Responsible for seven-layer HTTP/HTTPS routing

    Gateway API / Envoy Gateway
        Responsible for the new generation of entry and routing models

Common combinations:

    User
      |
      v
    VIP allocated by MetalLB
      |
      v
    ingress-nginx-controller Service(type=LoadBalancer)
      |
      v
    Ingress rules
      |
      v
    Service
      |
      v
    Pod

Or:

    User
      |
      v
    VIP allocated by MetalLB
      |
      v
    Envoy Gateway Service(type=LoadBalancer)
      |
      v
    Gateway/HTTPRoute
      |
      v
    Service
      |
      v
    Pod

Simple understanding:

    MetalLB solves "where does the entry IP come from"
    Ingress/Gateway API solves "how to forward HTTP requests"

---

## IV. Deployment Planning

### 4.1 Cluster Network Planning

This document's cluster node network segment:

    10.0.0.0/24

Existing node planning:

    ops-server        10.0.0.10
    k8s-master-01     10.0.0.20
    k8s-master-02     10.0.0.21
    k8s-master-03     10.0.0.22
    k8s-worker-01     10.0.0.23
    k8s-worker-02     10.0.0.24
    k8s-worker-03     10.0.0.25
    k8s-api-server    10.0.0.30

MetalLB address pool planning:

    10.0.0.100-10.0.0.120

Note: /think

10.0.0.30 is already used as Kubernetes APIServer VIP, do not add to MetalLB address pool.
10.0.0.20-10.0.0.25 are node IPs, do not add to MetalLB address pool.
10.0.0.100-10.0.0.120 must confirm not occupied by DHCP or other servers.
Production environment recommends explicitly reserving this address range at network layer or DHCP.

---

### 4.2 MetalLB Planning

| Item | Planning |
|---|---|
| Installation Method | Helm |
| Namespace | metallb-system |
| Operating Mode | Layer2 |
| Address Pool Name | ops-l2-pool |
| Address Pool Range | 10.0.0.100-10.0.0.120 |
| Broadcast Method | L2Advertisement |
| Test Service | nginx-metallb-demo |
| Test Namespace | metallb-demo |

---

## Five. Pre-deployment Checks

### 5.1 Check Cluster Status

Execute:

    kubectl get nodes -o wide

Requirements:

    All nodes Ready.

---

### 5.2 Check Helm

Execute:

    helm version

---

### 5.3 Check kube-proxy Mode

Execute:

    kubectl -n kube-system get cm kube-proxy -o jsonpath='{.data.config.conf}' | grep -E "mode:|strictARP|scheduler:"

Expected to see at least:

    mode: "ipvs"

If kube-proxy uses IPVS mode, MetalLB deployment requires enabling:

    strictARP: true

---

### 5.4 Check MetalLB Address Pool Occupancy

Test if address pool IPs are already in use on any node or operations machine.

Example:

    ping -c 2 10.0.0.100

    ping -c 2 10.0.0.101

    ping -c 2 10.0.0.120

No response doesn't guarantee absolute availability, but can serve as initial judgment.

More reliable method:

    1. Query DHCP address pool
    2. Query switch ARP table
    3. Query gateway ARP table
    4. Confirm this IP range is reserved for MetalLB

---

## Six. Configure kube-proxy strictARP

The cluster was already configured with kube-proxy in IPVS mode during deployment.

MetalLB recommends enabling strictARP when using IPVS mode.

---

### 6.1 Check Current strictARP

Execute:

    kubectl -n kube-system get cm kube-proxy -o jsonpath='{.data.config.conf}' | grep -E "mode:|strictARP|scheduler:"

If seeing:

    strictARP: false

Or no strictARP entry, need to change to:

    strictARP: true

---

### 6.2 Modify kube-proxy strictARP

Export kube-proxy configuration:

    kubectl -n kube-system get cm kube-proxy -o yaml > /tmp/kube-proxy-metallb.yaml

Backup:

    cp /tmp/kube-proxy-metallb.yaml /tmp/kube-proxy-metallb.yaml.bak.$(date +%F-%H%M%S)

Modify strictARP:

    sed -i.bak -E \
      -e 's/^([[:space:]]*)strictARP:.*$/\1strictARP: true/' \
      /tmp/kube-proxy-metallb.yaml

If original config lacks strictARP field, manually edit:

    vim /tmp/kube-proxy-metallb.yaml

Ensure config.conf under ipvs field contains:

    ipvs:
      scheduler: "rr"
      strictARP: true

Apply changes:

    kubectl replace -f /tmp/kube-proxy-metallb.yaml

Restart kube-proxy:

    kubectl -n kube-system delete pod -l k8s-app=kube-proxy

Wait for rebuild:

    kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide

---

### 6.3 Verify strictARP

Check configuration:

    kubectl -n kube-system get cm kube-proxy -o jsonpath='{.data.config.conf}' | grep -E "mode:|strictARP|scheduler:"

Expected:

    mode: "ipvs"
    scheduler: "rr"
    strictARP: true

Check kernel parameters on nodes:

    sysctl net.ipv4.conf.all.arp_ignore

    sysctl net.ipv4.conf.all.arp_announce

Common expected values:

    net.ipv4.conf.all.arp_ignore = 1
    net.ipv4.conf.all.arp_announce = 2

Notes:

    If just modified configuration, wait for kube-proxy rebuild before checking.
    Output may vary slightly by version, with strictARP: true in kube-proxy ConfigMap as the standard.

---

## Seven. Install MetalLB

### 7.1 Create Namespace

MetalLB speaker requires high privileges, so first create and label the namespace.

Execute:

    kubectl create namespace metallb-system

Add Pod Security label: /think

kubectl label namespace metallb-system \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/audit=privileged \
  pod-security.kubernetes.io/warn=privileged \
  --overwrite

Check:

  kubectl get namespace metallb-system --show-labels

---

### 7.2 Add MetalLB Helm Repository

Execute:

  helm repo add metallb https://metallb.github.io/metallb

  helm repo update

Check versions:

  helm search repo metallb/metallb --versions | head

---

### 7.3 Install MetalLB

Create directory:

  mkdir -p /root/k8s-yaml/metallb

  cd /root/k8s-yaml/metallb

Install:

  helm install metallb metallb/metallb \
    -n metallb-system

Check Helm Release:

  helm list -n metallb-system

Check Pods:

  kubectl -n metallb-system get pods -o wide

Common components:

  controller
  speaker

Notes:

  controller handles address allocation logic.
  speaker is responsible for announcing LoadBalancer IP on nodes.

---

### 7.4 Wait for MetalLB to be Ready

Execute:

  kubectl -n metallb-system get deployment

  kubectl -n metallb-system get daemonset

  kubectl -n metallb-system get pods -o wide

Expected:

  controller Running
  speaker Running on each node

Check logs:

  kubectl -n metallb-system logs deploy/metallb-controller --tail=100

  kubectl -n metallb-system logs -l app.kubernetes.io/component=speaker --tail=100

---

## VIII. Configure IPAddressPool and L2Advertisement

MetalLB will not automatically assign IP after installation.

Must create:

  IPAddressPool
  L2Advertisement

---

### 8.1 Create Address Pool

Create:

  cat <<EOF > metallb-ipaddresspool.yaml
  apiVersion: metallb.io/v1beta1
  kind: IPAddressPool
  metadata:
    name: ops-l2-pool
    namespace: metallb-system
  spec:
    addresses:
    - 10.0.0.100-10.0.0.120
    autoAssign: true
    avoidBuggyIPs: true
  EOF

Apply:

  kubectl apply -f metallb-ipaddresspool.yaml

Check:

  kubectl -n metallb-system get ipaddresspool

Check details:

  kubectl -n metallb-system describe ipaddresspool ops-l2-pool

Notes:

  addresses
      IP range MetalLB can assign to LoadBalancer Service

  autoAssign: true
      Allow automatic IP assignment from this pool

  avoidBuggyIPs: true
      Avoid using potentially incompatible addresses

---

### 8.2 Create L2Advertisement

Create:

  cat <<EOF > metallb-l2advertisement.yaml
  apiVersion: metallb.io/v1beta1
  kind: L2Advertisement
  metadata:
    name: ops-l2-advertisement
    namespace: metallb-system
  spec:
    ipAddressPools:
    - ops-l2-pool
  EOF

Apply:

  kubectl apply -f metallb-l2advertisement.yaml

Check:

  kubectl -n metallb-system get l2advertisement

Check details:

  kubectl -n metallb-system describe l2advertisement ops-l2-advertisement

Notes:

  L2Advertisement indicates using Layer2 to announce pool IPs.
  Clients on same subnet find VIP holder via ARP

---

## IX. Create LoadBalancer Test Service

### 9.1 Create Test Namespace

Execute:

  kubectl create namespace metallb-demo

---

### 9.2 Create nginx Test Application

Create Deployment:

  kubectl -n metallb-demo create deployment nginx-metallb-demo --image=nginx:1.25

Scale:

  kubectl -n metallb-demo scale deployment nginx-metallb-demo --replicas=2

Check Pods:

  kubectl -n metallb-demo get pods -o wide

---

### 9.3 Create LoadBalancer Service

Create:

```bash
cat <<EOF > svc-nginx-metallb-lb.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-metallb-lb
  namespace: metallb-demo
spec:
  type: LoadBalancer
  externalTrafficPolicy: Cluster
  selector:
    app: nginx-metallb-demo
  ports:
  - name: http
    port: 80
    targetPort: 80
    protocol: TCP
EOF
```

**Apply:**

```bash
kubectl apply -f svc-nginx-metallb-lb.yaml
```

**Check:**

```bash
kubectl -n metallb-demo get svc nginx-metallb-lb -o wide
```

**Expected output:**

```
EXTERNAL-IP: 10.0.0.100-10.0.0.120 range
```

**Example:**

```
NAME               TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)
nginx-metallb-lb   LoadBalancer   10.96.88.100    10.0.0.100     80:xxxxx/TCP
```

---

### 9.4 Access Verification

Assume the allocated EXTERNAL-IP is:

```
10.0.0.100
```

**Access:**

```bash
curl http://10.0.0.100
```

If the nginx default page is returned, it indicates that MetalLB LoadBalancer is working.

You can also directly retrieve the EXTERNAL-IP:

```bash
export LB_IP=$(kubectl -n metallb-demo get svc nginx-metallb-lb -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo ${LB_IP}
curl http://${LB_IP}
```

---

## Ten. Specifying Address Pool or Fixed IP

### 10.1 Specifying Address Pool

If there are multiple IPAddressPool in the cluster, you can specify which address pool to use via annotation.

**Example:**

```bash
cat <<EOF > svc-nginx-metallb-pool.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-metallb-pool
  namespace: metallb-demo
  annotations:
    metallb.io/address-pool: ops-l2-pool
spec:
  type: LoadBalancer
  externalTrafficPolicy: Cluster
  selector:
    app: nginx-metallb-demo
  ports:
  - name: http
    port: 80
    targetPort: 80
    protocol: TCP
EOF
```

**Apply:**

```bash
kubectl apply -f svc-nginx-metallb-pool.yaml
```

**Check:**

```bash
kubectl -n metallb-demo get svc nginx-metallb-pool
```

---

### 10.2 Fixed LoadBalancer IP

If you want a Service to use a specific IP, you can specify:

```
loadBalancerIP
```

**Example:**

```bash
cat <<EOF > svc-nginx-metallb-fixed-ip.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-metallb-fixed-ip
  namespace: metallb-demo
spec:
  type: LoadBalancer
  loadBalancerIP: 10.0.0.101
  externalTrafficPolicy: Cluster
  selector:
    app: nginx-metallb-demo
  ports:
  - name: http
    port: 80
    targetPort: 80
    protocol: TCP
EOF
```

**Apply:**

```bash
kubectl apply -f svc-nginx-metallb-fixed-ip.yaml
```

**Check:**

```bash
kubectl -n metallb-demo get svc nginx-metallb-fixed-ip
```

**Access:**

```bash
curl http://10.0.0.101
```

**Note:**

- The specified IP must be within an IPAddressPool range.
- The specified IP cannot be already occupied by another LoadBalancer Service.
- Before fixing IP in production, confirm DNS, load balancing, and firewall planning.

---

## Eleven. Optional: Change ingress-nginx to LoadBalancer

Previously, ingress-nginx used NodePort:

```
30080
30443
```

If MetalLB is installed, you can also change the ingress-nginx-controller Service to LoadBalancer.

**Note:**

- This is an optional operation.
- If current NodePort meets usage requirements, you can skip this step.

---

### 11.1 Check Current ingress-nginx Service

Execute:

```bash
kubectl -n ingress-nginx get svc ingress-nginx-controller -o wide
```

Check type:

```bash
kubectl -n ingress-nginx get svc ingress-nginx-controller -o jsonpath='{.spec.type}'
```

---

### 11.2 Modify Helm values

Enter the ingress-nginx values directory:

```bash
cd /root/k8s-yaml/ingress-nginx
```

Backup: /think

cp values-ingress-nginx-nodeport.yaml values-ingress-nginx-loadbalancer.yaml

Edit:

vim values-ingress-nginx-loadbalancer.yaml

Change controller.service to:

controller:
  service:
    type: LoadBalancer
    loadBalancerIP: 10.0.0.102
    externalTrafficPolicy: Cluster

If the following exists:

nodePorts:
  http: 30080
  https: 30443

You can delete or comment it out.

---

### 11.3 Upgrade ingress-nginx

Execute:

helm upgrade ingress-nginx ingress-nginx/ingress-nginx \
  -n ingress-nginx \
  -f values-ingress-nginx-loadbalancer.yaml

Check:

kubectl -n ingress-nginx get svc ingress-nginx-controller -o wide

Expected:

TYPE: LoadBalancer
EXTERNAL-IP: 10.0.0.102

---

### 11.4 Verify Ingress Access

If you previously had a demo.ops.local Ingress, you can access:

curl -H "Host: demo.ops.local" http://10.0.0.102/

Note:

After switching to LoadBalancer, external access no longer requires 30080.
You can directly use 80 / 443.

---

## Twelve. Using LoadBalancer for Gateway API (Optional)

If Envoy Gateway defaults to using LoadBalancer Service, after installing MetalLB, Gateway data plane Service can directly obtain EXTERNAL-IP.

Check Envoy Gateway's generated Service:

kubectl -n envoy-gateway-system get svc

If Service type is LoadBalancer:

kubectl -n envoy-gateway-system get svc -o wide

Expected to see:

EXTERNAL-IP

If previously changed to NodePort for bare-metal clusters, you can revert to LoadBalancer using Envoy Gateway's values or EnvoyProxy configuration.

Note:

Gateway API can be used with MetalLB.
MetalLB is responsible for assigning VIP to Envoy Gateway's data plane Service.
Gateway / HTTPRoute handles HTTP routing.

---

## Thirteen. Layer2 Mode Explanation

This document uses Layer2 mode.

Layer2 mode characteristics:

1. Simple configuration
2. No need for BGP routers
3. Suitable for bare-metal clusters with Layer2 network reachability
4. Announce VIP externally via ARP
5. A single node responds to VIP at a time

Suitable for:

1. Experimental environments
2. Small-scale production environments
3. Private delivery environments
4. Intranet clusters without BGP conditions

Note:

MetalLB's LoadBalancer IP must be reachable from client networks.
Layer2 mode requires network allowing ARP broadcasts.
If accessing across Layer3 networks, ensure network side correctly routes to this address range.
For large-scale or complex network environments, consider BGP mode.

---

## Fourteen. Common Issue Troubleshooting

### 14.1 LoadBalancer Service Stays Pending

Check Service:

kubectl -n metallb-demo get svc nginx-metallb-lb

Check MetalLB Pods:

kubectl -n metallb-system get pods -o wide

Check IPAddressPool:

kubectl -n metallb-system get ipaddresspool

Check L2Advertisement:

kubectl -n metallb-system get l2advertisement

Check controller logs:

kubectl -n metallb-system logs deploy/metallb-controller --tail=100

Common causes:

1. MetalLB not installed successfully
2. IPAddressPool not created
3. L2Advertisement not created
4. Address pool IP range written incorrectly
5. Address pool exhausted
6. Service lacks type: LoadBalancer

---

### 14.2 EXTERNAL-IP Exists but Access Fails

Check Service:

kubectl -n metallb-demo describe svc nginx-metallb-lb

Check Endpoints:

kubectl -n metallb-demo get endpoints nginx-metallb-lb

Check Pods:

kubectl -n metallb-demo get pods -o wide

Check kube-proxy:

kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide

Check IPVS:

sudo ipvsadm -Ln | grep 10.0.0.100

Check firewall:

sudo ufw status

Common causes:

1. Service selector does not match Pod
2. Endpoints are empty
3. Pod is not Running
4. kube-proxy anomaly
5. Firewall interception
6. Client network unreachable to 10.0.0.100
7. strictARP not enabled
8. Layer2 network does not support ARP broadcast

---

### 14.3 Address Pool IP Conflict

Symptoms:

Same IP used by other devices
Access anomalies
ARP drift anomalies
Sometimes works, sometimes not

Troubleshooting:

    ping 10.0.0.100

    arp -a | grep 10.0.0.100

    ip neigh | grep 10.0.0.100

Resolution:

    1. Immediately remove the conflicting IP from IPAddressPool
    2. Use an unused IP address
    3. Reserve MetalLB address range in DHCP or network side
    4. Check switch and gateway ARP tables

Production Requirements:

    MetalLB address pool must be planned in advance.
    Do not arbitrarily take a segment from the office network DHCP address range.

---

### 14.4 speaker Pod Abnormalities

Check:

    kubectl -n metallb-system get pods -o wide

Check DaemonSet:

    kubectl -n metallb-system get ds

Check logs:

    kubectl -n metallb-system logs -l app.kubernetes.io/component=speaker --tail=100

Common causes:

    1. Insufficient permissions
    2. Namespace lacks privileged label
    3. Node network anomalies
    4. CNI or host network restrictions
    5. Failed image pull

Check namespace labels:

    kubectl get namespace metallb-system --show-labels

Must include:

    pod-security.kubernetes.io/enforce=privileged
    pod-security.kubernetes.io/audit=privileged
    pod-security.kubernetes.io/warn=privileged

---

### 14.5 kube-proxy strictARP Not Enabled

Check:

    kubectl -n kube-system get cm kube-proxy -o jsonpath='{.data.config\.conf}' | grep strictARP

If not:

    strictARP: true

Need to modify according to Section 6 of this document.

After modification, restart kube-proxy:

    kubectl -n kube-system delete pod -l k8s-app=kube-proxy

---

### 14.6 LoadBalancer IP Not Accessible from Other Subnets

In Layer2 mode, MetalLB is mainly suitable for same Layer2 network access.

If client is in other subnet, need to confirm:

    1. Route reachability
    2. Gateway knows the path for 10.0.0.100-10.0.0.120
    3. Firewall rules allow traffic
    4. Network devices allow ARP/VIP drift
    5. Should switch to BGP mode

If network spans multiple Layer3 subnets, recommend consulting network team for routing plan.

---

## FifteenI don't know.Upgrade and Rollback

### 15.1 Check Helm Release

Execute:

    helm list -n metallb-system

Check status:

    helm status metallb -n metallb-system

Check history:

    helm history metallb -n metallb-system

---

### 15.2 Backup Configuration

Backup Helm values:

    helm get values metallb -n metallb-system -o yaml > metallb-values-backup.yaml

Backup MetalLB configuration:

    kubectl -n metallb-system get ipaddresspool -o yaml > metallb-ipaddresspool-backup.yaml

    kubectl -n metallb-system get l2advertisement -o yaml > metallb-l2advertisement-backup.yaml

---

### 15.3 Upgrade

Update repository:

    helm repo update

Check version:

    helm search repo metallb/metallb --versions | head

Upgrade:

    helm upgrade metallb metallb/metallb \
      -n metallb-system

Check status:

    kubectl -n metallb-system get pods -o wide

    kubectl -n metallb-system get ipaddresspool

    kubectl -n metallb-system get l2advertisement

Production recommendations:

    Confirm current LoadBalancer Service usage before upgrading MetalLB.
    Schedule upgrade during maintenance window.
    Check if EXTERNAL-IP remains unchanged after upgrade.

---

### 15.4 Rollback

Check history:

    helm history metallb -n metallb-system

Rollback:

    helm rollback metallb <REVISION> -n metallb-system

Check:

    helm status metallb -n metallb-system

---

## SixteenI don't know.Clean Up Test Resources

Delete test Service:

    kubectl delete -f svc-nginx-metallb-lb.yaml

Delete fixed IP test Service:

    kubectl delete -f svc-nginx-metallb-fixed-ip.yaml

Delete specified address pool test Service:

    kubectl delete -f svc-nginx-metallb-pool.yaml

Delete test application:

    kubectl -n metallb-demo delete deployment nginx-metallb-demo

Delete namespace:

    kubectl delete namespace metallb-demo

Check if address is released:

    kubectl get svc -A | grep LoadBalancer

---

## SeventeenI don't know.Uninstall MetalLB

Note:

    Must confirm no business Service is using MetalLB allocated EXTERNAL-IP before uninstallation.
    Do not directly uninstall in production environment.

Check LoadBalancer Service: /think

kubectl get svc -A | grep LoadBalancer

Delete MetalLB Address Pool Configuration:

    kubectl delete -f metallb-l2advertisement.yaml

    kubectl delete -f metallb-ipaddresspool.yaml

Uninstall Helm Release:

    helm uninstall metallb -n metallb-system

Delete Namespace:

    kubectl delete namespace metallb-system

Check CRD:

    kubectl get crd | grep metallb

If confirmed no longer in use, delete CRD:

    kubectl delete crd ipaddresspools.metallb.io

    kubectl delete crd l2advertisements.metallb.io

Note:

    Deleting CRD will remove MetalLB-related custom resources.
    If the cluster will still use MetalLB in the future, do not delete CRD.

---

## Eighteen. Post-Installation Checklist

After installation, execute:

    kubectl -n metallb-system get pods -o wide

    kubectl -n metallb-system get ipaddresspool

    kubectl -n metallb-system get l2advertisement

    kubectl -n kube-system get cm kube-proxy -o jsonpath='{.data.config\.conf}' | grep strictARP

    kubectl -n metallb-demo get svc nginx-metallb-lb -o wide

    curl http://${LB_IP}

Should meet:

    1. metallb-controller Running
    2. metallb-speaker Running on nodes
    3. kube-proxy strictARP=true
    4. IPAddressPool created
    5. L2Advertisement created
    6. LoadBalancer Service obtains EXTERNAL-IP
    7. EXTERNAL-IP is within 10.0.0.100-10.0.0.120
    8. curl access to EXTERNAL-IP succeeds
    9. Address pool has no conflicts with node IPs, APIServer VIP, or DHCP addresses

---

## Nineteen. Summary

This document completes the basic installation and verification of MetalLB.

Core content:

    1. Understanding why bare-metal clusters need MetalLB
    2. Installing MetalLB using Helm
    3. Configuring kube-proxy strictARP
    4. Creating IPAddressPool
    5. Creating L2Advertisement
    6. Creating LoadBalancer Service
    7. Verifying EXTERNAL-IP allocation
    8. Verifying business access
    9. Understanding MetalLB's relationship with Ingress/Gateway API
    10. Troubleshooting Pending, access issues, IP conflicts, and speaker anomalies

Production recommendations:

    1. MetalLB address pools must be planned in advance
    2. Avoid conflicts with node IPs, APIServer VIP, or DHCP addresses
    3. strictARP must be monitored when kube-proxy uses IPVS
    4. Layer2 mode is suitable for simple bare-metal environments
    5. Evaluate BGP mode for cross-layer networks or complex environments
    6. ingress-nginx/Gateway API can obtain LoadBalancer VIP via MetalLB
    7. MetalLB does not replace Ingress or Gateway API; it only provides LoadBalancer external IP

At this point, the cluster's base component installation directory can be closed:

    00-Cloud-Native Foundation: kubectl Operation Efficiency Tools and Helm Installation Guide
    01-metrics-server Installation: kubectl top, Resource Metrics, and HPA Dependencies
    02-ingress-nginx Production Installation: NodePort, IngressClass, and Access Verification
    03-cert-manager Installation: TLS Certificate Auto-Issuance, Renewal, and Secret Management
    04-Gateway API Getting Started Installation: Envoy Gateway, GatewayClass, Gateway, and HTTPRoute
    05-StorageClass Basic Installation: NFS Dynamic Provisioning and Default Storage Class
    06-Longhorn Installation: Distributed Block Storage, StorageClass, and PVC Verification
    07-ExternalDNS Installation: Ingress, Service, and External DNS Record Auto-Management
    08-MetalLB Installation: LoadBalancer Capability for Bare-Metal Clusters and Business VIP Entry