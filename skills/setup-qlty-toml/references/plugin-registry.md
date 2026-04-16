# Plugin Registry Notes

> **Source of truth:** https://github.com/qltysh/qlty/tree/main/qlty-plugins/plugins
>
> Always verify a plugin exists there before adding it to qlty.toml. The entries below are observations from past eval runs — they may become outdated as the registry evolves.

---

## Plugins confirmed NOT in the public registry

These cause **"Build errored"** if added to qlty.toml with `[[source]] default = true`.

| Plugin name | Notes | Last confirmed |
|---|---|---|
| `psalm` | PHP static analysis tool — despite `psalm.xml` being a common config file, no Qlty plugin exists. Use `phpstan` instead. | 2026-04-16 |
| `radarlint-java` | Enterprise-only — requires separate Qlty enterprise setup. Not resolvable from the public registry. | 2026-04-16 |
| `radarlint-kotlin` | Enterprise-only — same as radarlint-java. | 2026-04-16 |
| `radarlint-js` | Enterprise-only. | 2026-04-16 |
| `radarlint-python` | Enterprise-only. | 2026-04-16 |
| `radarlint-ruby` | Enterprise-only. | 2026-04-16 |
| `radarlint-php` | Enterprise-only. | 2026-04-16 |
| `radarlint-go` | Enterprise-only. | 2026-04-16 |

---

## Plugins with known caveats

### `tsc`
- Only enable if the project already passes `tsc --noEmit` in CI. On real-world TypeScript repos with uncertain compilation status, `tsc` causes "Build errored" in Qlty Cloud even with `skip_upstream = true`.
- When in doubt, skip `tsc` and rely on `eslint` with `@typescript-eslint` for TypeScript linting.
- Last confirmed problematic: evals 26-30 (2026-04-15)

### `zizmor`
- May not appear on docs.qlty.sh/plugins, but IS a supported Qlty plugin.
- Strong signal to include: `zizmor.yml` or `.github/workflows/zizmor.yml` already exists in the repo's CI.
- Last confirmed working: evals 41-50 (2026-04-16)

### `trivy` — valid `drivers` values
- `"config"` — IaC scanning (Terraform, Docker, K8s)
- `"secret"` — secret scanning (trufflehog is generally preferred for this)
- `"vuln"` — vulnerability scanning of lockfiles/containers
- **`"fs-vuln"` is NOT a valid value** — causes config parse error.
- Last confirmed: evals 31-40 (2026-04-16)

---

## `[[triage]]` caveats

- **`set.category`** — may cause "Build errored" in Qlty Cloud. Avoid until confirmed supported. Use `set.mode` instead for suppression.

---

## How to check the registry

To verify a plugin is available, browse:
```
https://github.com/qltysh/qlty/tree/main/qlty-plugins/plugins
```
Each plugin has its own subdirectory. If a subdirectory named `<plugin-name>` exists, the plugin is in the public registry.

---

## Updating this file

When a new eval run reveals a plugin is missing from the registry or has a caveat, add it here with a `Last confirmed` date. Remove or correct entries that become outdated.
