# 03-cert-manager Installation: Automatic TLS Certificate Issuance, Renewal, and Secret Management

Recommended path:

    04-Kubernetes/08-Operations/02-Cluster Base Component Installation/03-cert-manager Installation: Automatic TLS Certificate Issuance, Renewal, and Secret Management.md

Tags:

    #Kubernetes
    #cert-manager
    #TLS
    #HTTPS
    #Ingress
    #Secret
    #CertificateManagement
    #ClusterBasicComponents

---

## I. Document Description

This document records the installation, verification, and basic usage methods of cert-manager in a Kubernetes cluster.

cert-manager is used to automatically manage the TLS certificate lifecycle in Kubernetes, including:

    1. Certificate application
    2. Certificate issuance
    3. Certificate renewal
    4. Certificate Secret creation
    5. Certificate Secret update
    6. Integration with Ingress / Gateway API for HTTPS

Deployment method in this document:

    Installation method: Helm
    cert-manager version: v1.20.2
    Namespace: cert-manager
    CRD installation method: Helm parameter crds.enabled=true
    Testing method: SelfSigned ClusterIssuer + Certificate + Ingress TLS

Execution node:

    k8s-master-01

Prerequisites:

    1. Kubernetes cluster has been deployed
    2. Node status is Ready
    3. CoreDNS is functioning normally
    4. ingress-nginx has been installed
    5. Helm has been installed
    6. kubectl can access the cluster normally

---

## II. What Problem Does cert-manager Solve

Without cert-manager, configuring HTTPS for Ingress typically requires manually preparing:

    tls.crt
    tls.key

Then manually creating a Secret:

    kubectl create secret tls demo-tls \
      --cert=tls.crt \
      --key=tls.key \
      -n demo

Problems with this approach:

    1. Certificates need manual application
    2. Certificates need manual Secret creation
    3. Renewal is required manually when certificates are about to expire
    4. Maintenance becomes complex with multiple businesses, domains, and namespaces
    5. Expired certificates can cause HTTPS access anomalies

After using cert-manager, you can declare via YAML:

    1. Who will issue the certificate
    2. Which domain's certificate to apply for
    3. Where to save the certificate
    4. Which TLS Secret to use for Ingress

cert-manager will automatically complete certificate issuance and Secret management.

---

## III. Core Resource Objects

Common resource objects in cert-manager:

| Resource | Function |
|---|---|
| Issuer | Namespace-level certificate issuer |
| ClusterIssuer | Cluster-level certificate issuer |
| Certificate | Declaration of requested certificate |
| CertificateRequest | Actual certificate application request |
| Secret | Stores the final certificate and private key |
| Order | ACME certificate order |
| Challenge | ACME domain verification task |

This document will first use:

    ClusterIssuer
    Certificate
    Secret

For basic verification.

---

## IV. Pre-Installation Checks

### 4.1 Check Cluster Status

Execute:

    kubectl get nodes -o wide

Requirements:

    All nodes are Ready.

---

### 4.2 Check ingress-nginx

Execute:

    kubectl -n ingress-nginx get pods -o wide

    kubectl -n ingress-nginx get svc

    kubectl get ingressclass

Requirements:

    ingress-nginx-controller Pod is Running
    ingress-nginx-controller Service is normal
    IngressClass nginx exists

---

### 4.3 Check Helm

Execute:

    helm version

---

## V. Install cert-manager

This document prioritizes installation via Helm OCI.

Note:

    cert-manager images default from quay.io.
    If the current environment cannot access quay.io, the images need to be synchronized to internal Harbor in advance, or modify the image repository configuration according to helm show values output.

---

### 5.1 Create Installation Directory

Execute:

    mkdir -p /root/k8s-yaml/cert-manager

    cd /root/k8s-yaml/cert-manager

---

### 5.2 View cert-manager Chart

View chart information:

    helm show chart oci://quay.io/jetstack/charts/cert-manager --version v1.20.2

