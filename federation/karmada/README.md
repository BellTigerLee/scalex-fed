# federation/karmada

This directory owns federation-wide Karmada resources for ScaleX. These resources
are applied by Tower to the federation control plane and are not tied to a single
project app.

Use the subdirectories as follows:

- `policies/`: shared propagation policy patterns or federation-wide policies
- `overrides/`: shared override policy patterns or federation-wide overrides
- `placements/`: reusable placement and cluster selector documentation

Project-specific policy manifests belong under `projects/<project>/policies`.
