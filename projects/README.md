# projects

`projects/` contains project or feature deployment units. Each project keeps its
app source, target-specific values, and Karmada policies together.

Create a new project by copying `_template`:

```text
cp -R projects/_template projects/<project-name>
```

Then update:

- `app/values.yaml`
- `targets/common/values.yaml`
- `targets/b/values.yaml`
- `targets/c/values.yaml`
- `policies/propagation.yaml`
- `policies/override.yaml`

The template is safe and example-only. It must not be used as a production app
without replacing names, labels, and selectors.
