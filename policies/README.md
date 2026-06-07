# Policy Notes

Recommended production additions:

- Kyverno or Gatekeeper policies to require resource requests/limits.
- Restrict privileged pods and hostPath mounts with Pod Security Standards.
- Require immutable image digests for production namespaces.
- Deny cross-namespace Flux references except for approved platform sources.
- Enforce labels: `app.kubernetes.io/name`, `app.kubernetes.io/part-of`, `environment`.

