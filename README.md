# VideoRank MLOps Config Repo

This repository is the GitOps source of truth for Kubernetes runtime state.

The app repository builds immutable images and opens promotion pull requests here. Flux runs
inside the cluster, reads this repository, and reconciles the declared state.

## Repository Layout

- `clusters/dev`: Flux entrypoint for the dev GKE Autopilot lab cluster.
- `apps/videorank-api`: Helm chart and dev values for the API deployment.
- `infrastructure/controllers`: controllers installed by Flux, including KubeRay.
- `infrastructure/namespaces`: namespaces, quotas and default limits.
- `jobs/ray-training`: short RayJob used to discuss KubeRay in the interview.
- `secrets`: SOPS-encrypted secret templates.

## Bootstrap

```bash
flux bootstrap github \
  --owner="$GITHUB_OWNER" \
  --repository=videorank-mlops-config \
  --branch=main \
  --path=clusters/dev \
  --personal \
  --private=true
```

After bootstrap:

```bash
flux get all -A
kubectl get helmreleases -A
kubectl get rayjobs -A
```

