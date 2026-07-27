# 08-MetalLB Installation: LoadBalancer Capability for Bare Metal Clusters and Business Entrance VIPs

Recommended Path:

    04-Kubernetes/08-Operations/02-Cluster Basic Components Installation/08-MetalLB Installation: LoadBalancer Capability for Bare Metal Clusters and Business Entrance VIPs.md

Tags:

    #Kubernetes
    #MetalLB
    #LoadBalancer
    #Bare Metal Clusters
    #Layer2
    #IPAddressPool
    #L2Advertisement
    #Service Exposure
    #Cluster Basic Components

---

## I. Document Description

This document records the installation, configuration, and verification methods of MetalLB in a Kubernetes bare metal cluster.

The role of MetalLB is:

    To provide the LoadBalancer type capability for bare metal Kubernetes clusters.

In cloud-managed Kubernetes environments, when creating a LoadBalancer type Service, the cloud platform typically automatically generates an SLB/ELB.

However, in self-built kubeadm clusters, bare metal clusters, or private environments, there is no default cloud provider LoadBalancer capability.

In such cases, MetalLB can be used to assign an internal VIP to the LoadBalancer Service, for example:

    10.0.0.100
    10.0.0.101
    10.0.0.102

This document utilizes:

    MetalLB
    Helm
    Layer2 mode
    IPAddressPool
    L2Advertisement
    LoadBalancer Service
    nginx test service
    ingress-nginx LoadBalancer exposure verification

The objectives of this document are:

    1. To install MetalLB
    2. To configure kube-proxy strictARP
    3. To create an IPAddressPool
    4. To create an L2Advertisement
    5. To create a LoadBalancer Service
    6. To verify EXTERNAL-IP allocation
    7. To validate business access
    8. To understand the relationship between MetalLB and Ingress/Gateway APIs
    9. To master common troubleshooting methods

Execution Node:

    k8s-master-01

Prerequisites:

    1. The Kubernetes cluster has been fully deployed.
    2. All nodes are in the Ready state.
    3. CNI is functioning correctly.
    4. CoreDNS is operational.
    5. kube-proxy is working properly.
    6. Helm has been installed.
    7. kubectl can access the cluster without issues.

---

## II. What Problems Does MetalLB Solve

### 2.1 Without MetalLB

In a bare metal Kubernetes cluster, if you attempt to create a LoadBalancer with the type set to LoadBalancer, you will see:

    EXTERNAL-IP: <pending>

Example:

    NAME        TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)
    nginx-lb    LoadBalancer   10.96.10.100    <pending>     80:31234/TCP

Reason:

    In self-built kubeadm clusters, there is no cloud provider LoadBalancer controller.
    Kubernetes itself does not generate external load balancing IPs.

---

### 2.2 With MetalLB

After installing MetalLB and configuring the address pool, when you create a LoadBalancer Service, MetalLB will allocate an IP from the address pool.

Example:

    NAME        TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)
    nginx-lb    LoadBalancer   10.96.10.100    10.0.0.100     80:31234/TCP

You can then directly access it using:

    curl http://10.0.0.100

---

## III. The Relationship Between MetalLB and Ingress/Gateway APIs

MetalLB, Ingress, and Gateway APIs are not mutually replaceable.

Their relationships are as follows:

    MetalLB
        Is responsible for allocating external IPs to LoadBalancer Services.

    ingress-nginx
        Handles layer 7 HTTP/HTTPS routing.

    Gateway API / Envoy Gateway
        Handles the new generation of entry points and routing models.

Common combinations include:

    User
      |
      v
    VIP Assigned by MetalLB
      |
      v
    ingress-nginx-controller Service(type=LoadBalancer)
      |
      v
    Ingress Rules
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
    VIP Assigned by MetalLB
      |
      v
    Envoy Gateway Service(type=LoadBalancer)
      |
      v
    Gateway / HTTPRoute
      |
      v
    Service
      |
      v
    Pod

Simply put:

    MetalLB addresses the issue of "whereExport kube-proxy configuration:

```bash
kubectl -n kube-system get cm kube-proxy -o yaml > /tmp/kube-proxy-metallb.yaml
```

Backup the file:

```bash
cp /tmp/kube-proxy-metallb.yaml /tmp/kube-proxy-metallb.yaml.bak.$(date +%F-%H%M%S)
```

Modify the `strictARP` setting:

```bash
sed -i.bak -E \
  -e 's/^([[:space:]]*)strictARP:.*$/\1strictARP: true/' \
  /tmp/kube-proxy-metallb.yaml
```

If the `strictARP` field is not present in the original configuration, you can edit it manually:

```bash
vim /tmp/kube-proxy-metallb.yaml
```

Ensure that the `ipvs` section contains the following settings:

