# Known Cases

Accumulated learnings from past runs. Consult this when you encounter a situation that isn't obvious from the docs or the plugin registry. Add new entries when you resolve a non-obvious case — include what you tried, what failed, and what worked.

Plugin availability is always checked live from https://github.com/qltysh/qlty/tree/main/qlty-plugins/plugins. This file records **what the registry listing alone won't tell you**.

---

## `radarlint-*`

All `radarlint-*` variants (`radarlint-java`, `radarlint-kotlin`, `radarlint-js`, `radarlint-php`, `radarlint-python`, `radarlint-ruby`, `radarlint-go`, `radarlint-iac`, `radarlint-scala`) are in the public registry and work without enterprise access. Include them in `monitor` mode — they can be noisy on first runs but are valid to enable.

*Confirmed working: gson build passed with radarlint-java in comment mode (2026-04-16)*

---

## `tsc`

**Do not use.** The `tsc` plugin is marked `hidden = true` in the Qlty registry and has no `known_good` version defined. Including it causes config validation to fail immediately with:

```
The enabled plugin version is "known_good", but the known good version is unknown: tsc
```

This happens before any code runs — the build is rejected at the validate config step. Use `eslint` with `@typescript-eslint` for TypeScript linting instead.

*Confirmed broken at config validation (2026-04-16)*

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
