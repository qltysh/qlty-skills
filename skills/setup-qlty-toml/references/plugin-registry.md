# Known Cases

Accumulated learnings from past runs. Consult this when you encounter a situation that isn't obvious from the docs or the plugin registry. Add new entries when you resolve a non-obvious case — include what you tried, what failed, and what worked.

Plugin availability is always checked live from https://github.com/qltysh/qlty/tree/main/qlty-plugins/plugins. This file records **what the registry listing alone won't tell you**.

---

## `tsc`

**Only enable if the project already passes `tsc --noEmit` in CI.** On real-world TypeScript repos with uncertain compilation status, `tsc` causes "Build errored" in Qlty Cloud even with `skip_upstream = true`. When in doubt, skip `tsc` and rely on `eslint` with `@typescript-eslint` for TypeScript linting.

*Last seen: evals 26–30 (2026-04-15)*

---

## `trivy` — valid `drivers` values

Valid values: `"config"` (IaC scanning), `"secret"` (secrets), `"vuln"` (vulnerability scanning). **`"fs-vuln"` is NOT valid** — causes a config parse error. `qlty init` has been observed generating `"fs-vuln"` — strip it if present.

*Last seen: evals 31–40 (2026-04-16)*

---

## `[[triage]] set.category`

May cause "Build errored" in Qlty Cloud. Use `set.mode` instead for suppression.

*Last seen: evals 26–30 (2026-04-15)*

---

## `zizmor`

IS a supported Qlty plugin despite not appearing on docs.qlty.sh/plugins. Strong signal to include: `zizmor.yml` already exists in the repo's `.github/workflows/`.

*Last seen: evals 41–50 (2026-04-16)*
