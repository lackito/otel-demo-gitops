# AWS GitOps operations

[Overview](../README.md) · [AWS infrastructure operations](https://github.com/lackito/otel-demo-infra-aws/blob/main/docs/OPERATIONS.md)

## Configuration

The Application is `otel-demo-aws` in namespace `argocd`; workloads use
`opentelemetry-demo`. It combines the pinned upstream chart with
`applications/otel-demo/values.yaml` from this repository's `main` branch.
Terraform layer 04 registers the Application. Keep its source configuration and
the declarative [Application YAML](../argocd/applications/otel-demo.yaml) aligned
when changing chart version, sources, destination, sync policy, or finalizers.

Ad, Product Catalog, and Recommendation each have a custom ECR override under
`components.<service>.imageOverride`. Other workloads use chart defaults unless
explicitly overridden. Application changes belong in `otel-demo-apps`; values
changes belong here. Commit and push values changes so Argo CD can see them.

## Release a custom service

1. Provision AWS layers 02 → 03 → 04 and inspect the publisher role outputs.
2. In `otel-demo-apps`, configure `AWS_GITOPS_REPOSITORY_TOKEN` with Contents read/write access to this repository and set `AWS_RELEASES_ENABLED=true`.
3. Merge the tested feature into `otel-demo-apps/main`, or manually run its AWS release workflow from `main`.
4. Confirm the workflow publishes and smoke-tests its full-SHA ECR image, then pushes the corresponding values change here.
5. Wait for Argo CD `otel-demo-aws` to be `Synced` and `Healthy`.

Repeat for all three services after creating empty ECR repositories. Updating
only Ad does not automatically publish Product Catalog or Recommendation;
shared workflow inputs can trigger several service releases.

AWS authentication uses GitHub OIDC and a per-service publisher role. The
GitOps token is for this repository, not the infrastructure repository.
See [delivery setup](https://github.com/lackito/otel-demo-apps/blob/main/DELIVERY.md)
for workflow inputs and credentials.

## Verify the release

Run from `otel-demo-infra-aws`:

```bash
aws-vault exec admin -- make application-validate
```

For Ad, Product Catalog, and Recommendation, compare:

| Evidence | Expected match |
| --- | --- |
| Successful release job | Intended application commit SHA and ECR image URI |
| ECR tag | Full source commit SHA |
| Pushed values | Same service repository and tag |
| Argo CD revision | Generated GitOps commit |
| Deployment and pod image ID | Same image reference and ECR platform digest |

A tag can point to an image index while the pod reports a platform image digest
inside it. These digests can differ. Untagged entries in ECR can be platform
manifests or build attestations referenced by the tagged index.

Exercise the store and check all three services in Jaeger and Grafana. Rollout
validation does not prove telemetry delivery. OpenSearch is disabled in AWS;
stored logs are not currently validated. Use `kubectl logs` for container output.

## Troubleshooting

For `ImagePullBackOff`, inspect pod events and confirm the exact tag exists in
ECR. Rebuilding infrastructure deletes old ECR images; run each service's release
workflow to repopulate its repository.

For a skipped release, check it runs on `main` and `AWS_RELEASES_ENABLED=true`.
For a failed GitOps push, check the token's selected repository and Contents
permission. For a stalled sync, inspect the Application's sync errors and source
revision. Change the GitOps source to persist a fix; self-healing can undo manual
Deployment edits.

## Teardown

Set `AWS_RELEASES_ENABLED=false` while AWS is down. Destroy Terraform layer 04
first: the Application finalizer needs running Argo CD to clean up workloads.
Dev ECR repositories use `force_delete=true`, removing their images during
destroy. Then destroy 03 and 02. Retain the shared state backend and GitHub OIDC
provider. See the [full teardown sequence](https://github.com/lackito/otel-demo-infra-aws/blob/main/docs/OPERATIONS.md#tear-down).
