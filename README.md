# scalex-fed

`scalex-fed` is the ScaleX federation deployment-intent repository. It is not a
generic app catalog and it is not a child-cluster preset. It describes how Tower
should render project apps, which target values should be used, and which
Karmada policies should propagate the rendered resources to `b`, `c`, or both.

This repository change only shapes `scalex-fed`. It does not edit `tower-k8s`,
`b-k8s`, `c-k8s`, or `eecs-k8s`.

## Operating model

`tower-k8s` syncs `scalex-fed` resources. For each project, Tower renders the
project app source with base values plus exactly one target values file.
Karmada policy selects rendered resources by labels and propagates them to the
matching member clusters.

```text
tower-k8s Argo CD
  -> reads scalex-fed
  -> renders projects/<project>/app with targets/common, targets/b, or targets/c
  -> applies projects/<project>/policies
  -> Karmada propagates selected resources to b, c, or b+c
```

The target meaning is:

- `common`: render once for resources intended for both `b` and `c`
- `b`: render for resources intended only for member cluster `b`
- `c`: render for resources intended only for member cluster `c`

## Repository layout

```text
scalex-fed/
  README.md
  federation/
    karmada/
      README.md
      policies/
        README.md
      overrides/
        README.md
      placements/
        README.md
  projects/
    README.md
    _template/
      README.md
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

The old top-level `apps/`, `workloads/`, and `policies/` layout is intentionally
removed. The repo is now organized around federation-wide Karmada resources and
project-specific deployment units.

## Directory roles

### `federation/karmada/`

Federation-wide Karmada resources live here. Use it for resources that belong to
the federation control plane as a whole rather than to one project.

- `policies/`: shared propagation policy patterns and federation-wide policies
- `overrides/`: shared override policy patterns and federation-wide overrides
- `placements/`: reusable cluster placement and selector documentation

### `projects/<project>/`

Each project or feature owns its app source, target values, and project-specific
Karmada policies in one directory.

- `app/`: Helm or Kustomize app source rendered by Tower
- `targets/common/values.yaml`: values for resources propagated to both `b` and `c`
- `targets/b/values.yaml`: values for resources propagated only to `b`
- `targets/c/values.yaml`: values for resources propagated only to `c`
- `policies/`: Karmada policies selecting this project's rendered resources

Start a real project by copying `projects/_template` to `projects/<project-name>`
and replacing the example names, labels, values, and policy selectors.

## Label contract

Project app resources should expose labels that project policies can select.
The template uses this contract:

```yaml
scalex.io/project: <project-name>
scalex.io/target: common | b | c
```

Karmada member clusters are expected to have these labels from Tower member join
configuration:

```yaml
scalex.io/role: child
scalex.io/topology: baremetal
scalex.io/site: b
```

```yaml
scalex.io/role: child
scalex.io/topology: baremetal
scalex.io/site: c
```

## Safe defaults

The committed project is a template. It renders a harmless ConfigMap example,
not a production workload. No real secret, kubeconfig, bearer credential, or
cluster-specific runtime state belongs in this repository.

Before enabling a real project, decide:

1. Which app source lives in `projects/<project>/app`
2. Which target values should differ for `common`, `b`, and `c`
3. Which labels the Karmada policies should select
4. Whether propagation is shared or target-specific
5. Which overrides, if any, are required per target cluster

## Tower integration sketch

Tower should create Argo CD Applications that point at project paths in this
repo. A Helm app can be rendered once per target values file, for example:

```yaml
source:
  repoURL: https://github.com/BellTigerLee/scalex-fed.git
  path: projects/<project>/app
  targetRevision: main
  helm:
    valueFiles:
      - values.yaml
      - ../targets/common/values.yaml
```

Then Tower should apply `projects/<project>/policies` so Karmada can propagate
the rendered resources according to `scalex.io/target` and member cluster labels.

The template uses namespace-scoped Karmada `PropagationPolicy` and
`OverridePolicy`. Apply a project's app resources and that project's policy
resources to the same namespace unless you intentionally switch to
cluster-scoped policy kinds.