```yaml
ipvs:
  scheduler: "rr"
  strictARP: true
```

Apply the changes:

```bash
kubectl replace -f /tmp/kube-proxy-metallb.yaml
```

Restart kube-proxy:

```bash
kubectl -n kube-system delete pod -l k8s-app=kube-proxy
```

Wait for kube-proxy to be reinstalled and then check its status:

```bash
kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide
```

---

### 6.3 Verify strictARP settings

Check the configuration:

```bash
kubectl -n kube-system get cm kube-proxy -o jsonpath '{.data.config\.conf}' | grep -E "mode:|strictARP|scheduler:"
```

Expected output:

```
mode: "ipvs"
scheduler: "rr"
strictARP: true
```

Verify the kernel parameters on the nodes:

```bash
sysctl net.ipv4.conf.all.arp_ignore
sysctl net.ipv4.conf.all.arp_announce
```

Common expected values:

```
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 2
```

Note:

- If you just modified the configuration, wait for kube-proxy to be reinstalled before checking the results.
- The output may vary slightly depending on the version of kube-proxy; refer to the `strictARP: true` setting in the ConfigMap for accurate values.

---

## Section 7: Install MetalLB

### 7.1 Create a namespace

MetalLB requires elevated permissions, so create and label a dedicated namespace first:

```bash
kubectl create namespace metallb-system
```

Add Pod Security labels to the namespace:

```bash
kubectl label namespace metallb-system \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/audit=privileged \
  pod-security.kubernetes.io/warn=privileged \
  --overwrite
```

Verify the labels:

```bash
kubectl get namespace metallb-system --show-labels
```

---

### 7.2 Add the MetalLB Helm repository

Add the MetalLB Helm repository and update it:

```bash
helm repo add metallb https://metallb.github.io/metallb
helm repo update
```

Check the available versions:

```bash
helm search repo metallb/metallb --versions | head
```

---

### 7.3 Install MetalLB

Create a directory for installing MetalLB and navigate to it:

```bash
mkdir -p /root/k8s-yaml/metallb
cd /root/k8s-yaml/metallb
```

Install MetalLB using Helm:

```bash
helm install metallb metallb/metallb \
  -n metallb-system
```

Check the installed Helm Release:

```bash
helm list -n metallb-system
```

View the installed Pods:

```bash
kubectl -n metallb-system get pods -o wide
```

Common components include a `controller` and a `speaker`. The `controller` is responsible for address allocation logic, while the `speaker` announces the LoadBalancer IP addresses on the nodes.

---

### 7.4 Wait for MetalLB to be ready

Check the status of the related resources:

```bash
kubectl -n metallb-system get deployment
kubectl -n metallb-system get daemonset
kubectl -n metallb-system get pods -o wide
```

Expected status:

- `controller` should be in the `Running` state.
- `speaker` should also be in the `Running` state on all nodes.

Monitor logs for any errors or issues:

```bash
kubectl -n metallb-system logs deploy/metallb-controller --tail=100
kubectl -n metallb-system logs -l app.kubernetes.io/component=speaker --tail=100
```

---

## Section 8: Configure IPAddressPool and L2Advertisement

After installing MetalLB, IP addresses are not automatically allocated. You need to create anapiVersion: v1
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

Application:

    kubectl apply -f svc-nginx-metallb-lb.yaml

View:

    kubectl -n metallb-demo get svc nginx-metallb-lb -o wide

Expected output:

    EXTERNAL-IP: An IP address within the range 10.0.0.100-10.0.0.120

Example:

    NAME               TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)
    nginx-metallb-lb   LoadBalancer   10.96.88.100    10.0.0.100     80:xxxxx/TCP

---

### 9.4 Access Verification

Assume the assigned EXTERNAL-IP is:

    10.0.0.100

Access it using:

    curl http://10.0.0.100

If the default nginx page is displayed, it indicates that the MetalLB LoadBalancer is functioning correctly.

You can also retrieve the EXTERNAL-IP directly:

    export LB_IP=$(kubectl -n metallb-demo get svc nginx-metallb-lb -o jsonpath '{.status.loadBalancer.ingress[0].ip}')

    echo ${LB_IP}

    curl http://${LB_IP}

---

## Ten: Specifying an Address Pool or Fixed IP

### 10.1 Specifying an Address Pool

If there are multiple IPAddressPools in the cluster, you can specify which one to use through annotations.

Example:

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

Application:

    kubectl apply -f svc-nginx-metallb-pool.yaml

View:

    kubectl -n metallb-demo get svc nginx-metallb-pool

---

### 10.2 Fixing a LoadBalancer IP

If you want a particular Service to use a fixed IP address, you can specify it:

    loadBalancerIP

Example:

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

