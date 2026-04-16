# Plugin Registry Notes

> Plugin availability is checked live from https://github.com/qltysh/qlty/tree/main/qlty-plugins/plugins (Phase 2).
> This file records **behavioral caveats** discovered in past runs — things the registry listing alone won't tell you.

---

## Known behavioral caveats

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

## Updating this file

When a run reveals a new behavioral caveat (unexpected Build errored, wrong config field, broken driver value, etc.), add it here with a `Last confirmed` date. Do not record plugin availability here — check the live registry instead.
