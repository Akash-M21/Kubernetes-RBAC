# Kubernetes RBAC Setup for User `akash`

A complete hands-on guide to Kubernetes RBAC (Role-Based Access Control) using certificate-based authentication.

This project demonstrates:

- User Authentication using Certificates
- Kubernetes Authorization using RBAC
- Roles and RoleBindings
- ClusterRole and ClusterRoleBindings
- kubeconfig Context Management
- Namespace-based Access Control
- Real-world RBAC concepts used in production

---

# Table of Contents

- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [RBAC Architecture Flow](#rbac-authentication--authorization-flow)
- [Authentication vs Authorization](#authentication-vs-authorization)
- [Step 1: Generate Private Key](#step-1-generate-private-key)
- [Step 2: Generate CSR](#step-2-generate-csr-certificate-signing-request)
- [Step 3: Sign the Certificate](#step-3-sign-the-certificate-using-kubernetes-ca)
- [Step 4: Configure User Credentials](#step-4-configure-user-credentials-in-kubeconfig)
- [Step 5: Create Namespace](#step-5-create-namespace)
- [Step 6: Create Role](#step-6-create-role)
- [Step 7: Create RoleBinding](#step-7-create-rolebinding)
- [Step 8: Verify Permissions](#step-8-verify-rbac-permissions)
- [Step 9: Configure Cluster Information](#step-9-configure-cluster-information)
- [Step 10: Create Context](#step-10-create-context)
- [Step 11: Switch Context](#step-11-switch-context)
- [Step 12: Verify User Identity](#step-12-verify-user-identity)
- [Step 13: Test User Access](#step-13-test-user-access)
- [Step 14: View Contexts](#step-14-view-available-contexts)
- [Step 15: Switch Back to Admin Context](#step-15-switch-back-to-admin-context)
- [Step 16: Grant Cluster Admin Access](#step-16-grant-cluster-admin-access-optional)
- [Role vs ClusterRole](#role-vs-clusterrole)
- [User vs ServiceAccount](#user-vs-serviceaccount)
- [Real-World RBAC Usage](#real-world-rbac-usage)
- [Verification Commands](#verification-commands)
- [Common Errors & Fixes](#common-errors--fixes)
- [RBAC Best Practices](#rbac-best-practices)
- [Cleanup Resources](#cleanup-resources)
- [Learning Outcomes](#learning-outcomes)

---

# Prerequisites

Before starting, ensure you have:

- Kubernetes Cluster Installed
- Access to Control Plane Node
- `kubectl` configured with admin access
- OpenSSL installed
- Root or sudo access

---

# Project Structure

```text
.
├── akash.crt
├── akash.csr
├── akash.key
├── role-akash.yaml
├── akash-role-binding.yaml
└── README.md
```

---

# RBAC Authentication & Authorization Flow

```text
+-------------------+
|   User: akash     |
+-------------------+
          |
          | Certificate Authentication
          v
+-------------------+
| Kubernetes API    |
| Server            |
+-------------------+
          |
          | RBAC Authorization
          v
+-------------------+
| RoleBinding       |
| read-pods         |
+-------------------+
          |
          v
+-------------------+
| Role: pod-reader  |
+-------------------+
          |
          v
+-------------------+
| Allowed Actions   |
| get/list/watch    |
| pods              |
+-------------------+
```

---

# Authentication vs Authorization

## Authentication

Authentication verifies:

> "Who are you?"

Authentication methods:

- Certificates
- Tokens
- OIDC
- Service Accounts

Example:

```text
CN=akash
```

---

## Authorization

Authorization verifies:

> "What are you allowed to do?"

Authorization methods:

- RBAC
- ABAC
- Webhooks

This project uses:

```text
RBAC (Role-Based Access Control)
```

---

# Step 1: Generate Private Key

Generate a private key for user `akash`.

```bash
openssl genrsa -out akash.key 2048
```

Verify:

```bash
ls
```

Expected:

```text
akash.key
```

---

# Step 2: Generate CSR (Certificate Signing Request)

Create CSR for Kubernetes authentication.

```bash
openssl req -new \
-key akash.key \
-out akash.csr \
-subj "/CN=akash/O=devops"
```

## Explanation

| Field | Meaning |
|---|---|
| CN=akash | Kubernetes Username |
| O=devops | Kubernetes Group |

Verify:

```bash
ls
```

Expected:

```text
akash.csr
akash.key
```

---

# Step 3: Sign the Certificate Using Kubernetes CA

Sign the CSR using Kubernetes CA.

```bash
openssl x509 -req \
-in akash.csr \
-CA /etc/kubernetes/pki/ca.crt \
-CAkey /etc/kubernetes/pki/ca.key \
-CAcreateserial \
-out akash.crt \
-days 365
```

Verify:

```bash
ls
```

Generated files:

```text
akash.crt
akash.csr
akash.key
```

---

# Step 4: Configure User Credentials in Kubeconfig

Add user credentials.

```bash
kubectl config set-credentials akash \
--client-certificate=akash.crt \
--client-key=akash.key
```

---

# Step 5: Create Namespace

Create namespace:

```bash
kubectl create namespace devops
```

Verify:

```bash
kubectl get ns
```

---

# Step 6: Create Role

Create a Role allowing pod read access.

Create file:

```bash
cat > role-akash.yaml << EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role

metadata:
  namespace: devops
  name: pod-reader

rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
EOF
```

Apply Role:

```bash
kubectl apply -f role-akash.yaml
```

Verify:

```bash
kubectl get role -n devops
```

---

# Step 7: Create RoleBinding

Bind Role to user `akash`.

Create file:

```bash
cat > akash-role-binding.yaml << EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding

metadata:
  name: read-pods
  namespace: devops

subjects:
- kind: User
  name: akash
  apiGroup: rbac.authorization.k8s.io

roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
EOF
```

Apply:

```bash
kubectl apply -f akash-role-binding.yaml
```

Verify:

```bash
kubectl get rolebinding -n devops
```

---

# Step 8: Verify RBAC Permissions

## Before RoleBinding

```bash
kubectl auth can-i get pods --as akash -n devops
```

Expected:

```text
no
```

---

## After RoleBinding

```bash
kubectl auth can-i get pods --as akash -n devops
```

Expected:

```text
yes
```

---

## Cluster-Wide Access Check

```bash
kubectl auth can-i get pods --as akash
```

Expected:

```text
no
```

Because Roles are namespace scoped.

---

# Step 9: Configure Cluster Information

Add cluster details.

```bash
kubectl config set-cluster kubernetes \
--server=https://172.30.1.2:6443 \
--certificate-authority=/etc/kubernetes/pki/ca.crt
```

---

# Step 10: Create Context

Create context for user `akash`.

```bash
kubectl config set-context akash-context \
--cluster=kubernetes \
--user=akash \
--namespace=devops
```

---

# Step 11: Switch Context

Switch to user context.

```bash
kubectl config use-context akash-context
```

Verify:

```bash
kubectl config current-context
```

---

# Step 12: Verify User Identity

Check authenticated user.

```bash
kubectl auth whoami
```

Expected:

```text
ATTRIBUTE                                           VALUE
Username                                            akash
Groups                                              [devops system:authenticated]
```

---

# Step 13: Test User Access

List pods:

```bash
kubectl get pods
```

Check all permissions:

```bash
kubectl auth can-i --list
```

---

# Step 14: View Available Contexts

```bash
kubectl config get-contexts
```

Example:

```text
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
*         akash-context                 kubernetes   akash              devops
          kubernetes-admin@kubernetes   kubernetes   kubernetes-admin
```

---

# Step 15: Switch Back to Admin Context

```bash
kubectl config use-context kubernetes-admin@kubernetes
```

---

# Step 16: Grant Cluster Admin Access (Optional)

> WARNING:
>
> This provides full cluster access.

Create ClusterRoleBinding:

```bash
kubectl create clusterrolebinding akash-admin \
--clusterrole=cluster-admin \
--user=akash
```

Switch back to user:

```bash
kubectl config use-context akash-context
```

Now user can access:

```bash
kubectl get nodes
kubectl get pods -A
```

---

# Role vs ClusterRole

| Feature | Role | ClusterRole |
|---|---|---|
| Scope | Namespace | Entire Cluster |
| Used For | Namespace Resources | Cluster Resources |
| Binding Type | RoleBinding | ClusterRoleBinding |
| Example | Pods in dev namespace | Nodes, PVs |

---

# User vs ServiceAccount

| Feature | User | ServiceAccount |
|---|---|---|
| Used By | Humans | Applications |
| Authentication | Certificates/OIDC | Tokens |
| Namespace Scoped | No | Yes |
| Common Usage | Developers/Admins | Pods/CI-CD |

---

# Real-World RBAC Usage

## Developers

- Access only development namespace
- Cannot access production

---

## QA Team

- Read-only access in QA namespace

---

## DevOps Team

- Full cluster access

---

## Security Team

- Audit and read-only access

---

## CI/CD Pipelines

- Uses ServiceAccounts
- Limited deployment permissions

---

# Verification Commands

## Check Current User

```bash
kubectl auth whoami
```

---

## Check Current Context

```bash
kubectl config current-context
```

---

## Check All Permissions

```bash
kubectl auth can-i --list
```

---

## Check Specific Permission

```bash
kubectl auth can-i create pods --as akash
```

---

## List Roles

```bash
kubectl get roles -A
```

---

## List RoleBindings

```bash
kubectl get rolebindings -A
```

---

## List ClusterRoles

```bash
kubectl get clusterroles
```

---

## List ClusterRoleBindings

```bash
kubectl get clusterrolebindings
```

---

## View Certificate Details

```bash
openssl x509 -in akash.crt -text -noout
```

---

# Common Errors & Fixes

## Error

```text
User "akash" cannot list resource "pods"
```

### Cause

RoleBinding missing.

### Fix

```bash
kubectl apply -f akash-role-binding.yaml
```

---

## Error

```text
certificate signed by unknown authority
```

### Cause

Wrong CA certificate.

### Fix

Verify:

```bash
/etc/kubernetes/pki/ca.crt
```

---

## Error

```text
context not found
```

### Fix

Recreate context:

```bash
kubectl config set-context akash-context \
--cluster=kubernetes \
--user=akash
```

---

## Error

```text
Forbidden
```

### Cause

RBAC permissions not granted.

### Fix

Verify:

```bash
kubectl auth can-i --list
```

---

# RBAC Best Practices

- Follow Principle of Least Privilege
- Avoid unnecessary cluster-admin access
- Use namespaces for isolation
- Prefer Roles over ClusterRoles
- Use ServiceAccounts for applications
- Audit RBAC permissions regularly
- Avoid wildcard (`*`) permissions in production
- Separate dev, qa, and prod access

---

# Cleanup Resources

## Delete Role

```bash
kubectl delete role pod-reader -n devops
```

---

## Delete RoleBinding

```bash
kubectl delete rolebinding read-pods -n devops
```

---

## Delete ClusterRoleBinding

```bash
kubectl delete clusterrolebinding akash-admin
```

---

## Delete Namespace

```bash
kubectl delete namespace devops
```

---

# Learning Outcomes

After completing this project, you will understand:

- Kubernetes Authentication
- Kubernetes Authorization
- RBAC Architecture
- Roles & RoleBindings
- ClusterRoles & ClusterRoleBindings
- kubeconfig Contexts
- Certificate-based Authentication
- Namespace-based Access Control
- Real-world Kubernetes Security Practices

---

# Summary

This project demonstrated how to:

- Create Kubernetes users using certificates
- Authenticate users using kubeconfig
- Implement RBAC using Roles and RoleBindings
- Restrict access to namespaces
- Grant cluster-wide access using ClusterRoleBindings
- Verify permissions using `kubectl auth can-i`

RBAC is one of the most critical Kubernetes security concepts and is heavily used in production-grade Kubernetes environments.
