# 07-ExternalDNS Installation: Ingress, Service, and Automatic Management of External DNS Records

Recommended path:

    04-Kubernetes/08-Operations/02-Cluster Base Component Installation/07-ExternalDNS Installation: Ingress, Service, and Automatic Management of External DNS Records.md

Tags:

    #Kubernetes
    #ExternalDNS
    #DNS
    #Ingress
    #GatewayAPI
    #Service
    #DomainNameResolution
    #ClusterBasicComponents
    #DnsAutomanaging

---

## I. Document Description

This document records the basic installation, verification, and troubleshooting methods for ExternalDNS in a Kubernetes cluster.

The role of ExternalDNS is:

    Monitor Kubernetes Ingress, Service, Gateway / HTTPRoute resources
    According to domain information in the resources
    Automatically create, update, or delete DNS record with external DNS service provider

Common DNS service providers include:

    Alibaba Cloud DNS
    Cloudflare
    AWS Route53
    Azure DNS
    Google Cloud DNS
    Internal DNS platform
    DNS services supporting RFC2136

This document focuses on:

    1. Understanding the difference between ExternalDNS and CoreDNS
    2. Using Helm to install ExternalDNS
    3. First verify with dry-run mode
    4. Generate DNS records via Ingress
    5. Generate DNS records via Service annotation
    6. Master log, permission, domain filtering, and record ownership troubleshooting methods

Execution node:

    k8s-master-01

Prerequisites:

    1. Kubernetes cluster has been deployed
    2. kubectl can access the cluster normally
    3. Helm is installed
    4. ingress-nginx or Gateway API is installed
    5. A manageable domain is available
    6. DNS service provider API permissions are available

---

## II. Difference Between ExternalDNS and CoreDNS

| Component | Management Scope | Function |
|---|---|---|
| CoreDNS | Cluster internal DNS | Pod resolution for Service, e.g. nginx.default.svc.cluster.local |
| ExternalDNS | External DNS service provider | Automatically create external DNS records like app.example.com |

Simple understanding:

    CoreDNS manages within the cluster:
        Pod access to Service
        Service domain resolution
        svc.cluster.local

    ExternalDNS manages outside the cluster:
        Ingress domain resolution
        LoadBalancer Service domain resolution
        Gateway / HTTPRoute domain resolution
        A / CNAME / TXT records on cloud DNS or company DNS platform

ExternalDNS is not a DNS server.

It is merely a controller:

    Read resources from Kubernetes API
    Calculate required DNS records
    Call DNS service provider API
    Synchronize DNS records

---

## III. Applicable Scenarios

ExternalDNS is suitable for:

    1. Many business domains
    2. Frequent addition or change of Ingress domains
    3. Gateway API domains needing automatic synchronization
    4. Service LoadBalancer addresses needing automatic DNS entry
    5. Not wanting to manually log into DNS console to create resolution records
    6. Company internal DNS platform supports API
    7. Cloud vendor DNS supports API

ExternalDNS is not suitable for:

    1. No DNS service provider API permissions
    2. Only local hosts testing
    3. All domain resolution manually maintained
    4. Not allowing Kubernetes to automatically modify DNS records
    5. No clear business entry IP or LoadBalancer address yet

---

## IV. Deployment Plan in This Document

This document uses Alibaba Cloud DNS as an example, other DNS service providers need to replace provider and authentication parameters.

| Item | Plan |
|---|---|
| Component name | external-dns |
| Namespace | external-dns |
| Installation method | Helm |
| Chart | external-dns/external-dns |
| Chart version | 1.20.0 |
| Provider example | alibabacloud |
| Managed domain | example.com |
| Source | ingress, service |
| Registry | txt |
| TXT Owner ID | k8s-prod |
| Default policy | upsert-only |
| First installation | dry-run=true |

Important notes:

    example.com is an example domain.
    Must replace with real domain in actual environment.
    Do not directly copy to production environment.
    Production environment must first enable dry-run, confirm logs are correct before disabling dry-run.

---

## V. DNS Record Management Logic

ExternalDNS generally manages two types of records:

    1. Business DNS records
    2. TXT ownership records

