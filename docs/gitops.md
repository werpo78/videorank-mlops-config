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

## Flux Objects Used Here

- `GitRepository`: source object created by `flux bootstrap`.
- `Kustomization`: applies paths from this repo with dependency ordering,
  pruning, waiting and health checks.
- `HelmRepository`: points Flux to an external Helm chart repository.
- `HelmRelease`: installs/upgrades a Helm chart from declared values.
- `ImageRepository`: scans Artifact Registry tags for the API image.
- `ImagePolicy`: selects the newest sortable CI tag and reflects the digest.
- `ImageUpdateAutomation`: writes selected image values back to Git.

## Bootstrap Best Practices

Before bootstrap, run `flux check --pre` against the target cluster. Bootstrap is
idempotent, creates the Flux manifests, pushes them to Git and installs the
controllers. This lab uses:

```bash
flux bootstrap github \
  --owner="$GITHUB_OWNER" \
  --repository=videorank-mlops-config \
  --branch=main \
  --path=clusters/dev \
  --components-extra=image-reflector-controller,image-automation-controller \
  --read-write-key=true \
  --personal \
  --private=true
```

The `--components-extra` flag is intentional: image automation controllers are
not part of a minimal Flux install.

`--read-write-key=true` is required because `ImageUpdateAutomation` pushes
commits back to the repository. If the cluster was already bootstrapped with a
read-only deploy key, rotate the `flux-system` Secret and rerun bootstrap.

## Reconciliation Diagnostics

Useful commands:

```bash
flux get all -A
flux get kustomizations -A
flux get helmreleases -A
flux get image repository -A
flux get image policy -A
flux get image update -A
flux reconcile source git flux-system
flux reconcile kustomization videorank-api -n flux-system
flux reconcile helmrelease videorank-api -n videorank-dev --reset
flux events --for kustomization/videorank-api
```

Common failure interpretation:

- `kustomization path not found`: the Flux `path` does not exist in Git.
- `Source not ready`: debug the `GitRepository` or `HelmRepository` first.
- `Helm upgrade failed`: inspect the `HelmRelease` events, then reset/retry.
- `ImagePolicy` not ready: inspect `ImageRepository` auth and tag pattern.

## HelmRelease Best Practices

The API `HelmRelease` is declared from a chart stored in the config repo. It has:

- `valuesFrom` pointing to a generated values `ConfigMap`.
- explicit `install.remediation.retries`.
- explicit `upgrade.remediation.retries`.
- `strategy: rollback` and `remediateLastFailure: true`.
- `rollback.timeout` so failed upgrades do not remain ambiguous.

This is the production interview point: a HelmRelease should declare how failed
installs/upgrades behave. Otherwise operations depend on defaults and manual
interpretation during incidents.

## Image Automation

CI pushes two tags:

- the full Git SHA tag for traceability.
- `main-<run_number>-<short_sha>` for deterministic Flux selection.

Flux scans the image repository with `provider: gcp`, filters only sortable
`main-*` tags, chooses the highest run number, and writes updates to the
`flux/image-updates/dev` branch. It does not push directly to `main`.

The values file has required markers:

```yaml
repository: ... # {"$imagepolicy": "flux-system:videorank-api:name"}
tag: ... # {"$imagepolicy": "flux-system:videorank-api:tag"}
digest: ... # {"$imagepolicy": "flux-system:videorank-api:digest"}
```

The chart deploys by digest when `image.digest` is set. Tags are selection
inputs; digest is the immutable runtime identity.

## Promotion

The normal app repo CI updates
`apps/videorank-api/overlays/dev/values.yaml` with an immutable image digest.
Merging that PR promotes the application image.

Flux image automation is a learning path for the interview and an optional
staging helper: it can create an update branch from registry state, but humans
still review the PR before merge.

Model promotion is separate from image promotion. The app repo `promote-model`
workflow opens a PR that updates only model runtime values:

- `VIDEORANK_MODEL_URI`
- `VIDEORANK_MODEL_VERSION`
- `VIDEORANK_VERTEX_MODEL_RESOURCE`
- `VIDEORANK_DATA_SNAPSHOT_ID`
- `VIDEORANK_MODEL_GIT_SHA`
- `VIDEORANK_MODEL_PROMOTED_AT`

This lets a validated Vertex AI model candidate be promoted or rolled back
without rebuilding the serving image.

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
image digest or previous model URI, depending on what was promoted.

## Security Notes

- Secrets must be SOPS-encrypted before Git.
- Flux permissions should be constrained by namespace in multi-tenant clusters.
- Production should require immutable image digests.
- Manual cluster changes are temporary diagnostics, not durable fixes.
- For private Artifact Registry, prefer GKE Workload Identity for Flux image
  controllers instead of static registry credentials.
