# 03-cert-manager Installation: Automatic TLS Certificate Issuance, Renewal, and Secret Management

Recommended Path:

    04-Kubernetes/08-Operations/02-Cluster Basic Components Installation/03-cert-manager Installation: Automatic TLS Certificate Issuance, Renewal, and Secret Management.md

Tags:

    #Kubernetes
    #cert-manager
    #TLS
    #HTTPS
    #Ingress
    #Secret
    #Certificate Management
    #Cluster Basic Components

---

## I. Document Description

This document records the installation, verification, and basic usage of cert-manager in a Kubernetes cluster.

cert-manager is used to automatically manage the lifecycle of TLS certificates in Kubernetes, including:

    1. Certificate application
    2. Certificate issuance
    3. Certificate renewal
    4. Creation of Secret for certificates
    5. Update of Secret for certificates
    6. Use with Ingress/Gateway API for HTTPS

Deployment Method in This Document:

    Installation method: Helm
    cert-manager version: v1.20.2
    Namespace: cert-manager
    CRD installation method: Helm parameter crds.enabled=true
    Testing method: SelfSigned ClusterIssuer + Certificate + Ingress TLS

Execution Node:

    k8s-master-01

Prerequisites:

    1. The Kubernetes cluster has been deployed.
    2. All nodes are in the Ready state.
    3. CoreDNS is functioning normally.
    4. ingress-nginx has been installed.
    5. Helm is installed.
    6. kubectl can access the cluster without issues.

---

## II. Problems Cert-manager Solves

Without cert-manager, configuring HTTPS for Ingress usually requires manually preparing:

    tls.crt
    tls.key

And then manually creating a Secret:

    kubectl create secret tls demo-tls \
      --cert=tls.crt \
      --key=tls.key \
      -n demo

Problems with this approach include:

    1. Certificates need to be applied for manually.
    2. Secrets must be created manually for certificates.
    3. Renewals require manual intervention when certificates are about to expire.
    4. Maintenance becomes complex when dealing with multiple services, domains, and namespaces.
    5. Expired certificates can cause HTTPS access issues.

With cert-manager, these tasks can be managed through YAML declarations:

    1. Who will issue the certificates.
    2. Which domain's certificates are required.
    3. Where the certificates should be stored in Secrets.
    4. Which TLS Secret Ingress should use.

cert-manager automatically handles certificate issuance and Secret management.

---

## III. Core Resource Objects

Common resource objects in cert-manager include:

| Resource | Function |
|---|---|
| Issuer | Namespace-level certificate issuer |
| ClusterIssuer | Cluster-level certificate issuer |
| Certificate | Declares the certificates to be requested |
| CertificateRequest | Actual certificate application request |
| Secret | Stores the final certificates and private keys |
| Order | ACME certificate order |
| Challenge | ACME domain validation task |

In this document, we will first use:

    ClusterIssuer
    Certificate
    Secret

for basic verification.

---

## IV. Pre-Installation Checks

### 4.1 Check Cluster Status

Execute:

    kubectl get nodes -o wide

Requirement:

    All nodes must be in the Ready state.

---

### 4.2 Check ingress-nginx

Execute:

    kubectl -n ingress-nginx get pods -o wide

    kubectl -n ingress-nginx get svc

    kubectl get ingressclass

Requirement:

    The ingress-nginx-controller Pod must be Running.
    The ingress-nginx-controller Service must be functioning properly.
    The IngressClass nginx must exist.

---

### 4.3 Check Helm

Execute:

    helm version

---

## V. Install cert-manager

This document recommends using the Helm OCI installation method.

Note:

    The cert-manager image is typically sourced from quay.io.
    If the current environment cannot access quay.io, you need to synchronize the image to an internal Harbor or modify the mirror repository configuration based on the output of helm show values.

---

### 5.1 Create Installation Directory

Execute:

    mkdir -p /root/k8s-yaml/cert-manager

    cd /root/k8s-yaml/cert-manager

---

### 5.2 View cert-manager Chart Information

View Chart details:

    helm show chart oci://quay.io/jetstack/charts/cert-manager --version v1.20.2

View default values:

    helm show values oci://quay.io/jetstack/charts/cert-manager --version v1.20.2 > values-cert-manager-default.yaml

View image-related configurations:

    grep -n "repository:" values-cert-manager-default.yaml

    grep -n "tag:" valuesSelfSigned certificates are self-signed and are not trusted by browsers by default. It is not recommended to use SelfSigned certificates for producing public HTTPS services. In production environments, enterprise CAs, internal CAs, ACME, Let’s Encrypt, or cloud provider certificate systems are typically used.

### Creating a ClusterIssuer:

```bash
cat <<EOF > clusterissuer-selfsigned.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-cluster-issuer
spec:
  selfSigned: {}
EOF
```

### Applying it:

```bash
kubectl apply -f cluster issuer-selfsigned.yaml
```

### Checking it:

```bash
kubectl get clusterIssuer
```

### Viewing details:

```bash
kubectl describe clusterIssuer selfsigned-cluster-issuer
```

---

## Section 8: Creating a Test Certificate

