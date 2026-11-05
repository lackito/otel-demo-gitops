# AWS GitOps Operations Guide

Detailed setup, operating procedures, and implementation notes.
Return to the [project overview](../README.md).

## otel-demo-gitops-aws

## Overview

`otel-demo-gitops-aws` is the GitOps repository for the OpenTelemetry Demo Kubernetes deployment running on Amazon EKS.

This repository defines the **desired state** of Kubernetes applications.

Instead of Terraform directly managing application workloads, Argo CD continuously monitors this repository and synchronizes changes into the Kubernetes cluster.

## Environment boundary

This repository is AWS-only:

- the Argo CD instance running in Amazon EKS watches this repository;
- `applications/otel-demo/values.yaml` contains AWS and ECR configuration;
- `argocd/applications/otel-demo.yaml` keeps the declarative AWS Application
  definition with the GitOps desired state;
- Terraform in `otel-demo-infra-aws/terraform/04-applications` registers that
  Application with the cluster;
- no local kind cluster watches this repository.

The standalone local environment keeps its desired state in the separate
`otel-demo-gitops-kind` repository. Local values and Gateway resources should
not be added here.

---

## Architecture

```
┌─────────────────────────────┐
│ Developer                   │
│                             │
│ Application / Configuration │
└──────────────┬──────────────┘
               |
               v
┌─────────────────────────────┐
│ GitHub Repository           │
│                             │
│ otel-demo-gitops-aws              │
│                             │
│ Argo CD manifests           │
│ Helm values                 │
└──────────────┬──────────────┘
               |
               v
┌─────────────────────────────┐
│ Argo CD                    │
│                             │
│ Detects desired state       │
│ Performs synchronization    │
└──────────────┬──────────────┘
               |
               v
┌─────────────────────────────┐
│ Amazon EKS                  │
│                             │
│ OpenTelemetry Demo          │
│ Kubernetes Resources        │
└─────────────────────────────┘
```

---

## Repository Structure

```
otel-demo-gitops-aws/
├── argocd/
│   └── applications/
│       └── otel-demo.yaml
│
├── applications/
│   └── otel-demo/
│       └── values.yaml
│
└── README.md
```

---

## Responsibilities

This repository manages:

- Argo CD Application definitions
- Helm value overrides
- Application configuration
- Kubernetes desired state

It does **not** manage:

- AWS infrastructure
- VPC
- EKS cluster
- IAM resources
- ECR repositories
- Kubernetes platform components
- local kind workloads or local Kubernetes configuration

Those responsibilities belong to Terraform repositories.

---

## Deployment Flow

The complete deployment workflow:

```
Developer
    |
    v
Application Source Code
    |
    v
GitHub Actions
    |
    v
Docker Image Build
    |
    v
Amazon ECR
    |
    v
GitOps Repository Update
    |
    v
Argo CD Sync
    |
    v
Amazon EKS
```

---

## Argo CD Application

The declarative definition lives at:

```text
argocd/applications/otel-demo.yaml
```

The Terraform application stage in `otel-demo-infra-aws` performs cluster
registration. The Application references:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:
  name: otel-demo-aws
  namespace: argocd

spec:
  project: default

  destination:
    server: https://kubernetes.default.svc
    namespace: opentelemetry-demo
```

1. OpenTelemetry Demo Helm chart

```
https://open-telemetry.github.io/opentelemetry-helm-charts
```

2. This Git repository for values:

```
applications/otel-demo/values.yaml
```

---

## Helm Configuration

Application overrides are stored in:

```
applications/otel-demo/values.yaml
```

Example:

```yaml
components:
  recommendation:
    imageOverride:
      repository: 123456789012.dkr.ecr.us-east-1.amazonaws.com/recommendation
      tag: dev
```

This allows application images to be controlled independently from the upstream Helm chart.

---

## Current Application Status

## Recommendation Service

The recommendation service is the first application validated through the GitOps workflow.

Current flow:

```
Recommendation Source
        |
        v
Docker Build
        |
        v
Amazon ECR
        |
        v
Helm values update
        |
        v
Argo CD Sync
        |
        v
