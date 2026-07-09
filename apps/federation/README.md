# federation chart

`apps/federation` is the thin Tower-facing Helm entrypoint for the `scalex-fed`
repository. It is an Argo CD app-of-apps chart, not a Karmada policy chart.

Helm charts cannot include files outside their chart root, so this chart does
not render the top-level `workloads/` or `policies/` directories directly.
Instead, it can create child Argo CD `Application` resources that point back to
those repository paths.

## Safe defaults

Both child Applications are disabled by default.

```yaml
workloads:
  enabled: false
policies:
  enabled: false
```

With the default values, this chart renders no child Application resources.
No workload or policy is deployed unless a caller explicitly enables the child
Application that owns that path.

## Child Application sources

When enabled, the chart creates child Applications in the configured Argo CD
namespace.

| Values key | Default path | Sync wave | Purpose |
| --- | --- | --- | --- |
| `workloads` | `workloads/examples` | `10` | Harmless source workload examples selected by Karmada policies |
| `policies` | `policies/examples` | `20` | Karmada policy examples that select and propagate workloads |

The default Argo CD namespace is `argo`, matching the current Tower control-plane
namespace convention.
The workload child Application also sets `destination.namespace` to
`scalex-federation-examples`, and the example workload path includes the matching
Namespace manifest.

The default child destination is `tower`.

```yaml
destination:
  name: tower
```

Adjust this to the real Argo CD cluster destination name used by the Tower
control plane.

## tower-k8s usage sketch

This repository does not edit `tower-k8s`. A future Tower Application can point
at this chart path and keep both children disabled for an initial safe sync.

```yaml
source:
  repoURL: https://github.com/BellTigerLee/scalex-fed.git
  path: apps/federation
  targetRevision: main
  helm:
    valuesObject:
      workloads:
        enabled: false
      policies:
        enabled: false
```

Enable `workloads.enabled` and `policies.enabled` only after reviewing the
example resources and confirming the intended destination and selectors.
