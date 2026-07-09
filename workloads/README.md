# workloads

`workloads/` contains source resources and apps that Karmada policies can select
and propagate to member clusters such as `b` and `c`.

This directory is separate from `apps/` because `apps/` is reserved for Argo and
Helm entrypoints that Tower can sync. It is also separate from `policies/`,
which owns Karmada propagation and override policy resources.

Nothing in this directory is deployed by default. The Tower-facing chart at
`apps/federation` creates a child Application for workload paths only when
`workloads.enabled` is explicitly set to `true`.
