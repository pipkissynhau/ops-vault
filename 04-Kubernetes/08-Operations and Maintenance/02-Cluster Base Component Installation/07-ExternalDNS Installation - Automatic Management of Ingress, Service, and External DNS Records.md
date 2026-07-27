# 07-ExternalDNS Installation: Automatic Management of Ingress, Service, and External DNS Records

Recommended Path:

    04-Kubernetes/08-Ops/02-Cluster Basic Components Installation/07-ExternalDNS Installation: Automatic Management of Ingress, Service, and External DNS Records.md

Tags:

    #Kubernetes
    #ExternalDNS
    #DNS
    #Ingress
    #GatewayAPI
    #Service
    #Domain Name Resolution
    #Cluster Basic Components
    #DNS Automatic Management

---

## I. Document Description

This document records the basic installation, verification, and troubleshooting methods for ExternalDNS in a Kubernetes cluster.

The functions of ExternalDNS are:

    To monitor resources such as Ingress, Service, Gateway / HTTPRoute in Kubernetes.
    Based on the domain name information in these resources,
    to automatically create, update, or delete DNS resolution records with external DNS service providers.

Common DNS service providers include:

    Alibaba Cloud DNS
    Cloudflare
    AWS Route53
    Azure DNS
    Google Cloud DNS
    Internal DNS platforms
    DNS services that support RFC2136

Key points of this document:

    1. Understand the differences between ExternalDNS and CoreDNS.
    2. Install ExternalDNS using Helm.
    3. Verify the installation in dry-run mode first.
    4. Generate DNS records through Ingress.
    5. Generate DNS records via Service annotations.
    6. Master methods for troubleshooting issues related to logs, permissions, domain name filtering, and record ownership.

Execution Node:

    k8s-master-01

Prerequisites:

    1. The Kubernetes cluster has been deployed successfully.
    2. kubectl can access the cluster normally.
    3. Helm is installed.
    4. ingress-nginx or Gateway API is already installed.
    5. There are manageable domain names available.
    6. API permissions for the DNS service provider have been obtained.

---

## II. Differences between ExternalDNS and CoreDNS

| Component | Management Scope | Function |
|---|---|---|
| CoreDNS | Internal cluster DNS | Resolves Pod access to Services, e.g., nginx.default.svc.cluster.local. |
| ExternalDNS | External DNS service provider | Automatically creates external DNS records like app.example.com. |

Simply put:

    CoreDNS manages within the cluster:
        - Pod access to Services.
        - Service domain name resolution.
        - svc.cluster.local.

    ExternalDNS manages outside the cluster:
        - Ingress domain name resolution.
        - LoadBalancer Service domain name resolution.
        - Gateway / HTTPRoute domain name resolution.
        - A/CNAME/TXT records on cloud DNS or company DNS platforms.

ExternalDNS is not a DNS server itself. It acts as a controller that:

    Reads resources from the Kubernetes API.
    Calculates the required DNS records.
    Calls the DNS service provider's API.
    Synchronizes the DNS records.

---

## III. Use Cases

ExternalDNS is suitable for:

    1. Scenarios with numerous business domain names.
    2. Situations where Ingress domain names are frequently added or changed.
    3. Cases where Gateway API domain names need automatic synchronization.
    4. When Service LoadBalancer addresses require automatic addition to DNS.
    5. When you prefer not to manually log in to the DNS console to create resolution records.
    6. If your company's internal DNS platform supports APIs.
    7. If cloud provider DNS services offer API support.

ExternalDNS is not suitable for:

    1. Situations where you do not have API permissions for the DNS service provider.
    2. Local testing scenarios using only hosts files.
    3. Cases where all domain name resolutions are manually maintained.
    4. Environments where Kubernetes is not allowed to automatically modify DNS records.
    5. When the business entry IP or LoadBalancer address is not yet determined.

---

## IV. Deployment Plan in This Document

