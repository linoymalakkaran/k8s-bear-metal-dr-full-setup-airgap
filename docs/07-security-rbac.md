# Security and RBAC Configuration Guide

## Table of Contents
1. [Security Architecture](#security-architecture)
2. [RBAC Configuration](#rbac-configuration)
3. [Pod Security Standards](#pod-security-standards)
4. [Network Policies](#network-policies)
5. [Secrets Management](#secrets-management)
6. [Certificate Management](#certificate-management)
7. [Security Scanning](#security-scanning)
8. [Audit Logging](#audit-logging)
9. [Compliance](#compliance)

## Security Architecture

### Defense in Depth Strategy

```yaml
security_layers:
  infrastructure:
    - "Physical security (datacenter access)"
    - "Network segmentation (VLANs, firewalls)"
    - "OS hardening (SELinux, AppArmor)"
    - "Encrypted storage (LUKS, dm-crypt)"
  
  cluster:
    - "RBAC (Role-Based Access Control)"
    - "Pod Security Standards"
    - "Network Policies"
    - "Admission Controllers"
    - "API server authentication/authorization"
  
  application:
    - "Container image scanning"
    - "Runtime security (Falco)"
    - "Secrets encryption"
    - "Service mesh (mTLS)"
  
  data:
    - "Encryption at rest"
    - "Encryption in transit"
    - "Backup encryption"
    - "Key rotation"
```

### Security Zones

```
┌──────────────────────────────────────────────────────────────┐
│                    SECURITY ZONES                            │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  DMZ Zone (VLAN 100)                                   │ │
│  │  - Ingress Controllers (public-facing)                 │ │
│  │  - WAF/API Gateway                                     │ │
│  │  - Load Balancers                                      │ │
│  │  Security: Restricted egress, monitored                │ │
│  └────────────────────────────────────────────────────────┘ │
│                           ▲                                  │
│                           │ Controlled traffic               │
│  ┌────────────────────────▼───────────────────────────────┐ │
│  │  Application Zone (VLAN 200)                           │ │
│  │  - Application Pods                                    │ │
│  │  - Microservices                                       │ │
│  │  - Service-to-service communication                    │ │
│  │  Security: Network policies, mTLS                      │ │
│  └────────────────────────────────────────────────────────┘ │
│                           ▲                                  │
│                           │ Database connections             │
│  ┌────────────────────────▼───────────────────────────────┐ │
│  │  Data Zone (VLAN 300)                                  │ │
│  │  - Databases (PostgreSQL, MongoDB)                     │ │
│  │  - Cache (Redis)                                       │ │
│  │  - Message Queues                                      │ │
│  │  Security: No external access, encrypted connections   │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Management Zone (VLAN 10)                             │ │
│  │  - Rancher UI                                          │ │
│  │  - Monitoring dashboards                               │ │
│  │  - Jump boxes                                          │ │
│  │  Security: MFA required, audit logged                  │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

## RBAC Configuration

### User Roles Hierarchy

```yaml
# File: kubernetes/rbac/roles-hierarchy.yaml

rbac_roles:
  cluster_admin:
    scope: cluster-wide
    permissions: all
    users:
      - k8s-admin@company.com
      - platform-team@company.com
  
  namespace_admin:
    scope: specific namespaces
    permissions:
      - Full control within namespace
      - Cannot create namespaces
    users:
      - app-team-lead@company.com
  
  developer:
    scope: development namespaces
    permissions:
      - Deploy applications
      - View logs and metrics
      - Cannot modify RBAC
      - Cannot access secrets directly
    users:
      - dev-team@company.com
  
  viewer:
    scope: cluster-wide (read-only)
    permissions:
      - View resources
      - View logs
      - No write access
    users:
      - support-team@company.com
      - auditors@company.com
```

### ClusterRole Examples

```yaml
# File: kubernetes/rbac/clusterroles.yaml

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-admin-role
  labels:
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
- nonResourceURLs: ["*"]
  verbs: ["*"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: developer-role
rules:
# Pods
- apiGroups: [""]
  resources: ["pods", "pods/log", "pods/exec"]
  verbs: ["get", "list", "watch", "create", "delete"]

# Deployments
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets", "statefulsets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

# Services
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

# ConfigMaps (limited)
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch", "create", "update"]

# NO access to secrets
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]  # Read-only

# Ingress
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

# HPA
- apiGroups: ["autoscaling"]
  resources: ["horizontalpodautoscalers"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: viewer-role
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: []  # No access to secrets

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cicd-deployer-role
rules:
# For CI/CD pipelines - deployment only
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "patch", "update"]

- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]

- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "create", "update"]
```

### RoleBindings

```yaml
# File: kubernetes/rbac/rolebindings.yaml

---
# Bind cluster admin to platform team
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: platform-team-cluster-admin
subjects:
- kind: Group
  name: platform-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io

---
# Bind developers to production namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developers-production
  namespace: production
subjects:
- kind: Group
  name: dev-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: developer-role
  apiGroup: rbac.authorization.k8s.io

---
# Service account for CI/CD
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cicd-deployer
  namespace: production
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cicd-deployer-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: cicd-deployer
  namespace: production
roleRef:
  kind: ClusterRole
  name: cicd-deployer-role
  apiGroup: rbac.authorization.k8s.io
```

### RBAC Setup Script

```bash
#!/bin/bash
# File: scripts/security/setup-rbac.sh

# Setup RBAC for production cluster

set -e

export KUBECONFIG=/etc/rancher/rke2/rke2.yaml

echo "=== Setting up RBAC ==="

# Create ClusterRoles
kubectl apply -f kubernetes/rbac/clusterroles.yaml

# Create namespaces
kubectl create namespace production --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace staging --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace development --dry-run=client -o yaml | kubectl apply -f -

# Create RoleBindings
kubectl apply -f kubernetes/rbac/rolebindings.yaml

# Create service accounts for CI/CD
for ns in production staging development; do
    kubectl create serviceaccount cicd-deployer -n $ns --dry-run=client -o yaml | kubectl apply -f -
done

echo "✓ RBAC configuration complete"

# Display current roles
echo ""
echo "ClusterRoles:"
kubectl get clusterroles | grep -E "cluster-admin-role|developer-role|viewer-role"

echo ""
echo "Service Accounts:"
kubectl get sa -A | grep cicd-deployer
```

## Pod Security Standards

### Pod Security Policy (PSP) Replacement - Pod Security Admission

```yaml
# File: kubernetes/security/pod-security-standards.yaml

# Since PSP is deprecated in K8s 1.25+, use Pod Security Admission

# Label namespaces with security standards
---
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted

---
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted

---
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/audit: baseline
    pod-security.kubernetes.io/warn: baseline
```

### Kyverno Policies (Alternative to PSP)

```yaml
# File: kubernetes/security/kyverno-policies.yaml

---
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-non-root-user
spec:
  validationFailureAction: enforce
  background: true
  rules:
  - name: check-runAsNonRoot
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "Running as root is not allowed"
      pattern:
        spec:
          securityContext:
            runAsNonRoot: true

---
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-read-only-root-filesystem
spec:
  validationFailureAction: enforce
  rules:
  - name: check-readOnlyRootFilesystem
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "Root filesystem must be read-only"
      pattern:
        spec:
          containers:
          - securityContext:
              readOnlyRootFilesystem: true

---
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-privileged-containers
spec:
  validationFailureAction: enforce
  rules:
  - name: check-privileged
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "Privileged containers are not allowed"
      pattern:
        spec:
          containers:
          - securityContext:
              privileged: false

---
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-limits
spec:
  validationFailureAction: enforce
  rules:
  - name: check-limits
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "Resource limits are required"
      pattern:
        spec:
          containers:
          - resources:
              limits:
                memory: "?*"
                cpu: "?*"
```

### Install Kyverno

```bash
#!/bin/bash
# File: scripts/security/install-kyverno.sh

# Install Kyverno policy engine

set -e

export KUBECONFIG=/etc/rancher/rke2/rke2.yaml

echo "=== Installing Kyverno ==="

# Add Kyverno Helm repository
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update

# Install Kyverno
helm install kyverno kyverno/kyverno \
  --namespace kyverno \
  --create-namespace \
  --set replicaCount=3 \
  --set resources.limits.memory=512Mi \
  --set resources.requests.memory=256Mi

# Wait for Kyverno to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=kyverno -n kyverno --timeout=300s

echo "✓ Kyverno installed"

# Apply policies
kubectl apply -f kubernetes/security/kyverno-policies.yaml

echo "✓ Security policies applied"
```

## Network Policies

### Default Deny All Policy

```yaml
# File: kubernetes/network-policies/default-deny.yaml

---
# Deny all ingress traffic by default
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress

---
# Deny all egress traffic by default
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
```

### Allow Specific Traffic

```yaml
# File: kubernetes/network-policies/allow-policies.yaml

---
# Allow frontend to backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080

---
# Allow backend to database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-database
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 5432

---
# Allow DNS queries (required for service discovery)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-access
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53

---
# Allow ingress controller to applications
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-to-apps
  namespace: production
spec:
  podSelector:
    matchLabels:
      expose: "true"
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080
```

## Secrets Management

### Sealed Secrets (Encrypted Secrets in Git)

```bash
#!/bin/bash
# File: scripts/security/install-sealed-secrets.sh

# Install Sealed Secrets controller

set -e

export KUBECONFIG=/etc/rancher/rke2/rke2.yaml

echo "=== Installing Sealed Secrets ==="

# Install Sealed Secrets controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# Wait for controller
kubectl wait --for=condition=ready pod -l name=sealed-secrets-controller -n kube-system --timeout=300s

# Install kubeseal CLI
KUBESEAL_VERSION='0.24.0'
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v${KUBESEAL_VERSION}/kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz
tar -xvzf kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal

echo "✓ Sealed Secrets installed"

# Fetch the public key
kubeseal --fetch-cert > sealed-secrets-cert.pem

echo "Public certificate saved to sealed-secrets-cert.pem"
```

### Create and Use Sealed Secrets

```bash
#!/bin/bash
# File: scripts/security/create-sealed-secret.sh

# Create a sealed secret from regular secret

SECRET_NAME="database-credentials"
NAMESPACE="production"

# Create regular secret (NOT stored in Git)
kubectl create secret generic ${SECRET_NAME} \
  --from-literal=username=dbuser \
  --from-literal=password='SuperSecretPassword123!' \
  --namespace=${NAMESPACE} \
  --dry-run=client -o yaml > temp-secret.yaml

# Encrypt it with kubeseal
kubeseal --format=yaml --cert=sealed-secrets-cert.pem \
  < temp-secret.yaml > kubernetes/secrets/sealed-${SECRET_NAME}.yaml

# Remove temporary file
rm temp-secret.yaml

echo "✓ Sealed secret created: kubernetes/secrets/sealed-${SECRET_NAME}.yaml"
echo "This file CAN be committed to Git"

# Apply sealed secret
kubectl apply -f kubernetes/secrets/sealed-${SECRET_NAME}.yaml
```

### External Secrets Operator (Vault Integration)

```yaml
# File: kubernetes/security/external-secrets.yaml

# For integrating with HashiCorp Vault or AWS Secrets Manager
---
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: production
spec:
  provider:
    vault:
      server: "https://vault.company.internal:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "production-apps"

---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: database-credentials
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: database-credentials
    creationPolicy: Owner
  data:
  - secretKey: username
    remoteRef:
      key: database/prod/credentials
      property: username
  - secretKey: password
    remoteRef:
      key: database/prod/credentials
      property: password
```

## Certificate Management

### cert-manager Configuration

```yaml
# File: kubernetes/security/cert-manager-issuers.yaml

---
# Let's Encrypt Production Issuer
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@company.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx

---
# Let's Encrypt Staging Issuer (for testing)
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: admin@company.com
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - http01:
        ingress:
          class: nginx

---
# Internal CA Issuer (for internal services)
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: internal-ca
spec:
  ca:
    secretName: internal-ca-key-pair
```

### Auto-Renewing Certificates

```yaml
# File: kubernetes/security/certificate-example.yaml

---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: rancher-tls
  namespace: cattle-system
spec:
  secretName: tls-rancher-ingress
  duration: 2160h # 90 days
  renewBefore: 360h # 15 days before expiration
  commonName: rancher.k8s.internal
  dnsNames:
  - rancher.k8s.internal
  - rancher.company.com
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
```

## Security Scanning

### Trivy Installation

```bash
#!/bin/bash
# File: scripts/security/install-trivy.sh

# Install Trivy for vulnerability scanning

set -e

echo "=== Installing Trivy ==="

# Install Trivy CLI
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt update
sudo apt install trivy -y

# Scan example
echo "✓ Trivy installed"
echo "Scan images with: trivy image <image-name>"
```

### Scan Images in CI/CD

```bash
#!/bin/bash
# File: scripts/security/scan-image.sh

# Scan container image for vulnerabilities

IMAGE=$1

if [ -z "$IMAGE" ]; then
    echo "Usage: $0 <image-name>"
    exit 1
fi

echo "Scanning $IMAGE for vulnerabilities..."

# Scan and fail on HIGH and CRITICAL
trivy image \
  --severity HIGH,CRITICAL \
  --exit-code 1 \
  --no-progress \
  $IMAGE

if [ $? -eq 0 ]; then
    echo "✓ No HIGH or CRITICAL vulnerabilities found"
else
    echo "✗ Vulnerabilities detected! Build failed."
    exit 1
fi
```

## Audit Logging

### Enable Audit Policy

```yaml
# File: kubernetes/security/audit-policy.yaml

apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Log all requests at RequestResponse level
  - level: RequestResponse
    verbs: ["create", "update", "patch", "delete"]
    resources:
      - group: ""
        resources: ["secrets", "configmaps"]
  
  # Log authentication events
  - level: Metadata
    verbs: ["get", "list", "watch"]
    resources:
      - group: "authentication.k8s.io"
  
  # Log RBAC changes
  - level: RequestResponse
    verbs: ["create", "update", "patch", "delete"]
    resources:
      - group: "rbac.authorization.k8s.io"
  
  # Don't log read-only requests
  - level: None
    verbs: ["get", "list", "watch"]
```

### RKE2 Audit Configuration

```yaml
# File: /etc/rancher/rke2/config.yaml
# Add these lines to enable audit logging

audit-policy-file: /etc/rancher/rke2/audit-policy.yaml
audit-log-path: /var/lib/rancher/rke2/server/logs/audit.log
audit-log-maxage: 30
audit-log-maxbackup: 10
audit-log-maxsize: 100
```

## Compliance

### CIS Benchmark Scanning

```bash
#!/bin/bash
# File: scripts/security/run-cis-benchmark.sh

# Run CIS Kubernetes Benchmark scan

set -e

export KUBECONFIG=/etc/rancher/rke2/rke2.yaml

echo "=== Running CIS Kubernetes Benchmark ==="

# Install kube-bench
wget https://github.com/aquasecurity/kube-bench/releases/download/v0.7.0/kube-bench_0.7.0_linux_amd64.tar.gz
tar -xvf kube-bench_0.7.0_linux_amd64.tar.gz

# Run benchmark (RKE2 uses different paths)
sudo ./kube-bench run \
  --benchmark rke2-cis-1.23 \
  --json > cis-benchmark-report.json

# Generate summary
echo ""
echo "Benchmark Results:"
jq '.Totals' cis-benchmark-report.json

echo ""
echo "Full report saved to: cis-benchmark-report.json"
```

## Security Checklist

```yaml
security_checklist:
  rbac:
    - [ ] ClusterRoles defined and applied
    - [ ] RoleBindings configured
    - [ ] Service accounts created for apps
    - [ ] No pods running as root
    - [ ] Least privilege principle followed
  
  network:
    - [ ] Default deny network policies in place
    - [ ] Allow policies defined for required traffic
    - [ ] Network segmentation implemented
    - [ ] Ingress traffic restricted
  
  secrets:
    - [ ] Sealed Secrets or External Secrets configured
    - [ ] No plaintext secrets in Git
    - [ ] Secret rotation policy defined
    - [ ] Vault or similar secrets manager integrated
  
  containers:
    - [ ] All images scanned for vulnerabilities
    - [ ] Images from trusted registries only
    - [ ] Image pull secrets configured
    - [ ] Security contexts defined
  
  certificates:
    - [ ] cert-manager installed
    - [ ] TLS for all ingress endpoints
    - [ ] Certificate auto-renewal configured
    - [ ] Internal CA for internal services
  
  monitoring:
    - [ ] Audit logging enabled
    - [ ] Security events monitored
    - [ ] Falco runtime security installed (optional)
    - [ ] Alerts configured for security events
  
  compliance:
    - [ ] CIS benchmark scanned regularly
    - [ ] Pod Security Standards enforced
    - [ ] Compliance reports generated
    - [ ] Security policies documented
```

## Next Steps

After implementing security:
1. ➡️ Regular security audits and scans
2. 🔄 Periodic RBAC reviews
3. 📊 Monitor security metrics
4. 🔐 Implement key rotation
5. 📝 Document security procedures