View default values:

    helm show values oci://quay.io/jetstack/charts/cert-manager --version v1.20.2 > values-cert-manager-default.yaml

View image-related configuration:

    grep -n "repository:" values-cert-manager-default.yaml

    grep -n "tag:" values-cert-manager-default.yaml

---

### 5.3 Install cert-manager via Helm

Execute:

    helm install cert-manager oci://quay.io/jetstack/charts/cert-manager \
      --version v1.20.2 \
      --namespace cert-manager \
      --create-namespace \
      --set crds.enabled=true

Note:

---
namespace cert-manager
    Install cert-manager into the cert-manager namespace.

--create-namespace
    Automatically create the namespace if it does not exist.

--set crds.enabled=true
    Install the CRD required for cert-manager.

---

### 5.4 Alternative Method: Installing with Jetstack Helm Repository

If the OCI method is unavailable, you can use the traditional Helm repository method.

Add the repository:

    helm repo add jetstack https://charts.jetstack.io --force-update

Update the repository:

    helm repo update

Install:

    helm install cert-manager jetstack/cert-manager \
      --namespace cert-manager \
      --create-namespace \
      --version v1.20.2 \
      --set crds.enabled=true

---

## Six. Checking cert-manager Status

### 6.1 View Helm Release

Execute:

    helm list -n cert-manager

Expected output:

    cert-manager

---

### 6.2 View Pod

Execute:

    kubectl -n cert-manager get pods -o wide

Expected output:

    cert-manager
    cert-manager-cainjector
    cert-manager-webhook

Status should be:

    Running

---

### 6.3 View Service

Execute:

    kubectl -n cert-manager get svc

Common Services:

    cert-manager
    cert-manager-webhook

---

### 6.4 View CRD

Execute:

    kubectl get crd | grep cert-manager

Should see similar resources:

    certificaterequests.cert-manager.io
    certificates.cert-manager.io
    challenges.acme.cert-manager.io
    clusterissuers.cert-manager.io
    issuers.cert-manager.io
    orders.acme.cert-manager.io

---

### 6.5 View API Resources

Execute:

    kubectl api-resources | grep cert-manager

Confirm you can see:

    certificates
    certificaterequests
    issuers
    clusterissuers

---

## Seven. Creating a SelfSigned ClusterIssuer

SelfSigned is suitable for basic functionality verification of cert-manager.

Note:

    SelfSigned certificates are self-signed, and browsers will not trust them by default.
    It is not recommended to use SelfSigned certificates for public HTTPS in production.
    In production environments, enterprise CA, internal CA, ACME, Let’s Encrypt, or cloud provider certificate systems are typically used.

Create ClusterIssuer:

    cat <<EOF > clusterissuer-selfsigned.yaml
    apiVersion: cert-manager.io/v1
    kind: ClusterIssuer
    metadata:
      name: selfsigned-cluster-issuer
    spec:
      selfSigned: {}
    EOF

Apply:

    kubectl apply -f clusterissuer-selfsigned.yaml

Check:

    kubectl get clusterissuer

View details:

    kubectl describe clusterissuer selfsigned-cluster-issuer

---

## Eight. Creating a Test Certificate

### 8.1 Create a Test Namespace

Skip this if the demo namespace has already been created.

Execute:

    kubectl create namespace demo

---

### 8.2 Create Certificate

Create certificate request:

    cat <<EOF > certificate-demo-selfsigned.yaml
    apiVersion: cert-manager.io/v1
    kind: Certificate
    metadata:
      name: demo-selfsigned-cert
      namespace: demo
    spec:
      secretName: demo-selfsigned-tls
      duration: 2160h
      renewBefore: 360h
      commonName: demo.ops.local
      dnsNames:
      - demo.ops.local
      issuerRef:
        name: selfsigned-cluster-issuer
        kind: ClusterIssuer
    EOF

Apply:

    kubectl apply -f certificate-demo-selfsigned.yaml

---

