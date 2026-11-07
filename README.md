# otel-demo-gitops-aws

AWS desired state for the [OpenTelemetry Demo](https://opentelemetry.io/docs/demo/).
Argo CD combines the upstream Helm chart with these values and reconciles the
workloads on EKS.

## Files and ownership

| File | Purpose |
| --- | --- |
| [Application](argocd/applications/otel-demo.yaml) | Chart version, values source, destination, and sync policy |
| [Helm values](applications/otel-demo/values.yaml) | AWS settings and Ad, Product Catalog, and Recommendation ECR overrides |
| [Operations](docs/OPERATIONS.md) | Release verification, troubleshooting, and teardown |

[otel-demo-infra-aws](https://github.com/lackito/otel-demo-infra-aws) provisions
EKS, installs Argo CD, and registers the Application through Terraform.
[otel-demo-apps](https://github.com/lackito/otel-demo-apps) owns application source
and release workflows. This repository owns AWS application configuration.

## Releases

Each custom service has its own AWS workflow. A qualifying change on
`otel-demo-apps/main` builds and smoke-tests that service, publishes an immutable
full-SHA image to ECR, and updates its image override here. Argo CD watches
`main` and deploys the pushed values change with automated sync, pruning, and
self-healing.

All three custom services use ECR: **Ad, Product Catalog, and Recommendation**.
Verify the release SHA, ECR platform digest, GitOps commit, Argo CD revision,
and deployed image. From the infrastructure repository, run
`aws-vault exec admin -- make application-validate` for the deployment summary.

## Telemetry and lifecycle

Jaeger and Prometheus are enabled. OpenSearch is disabled in the current AWS
values, while the chart log exporter still targets it; stored logs need further
configuration. Container stdout remains available through `kubectl logs`.

The Application includes Argo CD's cascading-deletion finalizer, matching its
Terraform registration. Destroy layer 04 while Argo CD is running, then 03 and
02. The disposable dev ECR repositories delete their stored images during
Terraform destroy. Republish all three images after rebuilding.

## Related guides

- [AWS operations](https://github.com/lackito/otel-demo-infra-aws/blob/main/docs/OPERATIONS.md)
- [Branching](https://github.com/lackito/otel-demo-apps/blob/main/docs/BRANCHING.md)
- [Names and repositories](https://github.com/lackito/otel-demo-apps/blob/main/docs/NAMING.md)
- [kind desired state](https://github.com/lackito/otel-demo-gitops-kind)
