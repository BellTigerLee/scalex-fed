# scalex-federation chart

`apps/scalex-federation` is the federation intent chart in the `scalex-fed`
repository. It provides only a minimal Karmada policy skeleton so a future
`tower-k8s` Argo CD Application can use this path as its source.

## Safe defaults

`values.yaml` sets `examplePolicies.enabled` to `false` by default. With the
default values, the chart renders no `ClusterPropagationPolicy` or
`ClusterOverridePolicy` resources.

The default member selector documents the current Karmada member join labels.

```yaml
scalex.io/role: child
scalex.io/topology: baremetal
scalex.io/site: b or c
```

The example policy selectors intentionally target only the placeholder ConfigMap
named `scalex-federation-example-placeholder` in the
`scalex-federation-examples` namespace so they do not accidentally select a
production resource.

## Non-rendered examples

These files are documentation examples only. Helm does not render them.

- `examples/cluster-propagation-policy.child-baremetal.example.yaml`
- `examples/cluster-override-policy.child-baremetal.example.yaml`

## Future tower-k8s Argo Application snippet

This snippet is documentation only. Do not edit `tower-k8s` from this repository.
Karmada installs around sync wave 20 and member clusters join around sync wave 30,
so the future federation policy app should be placed after those steps, around
sync wave 40.

```yaml
apps:
  scalex-federation:
    enabled: true
    annotations:
      argocd.argoproj.io/sync-wave: "40"
    source:
      repoURL: https://example.invalid/scalex-fed.git
      path: apps/scalex-federation
      targetRevision: main
```

Choose the real `repoURL` and `targetRevision` according to tower control-plane
operations policy.

## Enabling example rendering

Enable example rendering explicitly. Do not apply the examples to a cluster
until the selectors are replaced with intentional production selectors.

```yaml
examplePolicies:
  enabled: true
```