Example:

    app.example.com    A      10.0.0.100
    app.example.com    TXT    heritage=external-dns;external-dns/owner=k8s-prod

Where:

    A / CNAME records:
        For business access use.

    TXT records:
        For ExternalDNS to determine if it manages this record.

Why TXT records are needed:

    1. Avoid accidental deletion of manually maintained DNS records
    2. Identify which ExternalDNS instance manages this record
    3. Avoid mutual record conflicts when multiple clusters share a DNS Zone
    4. Support for future record synchronization and cleanup

Production recommendations:

    txtOwnerId should not be changed once determined.
    If txtOwnerId is modified, old TXT records may not be recognized by new instances.

---

## VI. Pre-Deployment Checks

### 6.1 Check Cluster Status

Execute: /think

kubectl get nodes -o wide

Requirements:

    All nodes Ready.

---

### 6.2 Check Helm

Execute:

    helm version

---

### 6.3 Check Ingress

If using Ingress as source, first confirm ingress-nginx is normal:

    kubectl -n ingress-nginx get pods -o wide

    kubectl get ingressclass

---

### 6.4 Check Gateway API (Optional)

If using Gateway API as source, first confirm Gateway API resources exist:

    kubectl get gatewayclass

    kubectl get gateway -A

    kubectl get httproute -A

---

### 6.5 Check Domain

Confirm you have a manageable domain, for example:

    example.com

Confirm the DNS service provider console contains the domain's DNS Zone.

---

## VII. Prepare DNS Service Provider Permissions

### 7.1 Permission Principles

ExternalDNS needs to call DNS service provider API.

Minimum privilege principle:

    1. Only allow management of specified domains
    2. Only allow creation, update, and deletion of necessary records
    3. Do not use main account AccessKey
    4. Use independent RAM sub-account or role
    5. AccessKey not written to Git repository
    6. AccessKey not written to Markdown documents
    7. Production environment prefers temporary credentials, RAM Role, or workload identity capabilities

---

### 7.2 Common Permissions for Alibaba Cloud DNS

If using Alibaba Cloud DNS, RAM permissions typically need to include:

    alidns:AddDomainRecord
    alidns:DeleteDomainRecord
    alidns:UpdateDomainRecord
    alidns:DescribeDomainRecords
    alidns:DescribeDomains

If using Alibaba Cloud PrivateZone, also need:

    pvtz:AddZoneRecord
    pvtz:DeleteZoneRecord
    pvtz:UpdateZoneRecord
    pvtz:DescribeZoneRecords
    pvtz:DescribeZones
    pvtz:DescribeZoneInfo

Note:

    Actual permissions should be configured according to company security policies.
    Do not directly grant excessive DNS management permissions.
    Do not use Alibaba Cloud main account AccessKey.

---

## VIII. Create DNS Credential Secret

The following example uses Kubernetes Secret to save AccessKey.

Note:

    Here only variable names are written, not real keys.
    Replace placeholders with real values during actual operation.
    Do not submit real AccessKey to Git repository.

Create namespace:

    kubectl create namespace external-dns

Create Secret:

    kubectl -n external-dns create secret generic alidns-secret \
      --from-literal=ALIBABA_CLOUD_ACCESS_KEY_ID='<your-access-key-id>' \
      --from-literal=ALIBABA_CLOUD_ACCESS_KEY_SECRET='<your-access-key-secret>'

Check if Secret exists:

    kubectl -n external-dns get secret alidns-secret

Explanation:

    ALIBABA_CLOUD_ACCESS_KEY_ID
        Alibaba Cloud RAM user AccessKey ID.

    ALIBABA_CLOUD_ACCESS_KEY_SECRET
        Alibaba Cloud RAM user AccessKey Secret.

Security requirements:

    Do not kubectl get secret -o yaml everywhere.
    Do not write commands containing real keys to documents or script repositories.
    Production environment recommends integrating with External Secrets, Vault, cloud Secret Manager, or workload identity.

---

## IX. Add ExternalDNS Helm Repository

Add repository:

    helm repo add external-dns https://kubernetes-sigs.github.io/external-dns/

Update repository:

    helm repo update