### 8.3 View Certificate Status

Execute:

    kubectl -n demo get certificate

Expected:

    READY is True

View details:

    kubectl -n demo describe certificate demo-selfsigned-cert

---

### 8.4 View Automatically Generated Secret

Execute:

    kubectl -n demo get secret demo-selfsigned-tls

Check Secret type:

    kubectl -n demo get secret demo-selfsigned-tls -o yaml | grep "type:"

Expected:

    type: kubernetes.io/tls

Check data fields in the Secret:

kubectl -n demo get secret demo-selfsigned-tls -o jsonpath='{.data}' | head

Normal output should include:

    tls.crt
    tls.key

Explanation:

    This Secret is automatically generated by cert-manager based on the Certificate.
    Subsequent Ingress can directly reference this Secret to implement HTTPS.

---

## IX. Verifying HTTPS with Ingress

This section uses the demo application from the previous ingress-nginx tutorial.

If you haven't tested the application yet, you can first create it:

    kubectl -n demo create deployment nginx-demo --image=nginx:1.25

    kubectl -n demo expose deployment nginx-demo \
      --port=80 \
      --target-port=80 \
      --type=ClusterIP

Check:

    kubectl -n demo get pods -o wide

    kubectl -n demo get svc

    kubectl -n demo get endpoints nginx-demo

Requirements:

    endpoints must not be empty.

---

### 9.1 Creating HTTPS Ingress

Create Ingress:

    cat <<EOF > ingress-nginx-demo-https.yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: nginx-demo-https
      namespace: demo
    spec:
      ingressClassName: nginx
      tls:
      - hosts:
        - demo.ops.local
        secretName: demo-selfsigned-tls
      rules:
      - host: demo.ops.local
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx-demo
                port:
                  number: 80
    EOF

Apply:

    kubectl apply -f ingress-nginx-demo-https.yaml

Check:

    kubectl -n demo get ingress

    kubectl -n demo describe ingress nginx-demo-https

Confirm:

    tls.secretName is demo-selfsigned-tls
    ingressClassName is nginx
    backend service is nginx-demo:80

---

### 9.2 Verifying HTTPS Access

Test with curl:

    curl -k -H "Host: demo.ops.local" https://10.0.0.23:30443/

You can also test other nodes:

    curl -k -H "Host: demo.ops.local" https://10.0.0.24:30443/

    curl -k -H "Host: demo.ops.local" https://10.0.0.25:30443/

If it returns the nginx default page, it indicates:

    1. The ingress-nginx HTTPS entry is working
    2. The Ingress TLS configuration is correct
    3. The TLS Secret created by cert-manager can be used by Ingress

Note:

    -k indicates ignoring certificate trust verification.
    Since this uses a SelfSigned certificate, clients won't trust it by default.

---

### 9.3 Viewing Certificate Contents

View the certificate in the Secret:

    kubectl -n demo get secret demo-selfsigned-tls \
      -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -text | head -n 30

Check certificate validity period:

    kubectl -n demo get secret demo-selfsigned-tls \
      -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -dates

Check certificate SAN:

    kubectl -n demo get secret demo-selfsigned-tls \
      -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -text | grep -A2 "Subject Alternative Name"

---

## X. Difference Between Issuer and ClusterIssuer

| Object | Scope | Use Cases |
|---|---|---|
| Issuer | Single namespace | Used for a specific business namespace |
| ClusterIssuer | Entire cluster | Used for multiple namespaces uniformly |

Recommendations:

    1. Test environments can use ClusterIssuer for simplified management
    2. Multi-tenant environments can use Issuer for namespace isolation
    3. Production environments should design certificate issuance scope based on permission models

---

## XI. Production Certificate Management Strategy

### 11.1 Internal Systems

Common solutions for internal systems:

    1. Enterprise internal CA
    2. Self-built CA
    3. Vault
    4. Cloud vendor private certificate service
    5. Manually issued certificates + cert-manager managing Secret

