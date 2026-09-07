<!--
SPDX-FileCopyrightText: Copyright (c) 2026, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# OpenShift (Validated Patterns)

Deploy AI-Q on OpenShift through the [Validated Patterns](https://validatedpatterns.io/learn/) GitOps framework. Scaffolding was generated with [patternizer](https://validatedpatterns.io/learn/creating-patterns-with-patternizer/) (`pattern init` without `--with-secrets`). This path is single-cluster only: no ACM hub/spoke and no HashiCorp Vault / External Secrets Operator.

The application Helm chart is unchanged. Pattern values point Argo CD at `deploy/helm/deployment-k8s` and apply `overrides/values-aiq-openshift.yaml` (MaaS Granite config mount, `gp3-csi` Postgres PVC, Ingress disabled). Use [Kubernetes (Helm)](./kubernetes.md) for a direct `helm install`.

## Prerequisites

- An OpenShift cluster and `oc` logged in with enough privilege to install operators.
- [Podman](https://podman.io/) (the `./pattern.sh` wrapper runs make targets in the Validated Patterns utility container).
- A Git remote Argo CD can clone (typically your fork) and this branch pushed.
- A Kubernetes Secret named `aiq-credentials` in the destination namespace **before** the AI-Q application syncs.

Default destination namespace in this repository is `aiq-itay`. Change `clusterGroup.namespaces` and each application's `namespace` in `values-prod.yaml` if you use a different project.

## Create the credentials secret

Do not commit secrets. Copy `deploy/.env.example` to `deploy/.env` (gitignored) and create the secret from those values:

```bash
oc new-project aiq-itay --skip-config-write || oc project aiq-itay

set -a && source deploy/.env && set +a
oc create secret generic aiq-credentials -n aiq-itay \
  --from-literal=DB_USER_NAME="${DB_USER_NAME}" \
  --from-literal=DB_USER_PASSWORD="${DB_USER_PASSWORD}" \
  --from-literal=OPENAI_API_KEY="${OPENAI_API_KEY}" \
  --from-literal=TAVILY_API_KEY="${TAVILY_API_KEY}" \
  --from-literal=AIQ_INFERENCE_BASE_URL="${AIQ_INFERENCE_BASE_URL}" \
  --from-literal=MAAS_MODEL_NAME="${MAAS_MODEL_NAME}" \
  --dry-run=client -o yaml | oc apply -f -
```

`NVIDIA_API_KEY` is not required for the MaaS Granite profile (`configs/config_maas_granite.yml`). NGC images on this overlay are public enough for many clusters; add an image-pull secret if your cluster cannot pull `nvcr.io/nvidia/blueprint/*`.

## Install

From the repository root, on the branch Argo CD should track:

```bash
./pattern.sh make validate-prereq
./pattern.sh make validate-cluster
./pattern.sh make show
./pattern.sh make install
./pattern.sh make argo-healthcheck
```

`make install` installs the Validated Patterns Operator, OpenShift GitOps, and a `Pattern` custom resource. Argo CD then syncs `aiq-maas-config` (workflow ConfigMap) and `aiq` (umbrella Helm chart). `global.secretLoader.disabled` is `true`; skip `make load-secrets`.

Re-running `podman run ... quay.io/validatedpatterns/patternizer init` is idempotent. After it runs, keep `values-prod.yaml` pointed at `deploy/helm/deployment-k8s` and namespace `aiq-itay` — patternizer auto-discovers the child chart under `deploy/helm/helm-charts-k8s/aiq`, which does not include the web-profile values.

## Validate

```bash
oc get pods,pvc -n aiq-itay
oc -n aiq-itay port-forward svc/aiq-backend 8000:8000
# in another terminal:
curl -sf http://127.0.0.1:8000/live && echo
curl -sf http://127.0.0.1:8000/health && echo
```

Frontend Ingress is disabled on OpenShift (the chart defaults to `ingressClassName: nginx`). Port-forward `svc/aiq-frontend` on port 3000 if you need the UI.

The NGC backend image does not contain `configs/config_maas_granite.yml`. GitOps mounts it from ConfigMap `aiq-maas-config`. Keep `charts/aiq-maas-config/files/config_maas_granite.yml` identical to `configs/config_maas_granite.yml`.