Application:

    kubectl apply -f svc-nginx-metallb-fixed-ip.yaml

View:

    kubectl -n metallb-demo get svc nginx-metallb-fixed-ip

Access it using:

    curl http://10.0.0.101

Note:

    The specified IP must be within the range of the IPAddressPool.
    The IP should not already be in use by another LoadBalancer Service.
    In a production environment, confirm DNS, load balancing, and firewall configurations before assigning a fixed IP.

---

## Eleven: Changing ingress-nginx to a LoadBalancer (Optional)

Previously, ingress-nginx used NodePorts:

    30080
    30443

If MetalLB is installed, you can also change the ingress-nginx-controller Service to a LoadBalancer.

Note:

    This is an optional step.
    If the current NodePorts are sufficient, there is no need to make this change.

---

### 11.1 Checking the Current ingress-nginx Service

Execute:

    kubectl -n ingress-nginx get svc ingress-nginx-controller -o wide

To check the type:

    kubectl -n ingress-nginx get svc ingress-nginx-controller -o jsonpath '{.spec.type}'

---

### 11.2 Modifying Helm Values

Enter theThis document utilizes the Layer2 mode.

### Features of Layer2 Mode:

1. Simple configuration
2. No need for BGP routers
3. Suitable for bare-metal clusters with Layer2 network connectivity
4. Declares VIP addresses externally via ARP
5. Only one node can respond to external requests for a particular VIP at any given time

### Applications:

1. Experimental environments
2. Small-scale production environments
3. Private delivery environments
4. Intranet clusters without BGP capabilities

### Notes:

- The LoadBalancer IP address of MetalLB must be reachable from the client network.
- Layer2 mode requires that the network supports ARP broadcasting.
- If accessing from a Layer3 network, proper routing to this IP range is necessary on the network side.
- For large or complex network environments, BGP mode may be considered.

---

## Section 14: Troubleshooting Common Issues

### 14.1 The LoadBalancer Service Remains in a "Pending" State

- Check the Service:
  ```bash
  kubectl -n metallb-demo get svc nginx-metallb-lb
  ```
- Check the MetalLB Pod:
  ```bash
  kubectl -n metallb-system get pods -o wide
  ```
- Check the IPAddressPool:
  ```bash
  kubectl -n metallb-system get ipaddresspool
  ```
- Check the L2Advertisement:
  ```bash
  kubectl -n metallb-system get l2advertisement
  ```
- Check the controller logs:
  ```bash
  kubectl -n metallb-system logs deploy/metallb-controller --tail=100
  ```

**Common causes:**
1. MetalLB failed to install successfully.
2. The IPAddressPool was not created.
3. The L2Advertisement was not generated.
4. The IP range specified for the address pool was incorrect.
5. The address pool had already been exhausted.
6. The Service did not have the `type: LoadBalancer` configuration.

---

### 14.2 The EXTERNAL-IP Is Available, But Access Is Impossible

- Check the Service:
  ```bash
  kubectl -n metallb-demo describe svc nginx-metallb-lb
  ```
- Check the Endpoints:
  ```bash
  kubectl -n metallb-demo get endpoints nginx-metallb-lb
  ```
- Check the Pod:
  ```bash
  kubectl -n metallb-demo get pods -o wide
  ```
- Check the kube-proxy:
  ```bash
  kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide
  ```
- Check IPVS:
  ```bash
  sudo ipvsadm -Ln | grep 10.0.0.100
  ```
- Check the firewall settings:
  ```bash
  sudo ufw status
  ```

**Common causes:**
1. The Service selector does not match any Pod.
2. The Endpoints are set to none.
3. The Pod is not running.
4. There are issues with the kube-proxy.
5. The firewall is blocking access.
6. The network between the client and 10.0.0.100 is unavailable.
7. StrictARP is not enabled.
8. The Layer2 network does not support ARP broadcasting.

---

### 14.3 IP Conflicts in the Address Pool

**Phenomena:**
- The same IP address is being used by another device.
- Access encounters issues.
- ARP resolution behaves abnormally.
- Connectivity fluctuates randomly.

**Troubleshooting:**
- Ping the IP address in question:
  ```bash
  ping 10.0.0.100
  ```
- Check the ARP table:
  ```bash
  arp -a | grep 10.0.0.100
  ```
- Verify the IP addresses in use:
  ```bash
  ip neigh | grep 10.0.0.100
  ```

**Actions:**
1. Immediately remove the conflicting IP address from the IPAddressPool.
2. Select an unused IP address from the pool.
3. Ensure that the MetalLB address range is reserved on the DHCP or network side.
4. Check the ARP tables of switches and gateways.

**Production Note:**
- The MetalLB address pool must be planned in advance.
- Do not arbitrarily select an IP range from the office network's DHCP pool.

