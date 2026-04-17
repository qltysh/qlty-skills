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