### 8.1 Creating a Test Namespace

If you have already created a `demo` namespace, you can skip this step.

```bash
kubectl create namespace demo
```

---

### 8.2 Creating a Certificate:

Create a certificate application:

```bash
cat <<EOF > certificate-demo-selfsigned.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: demo-self-signed-cert
  namespace: demo
spec:
  secretName: demo-selfsigned-tls
  duration: 2160h
  renewBefore: 360h
  commonName: demo.ops.local
  dnsNames:
    - demo ops.local
  issuerRef:
    name: selfsigned-cluster-issuer
    kind: ClusterIssuer
EOF
```

### Applying it:

```bash
kubectl apply -f certificate-demo-self-signed.yaml
```

---

### 8.3 Checking the Certificate Status:

```bash
kubectl -n demo get certificate
```

The expected status should be `READY` set to `True`.

### 8.4 Viewing the Automatically Generated Secret:

```bash
kubectl -n demo get secret demo-selfsigned-tls
```

To check the type of the Secret:

```bash
kubectl -n demo get secret demo-selfsigned-tls -o yaml | grep "type:"
```

The expected output should contain `type: kubernetes.io/tls`.

To view the data fields in the Secret:

```bash
kubectl -n demo get secret demo-selfsigned-tls -o jsonpath '{.data}' | head
```

The output should include `tls.crt` and `tls.key`.

**Note:** This Secret is automatically generated by cert-manager based on the created Certificate. Subsequent Ingress configurations can directly use this Secret to enable HTTPS.

---

## Section 9: Verifying HTTPS with Ingress

In this section, we will use the `demo` application from the previous Ingress-nginx example.

If you haven’t created a test application yet, you can first create one:

```bash
kubectl -n demo create deployment nginx-demo --image=nginx:1.25
kubectl -n demo expose deployment nginx-demo \
  --port=80 \
  --target-port=80 \
  --type=ClusterIP
```

### Checking the Setup:

```bash
kubectl -n demo get pods -o wide
kubectl -n demo get svc
kubectl -n demo get endpoints nginx-demo
```

The `endpoints` should not be empty.

---

### 9.1 Creating an HTTPS Ingress:

Create an Ingress configuration:

```bash
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
  - host: demo ops.local
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
```

### Applying it:

```bash
kubectl apply -f ingress-nginx-demo-https.yaml
```

### Checking it:

```bash
kubectl -n demo get ingress
kubectl -n demo describe ingress nginx-demo-https
```

Confirm that:

- `tls.secretName` is set to `demo-selfsigned-tls`.
- `ingressClassName` is `nginx`.
- The backend service is `nginx-demo:80`.

---

### 9.2 Verifying HTTPS Access:

Use `curl` to test the connection:

```bash
curl -k -H "Host: demo.ops.local" https://10.0.0.23:30443/
```

You can also test from other nodes:

```bash
curl -k -H "Host: demoOPS.local" https://10.0.3. Private Key Permission Control
4. Isolation of Different Business Namespaces
5. Prevention of Misuse of Certificates across Businesses    kubectl delete crd orders.acme.cert-manager.io

Delete the namespace:

    kubectl delete namespace cert-manager

---

## Section Fifteen: Post-Installation Verification Checklist

After installation, execute the following commands:

    helm list -n cert-manager

    kubectl -n cert-manager get pods -o wide

    kubectl get crd | grep cert-manager

    kubectl api-resources | grep cert-manager

    kubectl get clusterIssuer

    kubectl -n demo get certificate

    kubectl -n demo get secret demo-selfsigned-tls

    curl -k -H "Host: demo.ops.local" https://10.0.0.23:30443/

The following should be confirmed:

    1. The cert-manager Pod is running.
    2. The cert-manager-cainjector Pod is running.
    3. The cert-manager-webhook Pod is running.
    4. The cert-manager CRD has been successfully installed.
    5. The ClusterIssuer was created successfully.
    6. The Certificate is ready and available.
    7. The TLS Secret was generated automatically.
    8. The Ingress can reference the TLS Secret.
    9. HTTPS access tests have been successful.

---

## Section Sixteen: Summary

This document outlines the basic installation and verification process for cert-manager.

Key points include:

    1. Using Helm to install cert-manager.
    2. Installing the cert-manager CRD.
    3. Creating a SelfSigned ClusterIssuer.
    4. Generating a Certificate.
    5. Automatically generating a TLS Secret.
    6. Referring to the TLS Secret in Ingress settings.
    7. Verifying HTTPS access.
    8. Troubleshooting common issues with Pods, webhooks, Certificates, Secrets, and Ingresses related to TLS.

Production recommendations include:

    1. In private networks, consider using an enterprise CA or self-built CA.
    2. In public networks, integrate ACME / Let’s Encrypt for certificate management.
    3. In multi-service environments, clearly define the usage boundaries between Issuers and ClusterIssuers.
    4. Strictly control access to TLS Secrets.
    5. Incorporate certificate renewal and expiration alerts into monitoring systems.

Further suggestions include:

    04-Gateway API Introduction and Installation: Envoy Gateway, GatewayClass, Gateway, and HTTPRoute.md