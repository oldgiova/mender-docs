---
title: Mender Server
taxonomy:
    category: docs
    label: tutorial
---

! Please note the code snippets in this section reuses the environment variables
! you set up when progressing through the tutorial, including the optional step
! of installing Minio. Please make sure you correctly define them or adapt the
! snippet to your specific use case.

## Prerequisites

### Optional: external services
The Mender Helm chart is packaged with required external services:
* [MongoDB](https://bitnami.com/stack/mongodb/helm)
* [NATS](https://nats-io.github.io/k8s/)

Using these packages is fine for test or PoC setups.
For production setups, however, it's recommended to use external dedicated services.

### Device authentication keys
The Mender Server deployment requires generating keys that are used for user and
device authentication. The following snippet uses `openssl` to generate the
required keys:

[ui-tabs position="top-left" active="0" theme="default" ]
[ui-tab title="Open Source"]
```bash
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:3072 | openssl rsa -out device_auth.key
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:3072 | openssl rsa -out useradm.key
```
[/ui-tab]
[ui-tab title="Enterprise"]
```bash
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:3072 | openssl rsa -out device_auth.key
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:3072 | openssl rsa -out useradm.key
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:3072 | openssl rsa -out tenantadm.key
```
[/ui-tab]
[/ui-tabs]


## Installing the Mender Helm chart

Before installing the Mender Server on the Kubernetes cluster using the
[Mender Helm chart](https://github.com/mendersoftware/mender-helm), add the
Mender Helm Chart repository:

```bash
helm repo add mender https://charts.mender.io
helm repo update
```

You can now install the Mender Server. You also have to decide how to expose
the Mender API Gateway service. This Helm Chart includes a configurable Ingress resource
for your convenience. Some sample Ingress values are detailed on the later section.

[ui-tabs position="top-left" active="0" theme="default" ]
[ui-tab title="Open Source"]
<!--AUTOVERSION: "export MENDER_VERSION_TAG=\"mender-%\""/ignore -->
<!--AUTOVERSION: "cat >mender-%.yml <<EOF"/integration "helm upgrade --install mender mender/mender -f mender-%.yml"/integration -->
```bash
export MENDER_SERVER_DOMAIN="mender.example.com"
export MENDER_SERVER_URL="https://${MENDER_SERVER_DOMAIN}"
export MENDER_VERSION_TAG="mender-3.4"
export MONGODB_ROOT_PASSWORD=$(pwgen 32 1)
export MONGODB_REPLICA_SET_KEY=$(pwgen 32 1)

cat >mender-master.yml <<EOF
global:
  enterprise: false
  image:
    tag: ${MENDER_VERSION_TAG}
  mongodb:
    URL: ""
  nats:
    URL: ""
  s3:
    AWS_URI: "https://${MINIO_DOMAIN_NAME}"
    AWS_BUCKET: "mender-artifact-storage"
    AWS_ACCESS_KEY_ID: "${MINIO_ACCESS_KEY}"
    AWS_SECRET_ACCESS_KEY: "${MINIO_SECRET_KEY}"
  url: "${MENDER_SERVER_URL}"

# This enables bitnami/mongodb sub-chart
mongodb:
  enabled: true
  auth:
    enabled: true
    rootPassword: ${MONGODB_ROOT_PASSWORD}
    replicaSetKey: ${MONGODB_REPLICA_SET_KEY}

# This enabled nats sub-chart
nats:
  enabled: true

api_gateway:
  env:
    SSL: false

device_auth:
  certs:
    key: |-
$(cat device_auth.key | sed -e 's/^/      /g')

useradm:
  certs:
    key: |-
$(cat useradm.key | sed -e 's/^/      /g')
EOF

helm upgrade --install mender mender/mender -f mender-master.yml
```
[/ui-tab]
[ui-tab title="Enterprise"]


!!!!! The following deployment requires access to the Mender Enterprise
!!!!! Container Registry. Please email [contact@mender.io](mailto:contact@mender.io) to
!!!!! receive an evaluation account.

<!--AUTOVERSION: "export MENDER_VERSION_TAG=\"mender-%\""/ignore -->
<!--AUTOVERSION: "cat >mender-%.yml <<EOF"/integration "helm upgrade --install mender mender/mender -f mender-%.yml"/integration -->
```bash
export MENDER_REGISTRY_USERNAME="replace-with-your-username"
export MENDER_REGISTRY_PASSWORD="replace-with-your-password"
export MENDER_SERVER_DOMAIN="mender.example.com"
export MENDER_SERVER_URL="https://${MENDER_SERVER_DOMAIN}"
export MENDER_VERSION_TAG="mender-3.4"
export MONGODB_ROOT_PASSWORD=$(pwgen 32 1)
export MONGODB_REPLICA_SET_KEY=$(pwgen 32 1)

cat >mender-master.yml <<EOF
global:
  enterprise: true
  image:
    username: "${MENDER_REGISTRY_USERNAME}"
    password: "${MENDER_REGISTRY_PASSWORD}"
    tag: ${MENDER_VERSION_TAG}
  mongodb:
    URL: ""
  nats:
    URL: ""
  s3:
    AWS_URI: "https://${MINIO_DOMAIN_NAME}"
    AWS_BUCKET: "mender-artifact-storage"
    AWS_ACCESS_KEY_ID: "${MINIO_ACCESS_KEY}"
    AWS_SECRET_ACCESS_KEY: "${MINIO_SECRET_KEY}"
  url: "${MENDER_SERVER_URL}"

# This enables bitnami/mongodb sub-chart
mongodb:
  enabled: true
  auth:
    enabled: true
    rootPassword: ${MONGODB_ROOT_PASSWORD}
    replicaSetKey: ${MONGODB_REPLICA_SET_KEY}

# This enabled nats sub-chart
nats:
  enabled: true

api_gateway:
  env:
    SSL: false

device_auth:
  certs:
    key: |-
$(cat device_auth.key | sed -e 's/^/      /g')

tenantadm:
  certs:
    key: |-
$(cat tenantadm.key | sed -e 's/^/      /g')

useradm:
  certs:
    key: |-
$(cat useradm.key | sed -e 's/^/      /g')
EOF

helm upgrade --install mender mender/mender -f mender-master.yml
```
[/ui-tab]
[/ui-tabs]

### Supported artifact storage types

Mender supports the following Artifact storage types:
* AWS S3 (or AWS S3 API-compatible storage layers, e.g., MinIO)
* Azure Blob Storage

[ui-tabs position="top-left" active="0" theme="lite" ]
[ui-tab title="AWS S3"]
<!--AUTOMATION: ignore -->
> To store the Mender Artifacts in an *AWS S3 bucket* instead of relying on a Minio service, update
> the examples above as follows:
>
> ```yaml
> global:
>   s3:
>     AWS_URI: "https://s3.<your-aws-region>.amazonaws.com"
>     AWS_BUCKET: "<name-of-your-bucket>"
>     AWS_REGION: "<your-aws-region>"
>     AWS_ACCESS_KEY_ID: "<your-access-key-id>"
>     AWS_SECRET_ACCESS_KEY: "<your-secret-access-key>"
>     AWS_FORCE_PATH_STYLE: "false"
> ```
[/ui-tab]
[ui-tab title="Azure Blob Storage"]

> To store the Mender Artifacts in an *Azure Blob Storage* container using a *connection string*, update the values file as follows:
>
> ```yaml
> global:
>   storage: "azure"
>   azure:
>     AUTH_CONNECTION_STRING: "BlobEndpoint=https://<name-of-your-storage>.blob.core.windows.net;SharedAccessSignature=..."
>     CONTAINER_NAME: "<name-of-your-container>"
> ```

Instead of a connection string, you can also specify the following parameters:
>
> ```yaml
> global:
>   storage: "azure"
>   azure:
>     AUTH_SHARED_KEY_ACCOUNT_NAME: "<account-name>"
>     AUTH_SHARED_KEY_ACCOUNT_KEY: "<account-key>"
>     CONTAINER_NAME: "<name-of-your-container>"
> ```
[/ui-tab]
[/ui-tabs]

### Exposing the service

You must configure
an [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) or a
[Load Balancer](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer)
to expose the Mender Server outside the Kubernetes cluster.

You can refer to your Infrastructure Provider's documentation for how to create an Ingress resource.
Starting from the Mender Helm Chart version 5.1.0, a sample Ingress resource is included. 

For example, here's an Ingress for the AWS EKS Provider:

<!--AUTOVERSION: "cat <<-EOF >> mender-%.yml <<EOF"/ignore -->
```bash
cat <<-EOF >> mender-3.6.0-build3.yml
ingress:
  enabled: true
  annotations:
    cert-manager.io/issuer: "letsencrypt"
    alb.ingress.kubernetes.io/healthcheck-path: /ui/
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/actions.ssl-redirect: '{"Type": "redirect", "RedirectConfig":{ "Protocol": "HTTPS", "Port": "443", "StatusCode": "HTTP_301"}}'
    alb.ingress.kubernetes.io/backend-protocol: HTTP
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS":443}]'
    alb.ingress.kubernetes.io/load-balancer-attributes: routing.http2.enabled=true,idle_timeout.timeout_seconds=600
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS-1-2-2017-01
    alb.ingress.kubernetes.io/target-type: ip
  path: /
  extraPaths:
    - path: /
      backend:
        serviceName: ssl-redirect
        servicePort: use-annotation

  hosts:
    - ${MENDER_SERVER_DOMAIN}
  tls:
  # this secret must exists or it can be created from a working cert-manager instance
   - secretName: mender-ingress-tls
     hosts:
       - ${MENDER_SERVER_DOMAIN}
EOF
```

For example, here's an Ingress for Azure AKS Provider:

<!--AUTOVERSION: "cat <<-EOF >> mender-%.yml <<EOF"/ignore -->
```bash
cat <<-EOF >> mender-3.6.0-build3.yml
ingress:
  enabled: true
  annotations:
    appgw.ingress.kubernetes.io/backend-protocol: http
    appgw.ingress.kubernetes.io/health-probe-path: /ui/
    appgw.ingress.kubernetes.io/request-timeout: "600"
    appgw.ingress.kubernetes.io/ssl-redirect: "true"
    kubernetes.io/ingress.class: azure/application-gateway
  path: /
  hosts:
    - ${MENDER_SERVER_DOMAIN}
  tls:
  # this secret must exists or it can be created from a working cert-manager instance
   - secretName: mender-ingress-tls
     hosts:
       - ${MENDER_SERVER_DOMAIN}
EOF
```

You can now update the Helm Chart with the included ingress:
<!--AUTOVERSION: "helm upgrade --install mender mender/mender -f mender-%.yml"/ignore -->
```bash
helm upgrade --install mender mender/mender -f mender-3.6.0-build3.yml
```

Alternatively, you can create your own Ingress resource. Some references:
* [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
* [AWS EKS Ingress](https://docs.aws.amazon.com/eks/latest/userguide/alb-ingress.html)
* [Azure Ingress](https://learn.microsoft.com/en-us/azure/aks/ingress-basic?tabs=azure-cli)

## Create the admin user

[ui-tabs position="top-left" active="0" theme="default" ]
[ui-tab title="Open Source"]
Create the initial user using the `useradm` pod:
```bash
USERADM_POD=$(kubectl get pod -l 'app.kubernetes.io/name=useradm' -o name | head -1)
kubectl exec $USERADM_POD -- useradm create-user --username "demo@mender.io" --password "demodemo"
```
[/ui-tab]
[ui-tab title="Enterprise"]

Create the administrator user using the `tenantadm` pod:
```bash
TENANTADM_POD=$(kubectl get pod -l 'app.kubernetes.io/name=tenantadm' -o name | head -1)
TENANT_ID=$(kubectl exec $TENANTADM_POD -- tenantadm create-org --name demo --username "admin@mender.io" --password "adminadmin" --plan enterprise)
```

You can create additional users from the command line of the `useradm` pod:

```bash
USERADM_POD=$(kubectl get pod -l 'app.kubernetes.io/name=useradm' -o name | head -1)
kubectl exec $USERADM_POD -- useradm-enterprise create-user --username "demo@mender.io" --password "demodemo" --tenant-id $TENANT_ID
```
[/ui-tab]
[/ui-tabs]

## Pre-release version

<!--AUTOVERSION: "pre-release (%)"/ignore-->
To use a pre-release (master) version of the backend, please refer to [the Development section of
the
documentation](https://docs.mender.io/development/server-installation/production-installation-with-kubernetes/mender-server).
