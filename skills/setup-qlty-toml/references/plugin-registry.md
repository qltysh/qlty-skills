# Known Cases

Caveats about **Qlty's own behavior** — things the plugin README won't tell you. Consult this before writing any plugin block.

Do NOT record plugin-specific facts here (supported versions, config syntax, package names). Those belong in the plugin's README at `https://raw.githubusercontent.com/qltysh/qlty/main/qlty-plugins/plugins/{plugin-name}/README.md`. Record only what is specific to how Qlty runs, caches, or installs plugins — behaviors that are invisible from reading the plugin README alone.

Add a new entry when you discover a non-obvious Qlty behavior. Include what failed, what worked, and when it was observed (so stale entries can be identified).

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

---

## `eslint` — ESLint 9 flat config required

Qlty's eslint plugin uses **ESLint 9.7.0** (flat config). Legacy `.eslintrc.js` configs are not loaded by ESLint 9. When you specify `config_files = [".qlty/configs/.eslintrc.js"]`, Qlty tries to pass `--config ""` (empty), which fails with:

```
Value for 'config' of type 'path::String' required.
You're using eslint.config.js, some command line flags are no longer available.
```

**Fix:** Use a flat config file (`eslint.config.js`) in `config_files`, not a legacy `.eslintrc.js`.
You cannot override the eslint version via `extra_packages` — Qlty validates the installed version matches the plugin's known_good version and rejects mismatches.

Flat config example for `@typescript-eslint` (compatible with v5 in CJS repos):
```js
const tsParser = require("@typescript-eslint/parser");
const tsPlugin = require("@typescript-eslint/eslint-plugin");
module.exports = [
  {
    files: ["**/*.ts"],
    languageOptions: { parser: tsParser },
    plugins: { "@typescript-eslint": tsPlugin },
    rules: { "@typescript-eslint/no-explicit-any": "warn" },
  },
];
```

*Confirmed 2026-04-17*

---

## `vale` — requires `vale sync` before running

Vale requires a `.vale/styles` directory populated by `vale sync` (downloads style packages). If the styles directory is missing, the build fails at runtime with:

```
[E100] [NewE201] Runtime error
The path '/path/to/repo/.vale/styles' does not exist.
```

**Only enable vale if `.vale/styles` is committed to the repo** (or if the CI runs `vale sync` before the Qlty check). Most repos do not commit the styles directory — skip vale unless you can confirm styles are available.

*Confirmed 2026-04-17*
---

## `trivy` — pin version explicitly to avoid 404 on `known_good` resolution

When Qlty resolves `trivy` at `known_good`, it may attempt to download an older cached version (e.g. `0.67.2`) that no longer exists on GitHub Releases, producing:

```
Error installing trivy@0.67.2.
https://api.github.com/repos/aquasecurity/trivy/releases/tags/0.67.2: status code 404
```

**Fix:** Always pin `trivy` to the explicit version listed in the plugin.toml `known_good_version` field (e.g. `version = "0.69.2"`). Check the current value at https://github.com/qltysh/qlty/blob/main/qlty-plugins/plugins/linters/trivy/plugin.toml before setting.

*First seen: express eval (2026-04-23)*

---

## `mypy` — fails when `pyproject.toml` has `plugins = ["pydantic.mypy"]`

Qlty runs `mypy` in an isolated venv that does not include the project's runtime dependencies. If `pyproject.toml` (or `mypy.ini`) declares `plugins = ["pydantic.mypy"]`, mypy will fail with:

```
pyproject.toml:1:1: error: Error importing plugin "pydantic.mypy": No module named 'pydantic'  [misc]
```

Exit code is 2, which qlty reports as an Error.

**Fix:** Comment out the mypy plugin block with a note. The issue would require installing pydantic (and other project deps) into the qlty mypy venv, which is not currently possible via config alone.

*First seen: fastapi eval (2026-04-23)*

---

## `trivy` — `fs-vuln` IS a valid driver (contradicts earlier entry)