View Chart:

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

---

## X. Create ExternalDNS values File

### 10.1 dry-run Version values

Recommended to enable dry-run for initial installation.

In dry-run mode:

    ExternalDNS will read Kubernetes resources
    Will calculate needed DNS record creations or updates
    Will output planned actions in logs
    Will not actually modify DNS service provider records

Create values file:

    cat <<EOF > values-external-dns-alidns-dryrun.yaml
    fullnameOverride: external-dns

    provider:
      name: alibabacloud

    sources:
      - ingress
      - service /think

```yaml
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
```

---

Must modify:

  - example.com
      Change to the actual domain to be managed.

  - k8s-prod
      Change to the current cluster's unique and stable owner ID.

Explanation:

  - provider.name: alibabacloud
      Use Alibaba Cloud DNS Provider.

  - sources:
      Monitor ingress and service.

  - domainFilters:
      Restrict management to only example.com domain suffix to avoid accidental operations on other domains.

  - policy: upsert-only
      Only create and update records, do not actively delete records. More secure in initial phase.

  - registry: txt
      Use TXT record for ownership management.

  - txtOwnerId:
      Current ExternalDNS instance's ownership ID.

  - --dry-run
      Only output planned actions, do not actually modify DNS.

  - --alibaba-cloud-zone-type=public
      Synchronize only Alibaba Cloud public DNS Zone.

---

### 10.2 Formal Write Version values

After confirming dry-run logs are correct, prepare the formal version.

Create:

  - cp values-external-dns-alidns-dryrun.yaml values-external-dns-alidns.yaml

Remove dry-run parameter:

  - sed -i.bak '/--dry-run/d' values-external-dns-alidns.yaml

Check:

  - grep -n "dry-run" values-external-dns-alidns.yaml

If no output, it means dry-run has been removed.

Note:

  - It is recommended to still use policy: upsert-only for production first deployment.
  - Do not recommend starting with policy: sync.
  - sync will let ExternalDNS attempt to clean up records it considers unnecessary, higher risk.

---

## ElevenI don't know.Install ExternalDNS

### 11.1 dry-run Installation

Execute:

  - helm upgrade --install external-dns external-dns/external-dns \
    -n external-dns \
    -f values-external-dns-alidns-dryrun.yaml \
    --version 1.20.0

Check Helm Release:

  - helm list -n external-dns

Check Pod:

  - kubectl -n external-dns get pods -o wide

Check logs:

  - kubectl -n external-dns logs deploy/external-dns --tail=100

---

### 11.2 Confirm dry-run Logs

Focus on logs for:

  1. Whether can connect to DNS Provider normally
  2. Whether only scan specified domainFilters
  3. Whether discover Ingress / Service domain names
  4. Whether output planned A / CNAME / TXT record creations
  5. Whether appear permission errors
  6. Whether appear authentication errors

If domain scope is incorrect in logs, immediately stop:

  - helm uninstall external-dns -n external-dns

Modify values and re-dry-run after.

---

### 11.3 Enable ExternalDNS Formally

After confirming dry-run is correct, execute:

  - helm upgrade external-dns external-dns/external-dns \
    -n external-dns \
    -f values-external-dns-alidns.yaml \
    --version 1.20.0

Check logs:

  - kubectl -n external-dns logs deploy/external-dns --tail=100 -f

---

## TwelveI don't know.Verify DNS Auto Creation via Ingress

### 12.1 Create Test Application

Create namespace:

  - kubectl create namespace external-dns-demo

Create application:

  - kubectl -n external-dns-demo create deployment nginx-dns-demo --image=nginx:1.25

Create Service:

  - kubectl -n external-dns-demo expose deployment nginx-dns-demo \
    --port=80 \
    --target-port=80 \
    --type=ClusterIP

Check:

  - kubectl -n external-dns-demo get pods -o wide