This document uses Alibaba Cloud DNS as an example. For other DNS service providers, you need to replace the provider and authentication parameters accordingly.

| Item | Planning |
|---|---|
| Component Name | external-dns |
| Namespace | external-dns |
| Installation Method | Helm |
| Chart | external-dns/external-dns |
| Chart Version | 1.20.0 |
| Provider Example | alibabacloud |
| Managed Domain Name | example.com |
| Source | ingress, service |
| Registry | txt |
| TXT Owner ID | k8s-prod |
| Default Policy | upsert-only |
| First Installation | dry-run=true |

Important Notes:

    example.com is just an example domain name. Replace it with your actual domain name in a production environment.
    Do not directly apply this deployment plan to a production environment. Always perform a dry-run first and verify the logs before proceeding.

---

## V. DNS```markdown
kubectl -n external-dns create secret generic alidns-secret \
      --from-literal=ALIBABA_CLOUD_ACCESS_KEY_ID '<your-access-key-id>' \
      --from-literal=ALIBABA_CLOUD_ACCESS_KEY_SECRET '<your-access-key-secret>'

Check if the Secret exists:

    kubectl -n external-dns get secret alidns-secret

Explanation:

    ALIBABA_CLOUD_ACCESS_KEY_ID
        The AccessKey ID for your Alibaba Cloud RAM account.

    ALIBABA_CLOUD_ACCESS_KEY_SECRET
        The AccessKey Secret for your Alibaba Cloud RAM account.

Security Requirements:

    Do not copy the command `kubectl get secret -o yaml` and distribute it elsewhere.
    Do not include commands containing actual secrets in documentation or script repositories.
    In a production environment, it is recommended to use External Secrets, Vault, cloud Secret Manager, or workload identities instead.
```

---

## Section 9: Adding an ExternalDNS Helm Repository

Add the repository:

    helm repo add external-dns https://kubernetes-sigs.github.io/external-dns/

Update the repository:

    helm repo update

View charts:

    helm search repo external-dns/external-dns

View default values:

    mkdir -p /root/k8s-yaml/external-dns

    cd /root/k8s-yaml/external-dns

    helm show values external-dns/external-dns --version 1.20.0 > values-external-dns-default.yaml

View key configurations:

    grep -n "provider:" values-external-dns-default.yaml

    grep -n "sources:" values-external-dns-default.yaml

    grep -n "registry:" values-external-dns-default.yaml

    grep -n "txtOwnerId:" values-external-dns-default.yaml
```

---

## Section 10: Creating ExternalDNS Values Files

### 10.1 Dry-run Version of Values File

It is recommended to enable dry-run during the initial installation.

In dry-run mode:

    ExternalDNS will read Kubernetes resources,
    calculate the DNS records that need to be created or updated,
    and log the planned actions, but it will not actually modify the DNS service provider's records.

Create the values file:

    cat <<EOF > values-external-dns-alidns-dryrun.yaml
    fullnameOverride: external-dns

    provider:
      name: alibabacloud

    sources:
      - ingress
      - service

    domainFilters:
      - example.com

    policy: upsert-only

    registry: txt

    txtOwnerId: k8s-prod

    interval: 1m

    logLevel: info

    extraArgs:
      - --dry-run
      - --alibaba-cloud-zone-type=public

    env:
      - name: ALIBABA_CLOUD_ACCESS_KEY_ID
        valueFrom:
          secretKeyRef:
            name: alidns-secret
            key: ALIBABA_CLOUD_ACCESS_KEY_ID
      - name: ALIBABA_CLOUD_ACCESS_KEY_SECRET
        valueFrom:
          secretKeyRef:
            name: alidns-secret
            key: ALIBABA_CLOUD_ACCESS_KEY_SECRET

    resources:
      requests:
        cpu: 50m
        memory: 64Mi
      limits:
        cpu: 200m
        memory: 256Mi
    EOF

