# VideoRank MLOps Config Repo

This repository is the GitOps source of truth for Kubernetes runtime state.

The app repository builds immutable images and opens promotion pull requests here. Flux runs
inside the cluster, reads this repository, and reconciles the declared state.

## Repository Layout

- `clusters/dev`: Flux entrypoint for the dev GKE Autopilot lab cluster.
- `apps/videorank-api`: Helm chart plus Kustomize base and dev overlay for the
  API deployment.
- `infrastructure/controllers`: controllers installed by Flux, including KubeRay.
- `infrastructure/image-automation`: Flux `ImageRepository`, `ImagePolicy` and
  `ImageUpdateAutomation` objects for the API image update lab.
- `infrastructure/namespaces`: namespaces, quotas and default limits.
- `jobs/ray-training`: short RayJob used to discuss KubeRay in the interview.
- `secrets`: SOPS-encrypted secret templates.

## Kustomize Validation

Render the cluster entrypoint before opening or merging a GitOps PR:

```bash
kubectl kustomize clusters/dev
```

Render the API overlay directly when reviewing application values:

```bash
kubectl kustomize apps/videorank-api/overlays/dev
```

The generated `videorank-api-values` `ConfigMap` intentionally keeps its
Kustomize hash suffix. The custom transformer config in
`apps/videorank-api/base/kustomizeconfig.yaml` propagates that generated name
into `HelmRelease.spec.valuesFrom[].name`.

## Bootstrap

Flux image automation controllers are optional Flux components. This lab
installs them explicitly because the repo declares `ImageRepository`,
`ImagePolicy` and `ImageUpdateAutomation` resources.

```bash
flux check --pre
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

After bootstrap:

```bash
flux get all -A
flux get image repository -A
flux get image policy -A
flux get image update -A
kubectl get helmreleases -A
kubectl get rayjobs -A
```

## Image Automation Mode

`ImageUpdateAutomation` pushes changes to `flux/image-updates/dev`, not `main`.
That keeps promotion reviewable: create a PR from that branch to `main`, inspect
updated `repository`, `tag` and `digest` markers, then merge if the image should
run in the lab cluster.

The API chart still deploys by digest when `image.digest` is set. Tags are used
for discovery and policy selection; the digest pins the exact image content.

For private Google Artifact Registry access, the Flux `ImageRepository` uses
`provider: gcp`. The GKE lab must grant the image reflector identity Artifact
Registry read access, usually with GKE Workload Identity or suitable node scopes.