Key focuses for internal systems:

    1. Trustworthy certificate chain
    2. Automatic certificate renewal
    3. Private key access control
    4. Isolation between different business namespaces
    5. Certificate Secret not misused across business areas

---

### 11.2 Public Systems

Common solutions for public systems:

    1. ACME
    2. Let’s Encrypt
    3. DNS-01 Challenge
    4. HTTP-01 Challenge

Note:

    HTTP-01 typically requires public access to the Ingress's 80 port.
    DNS-01 usually requires DNS service provider API permissions.
    Internal network environments, private environments, and environments without public domain names are not suitable for directly applying public ACME processes.

Recommendations:

    ACME / Let’s Encrypt related content should be organized separately.
    This article only completes the basic installation of cert-manager and self-signed certificate verification.

---

## Twelve. Common Issues Troubleshooting

### 12.1 cert-manager Pod ImagePullBackOff

Check:

    kubectl -n cert-manager get pods -o wide

Check events:

    kubectl -n cert-manager describe pod <pod-name>

Common causes:

    1. Unable to access quay.io
    2. Unable to access the image repository
    3. The image is not synchronized to internal Harbor
    4. containerd fails to pull the image

Handling approach:

    1. Check the default values in the chart
    2. Find the image repository configuration
    3. Synchronize the image to internal Harbor
    4. Use helm upgrade to modify the image address

Check default values:

    helm show values oci://quay.io/jetstack/charts/cert-manager --version v1.20.2 > values-cert-manager-default.yaml

Check image fields:

    grep -n "repository:" values-cert-manager-default.yaml

---

### 12.2 webhook startup failure

Check Pod:

    kubectl -n cert-manager get pods -o wide

Check webhook logs:

    kubectl -n cert-manager logs deploy/cert-manager-webhook

Check Service:

    kubectl -n cert-manager get svc cert-manager-webhook

Check Endpoint:

    kubectl -n cert-manager get endpoints cert-manager-webhook

Common causes:

    1. webhook Pod has not started
    2. webhook Service has no endpoints
    3. Network policy restricts access
    4. Image pull failure
    5. API Server cannot access webhook

---

### 12.3 Certificate remains Pending

Check Certificate:

    kubectl -n demo describe certificate demo-selfsigned-cert

Check CertificateRequest:

    kubectl -n demo get certificaterequest

    kubectl -n demo describe certificaterequest <name>

Check Issuer:

    kubectl get clusterissuer

    kubectl describe clusterissuer selfsigned-cluster-issuer

Common causes:

    1. issuerRef is incorrect
    2. ClusterIssuer does not exist
    3. cert-manager controller is abnormal
    4. webhook is abnormal
    5. CertificateRequest has not been approved or issuance failed

---

### 12.4 Secret not generated

Check Certificate:

    kubectl -n demo get certificate

    kubectl -n demo describe certificate demo-selfsigned-cert

Check Secret:

    kubectl -n demo get secret demo-selfsigned-tls

Common causes:

    1. Certificate is not Ready
    2. issuerRef is incorrect
    3. cert-manager Pod is abnormal
    4. Secret name is incorrect
    5. Certificate issuance failed

---

### 12.5 Ingress HTTPS access failure

Check Ingress:

    kubectl -n demo describe ingress nginx-demo-https

Check TLS Secret:

    kubectl -n demo get secret demo-selfsigned-tls

Check ingress-nginx logs:

    kubectl -n ingress-nginx logs -l app.kubernetes.io/component=controller --tail=100

Check access method:

    curl -k -H "Host: demo.ops.local" https://10.0.0.23:30443/

Common causes:

    1. Ingress has no TLS configuration
    2. secretName is incorrect
    3. TLS Secret does not exist
    4. TLS Secret type is not kubernetes.io/tls
    5. IngressClass mismatch
    6. Host does not match during access
    7. Using self-signed certificate but curl lacks -k

---

## Thirteen. Upgrade and Rollback