You must modify the following values:

    example.com
        Replace this with the actual domain name you want to manage.

    k8s-prod
        Replace this with the unique and stable owner ID of your current cluster.

Explanation:

    provider.name: alibabacloud
        Uses the Alibaba Cloud DNS Provider.

    sources:
        Monitors Ingress and Service resources.

    domainFilters:
        Limits management to only the example.com domain name to prevent accidental operations on other domains.

    policy: upsert-only
        Only creates and updates records; does not delete them, which is safer in the initial phase.

    registry: txt
        Uses TXT records for ownership management.

    txtOwnerId:
        Identifies the owner of the current ExternalDNS instance.

    --dry-run
        Only outputs planned actions without actually modifying DNS records.

    --alibaba-cloud-zone-type=public
        Synchronizes with Alibaba Cloud's public DNS zone.
```

### 10.2 Official Version of Values File

After confirming that the dry-run logs are correct, prepare the official version of the values file.

Create a copy:

    cp values-external-dns-alidns-dryrun.yaml values-external-dns-alidns.yaml

Remove the dry-run parameters:

    sed -i.bak '/--dry-run/d' values-external-dns-alidns.yaml

Check:

    grep -n "dry-run" values-external-dns-alidns.yaml

If no result is displayed, it means that### 12.2 Creating an Ingress

Note:

You must replace `external-dns-demo.example.com` with your actual domain name.
The domain name suffix must match the `domainFilters`.

Create an Ingress:

```bash
cat <<EOF > ingress-external-dns-demo.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-dns-demo
  namespace: external-dns-demo
  annotations:
    external-dns.alpha.kubernetes.io/ttl: "60"
    external-dns_alpha.kubernetes.io/target: "10.0.0.100"
spec:
  ingressClassName: nginx
  rules:
  - host: external-dns-demo.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-dns-demo
            port:
              number: 80
