# otel-demo-gitops-aws

AWS desired state for the [OpenTelemetry Demo](https://opentelemetry.io/docs/demo/).
Argo CD combines the upstream Helm chart with these values and reconciles the
workloads on EKS.

## Files and ownership

| File | Purpose |
| --- | --- |
| [Application](argocd/applications/otel-demo.yaml) | Chart version, values source, destination, and sync policy |
| [Helm values](applications/otel-demo/values.yaml) | AWS settings for the upstream baseline |
| [Operations](docs/OPERATIONS.md) | Release verification, troubleshooting, and teardown |

[otel-demo-infra-aws](https://github.com/lackito/otel-demo-infra-aws) provisions
EKS, installs Argo CD, and registers the Application through Terraform.
[otel-demo-apps](https://github.com/lackito/otel-demo-apps) owns application source
and release workflows. This repository owns AWS application configuration.

## Releases

Chart `0.42.6` supplies the upstream `3.1.0` images for Ad, Product Catalog,
and Recommendation. Keep custom image publishing disabled while verifying
this baseline. Reintroduce each service separately using the release workflow
in the operations guide.

## Telemetry and lifecycle

Jaeger and Prometheus are enabled. OpenSearch is disabled in the current AWS
values, while the chart log exporter still targets it; stored logs need further
configuration. Container stdout remains available through `kubectl logs`.

The Application includes Argo CD's cascading-deletion finalizer, matching its
Terraform registration. Destroy layer 04 while Argo CD is running, then 03 and
02. The disposable dev ECR repositories delete their stored images during
Terraform destroy. Republish custom images when reintroducing those services.

## Related guides

- [AWS operations](https://github.com/lackito/otel-demo-infra-aws/blob/main/docs/OPERATIONS.md)
- [Branching](https://github.com/lackito/otel-demo-apps/blob/main/docs/BRANCHING.md)
- [Names and repositories](https://github.com/lackito/otel-demo-apps/blob/main/docs/NAMING.md)
- [kind desired state](https://github.com/lackito/otel-demo-gitops-kind)
