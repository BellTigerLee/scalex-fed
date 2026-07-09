# federation/karmada/placements

Store shared placement and cluster selector documentation here.

Current member-cluster label convention:

```yaml
scalex.io/role: child
scalex.io/topology: baremetal
scalex.io/site: b or c
```

Use `scalex.io/site` to target one member and a selector with both `b` and `c`
for common deployments.