Recommendation Pod
```

Example image:

```
123456789012.dkr.ecr.us-east-1.amazonaws.com/recommendation:dev
```

---

## Product Catalog and Ad delivery readiness

Implementation is prepared in `otel-demo-apps`; ECR repositories and scoped
publisher roles are prepared in infrastructure layer 04. This is not a record
of publication or deployment. Only Recommendation currently has a custom
image override in active values.

Chart **0.38.4 remains pinned**. Product Catalog uses the chart's existing
`product-catalog-products` ConfigMap mounted at `/usr/src/app/products`, with
atomic JSON reloads. No PostgreSQL or chart upgrade is part of this work.
Ad's missing protobuf build input and Java agent version are supplied in the
application repository, with provenance and license retained.

Before activation, apply the reviewed layer-04 plan, configure the GitOps token,
and complete application CI. Merging the release changes to apps main can
publish and promote images, so schedule that merge as a release action.
Each new service's workflow publishes a full-SHA ECR image, pulls and smoke-tests
it, then adds only `components.product-catalog.imageOverride` or
`components.ad.imageOverride`. Concurrent promotions use fetch/replay retries
and ordinary non-force pushes. Do not add example image references here.

After promotion, observe Argo CD sync/health and Deployment rollouts; confirm
pod image digests match ECR and exercise catalog list/get/search and ads.
Check gRPC health plus traces, metrics, and logs, and record timestamps,
workflow run links, source SHAs, ECR digests, and observed cluster results.
Local container tests and Helm rendering alone do not establish EKS health.

Rollback only the affected override to a previously verified SHA. Removing
its override restores the pinned chart's default image. Preserve unrelated
promotions. Full build, release, trust, and verification instructions are in
`otel-demo-apps/DELIVERY.md`.

The existing disabled-OpenSearch/default-Collector-log-exporter mismatch is
separate and remains unresolved; verify downstream log storage independently.

---

## Synchronization

## Manual Sync

From the Argo CD UI:

1. Open application
2. Select Sync
3. Review resources
4. Synchronize

---

## Automatic Sync

The current Application configuration enables:

```yaml
syncPolicy:

  automated:
    prune: true
    selfHeal: true
```

This means:

### Self Heal

If Kubernetes resources drift from Git:

```
Git State != Cluster State
```

Argo CD restores the desired state.

---

### Prune

If resources are removed from Git:

```
Git
 |
 v
Resource deleted
 |
 v
Argo CD removes resource
```

---

## Connecting Argo CD

The cluster must have:

- Argo CD installed
- Network access to this repository
- Kubernetes credentials configured

For private repositories, configure repository credentials:

```bash
argocd repo add <repository-url>
```

For public repositories no credentials are required.

---

## Validation

Check Argo CD applications:

```bash
kubectl get applications \
-n argocd
```

Expected:

```
NAME        SYNC STATUS   HEALTH STATUS
otel-demo   Synced        Healthy
```

---

Check workloads:

```bash
kubectl get pods \
-n opentelemetry-demo
```

Expected:

```
Running
```

## Validate the AWS GitOps delivery trail

For a Recommendation release from `otel-demo-apps/main`, verify all three
systems:

1. In **otel-demo-apps → Actions**, open **Release recommendation service**
   and confirm the ECR push and GitOps update steps succeeded.
2. In this repository, confirm that CI created a commit named
   `chore(recommendation): deploy <commit-sha>` and changed only
   `applications/otel-demo/values.yaml`:

   ```bash
   git fetch origin main
   git log --oneline -3 origin/main -- applications/otel-demo/values.yaml
   git show origin/main:applications/otel-demo/values.yaml
   ```

3. In the AWS Argo CD GUI, open `otel-demo`, refresh it, and verify the
   application moves back to `Synced` and `Healthy`. Inspect **History and
   Rollback** and the Recommendation Deployment to confirm the ECR image tag
   matches the application commit SHA.

This AWS audit trail is separate from the local one: no local release should
create a commit in `otel-demo-gitops-aws`.

---

## Relationship With Terraform

The project intentionally separates ownership:

| Component | Owner |
|---|---|
| VPC | Terraform |
| EKS | Terraform |
| IAM | Terraform |
| AWS Load Balancer Controller | Terraform |
| ECR | Terraform |
| Kubernetes Applications | Argo CD |
| Helm Values | Git |
| Container Images | CI/CD |

---

## Operational Workflow

Typical change:

1. Developer changes application code.
2. CI builds a new container image.
3. Image is pushed to ECR.
4. GitOps values file is updated.
5. Argo CD detects the change.
6. Kubernetes deployment is updated.

---

## Lessons Learned

During development:

- Terraform should not directly manage application workloads when using GitOps.
- Argo CD should be the Kubernetes application owner.
- Public Git repositories simplify initial GitOps setup.
- Private repositories require Argo CD repository credentials.
- Helm chart values provide clean separation between upstream charts and environment customization.
- Application lifecycle is independent from infrastructure lifecycle.

---

## Future Enhancements

Potential improvements:

- Private GitHub repository with Argo CD credentials
- Image promotion workflow
- Development/staging/production environments
- Application health checks
- Progressive delivery using Argo Rollouts
- Security scanning integration
- SBOM generation

---

## Project Lifecycle

Current completed phases:

✅ Terraform backend  
✅ AWS infrastructure  
✅ Amazon EKS cluster  
✅ Kubernetes platform services  
✅ AWS Load Balancer Controller  
✅ Amazon ECR  
✅ Argo CD installation  
✅ GitOps repository integration  
✅ Recommendation service deployment  

Release workflow described in this guide:

GitHub Actions automation and GitOps image updates are documented above.
Verify individual releases using the delivery trail; the checklist records
project milestones, not current cluster health.

```
GitHub Actions
        |
        v
Build Image
        |
        v
Push ECR
        |
        v
Update GitOps
        |
        v
Argo CD Deployment
```
