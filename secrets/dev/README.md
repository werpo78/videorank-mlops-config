# SOPS Secrets

Do not commit cleartext Kubernetes Secrets.

For the lab, Age is the fastest setup:

```bash
age-keygen -o age.key
export SOPS_AGE_KEY_FILE=$PWD/age.key
```

Copy the public key into `.sops.yaml`, then create a secret:

```bash
kubectl create secret generic videorank-api-runtime \
  --from-literal=example=value \
  --dry-run=client -o yaml > secrets/dev/videorank-api-runtime.yaml
sops --encrypt --in-place secrets/dev/videorank-api-runtime.yaml
```

Production answer: use GCP KMS and IAM/Workload Identity so Flux can decrypt
without a static private key stored in the cluster.

Secret ownership model:

- GitHub Actions secrets stay in `videorank-mlops-app`; the app workflow owns
  build and promotion.
- SOPS files stay in `videorank-mlops-config`; Flux owns Kubernetes desired
  state.
- Runtime cloud secrets should come from Secret Manager or a Kubernetes secret
  controller, not from plaintext manifests.
