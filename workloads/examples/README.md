# workload examples

This directory contains harmless example source workloads for validating the
ScaleX federation flow.

The placeholder ConfigMap is intentionally small and non-production. It exists
only so the policy examples can target a known resource name:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: scalex-federation-examples
  name: scalex-federation-example-placeholder
```

Do not put production workloads, credentials, kubeconfigs, or cluster-specific
runtime state in this example directory.

The directory also includes `namespace.yaml` so the example path is self-contained
when a child Argo CD Application syncs it.