---

### 14.4 Issues with the speaker Pod

- Check the Pods:
  ```bash
  kubectl -n metallb-system get pods -o wide
  ```
- Check the DaemonSet:
  ```bash
  kubectl -n metallb-system get ds
  ```
- Check the logs:
  ```bash
  kubectl -n metallb-system logs -l app.kubernetes.io## Seventeen. Uninstalling MetalLB

Note:

Before uninstalling, make sure that no business services are currently using the EXTERNAL-IP addresses allocated by MetalLB.
Do not uninstall it in a production environment.

Check the LoadBalancer service:

```bash
kubectl get svc -A | grep LoadBalancer
```

Delete the MetalLB address pool configurations:

```bash
kubectl delete -f metallb-l2advertisement.yaml
kubectl delete -f metallb-ipaddresspool.yaml
```

Uninstall the Helm release:

```bash
helm uninstall metallb -n metallb-system
```

Delete the namespace:

```bash
kubectl delete namespace metallb-system
```

Check the CRDs:

```bash
kubectl get crd | grep metallb
```

If you confirm that MetalLB is no longer needed, you can delete the CRDs:

```bash
kubectl delete crd ipaddresspools.metallb.io
kubectl delete crd l2advertisements.metallb.io
```

Note:

Deleting the CRDs will remove all related custom MetalLB resources.
If the cluster may still need MetalLB in the future, do not delete these CRDs.

---

## Eighteen. Post-Installation Verification Checklist

After completing the installation, perform the following checks:

```bash
kubectl -n metallb-system get pods -o wide
kubectl -n metallb-system get ipaddresspool
kubectl -n metallb-system get l2advertisement
kubectl -n kube-system get cm kube-proxy -o jsonpath '{.data.config\.conf}' | grep strictARP
kubectl -n metallb-demo get svc nginx-metallb-lb -o wide
curl http://${LB_IP}
```

The following should be confirmed:

1. The `metallb-controller` is running.
2. The `metallb-speaker` is running on the nodes.
3. `kube-proxy` has `strictARP` set to `true`.
4. The `IPAddressPool` has been created successfully.
5. The `L2Advertisement` has been created successfully.
6. The LoadBalancer service is able to obtain an EXTERNAL-IP address.
7. The EXTERNAL-IP address falls within the range 10.0.0.100-10.0.0.120.
8. Accessing the EXTERNAL-IP address via `curl` is successful.
9. There are no conflicts between the address pool and node IP addresses, API Server VIPs, or DHCP addresses.

---

## Nineteen. Summary

This document covers the basic installation and verification of MetalLB in a bare-metal cluster.

Key points:

1. Understanding why MetalLB is necessary for bare-metal clusters.
2. Installing MetalLB using Helm.
3. Configuring `kube-proxy` with strictARP.
4. Creating an `IPAddressPool`.
5. Generating an `L2Advertisement`.
6. Setting up a LoadBalancer service.
7. Verifying the allocation of EXTERNAL-IP addresses.
8. Checking business access functionality.
9. Understanding the relationship between MetalLB and Ingress/Gateway APIs.
10. Troubleshooting issues such as Pending status, access failures, IP address conflicts, or speaker exceptions.

Production recommendations:

1. Plan the MetalLB address pool in advance.
2. Ensure there are no conflicts with node IP addresses, API Server VIPs, or DHCP addresses.
3. When using `kube-proxy` with IPVS, pay close attention to `strictARP`.
4. The Layer 2 mode is suitable for simple bare-metal environments.
5. For multi-layer or complex network setups, consider using the BGP mode.
6. Ingress-nginx/Gateway APIs can obtain the LoadBalancer VIP through MetalLB.
7. MetalLB does not replace Ingress or Gateway APIs; it only provides an external IP address for the LoadBalancer.

With these steps, the basic installation of cluster components is complete:

- **00-Cloud-Native Foundation:** Kubectl operations tools and Helm installation guide.
- **01-Metrics-Server Installation:** `kubectl top`, resource metrics, and HPA basics.
- **02-Ingress-nginx Production Installation:** NodePort, IngressClass, and access verification.
- **03-Cert-Manager Installation:** TLS certificate automation, renewal, and Secret management.
- **04-Gateway API Introduction:** Envoy Gateway, GatewayClass, Gateway, and HTTPRoute.
- **05-StorageClass Basics:** NFS dynamic provisioning and default storage classes.
- **06-Longhorn Installation:** Distributed block storage, StorageClass, and PVC verification.
- **07-ExternalDNS Installation:** Ingress, Service, and external DNS record automation.
- **08-MetalLB Installation:** Bare-metal cluster LoadBalancer capabilities and business entry VIPs.