### 13.1 Check current version

Check Helm release:

    helm list -n cert-manager

Check status:

    helm status cert-manager -n cert-manager

Check image:

    kubectl -n cert-manager get deploy -o yaml | grep image:

---

### 13.2 Upgrade cert-manager

Before upgrading, it is recommended to back up current values:

    helm get values cert-manager -n cert-manager -o yaml > cert-manager-values-backup.yaml

Upgrade example: /think

helm upgrade cert-manager oci://quay.io/jetstack/charts/cert-manager \
  --version v1.20.2 \
  --namespace cert-manager \
  --set crds.enabled=true

Viewing historical versions:

  helm history cert-manager -n cert-manager

---

### 13.3 Rollback cert-manager

Viewing historical versions:

  helm history cert-manager -n cert-manager

Rollback:

  helm rollback cert-manager <REVISION> -n cert-manager

Viewing:

  helm status cert-manager -n cert-manager

---

## Fourteen, Uninstall cert-manager

Note:

  Before uninstalling cert-manager, confirm whether there are any business certificate resources still in use.
  Do not directly delete CRD, as this may cause Certificate, Issuer, and other resources to be lost.

Viewing cert-manager related resources:

  kubectl get issuers,clusterissuers,certificates,certificaterequests,orders,challenges -A

Deleting test resources:

  kubectl delete -f ingress-nginx-demo-https.yaml

  kubectl delete -f certificate-demo-selfsigned.yaml

  kubectl delete -f clusterissuer-selfsigned.yaml

Uninstalling Helm release:

  helm uninstall cert-manager -n cert-manager

Viewing CRD:

  kubectl get crd | grep cert-manager

If confirmed that cert-manager is no longer in use, delete CRD:

  kubectl delete crd certificaterequests.cert-manager.io
  kubectl delete crd certificates.cert-manager.io
  kubectl delete crd challenges.acme.cert-manager.io
  kubectl delete crd clusterissuers.cert-manager.io
  kubectl delete crd issuers.cert-manager.io
  kubectl delete crd orders.acme.cert-manager.io

Deleting namespace:

  kubectl delete namespace cert-manager

---

## Fifteen, Installation Completion Checklist

After installation, execute:

  helm list -n cert-manager

  kubectl -n cert-manager get pods -o wide

  kubectl get crd | grep cert-manager

  kubectl api-resources | grep cert-manager

  kubectl get clusterissuer

  kubectl -n demo get certificate

  kubectl -n demo get secret demo-selfsigned-tls

  curl -k -H "Host: demo.ops.local" https://10.0.0.23:30443/

Should meet:

  1. cert-manager Pod Running
  2. cert-manager-cainjector Pod Running
  3. cert-manager-webhook Pod Running
  4. cert-manager CRD is installed
  5. ClusterIssuer created successfully
  6. Certificate Ready=True
  7. TLS Secret automatically generated
  8. Ingress can reference TLS Secret
  9. HTTPS access test successful

---

## Sixteen, Summary

This document completes the basic installation and verification of cert-manager.

Core content:

  1. Install cert-manager using Helm
  2. Install cert-manager CRD
  3. Create SelfSigned ClusterIssuer
  4. Create Certificate
  5. Automatically generate TLS Secret
  6. Ingress references TLS Secret
  7. Verify using HTTPS access
  8. Troubleshoot common issues with Pod, webhook, Certificate, Secret, and Ingress TLS

Production recommendations:

  1. In internal environments, combine with enterprise CA or self-built CA
  2. In public environments, combine with ACME / Let’s Encrypt
  3. In multi-business environments, recommend defining boundaries for Issuer / ClusterIssuer usage
  4. TLS Secret permissions need strict control
  5. Certificate renewal and expiration alerts should be included in monitoring systems

Subsequent recommendations for further organization:

  04-Gateway API Introduction Installation: Envoy Gateway, GatewayClass, Gateway and HTTPRoute.md /think