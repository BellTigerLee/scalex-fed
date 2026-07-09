# policies

`policies/` contains Karmada policy examples and templates for deciding how
ScaleX federation workloads are propagated to member clusters.

Policy examples use the existing member cluster label convention:

```yaml
scalex.io/role: child
scalex.io/topology: baremetal
scalex.io/site: b or c
```

The example policies select only the placeholder ConfigMap named
`scalex-federation-example-placeholder` in the `scalex-federation-examples`
namespace. They are examples, not production placement policy.

## Direct examples

`policies/examples/` contains plain YAML manifests suitable for Argo CD directory
sync after they are explicitly enabled through `apps/federation`.

## Template examples

`policies/templates/cluster-policies.example.helm.yaml` preserves the previous
Helm-style policy template as a source example. It is not rendered by the
`apps/federation` chart because that chart must stay within its own chart root.