The plugin.toml at https://github.com/qltysh/qlty/blob/main/qlty-plugins/plugins/linters/trivy/plugin.toml defines three drivers: `fs-vuln`, `fs-secret`, and `config`. **`vuln` and `secret` are NOT valid driver names.** The earlier entry saying `"fs-vuln" is NOT valid` was incorrect — `fs-vuln` is the correct driver for lockfile vulnerability scanning. `qlty init` emitting `"fs-vuln"` is actually correct. The earlier guidance to strip it was wrong.

**Summary of valid trivy driver values:** `"config"`, `"fs-vuln"`, `"fs-secret"`

*Corrected 2026-04-23 (mux eval)*

---

## `editorconfig-checker` — not found in local qlty plugin registry (v0.610.0)

`editorconfig-checker` appears in the GitHub plugin registry (https://github.com/qltysh/qlty/tree/main/qlty-plugins/plugins/linters/editorconfig-checker) but causes an immediate `Plugin definition not found for editorconfig-checker` error when included in `qlty.toml` with qlty CLI v0.610.0. The plugin definition exists in source but is not available in this CLI version's bundled registry.

**Fix:** Comment out the `editorconfig-checker` block. Check again if the CLI is upgraded.

*First seen: mux eval (2026-04-23)*

---

## `rustfmt` — fails on Rust workspace crates with `mod` declarations

When qlty runs `rustfmt` on individual files from a Rust workspace, it fails with:

```
Error writing files: failed to resolve mod `<name>`: /path/to/tmp/crates/.../src/submodule.rs does not exist
```

This happens because `rustfmt` needs to resolve sibling module files (`mod pool;` etc.) relative to the file being formatted, but qlty runs it in a temp directory with only the target files copied. Projects with `mod` declarations in large workspaces (like ruff/astral-sh) will hit this on nearly every Rust file.

Additionally, `rustfmt.toml` options like `style_edition = "2024"` require a recent nightly/stable version; qlty's bundled rustfmt (1.77.2 as of v0.610.0) does not support them.

**Fix:** Comment out the `rustfmt` plugin for Rust workspaces. The plugin works for simple single-file crates but is unreliable for large multi-crate workspaces.

*First seen: ruff/laura-mlg eval (2026-04-23)*

---

## `eslint-plugin-react` — v7.33+ required for flat config (`.configs.flat` API)

The `eslint-plugin-react` package only added the flat config API (`.configs.flat.recommended`, `.configs.flat["jsx-runtime"]`) in v7.33.0. Using v7.31.x or earlier with a flat config file that calls `react.configs.flat.recommended` will fail with:

```
TypeError: Cannot read properties of undefined (reading 'recommended')
```

**Fix:** Use `eslint-plugin-react@7.37.5` (latest as of 2026-04-23) or at minimum `7.33.0` in `extra_packages` when the project's ESLint flat config uses the flat config API.

*First seen: ruff/laura-mlg eval (2026-04-23)*

---

## `eslint` — `@eslint/js` version must match ESLint 9.7.0

When adding `@eslint/js` to `extra_packages`, use the **9.x** line (e.g. `@eslint/js@9.7.0`). Using `@eslint/js@10.x` (the ESLint 10 version) causes a cloud build error despite passing `qlty check --sample` locally. The local pass is a false negative — npm resolves a cached fallback, but the cloud uses a fresh install environment that rejects the version mismatch.

**Fix:** `extra_packages = ["@eslint/js@9.7.0", ...]`

*First seen: axios eval (2026-04-29)*

---

## `eslint` — Always use CJS syntax in `.qlty/configs/eslint.config.js` (even for ESM projects)

Qlty copies config files to its cache as `.js` regardless of the source file extension. An `eslint.config.mjs` with ESM `import` syntax gets renamed to `eslint.config.js` in the cache and fails with:

```
SyntaxError: Cannot use import statement outside a module
```

**Rule:** Always write `eslint.config.js` (CJS format: `const x = require(...); module.exports = [...]`) in `.qlty/configs/`. This applies even if the project has `"type": "module"` in `package.json` — Qlty's ESLint runtime is a CJS process independent of the project's module type. Never use ESM `import`/`export` in the qlty eslint config file.

*Confirmed: axios eval (2026-04-29) — .mjs was stripped to .js and caused local failure after cache bust*
