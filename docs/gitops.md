# GitOps Interview Notes

## Core Principle

Git contains the desired runtime state. Flux runs inside the cluster and pulls
that state. If the live cluster drifts, Flux reconciles it back to Git.

## Why App Repo + Config Repo

The app repo owns source and artifact production. The config repo owns runtime
promotion. This keeps permissions and responsibilities clean:

- Developers can open app PRs without directly deploying to Kubernetes.
- Platform or ML owners can review promotions in the config repo.
- Rollback is a Git revert.
- The cluster is not dependent on a one-shot CI job for long-term correctness.

## Objects Used Here

- `GitRepository`: source object created by `flux bootstrap`.
- `Kustomization`: applies paths from this repo with dependency ordering,
  pruning and health checks.
- `HelmRepository`: points Flux to an external Helm chart repository.
- `HelmRelease`: installs/upgrades a Helm chart from declared values.

## Promotion

The app repo CI updates
`apps/videorank-api/overlays/dev/values.yaml` with an immutable image digest.
Merging that change promotes the application.

## Kustomize Layout

The API follows the base + overlay pattern:

- `apps/videorank-api/base`: reusable `HelmRelease` contract.
- `apps/videorank-api/overlays/dev`: dev namespace, labels, annotations,
  generated values `ConfigMap` and environment patch.

The values `ConfigMap` keeps Kustomize's default name suffix hash. The
`base/kustomizeconfig.yaml` file teaches Kustomize that
`HelmRelease.spec.valuesFrom[].name` references a `ConfigMap`, so the generated
hash is propagated into the `HelmRelease`. This avoids disabling the hash while
still letting Flux notice values changes cleanly.

We do not use `namePrefix` for this release because the environment is isolated
by namespace and Flux health checks expect a stable `HelmRelease` name.

## Rollback

Revert the promotion commit in this repository. Flux will apply the previous
image digest.

## Security Notes

- Secrets must be SOPS-encrypted before Git.
- Flux permissions should be constrained by namespace in multi-tenant clusters.
- Production should require immutable image digests.
- Manual cluster changes are temporary diagnostics, not durable fixes.

