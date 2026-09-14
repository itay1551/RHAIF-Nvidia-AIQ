<!--
SPDX-FileCopyrightText: Copyright (c) 2025-2026, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# NVIDIA AI-Q on OpenShift (Validated Pattern)

## Introduction

[NVIDIA AI-Q](https://github.com/NVIDIA-AI-Blueprints/aiq) is an enterprise research agent built on the
[NVIDIA NeMo Agent Toolkit](https://docs.nvidia.com/nemo/agent-toolkit/latest/) and
[LangChain Deep Agents](https://docs.langchain.com/oss/python/deepagents/overview). It provides
**quick, cited answers** and **in-depth, report-style research**. Teams self-host the application
boundary and connect their own models, data sources, authentication, storage, and observability.
AI-Q is a governed research workflow, not a general-purpose coding-agent harness.

This repository deploys AI-Q on [Red Hat OpenShift](https://www.redhat.com/en/technologies/cloud-computing/openshift)
through the [Validated Patterns](https://validatedpatterns.io/) GitOps framework. `./pattern.sh make install`
installs the Validated Patterns Operator and OpenShift GitOps, then Argo CD syncs the AI-Q Helm chart
into the `aiq` namespace.

The path is **single-cluster only**: no ACM hub/spoke and no HashiCorp Vault / External Secrets Operator.
Secrets use the Validated Patterns `none` backend, which writes Kubernetes Secret `aiq-credentials` from
a local file. The default overlay targets an OpenAI-compatible
[OpenShift AI Model as a Service (MaaS)](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/)
Granite endpoint; no cluster GPU nodes are required for that profile.

CLI setup, MCP, Jupyter notebooks, benchmarks, and local development are documented in the
[upstream NVIDIA AI-Q repository](https://github.com/NVIDIA-AI-Blueprints/aiq).

## Architecture

AI-Q uses a LangGraph-based state machine: an orchestration node classifies intent and research depth,
a shallow researcher answers quickly with citations, and a deep researcher plans, gathers sources, and
writes citation-backed reports.

<p align="center">
<img src="./docs/assets/AIQ-arch-light.png" alt="AI-Q Architecture" width="800">
</p>

_Figure 1. NVIDIA AI-Q research-agent architecture._

On OpenShift, Argo CD applies the unchanged application chart at `deploy/helm/deployment-k8s` plus
`overrides/values-aiq-openshift.yaml` (MaaS Granite config mount, Postgres PVC on the cluster default
StorageClass, Ingress disabled). See
[OpenShift (Validated Patterns)](docs/source/deployment/validated-patterns.md) for the GitOps overlay.

## Key Features

- **Orchestration** — One node classifies intent (meta vs. research) and research depth (shallow vs. deep).
- **Shallow research** — Bounded, faster researcher with tool-calling and source citation.
- **Structured deep research** — Advisory source routing, structured planning, concurrent researchers, and a dedicated writer produce citation-backed reports.
- **Report follow-up** — Ask questions about a completed report, rewrite it, or run delta research with the parent report as context.
- **Workflow configuration** — YAML configs define agents, tools, LLMs, and routing without code changes.
- **Data sources** — Web, paper, enterprise, collaboration, and knowledge-layer sources can be selected per request.
- **Production API and web UI** — REST endpoints, async jobs, and a Next.js frontend for interactive research.
- **GitOps on OpenShift** — Validated Patterns Operator, OpenShift GitOps, and Argo CD deploy and reconcile the stack.

## Components deployed

Argo CD syncs two applications into namespace `aiq`:

- **aiq-backend** — NVIDIA AI-Q agent (`nvcr.io/nvidia/blueprint/aiq-agent`). Serves the research API on port 8000. The MaaS Granite workflow YAML is mounted from ConfigMap `aiq-maas-config`.
- **aiq-frontend** — Web UI (`nvcr.io/nvidia/blueprint/aiq-frontend`) on port 3000.
- **aiq-postgres** — In-cluster PostgreSQL with a PVC for jobs and checkpoints.
- **aiq-maas-config** — ConfigMap sourced from `charts/aiq-maas-config/files/config_maas_granite.yml`. The NGC backend image does not contain this overlay file.

`make install` also installs the Validated Patterns Operator, OpenShift GitOps, and a `Pattern` custom resource. Secret `aiq-credentials` is loaded into `aiq` from `~/values-secret-aiq.yaml`.

## Prerequisites

- [Podman](https://podman.io/) (the `./pattern.sh` wrapper runs make targets in the Validated Patterns utility container).
- An OpenShift cluster and `oc` logged in with enough privilege to install operators.
- Git, and a remote Argo CD can clone (this repository or your fork) with the branch pushed.
- A reachable OpenAI-compatible LLM endpoint (`AIQ_INFERENCE_BASE_URL`) and model name (`MAAS_MODEL_NAME`). Tavily is optional for web research.

Default destination namespace is `aiq`.

## Deploying

### Login to OpenShift

```bash
oc login --token=<token> --server=<api_server_url>
```

### Clone the repository

```bash
git clone https://github.com/itay1551/RHAIF-Nvidia-AIQ
cd RHAIF-Nvidia-AIQ
```

Use the branch Argo CD should track (typically `develop`).

### Configure secrets

Do not commit secrets. Copy the template out of Git and fill in real values:

```bash
cp values-secret.yaml.template ~/values-secret-aiq.yaml
# edit ~/values-secret-aiq.yaml
# set DB_USER_NAME, DB_USER_PASSWORD, OPENAI_API_KEY,
# AIQ_INFERENCE_BASE_URL, MAAS_MODEL_NAME; TAVILY_API_KEY may be empty
```

`./pattern.sh make install` looks for `~/values-secret-aiq.yaml` before falling back to the in-repo template.
`NVIDIA_API_KEY` is not required for the MaaS Granite profile.

### Install

From the repository root:

```bash
./pattern.sh make validate-prereq
./pattern.sh make validate-cluster
./pattern.sh make install
./pattern.sh make argo-healthcheck
```

`make install` installs the Validated Patterns Operator and OpenShift GitOps (`vp-gitops`), creates the `Pattern` resource, and
loads `aiq-credentials`. `global.singleArgoCD: true` keeps clustergroup Applications in that Argo CD instance. Argo CD then syncs `aiq-maas-config` and `aiq`.

## Verify the installation

```bash
oc get pods,pvc -n aiq
```

Expect backend, frontend, and postgres pods `Running`, and a Postgres PVC bound.

Port-forward the backend and check health:

```bash
oc -n aiq port-forward svc/aiq-backend 8000:8000
# in another terminal:
curl -sf http://127.0.0.1:8000/live && echo
curl -sf http://127.0.0.1:8000/health && echo
```

## Launch the UI

Frontend Ingress is disabled on this overlay (`ingressClassName: nginx` is not used on OpenShift).
If you enable an OpenShift Route for `svc/aiq-frontend`, open that URL. Otherwise port-forward:

```bash
oc -n aiq port-forward svc/aiq-frontend 3000:3000
```

Then open http://127.0.0.1:3000.

## Further documentation

- [OpenShift (Validated Patterns)](docs/source/deployment/validated-patterns.md) — secrets, overlay, Argo CD, and troubleshooting for this deployment.
- [NVIDIA AI-Q Blueprint](https://github.com/NVIDIA-AI-Blueprints/aiq) — upstream product docs, CLI, MCP, Jupyter, benchmarks, and development.
- [Validated Patterns](https://validatedpatterns.io/learn/) — GitOps framework used by this repository.

## License

This project is licensed under the [Apache License, Version 2.0](https://www.apache.org/licenses/LICENSE-2.0).
See [LICENSE](LICENSE). Additional third-party terms are listed in [LICENSE-THIRD-PARTY](LICENSE-THIRD-PARTY).