```

kubectl -n external-dns-demo get svc

kubectl -n external-dns-demo get endpoints nginx-dns-demo

---

### 12.2 Creating Ingress

Note:

    external-dns-demo.example.com must be replaced with your actual domain.
    The domain suffix must match domainFilters.

Create Ingress:

    cat <<EOF > ingress-external-dns-demo.yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: nginx-dns-demo
      namespace: external-dns-demo
      annotations:
        external-dns.alpha.kubernetes.io/ttl: "60"
        external-dns.alpha.kubernetes.io/target: "10.0.0.100"
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

Apply:

    kubectl apply -f ingress-external-dns-demo.yaml

Check:

    kubectl -n external-dns-demo get ingress

    kubectl -n external-dns-demo describe ingress nginx-dns-demo

Explanation:

    external-dns.alpha.kubernetes.io/target
        Used to explicitly specify the IP or domain that the DNS record points to.
        Commonly used in environments like NodePort, self-hosted clusters, or without LoadBalancer IP.

    If production has an independent business entry VIP, for example:
        10.0.0.100

    Then ExternalDNS can make:
        external-dns-demo.example.com

    Resolve to:
        10.0.0.100

---

### 12.3 Checking ExternalDNS Logs

Execute:

    kubectl -n external-dns logs deploy/external-dns --tail=100 -f

Watch for the following appearing in the logs:

    CREATE
    UPDATE
    desired change
    external-dns-demo.example.com

---

### 12.4 Verifying DNS Resolution

Wait for some time, then run the following on any client:

    nslookup external-dns-demo.example.com

Or:

    dig external-dns-demo.example.com

If it resolves to:

    10.0.0.100

It indicates that the Ingress successfully triggered automatic DNS creation.

---

## ThirteenI don't know.Verifying DNS Automatic Creation via Service

Service method is commonly used for:

    LoadBalancer Service
    Services with explicit external-dns annotations

In self-hosted clusters without LoadBalancer, you can specify target via annotation.

---

### 13.1 Creating a Service with Annotation

Create Service:

    cat <<EOF > svc-external-dns-demo.yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: nginx-dns-svc
      namespace: external-dns-demo
      annotations:
        external-dns.alpha.kubernetes.io/hostname: svc-dns-demo.example.com
        external-dns.alpha.kubernetes.io/target: "10.0.0.100"
        external-dns.alpha.kubernetes.io/ttl: "60"
    spec:
      type: ClusterIP
      selector:
        app: nginx-dns-demo
      ports:
      - name: http
        port: 80
        targetPort: 80
    EOF

Apply:

    kubectl apply -f svc-external-dns-demo.yaml

Check:

    kubectl -n external-dns-demo get svc nginx-dns-svc -o yaml

---

### 13.2 Checking ExternalDNS Logs

Execute:

    kubectl -n external-dns logs deploy/external-dns --tail=100 -f

Watch for:

    svc-dns-demo.example.com

---

### 13.3 Verifying Resolution

Execute:

    nslookup svc-dns-demo.example.com

Or:

    dig svc-dns-demo.example.com

Expected resolution to:

    10.0.0.100

---

## FourteenI don't know.Notes on Integration with Gateway API

ExternalDNS can also integrate with Gateway API, but actual support depends on ExternalDNS version, source configuration, and Gateway Controller implementation.

Basic understanding: /think

# Gateway Definition
# HTTPRoute Definition of Domain and Routing
# ExternalDNS Generates DNS Records Based on Hostname in Gateway / HTTPRoute

Production Recommendations:

    1. First ensure Ingress scenario works
    2. Then verify Gateway API scenario separately
    3. Start with dry-run
    4. Enable real write after verification
    5. Only manage one test domain suffix at a time

If you need to enable Gateway API source later, confirm sources configuration based on current ExternalDNS version, for example:

    sources:
      - gateway-httproute

Or use source supported by current version documentation for Gateway.

This article will focus on Ingress and Service first, without expanding Gateway API details.

---

## Fifteen, Common Annotations

### 15.1 Specify Domain

Service commonly uses:

    external-dns.alpha.kubernetes.io/hostname: app.example.com

Ingress typically uses:

    spec.rules.host

You can also use annotation to override or supplement.

---

### 15.2 Specify Target Address

Commonly used in self-built clusters without LoadBalancer IP:

    external-dns.alpha.kubernetes.io/target: "10.0.0.100"

Can also point to CNAME:

    external-dns.alpha.kubernetes.io/target: "lb.example.com"

---

### 15.3 Specify TTL

Example:

    external-dns.alpha.kubernetes.io/ttl: "60"

Notes:

    Smaller TTL means faster DNS change propagation.
    Smaller TTL means higher DNS query pressure.
    Production environment needs to set based on DNS platform and business requirements.

---

## Sixteen, Common Troubleshooting

### 16.1 ExternalDNS Pod ImagePullBackOff

Check Pod:

    kubectl -n external-dns get pods -o wide

Check events:

    kubectl -n external-dns describe pod <pod-name>

Common causes:

    1. Unable to access registry.k8s.io
    2. Unable to pull external-dns image
    3. Internal Harbor not synced with image
    4. containerd network or proxy issues

Resolution:

    helm show values external-dns/external-dns --version 1.20.0 > values-default.yaml

Check image field:

    grep -n "image:" values-default.yaml

    grep -n "repository:" values-default.yaml

After syncing image to internal Harbor, modify image.repository in values.

---

### 16.2 No DNS Record Created

Check logs:

    kubectl -n external-dns logs deploy/external-dns --tail=200

Check resources:

    kubectl get ingress -A

    kubectl get svc -A

Check if domain matches domainFilters:

    grep -n "domainFilters" values-external-dns-alidns.yaml

Common causes:

    1. domainFilters not matching
    2. Ingress has no host
    3. Service lacks external-dns hostname annotation
    4. No target address
    5. DNS Provider authentication failed
    6. dry-run not closed
    7. policy configuration prevents expected actions

---

### 16.3 Authentication Failed

Check logs:

    kubectl -n external-dns logs deploy/external-dns --tail=200

Check Secret:

    kubectl -n external-dns get secret alidns-secret

Check environment variables injection:

    kubectl -n external-dns get deploy external-dns -o yaml | grep -A20 "env:"

Common causes:

    1. Secret name error
    2. Secret key name error
    3. Invalid AccessKey
    4. AccessKey insufficient permissions
    5. Main account restricted by security policy
    6. DNS provider API blocked by network or outbound policies

---

### 16.4 Insufficient Permissions

Common logs:

    Forbidden
    AccessDenied
    Unauthorized
    PermissionDenied

Resolution approach:

    1. Check RAM user permissions
    2. Check if AddDomainRecord is allowed
    3. Check if UpdateDomainRecord is allowed
    4. Check if DeleteDomainRecord is allowed
    5. Check if DescribeDomainRecords is allowed
    6. Check if resource scope is incorrectly limited

---

### 16.5 Resolution Result Not Expected IP

Check Ingress annotation:

    kubectl -n external-dns-demo get ingress nginx-dns-demo -o yaml | grep -A5 annotations

Check Service annotation:

    kubectl -n external-dns-demo get svc nginx-dns-svc -o yaml | grep -A10 annotations

Check ExternalDNS logs:

    kubectl -n external-dns logs deploy/external-dns --tail=200

Common causes:

1. Target annotation is written incorrectly  
2. Ingress Controller has no identifiable address  
3. Service is not LoadBalancer  
4. Old DNS cache has not expired  
5. Old manual records exist in DNS platform  

---

### 16.6 TXT Owner ID Inconsistency  

Phenomenon:  

    ExternalDNS does not update old records  
    Or logs indicate record ownership mismatch  

Check values:  

    grep -n "txtOwnerId" values-external-dns-alidns.yaml  

Handling principles:  

    1. txtOwnerId should not be modified arbitrarily once deployed  
    2. Multi-cluster environments must use different txtOwnerId  
    3. When migrating ExternalDNS, retain the original txtOwnerId  
    4. Do not manually delete TXT records unless the impact scope is confirmed  

---

### 16.7 Risk Control for Accidental DNS Deletion or Modification  

Must be done before deployment:  

    1. Use domainFilters to restrict domain scope  
    2. Use dry-run to observe logs  
    3. Use upsert-only instead of sync  
    4. Use independent test subdomains  
    5. Use independent txtOwnerId  
    6. Do not use main account AccessKey  
    7. Do not manage root domains on first deployment  

Recommended test domain:  

    k8s-test.example.com  

Not recommended to manage initially:  

    example.com  
    www.example.com  
    api.example.com  

---

## SeventeenI don't know.Upgrade and Rollback  

### 17.1 Check Current Version  

Check Helm Release:  

    helm list -n external-dns  

Check status:  

    helm status external-dns -n external-dns  

Check history:  

    helm history external-dns -n external-dns  

---

### 17.2 Backup Current values  

Execute:  

    helm get values external-dns -n external-dns -o yaml > external-dns-values-backup.yaml  

---

### 17.3 Upgrade  

Update repository:  

    helm repo update  

Upgrade:  

    helm upgrade external-dns external-dns/external-dns \
      -n external-dns \
      -f values-external-dns-alidns.yaml \
      --version 1.20.0  

Check status:  

    kubectl -n external-dns get pods -o wide  

    kubectl -n external-dns logs deploy/external-dns --tail=100  

---

### 17.4 Rollback  

Check history:  

    helm history external-dns -n external-dns  

Rollback:  

    helm rollback external-dns <REVISION> -n external-dns  

Check:  

    helm status external-dns -n external-dns  

---

## EighteenI don't know.Cleanup Test Resources  

Delete Ingress:  

    kubectl delete -f ingress-external-dns-demo.yaml  

Delete test Service:  

    kubectl delete -f svc-external-dns-demo.yaml  

Delete application:  

    kubectl -n external-dns-demo delete svc nginx-dns-demo  

    kubectl -n external-dns-demo delete deployment nginx-dns-demo  

Delete namespace:  

    kubectl delete namespace external-dns-demo  

Note:  

    If policy is upsert-only, ExternalDNS may not automatically delete DNS records.  
    If automatic deletion is required, evaluate whether to use policy: sync.  
    Do not use sync in production environments initially.  

---

## NineteenI don't know.Uninstall ExternalDNS  

Uninstall:  

    helm uninstall external-dns -n external-dns  

Delete Secret:  

    kubectl -n external-dns delete secret alidns-secret  

Delete namespace:  

    kubectl delete namespace external-dns  

Note:  

    Uninstalling ExternalDNS does not automatically delete DNS provider records.  
    You must confirm in DNS console whether to retain or clean up test records.  

---

## TwentyI don't know.Installation Completion Checklist  

Execute:  

    helm list -n external-dns  

    kubectl -n external-dns get pods -o wide  

    kubectl -n external-dns logs deploy/external-dns --tail=100  

    kubectl get ingress -A  

    kubectl get svc -A  

    nslookup external-dns-demo.example.com  

    nslookup svc-dns-demo.example.com  

Should satisfy:  

    1. ExternalDNS Pod Running  
    2. No authentication errors in logs  
    3. No permission errors in logs  
    4. domainFilters only include expected domains  
    5. txtOwnerId is fixed  
    6. Ingress domain can generate DNS records  
    7. Service annotation can generate DNS records  
    8. DNS resolution results match expectations  
    9. DNS console shows corresponding A/CNAME and TXT records  

---

## Twenty-oneI don't know.Summary  

This document completes the basic installation and verification of ExternalDNS.  

Core content:

1. Differences Between ExternalDNS and CoreDNS  
2. Install ExternalDNS Using Helm  
3. Use dry-run for Safe Validation  
4. Use domainFilters to Limit Management Scope  
5. Use txtOwnerId to Identify Record Ownership  
6. Automatically Generate DNS Records via Ingress  
7. Automatically Generate DNS Records via Service Annotation  
8. Troubleshoot Common Issues: Authentication, Permissions, domainFilters, Target, TXT Ownership, etc  

Production Recommendations:  

1. Dry-run is mandatory on initial deployment  
2. Use upsert-only mode recommended for initial deployment  
3. domainFilters must be configured  
4. Stable txtOwnerId must be configured  
5. Do not use main account AccessKey  
6. Do not write real AccessKey into Git or documentation  
7. Do not share the same txtOwnerId across multi-cluster environments  
8. Validate with test subdomains first, then proceed to production domains