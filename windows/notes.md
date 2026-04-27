# Deployment Notes

## Overview

This deployment is intended for local development. It includes the following
resources/components:

- SEP cluster with coordinator and two worker pods
- LDAP authentication
- Insights and local Insights database (Postgres)
- Local OpenLDAP server secured with TLS
- BIAC is configured

## Deployment Instructions

Read the notes below to make use of this environment. Be sure to follow the
steps in order.

Preliminary commands:

```bash
# Spin up local K8s cluster
kind create cluster --name 'local'

create-registry-secret # // OR:

kubectl create secret docker-registry registry-credentials \
    --docker-server=harbor.starburstdata.net \
    --docker-username=${USER} \
    --docker-password=${PASSWORD} \
    --docker-email=${EMAIL}

kubectl apply -f k8s/manifests/
kubectl create secret generic starburst-license \
    --from-file="${HOME}"/Work/license/starburstdata.license

# Provisions local Insights DB and links Docker/K8s networks
docker compose -f capstone-services/docker-compose.yaml up -d

docker network connect \
    capstone-services_default local-control-plane

# Install SEP chart
helm upgrade sep oci://harbor.starburstdata.net/starburstdata/charts/starburst-enterprise \
    --install \
    --version 453.4.0 \
    --values k8s/helm/values.yaml

# Forward coordinator port
kubectl port-forward pod/<coordinator> 8443:8443

# NOTE: must type 'thisisunsafe' in browser due to self-signed cert
```