EOF
```

Apply the configuration:

```bash
kubectl apply -f ingress-external-dns-demo.yaml
```

Check the Ingress:

```bash
kubectl -n external-dns-demo get ingress
kubectl -n external-dns-demo describe ingress nginx-dns-demo
```

Explanation:

`external-dns.alpha.kubernetes.io/target` is used to explicitly specify which IP or domain name the DNS record should point to. This is particularly useful in environments without a LoadBalancer IP, such as self-built clusters.

If you have an independent business entry VIP in production, for example `10.0.0.100`, you can use this to direct `external-dns-demo.example.com` to resolve to `10.0.0.100`.4. Abnormalities in Containerd Networking or Proxy

Resolution:

    helm show values external-dns/external-dns --version 1.20.0 > values-default.yaml

Check the image fields:

    grep -n "image:" values-default.yaml

    grep -n "repository:" values-default.yaml

After synchronizing the image to the internal Harbor, modify the image.repository in the values file.

---

### 16.2 Failure to Create DNS Records

View the logs:

    kubectl -n external-dns logs deploy/external-dns --tail=200

Check the resources:

    kubectl get ingress -A

    kubectl get svc -A

Verify if the domain names match the domainFilters set in values-external-dns-alidns.yaml:

    grep -n "domainFilters" values-external-dns-alidns.yaml

Common causes:

    1. Mismatch between domainFilters and actual domain names
    2. Ingress resource lacks a host field
    3. Service resource does not have the external-dns hostname annotation
    4. Absence of a target address
    5. Authentication failure with the DNS provider
    6. Dry-run mode still enabled
    7. Policy configuration preventing the expected actions

---

### 16.3 Authentication Failure

Check the logs:

    kubectl -n external-dns logs deploy/external-dns --tail=200

Verify the Secret configured for authentication:

    kubectl -n external-dns get secret alidns-secret

Confirm if the environment variables have been correctly injected into the application:

    kubectl -n external-dns get deploy external-dns -o yaml | grep -A20 "env:"

Common issues:

    1. Incorrect name of the Secret
    2. Incorrect name of the AccessKey
    3. Invalid AccessKey
    4. Insufficient permissions for the AccessKey
    5. Use of a root account with restricted security policies
    6. DNS provider API blocked by network or outbound policies

---

### 16.4 Insufficient Permissions

Common error messages:

    Forbidden
    AccessDenied
    Unauthorized
    PermissionDenied

Solution steps:

    1. Verify the permissions of the current user
    2. Check if operations like AddDomainRecord, UpdateDomainRecord, and DeleteDomainRecord are allowed
    3. Ensure that DescribeDomainRecords is also permitted
    4. Verify if the specified resource scope is correct

---

### 16.5 Unexpected IP Address in Resolution Results

Check the Ingress annotations:

    kubectl -n external-dns-demo get ingress nginx-dns-demo -o yaml | grep -A5 annotations

Inspect the Service annotations:

    kubectl -n external-dns-demo get svc nginx-dns-svc -o yaml | grep -A10 annotations

Review the ExternalDNS logs:

    kubectl -n external-dns logs deploy/external-dns --tail=200

Possible reasons:

    1. Incorrect target annotation
    2. Ingress Controller unable to resolve the address
    3. Service not configured as a LoadBalancer
    4. Old DNS cache not cleared
    5. Manual entries in the DNS platform remain

---

### 16.6 Inconsistent TXT Owner ID

Issues:

    ExternalDNS fails to update old records
    Logs indicate that record ownership does not match

Check the values file:

    grep -n "txtOwnerId" values-external-dns-alidns.yaml

Guidelines for resolution:

    1. Once the txtOwnerId is set, do not modify it arbitrarily
    2. Different txtOwnerIDs must be used for multiple clusters
    3. Retain the original txt Ownership ID when migrating ExternalDNS
    4. Do not manually delete TXT records unless you are certain it will not affect other systems

---

### 16.7 Risk Control Measures for Accidental Deletion or Modification of DNS Records

Pre-production steps:

    1. Use domainFilters to restrict the range of domain names
    2. Execute dry-run tests to monitor logs
    3. Use upsert-only mode instead of sync for data updates
    4. Use dedicated test subdomains
    5. Assign unique txtOwnershipIDs
    6. Avoid using root account AccessKeys
    7. Do not manage root domain names during initial deployment

Recommended test domain:

    k8s-test.example.com

Avoid managing directly for initial use:

    example.com
    www.example.com
    api.example.com

---

## Section Seventeen: Upgrade and Rollback

### 17.1 Checking the Current Version

View Helm releases:

    helm list -n external-dns

Check the current status:

    helm status external-dns -n external-dns

Review historical6. Ingress domain names can generate DNS records.
7. Service annotations can generate DNS records.
8. The DNS resolution results meet expectations.
9. Corresponding A/CNAME and TXT records can be seen in the DNS console.

---

## 21. Summary

This document outlines the basic installation and verification of ExternalDNS.

Key points:

    1. Differences between ExternalDNS and CoreDNS.
    2. Installation of ExternalDNS using Helm.
    3. Security verification through dry-run tests.
    4. Use of domainFilters to restrict management scope.
    5. Utilization of txtOwnerId to identify record ownership.
    6. Automatic generation of DNS records via Ingress.
    7. Automatic generation of DNS records using Service annotations.
    8. Troubleshooting common issues related to authentication, permissions, domainFilters, targets, and TXT record ownership.

Production recommendations:

    1. Always perform dry-run tests before deploying for the first time.
    2. It is recommended to use upsert-only mode during initial deployment.
    3. Domain Filters must be configured correctly.
    4. A stable txtOwnerId must be assigned.
    5. Never use the root account's AccessKey for deployment.
    6. Do not store real AccessKeys in Git or documentation.
    7. Different clusters should not share the same txtInstanceId.
    8. Always test with a trial subdomain before moving to production domains.