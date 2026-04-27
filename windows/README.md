# Starburst Enterprise Platform — Kubernetes Deployment Capstone

A hands-on platform engineering capstone demonstrating the deployment, configuration, and troubleshooting of **Starburst Enterprise Platform (SEP)** on a local Kubernetes cluster. The project involves standing up a production-like SEP environment from scratch, diagnosing four distinct deployment failures, and resolving each through targeted infrastructure and configuration changes.

> **Note:** All license files, private keys, and internal registry credentials have been excluded from this repository. Registry references to `harbor.starburstdata.net` are standard Starburst Helm chart source syntax and contain no credentials.

---

## Architecture

```
kind (local K8s cluster)
│
├── SEP Coordinator Pod  (port-forwarded: 8443)
├── SEP Worker Pod 1
├── SEP Worker Pod 2
│   └── Secrets injected via Kubernetes secretRef (envFrom)
│
├── OpenLDAP (Docker Compose, TLS-secured)
│   └── Certificates: dynamically generated, mounted via ConfigMap
│
├── Insights Postgres DB (Docker Compose)
│
└── AWS Aurora (external, SSL/TLS connection)
```

**Tech Stack:** Kubernetes (kind) · Helm · Docker Compose · OpenLDAP · Python · Bash · AWS Aurora · Starburst Enterprise Platform 453.4.0

---

## Repository Structure

```
├── k8s/
│   ├── helm/
│   │   ├── values.yaml                   # Base Helm values
│   │   └── edited/
│   │       └── values.yaml               # Corrected values (fixes applied)
│   └── manifests/
│       ├── sep-secrets.yaml              # K8s Secret definitions
│       ├── sep-groups.yaml               # BIAC group configuration
│       ├── fetch-certs.yaml              # ConfigMap: dynamic cert generation script
│       └── edited/
│           └── fetch-certs.yaml          # Corrected fetch-certs manifest
│
├── capstone-services/
│   ├── docker-compose.yaml               # Insights Postgres + OpenLDAP services
│   └── ldap/                             # OpenLDAP configuration
│
├── notes.md                              # Step-by-step deployment guide
├── debug.yaml                            # Diagnostic manifest used during troubleshooting
├── rendered.yaml                         # Rendered Helm output for inspection
└── Solution Notes.docx                   # Client-facing write-up of all four issues and resolutions
```

---

## Deployment Overview

### Prerequisites

- Docker Desktop with Kubernetes enabled, or `kind` installed
- `kubectl` and `helm` CLI tools
- Starburst license file (not included — obtain from Starburst)
- Access to the Starburst Helm chart registry

### Quick Start

```bash
# 1. Create local K8s cluster
kind create cluster --name 'local'

# 2. Create registry credentials secret
kubectl create secret docker-registry registry-credentials \
    --docker-server=harbor.starburstdata.net \
    --docker-username=${USER} \
    --docker-password=${PASSWORD} \
    --docker-email=${EMAIL}

# 3. Apply K8s manifests (secrets, groups, cert ConfigMap)
kubectl apply -f k8s/manifests/

# 4. Create license secret
kubectl create secret generic starburst-license \
    --from-file=path/to/starburstdata.license

# 5. Start supporting services (Insights DB + OpenLDAP)
docker compose -f capstone-services/docker-compose.yaml up -d

# 6. Bridge Docker and Kubernetes networks
docker network connect capstone-services_default local-control-plane

# 7. Install SEP via Helm
helm upgrade sep oci://harbor.starburstdata.net/starburstdata/charts/starburst-enterprise \
    --install \
    --version 453.4.0 \
    --values k8s/helm/edited/values.yaml

# 8. Port-forward the coordinator
kubectl port-forward pod/<coordinator-pod-name> 8443:8443
```

> **Note:** On first access, the browser will warn about a self-signed certificate. Type `thisisunsafe` in Chrome to bypass.

---

## Issues Resolved

This capstone was structured around diagnosing and fixing four deployment failures. Full analysis and resolution steps are documented in `Solution Notes.docx`.

### Issue 1 — Resource Scheduling: Worker Pods Pending

**Symptom:** Worker pods stuck in `Pending` state; coordinator healthy.

**Root Cause:** Helm values specified resource requests exceeding available node capacity on the local `kind` cluster. The scheduler could not place both workers simultaneously.

**Resolution:** Adjusted `resources.requests` and `resources.limits` in `k8s/helm/edited/values.yaml` to fit within single-node constraints. Also tuned JVM heap settings (`-Xmx`, `-Xms`) to prevent out-of-memory conditions under the reduced resource envelope.

---

### Issue 2 — Worker Pod Secrets: Environment Variables Not Injected

**Symptom:** Worker pods started but catalog connections failed; environment variables missing at runtime.

**Root Cause:** The Helm values referenced a Kubernetes Secret via `envFrom.secretRef`, but the Secret name in `sep-secrets.yaml` did not match the reference in the values file. Kubernetes silently skipped the injection.

**Resolution:** Aligned the `secretRef.name` in `k8s/helm/edited/values.yaml` with the correct Secret name defined in `k8s/manifests/sep-secrets.yaml`. Verified injection with `kubectl exec -- env | grep <VAR>`.

---

### Issue 3 — LDAP → LDAPS: TLS Certificate Handshake Failure

**Symptom:** SEP could not authenticate users; LDAP connection errors in coordinator logs.

**Root Cause:** The deployment used plain LDAP (port 389). Switching to LDAPS (port 636) required TLS certificates that were not present in the OpenLDAP container at startup.

**Resolution:** Implemented a dynamic certificate generation approach using `fetch-certs.yaml` — a Kubernetes Job that generates a self-signed certificate at deploy time and mounts it into both the OpenLDAP container and the SEP coordinator as a ConfigMap. Updated Helm values to point SEP's LDAP configuration at `ldaps://` and the correct certificate path.

---

### Issue 4 — AWS Aurora SSL: DH Key Incompatibility

**Symptom:** SEP catalog connector failed to establish a TLS connection to AWS Aurora; handshake error in logs referencing DH key size.

**Root Cause:** Modern Java (17+) rejects DHE cipher suites with keys below 1024 bits. AWS Aurora's SSL configuration offered a short DH key that Java refused during the TLS handshake.

**Resolution:** Added JVM flags (`-Djdk.tls.ephemeralDHKeySize=2048` and disabling legacy DHE suites via `jdk.tls.disabledAlgorithms`) to the SEP coordinator configuration in the Helm values. Also validated that the correct Aurora SSL root certificate was referenced in the catalog connector properties.

---

## Key Skills Demonstrated

- Kubernetes pod scheduling, resource management, and secret injection patterns
- Helm chart customization and values file management
- TLS/SSL certificate lifecycle — generation, mounting, and Java truststore configuration
- LDAP/LDAPS authentication integration with enterprise platforms
- Systematic troubleshooting methodology: log analysis → root cause isolation → targeted fix → verification
- Platform engineering documentation for client-facing and technical audiences
