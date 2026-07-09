# project template

This template shows the expected project layout for `scalex-fed`.

```text
_template/
  app/
    Chart.yaml
    values.yaml
    templates/
      configmap.yaml
  targets/
    common/
      values.yaml
    b/
      values.yaml
    c/
      values.yaml
  policies/
    propagation.yaml
    override.yaml
```

Tower renders `app/` with `app/values.yaml` and one target values file. Karmada
policies then select the rendered resources by `scalex.io/project` and
`scalex.io/target` labels.

The template policies are namespace-scoped. Keep the rendered app resources and
the policy resources in the same namespace, or replace them with cluster-scoped
policy kinds for a different operating model.

This template is not a production deployment.
