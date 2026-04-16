---
name: setup-qlty-toml
description: Configure and fine-tune qlty.toml for static analysis using the Qlty CLI and Qlty Cloud. Use this skill when a user wants to set up Qlty for the first time, improve an existing qlty.toml configuration, choose which linters and security scanners to enable, configure plugin modes (block/comment/monitor), set code smell thresholds, or tune exclude patterns. Handles single repos and monorepos. Supports all Qlty-supported languages: JavaScript, TypeScript, Python, Ruby, Go, Java, Kotlin, PHP, Rust, Swift, Shell, CSS, SQL, Terraform, Docker, and more. Do NOT use for code coverage setup (use setup-coverage instead), writing or running tests, or general CI/CD changes unrelated to Qlty analysis.
---

# Set Up qlty.toml

You are a senior software engineer configuring Qlty static analysis for this repository. Your goal is to produce a well-tuned `qlty.toml` that reflects the actual code in the repo and matches the team's preferences. Work through these phases in order.

---

## Phase 1: Analyze the Repository

### 1a. Detect current Qlty state

Check whether `.qlty/qlty.toml` already exists:

- **Does not exist**: Run `qlty init` to generate the baseline configuration, then proceed. Note: `qlty init` auto-detects file types and enables a starter set of plugins.
- **Already exists**: Read the current config carefully. Note which plugins are enabled, their modes, existing exclude patterns, and any smells/triage blocks. You will be refining this, not replacing it from scratch.

### 1b. Catalog the repository

Gather the following. Do NOT make changes yet — just collect facts.

**Languages and file types:**
Scan the repo for all significant file extensions and note:
- Primary source language(s)
- Whether there are Dockerfiles, Terraform `.tf` files, YAML, SQL, OpenAPI specs
- Whether GitHub Actions workflows exist (`.github/workflows/`)
- Is this a monorepo? (Multiple `package.json`, `Cargo.toml`, `go.mod`, `Gemfile`, `pom.xml`, `build.gradle`, `pyproject.toml`, or similar manifest files in different subdirectories signal a monorepo)

**Existing linter/formatter configs:**
Search the repo for plugin-specific config files. These tell you which tools the team already uses AND whether `extra_packages`, `config_files`, or `package_file` entries are needed in the plugin block.

| Config file(s) | Plugin | Notes |
|---|---|---|
| `eslint.config.js`, `eslint.config.mjs`, `eslint.config.cjs`, `.eslintrc`, `.eslintrc.js`, `.eslintrc.json`, `.eslintrc.yml`, `.eslintrc.yaml` | eslint | Read it — detect any plugins listed (e.g. `@typescript-eslint`, `eslint-plugin-react`, `eslint-plugin-import`, `eslint-plugin-vue`) |
| `package.json` with `"eslintConfig"` key | eslint | Use `package_file = "package.json"` |
| `.prettierrc`, `.prettierrc.json`, `.prettierrc.json5`, `.prettierrc.yaml`, `.prettierrc.yml`, `.prettierrc.toml`, `.prettierrc.js`, `.prettierrc.cjs`, `prettier.config.js`, `prettier.config.cjs` | prettier | Read for parser plugins (e.g. `prettier-plugin-tailwindcss`, `prettier-plugin-svelte`) |
| `biome.json`, `biome.jsonc` | biome | **EXCLUSIVE:** If `biome.json` or `biome.jsonc` exists, use ONLY biome — do NOT add eslint or prettier |
| `.stylelintrc`, `.stylelintrc.json`, `.stylelintrc.yaml`, `.stylelintrc.yml`, `.stylelintrc.js`, `.stylelintrc.cjs`, `.stylelintrc.mjs`, `stylelint.config.js`, `stylelint.config.cjs`, `stylelint.config.mjs` | stylelint | Read for extends (e.g. `stylelint-config-standard-scss`) — those become `extra_packages` |
| `knip.json`, `knip.jsonc`, `.knip.json`, `.knip.jsonc`, `knip.ts`, `knip.js`, `knip.config.js` | knip | Detects unused exports/dependencies in JS/TS |
| `tsconfig.json` | tsc | TypeScript type-checking — only enable if the project compiles with `tsc` |
| `oxlintrc.json` | oxc | Fast JS/TS linter, may coexist with eslint |
| `.rubocop.yml`, `.rubocop_*.yml`, `.rubocop-*.yml` | rubocop | Ruby linter/formatter |
| `.standard.yml` | standardrb | Ruby style — mutually exclusive with rubocop |
| `.reek.yml` | reek | Ruby code smell detection |
| `.haml-lint.yml` | haml-lint | Ruby HAML templates |
| `ruff.toml`, `pyproject.toml` (check `[tool.ruff]`) | ruff | Python linter+formatter — replaces flake8+black for most projects |
| `.flake8`, `setup.cfg` (check `[flake8]`) | flake8 | Python linter — don't enable both ruff and flake8 |
| `pyproject.toml` (check `[tool.black]`), `.black` | black | Python formatter — skip if ruff handles formatting |
| `mypy.ini`, `.mypy.ini`, `pyproject.toml` (check `[tool.mypy]`) | mypy | Python type checker |
| `.bandit`, `pyproject.toml` (check `[tool.bandit]`) | bandit | Python security scanner |
| `phpcs.xml`, `.phpcs.xml` | php-codesniffer | PHP style — check for custom sniffs in config |
| `.php-cs-fixer.dist.php`, `.php-cs-fixer.php` | php-cs-fixer | PHP formatter — mutually exclusive with php-codesniffer for formatting |
| `phpstan.neon`, `phpstan.neon.dist`, `phpstan.dist.neon` | phpstan | PHP static analysis |
| `.golangci.yml`, `.golangci.yaml`, `.golangci.json`, `.golangci.toml` | golangci-lint | Go meta-linter — replaces standalone gofmt for many teams |
| (no config needed) | gofmt | Go formatter — use golangci-lint instead if `.golangci.*` exists |
| `.clippy.toml`, `clippy.toml` | clippy | Rust linter |
| `rustfmt.toml`, `.rustfmt.toml` | rustfmt | Rust formatter |
| `.swiftlint.yml`, `.swiftlint.yaml`, `.swiftlint` | swiftlint | Swift linter |
| `.swiftformat` | swiftformat | Swift formatter |
| `.hadolint.yaml`, `.hadolint.yml` | hadolint | Dockerfile linter |
| `.shellcheckrc`, `shellcheckrc` | shellcheck | Shell script linter |
| (no config needed) | shfmt | Shell script formatter |
| `.markdownlint.json`, `.markdownlint.yml`, `.markdownlint.yaml` | markdownlint | Markdown linter |
| `.yamllint`, `.yamllint.yml`, `.yamllint.yaml` | yamllint | YAML linter |
| `.github/actionlint.yaml`, `.github/actionlint.yml` | actionlint | GitHub Actions workflow linter |
| `.tflint.hcl` | tflint | Terraform linter |
| (no config needed) | terraform | Terraform formatter |
| `checkov.yml`, `.checkov.yml`, `.checkov.yaml` | checkov | IaC security scanner (Terraform, Docker, K8s, CloudFormation) |
| `trivy.yaml`, `trivy-secret.yaml` | trivy | Container/dependency security scanner |
| `osv-scanner.toml` | osv-scanner | Open Source Vulnerability scanner |
| `.gitleaks.toml`, `.gitleaks.config` | gitleaks | Git history secrets scanner — alternative to trufflehog |
| `.semgrep.yaml`, `.semgrepignore`, `.semgrep` | semgrep | Multi-language semantic code scanner |
| `sqlfluff.cfg`, `.sqlfluff` | sqlfluff | SQL linter |
| `.spectral.yml`, `.spectral.yaml`, `.spectral.json`, `.spectral.js` | spectral | OpenAPI/AsyncAPI spec linter |
| `.vale.ini` | vale | Prose/documentation linter |
| `sgconfig.yml` | ast-grep | Structural code search/lint (language-agnostic) |
| `.kube-linter.yaml`, `.kube-linter.yml` | kube-linter | Kubernetes manifest linter |
| `zizmor.yml`, `.github/workflows/zizmor.yml` | zizmor | GitHub Actions security scanner (more thorough than actionlint for security). **Note: zizmor may not appear on docs.qlty.sh/plugins — it IS a supported Qlty plugin.** |
| `coffeelint.json` | coffeelint | CoffeeScript linter |
| `.editorconfig` | editorconfig-checker | Checks files conform to .editorconfig rules |
| `brakeman.ignore` | brakeman | Ruby on Rails security scanner |
| `.pmd/` or `pmd.xml` | pmd | Java/Kotlin static analysis — if using pmd standalone |
| `radarlint.properties` | radarlint-{java,kotlin,js,python,ruby,php,go} | Deep code quality — see `references/plugin-registry.md` before adding |
| (no config needed) | google-java-format | Java formatter |
| (no config needed) | ktlint | Kotlin formatter/linter |
| (no config needed) | prisma | Prisma schema formatter |
| (no config needed) | dotenv-linter | `.env` file linter |
| `.ripgreprc` | ripgrep | Text pattern search (custom rules via ripgrep) |

**For npm-based plugins** (`eslint`, `prettier`, `stylelint`, `markdownlint`, `biome`, `knip`, `oxc`, `tsc`, `coffeelint`, `spectral`): read the `package.json` devDependencies carefully. Any tool-specific packages (plugins, configs, parsers, extends) need to be reflected in `extra_packages` or `package_file` in the plugin block.

**For Python-based plugins** (`ruff`, `flake8`, `bandit`, `mypy`, `sqlfluff`, `semgrep`, `yamllint`, `checkov`, `zizmor`): check `pyproject.toml`, `requirements.txt`, or `requirements-dev.txt` for pinned versions.

**Generated and vendored directories:**
Look for directories likely to contain generated, vendored, or third-party code:
- `vendor/`, `node_modules/`, `dist/`, `build/`, `target/`, `generated/`, `protos/`, `.yarn/`
- Any directories not committed as source (check `.gitignore`)
- Minified files (`*.min.js`, `*-min.css`, `*.min.*`)
- TypeScript declaration files (`**/*.d.ts`)

**CI configuration:**
Check `.github/workflows/` or other CI config. Note whether `qlty check` is already wired into CI.

Produce a brief summary of findings, then move on to Phase 2.

---

## Phase 2: Fetch Documentation

Before making decisions, fetch what you need from these sources:

**Plugin availability — always check this first:**
- https://github.com/qltysh/qlty/tree/main/qlty-plugins/plugins — authoritative list of available plugins; each subdirectory name is a valid plugin name

**Config reference** (useful but may not be complete or fully up to date):
- https://docs.qlty.sh/qlty-toml — TOML field reference
- https://docs.qlty.sh/analysis-configuration — maintainability checks and thresholds
- https://docs.qlty.sh/excluding-files — exclude patterns and per-plugin exclusions
- https://docs.qlty.sh/silencing-issues — `qlty-ignore` inline comment syntax
- https://docs.qlty.sh/cli/linter-extensions — `extra_packages` vs `package_file` vs `package_filters`

**For deeper config questions:** If the docs don't cover a specific plugin's options (valid fields, driver values, version constraints), check the plugin's source directly at https://github.com/qltysh/qlty — the source is always more complete than the docs.

**Accumulated learnings:** Read `references/plugin-registry.md` for known cases and caveats from past runs — specific situations that aren't obvious from the docs or registry alone.

---

## Phase 3: Interactive Configuration Decisions

Work through each decision below. For each one: **briefly explain what it does, state your recommendation with reasoning, and ask the user to confirm or change it before moving on.** Group related decisions in one message to reduce back-and-forth.

**If running autonomously** (e.g., in an eval or non-interactive context): apply all recommended defaults from each decision below and proceed without asking for confirmation.

### Decision 1: Plugin Selection

Based on detected languages and existing config files, propose a full plugin list organized by category. Present as a table:

| Plugin | Category | Why recommended | Enabled? |
|---|---|---|---|
| trufflehog | Security | Secrets detection — all repos benefit | Yes |
| eslint | Linter | TypeScript/JavaScript found | Yes |
| ... | ... | ... | ... |

**How to select plugins:**

1. **Only propose plugins you confirmed exist** in the GitHub registry fetched in Phase 2. If a plugin name doesn't appear as a subdirectory there, don't add it — check `references/plugin-registry.md` for any known caveats on why.

2. **Mutual-exclusion rules:**
   - **biome** replaces eslint and prettier — if `biome.json`/`biome.jsonc` present, do NOT also add eslint or prettier.
   - **standardrb** replaces rubocop — if `.standard.yml` present, use standardrb, not rubocop.
   - **golangci-lint** replaces gofmt — if `.golangci.*` present, do NOT also add gofmt.
   - **oxc** coexists with eslint — unlike biome, both can be enabled together.

3. **Security baseline (always recommend):** `trufflehog` for secrets. `osv-scanner` or `trivy` if lockfiles present. `zizmor` if GitHub Actions workflows present.

4. **Cross-language tools:** Recommend based on what's in the repo — `markdownlint` (`.md` files), `actionlint` + `yamllint` (`.github/workflows/`), `checkov` (Docker/Terraform/K8s), `kube-linter` (K8s manifests), `spectral` (OpenAPI specs), `vale` (prose docs).

Ask the user to confirm the list or add/remove plugins before proceeding.

### Decision 2: Plugin Modes

Explain the four modes once, clearly:

- **`block`**: Fails the Qlty Quality Gate — the PR cannot be merged until issues are resolved. Use for issues you always want fixed.
- **`comment`**: Posts a code review comment on the PR, but does not block merging. Use for important feedback without hard enforcement.
- **`monitor`**: Issues are visible on Qlty Cloud but not posted to GitHub at all. Use for noisy tools while you build trust in their signal.
- **`disabled`**: Plugin does not run.

Present your mode recommendations in a table using these defaults:

| Plugin(s) | Default mode | Reasoning |
|---|---|---|
| `trufflehog`, `gitleaks` | `block` | Secrets are always urgent and high-signal — block immediately |
| `zizmor` | `block` | GitHub Actions security issues are high value and low noise |
| `eslint`, `rubocop`, `ruff`, `flake8`, `mypy`, `phpstan`, `golangci-lint`, `clippy`, `swiftlint`, `shellcheck`, `sqlfluff`, `spectral`, `knip`, `oxc`, `tsc`, `haml-lint`, `reek`, `brakeman`, `coffeelint`, `vale`, `ast-grep` | `comment` | Important linting feedback, non-blocking by default |
| `prettier`, `black`, `gofmt`, `rustfmt`, `shfmt`, `ktlint`, `google-java-format`, `swiftformat`, `ruby-stree`, `standardrb`, `php-cs-fixer`, `dockerfmt`, `dotenv-linter`, `prisma`, `terraform` (format driver) | `comment` | Formatting is style, not correctness — visible but non-blocking |
| `markdownlint`, `yamllint`, `actionlint`, `hadolint`, `editorconfig-checker` | `comment` | Config/docs quality is useful but not blocking |
| `osv-scanner`, `trivy`, `checkov`, `semgrep`, `bandit` | `block` | Security/vulnerability scanners — high-value, low-noise; block by default |
| `tflint` | `comment` | Terraform linter — important feedback, non-blocking |
| `pmd`, `php-codesniffer`, `radarlint-*`, `kube-linter`, `ripgrep` | `monitor` | Can be noisy on first run — observe before enforcing |

Ask: "Do any of these feel too strict or too lenient for your team? If you're starting fresh and want to ease in gradually, we can move security scanners to `monitor`. If you want stricter enforcement, we can promote linters to `block`."

### Decision 3: Extra Packages and Config Files

For plugins that require external packages or have config files the team has already written, propose the right `extra_packages`, `config_files`, and `package_file` entries.

**npm-based plugins** — check devDependencies in `package.json` and the plugin config:

| Plugin | Commonly needed `extra_packages` |
|---|---|
| eslint | `@typescript-eslint/parser`, `@typescript-eslint/eslint-plugin`, `eslint-plugin-react`, `eslint-plugin-react-hooks`, `eslint-plugin-import`, `eslint-plugin-jsx-a11y`, `eslint-plugin-vue`, `eslint-config-prettier`, `@next/eslint-plugin-next`, etc. |
| stylelint | `stylelint-config-standard`, `stylelint-config-standard-scss`, `stylelint-config-sass-guidelines`, `stylelint-config-recommended`, `stylelint-order`, etc. |
| prettier | `prettier-plugin-tailwindcss`, `prettier-plugin-svelte`, `prettier-plugin-organize-imports`, etc. |
| markdownlint | `markdownlint-rule-*` (custom rules) |
| spectral | `@stoplight/spectral-formats`, `@stoplight/spectral-rulesets`, etc. |

For each npm-based plugin, read the existing config file to identify all `extends`, `plugins`, and `parser` entries. Map each one to the corresponding npm package and version. Propose the `extra_packages` list.

**Version pinning:** Use versions already pinned in `package.json` devDependencies wherever possible. Fall back to the latest stable version only if the package is not already in devDependencies.

**`@typescript-eslint` version cap:** The scoped packages `@typescript-eslint/parser` and `@typescript-eslint/eslint-plugin` only exist up to v6.x. In v7+, the project was rebranded to the unscoped `typescript-eslint` package. Always pin to v5.x or v6.x for the scoped packages (e.g., `@typescript-eslint/parser@5.62.0` or `6.x`). Do NOT use v7+ for the `@typescript-eslint/` scoped packages.

**IMPORTANT: Always use `config_files = ["path"]` when referencing a plugin's config file. Never use `config = "path"` — that is not a valid TOML field.**

**Config file references** — if a standalone config file exists, add `config_files`:
```toml
[[plugin]]
name = "eslint"
config_files = ["eslint.config.js"]
extra_packages = ["@typescript-eslint/eslint-plugin@8.0.0"]

[[plugin]]
name = "stylelint"
config_files = [".stylelintrc.json"]
extra_packages = ["stylelint-config-standard-scss@13.0.0"]
```

If the config is embedded in `package.json` (e.g. eslint under `"eslintConfig"`), use `package_file = "package.json"` instead of `config_files`.

**`package_file` + `package_filters`** — When the project's `package.json` (or `Gemfile`, `composer.json`) already declares all needed linter plugins, prefer referencing it over duplicating versions in `extra_packages`. Use `package_filters` to limit which packages get installed from that file:

```toml
[[plugin]]
name = "eslint"
package_file = "package.json"
package_filters = ["eslint"]
```

**Warning: `package_filters` is a prefix filter and does NOT match scoped npm packages.** A filter of `"eslint"` will NOT match `@typescript-eslint/parser` or `@typescript-eslint/eslint-plugin` because those package names start with `@`, not `eslint`. If your eslint config requires scoped packages (`@typescript-eslint/*`, `@babel/*`, etc.), use `extra_packages` with explicit versions instead of `package_file` + `package_filters`.

Choose `package_file` when: the project's package manager file already lists all needed plugins and you want versions to stay in sync, AND all needed packages are unscoped (no `@scope/` prefix). Choose `extra_packages` when: the project has no package file, you need specific versions independent of the project, or any required packages are scoped npm packages.

**Note on lock files:** With `package_file`, Qlty respects locked versions. With `package_file` + `package_filters`, lock files are ignored. With `extra_packages`, lock files are not used at all.

**`version`** — Pin the tool version (the linter itself, not its extensions). Use when a specific tool version is needed for compatibility with the project's config:

```toml
[[plugin]]
name = "golangci-lint"
version = "1.55.2"
```

**`affects_cache`** — Additional files whose changes should invalidate the plugin's analysis cache. Use when a plugin reads config from files not automatically detected:

```toml
[[plugin]]
name = "eslint"
affects_cache = [".eslintignore", "tsconfig.json"]
```

**`skip_upstream`** — Set to `true` to skip analysis of files outside the current diff (upstream dependencies). Useful for TypeScript projects where `tsc` would otherwise type-check all imported files:

```toml
[[plugin]]
name = "tsc"
skip_upstream = true
```

**Warning on `tsc` plugin:** Only enable `tsc` if the project already passes `tsc --noEmit` in CI. When in doubt, skip `tsc` and rely on `eslint` with `@typescript-eslint` instead. See `references/plugin-registry.md` for details. Also: `xo`-based projects (those with `"xo"` in devDependencies) use xo as a CLI linter wrapping eslint — do NOT add the eslint plugin for xo projects, as xo bundles its own eslint internals.

**`.qlty/configs/` directory** — If you want to store a plugin's config file inside the `.qlty/` directory (to keep it out of the repo root), Qlty will automatically provision it during analysis. Reference it with a relative path in `config_files`:

```toml
[[plugin]]
name = "rubocop"
config_files = [".qlty/configs/.rubocop.yml"]
```

**Warning for ESM projects:** If the project has `"type": "module"` in `package.json`, a `.eslintrc.js` stored in `.qlty/configs/` will fail because Node treats `.js` as ESM but the config uses `module.exports` (CommonJS). Use `.eslintrc.cjs` (explicit CJS extension) instead: `config_files = [".qlty/configs/.eslintrc.cjs"]`.

Show each proposed entry and explain why. Ask the user to confirm or adjust package versions.

### Decision 4: Exclude Patterns

Explain: "Exclude patterns tell Qlty to completely skip certain files and directories — no plugin will analyze them. This is useful for generated code, vendored dependencies, minified assets, and test fixtures with intentional violations."

Show the current `exclude_patterns` (from `qlty init` or the existing config).

Based on Phase 1 findings, propose additions for directories not already covered:

**Common candidates:**
- `**/generated/**` — protobuf or other code-gen output
- `**/protos/**` — protobuf definitions
- `**/fixtures/**` — test fixtures with intentional code violations
- `*_min.*`, `*-min.*`, `*.min.*` — minified files
- `**/*.d.ts` — TypeScript declaration files
- `**/migrations/**` — database migration files (often intentionally non-idiomatic)
- `**/vendor/**` — vendored third-party code (Go, PHP, etc.)
- `**/bower_components/**` — legacy JS dependencies

**Warning:** Avoid broad excludes like `**/config/**` or `**/templates/**` — these names are commonly used for actual source code in framework/library repos. Only add them if you have confirmed the directory contains generated or third-party code.

For **per-plugin exclusions** (suppress one plugin on specific paths without excluding those paths from all plugins), use `[[exclude]]` blocks:
```toml
[[exclude]]
plugins = ["osv-scanner", "trufflehog"]
file_patterns = ["**/test/fixtures/**"]
```

Ask: "Are there directories with generated code, third-party code you don't own, or files with intentional violations that should be excluded?"

### Decision 5: Test Patterns

Show the current `test_patterns`. These help Qlty Cloud distinguish test code from production code for more accurate quality metrics.

Defaults cover most frameworks (`**/test/**`, `**/spec/**`, `**/*.test.*`, `**/*.spec.*`, etc.).

Ask if they need additions for custom test directories (e.g., `**/e2e/**`, `**/integration_tests/**`, `**/features/**` for Cucumber).

### Decision 6: Code Smells and Complexity Thresholds

Explain: "Qlty has built-in maintainability analysis — separate from any linter plugin. It flags things like overly complex functions, too many parameters, deeply nested code, and duplicate logic. These are called 'smells'."

Present the configurable smells:

| Smell | What it flags | Default threshold |
|---|---|---|
| `function_complexity` | Cyclomatic complexity of a function | 15 |
| `function_parameters` | Number of parameters in a function | 4 |
| `return_statements` | Number of return statements in a function | 4 |
| `file_length` | Lines in a file | 250 |
| `nested_control_flow` | Nesting depth of if/for/while blocks | 4 |
| `identical_code` | Copy-pasted identical code blocks | enabled |
| `similar_code` | Structurally similar code blocks | enabled |

**Disabling individual smells** — to turn off a specific check without disabling smells entirely:

```toml
[smells.identical_code]
enabled = false
```

**Duplication fine-tuning** — if the default duplication sensitivity is too aggressive:

```toml
[smells.duplication]
nodes_threshold = 100      # higher = less sensitive (default ~45 nodes)
filter_patterns = ["**/migrations/**", "**/fixtures/**"]  # skip these paths for duplication only
```

**Note:** `min_duplication_lines` is NOT a valid field — use `nodes_threshold`. A "node" in the AST roughly corresponds to 1–3 lines of code, so `nodes_threshold = 100` is roughly equivalent to ~50–100 lines.

**Per-language overrides** — to set different thresholds for a specific language without affecting others:

```toml
[language.python.smells]
function_parameters.threshold = 8    # data science functions often take more params

[language.java.smells]
function_complexity.threshold = 20   # enterprise Java often has higher complexity

[language.go.smells]
function_parameters.threshold = 6    # Go error-handling patterns use more params
```

Ask:
1. Do you want to enable smells? (Recommended: yes, `mode = "comment"`)
2. Do any thresholds need adjustment for your codebase? (e.g., lower `function_parameters` to 3 for strict teams, or raise `function_complexity` to 20 for data-heavy code)
3. Are there specific smells to disable entirely? (e.g., `identical_code` for repos with intentional duplication like migrations or fixtures)
4. Do you want per-language overrides? (e.g., Python data science, Java enterprise, Go error-handling patterns all commonly need higher thresholds)

### Decision 7: Monorepo Configuration (skip if not a monorepo)

If the repo has distinct services or packages in subdirectories with different language stacks:

**Same-language workspaces** (e.g., a Cargo workspace with multiple Rust crates, a Lerna monorepo with all-JS packages): do NOT scope by `prefix` — a single config covers all packages. Use `prefix` only when different subdirectories use different languages or need different plugin configs.

Explain: "The `prefix` field on a plugin scopes it to run only within a specific subdirectory. This is useful when different parts of the repo use different languages or have separate linter configs."

```toml
[[plugin]]
name = "eslint"
prefix = "frontend"
config_files = ["frontend/eslint.config.js"]

[[plugin]]
name = "ruff"
prefix = "backend"
```

Also explain `[[exclude]]` for suppressing a plugin on specific paths:
```toml
[[exclude]]
plugins = ["tsc", "knip"]
file_patterns = ["backend/**", "infra/**"]
```

Ask: "Should any plugins be scoped to specific subdirectories? Do you want to exclude any plugins from running in specific parts of the repo?"

### Decision 8: Triage Rules (advanced — skip if not needed)

`[[triage]]` blocks let you modify how specific issues are reported without changing the entire plugin's mode. They are evaluated in order — first match wins. Use them when:

- A specific rule is too noisy in general, but you don't want to move the whole plugin to `monitor`
- You want to promote a specific finding from `comment` → `block` without promoting the whole plugin
- A rule should be silenced only in certain file paths (e.g., ignore `no-console` in test files)
- You want to recategorize findings (change `level` from warning → error, or adjust `category`)

```toml
# Downgrade a noisy rubocop style rule to monitor-only
[[triage]]
plugins = ["rubocop"]
rules = ["Style/StringLiterals"]
set.mode = "monitor"

# Promote a high-severity semgrep finding to block
[[triage]]
plugins = ["semgrep"]
rules = ["python.django.security.injection.sql"]
set.mode = "block"
set.level = "error"

# Ignore no-console violations in test files only
[[triage]]
plugins = ["eslint"]
rules = ["no-console"]
file_patterns = ["**/*.test.*", "**/*.spec.*"]
set.ignored = true

# Downgrade all pmd findings (noisy for first-time Java setup)
[[triage]]
plugins = ["pmd"]
set.mode = "monitor"
```

**Match conditions** (all are optional, combined as AND):
- `plugins = ["name"]` — match by plugin
- `rules = ["rule-id"]` — match by rule identifier
- `levels = ["error"]` — match by current issue level
- `file_patterns = ["glob"]` — match by file path

**Set values** (what to change when matched):
- `set.mode` — override to `"block"`, `"comment"`, `"monitor"`, or `"disabled"` ← **prefer this for suppression**
- `set.level` — override to `"error"`, `"warning"`, or `"note"`
- `set.ignored = true` — suppress entirely (use `set.mode = "disabled"` as a safer alternative; `set.ignored` behavior may not be consistent across Qlty versions)
- `set.category` — recategorize the finding (**use with caution** — may cause "Build errored" in Qlty Cloud; avoid until confirmed supported)

Ask: "Are there specific rules from any plugin that are too noisy, need to be promoted to blocking, or should be silenced in certain paths? If so, we can add `[[triage]]` blocks to handle them without touching the plugin's overall mode."

---

## Phase 4: Implement and Open PR

Once all decisions are confirmed:

### 1. Write `.qlty/qlty.toml`

**CRITICAL — AVOID THESE COMMON FORMAT ERRORS (all cause "Build errored"):**

- **DO NOT include `[cli]` or `[settings]` sections.** If you ran `qlty init`, its output contains these internal CLI metadata sections — they are NOT valid qlty.toml fields. Strip them out.
- **DO NOT use the old `[[sources.community]]` format** with `repository =` and `tag =` fields — that is a deprecated source format. Use `[[source]] name = "default" default = true` (see template below).
- **DO NOT use the old `[plugins.enabled]` / `[[plugins.enabled]]` format** with a separate `[plugins.releases]` version table — that is the deprecated qlty.toml v0 schema. Always use `[[plugin]]` blocks.
- **DO NOT nest `exclude_patterns` as a section** (`[exclude_patterns] patterns = [...]`). It must be a top-level array: `exclude_patterns = ["..."]`.
- **The `[[source]]` block is required.** Without it, Qlty cannot resolve plugin definitions and the build will error. Use exactly: `[[source]]` / `name = "default"` / `default = true`. Do NOT set `directory` to a URL — `directory` is for local filesystem paths only.

Apply all confirmed decisions. Structure the file in this order:

```toml
# This configuration was generated with the setup-qlty-toml skill.
# For reference: https://docs.qlty.sh/qlty-toml
config_version = "0"

exclude_patterns = [
  # ... confirmed patterns
]

test_patterns = [
  # ... confirmed patterns
]

[smells]
mode = "comment"

# Per-smell overrides (only if non-default)
[smells.function_parameters]
threshold = X

# Per-language overrides (only if requested)
[language.python.smells]
function_parameters.threshold = X

# REQUIRED — points to the built-in Qlty plugin registry. Do not omit this block.
[[source]]
name = "default"
default = true

# Language linters
[[plugin]]
name = "..."

# Formatters
[[plugin]]
name = "..."

# Security scanners
[[plugin]]
name = "..."

# Cross-language tools
[[plugin]]
name = "..."

# Per-plugin exclusions (if any)
[[exclude]]
plugins = [...]
file_patterns = [...]
```

Only include sections with non-default values or explicit user decisions. Include `[[triage]]` blocks only when Decision 8 identified specific rules to adjust. Don't add empty blocks.

### 2. Create a branch and open a PR

- Branch: `qlty-setup` (or `qlty-config` if that already exists)
- Commit: `Add .qlty/qlty.toml configuration`
- PR description should include:
  - Every plugin enabled and its mode
  - Any `extra_packages` added and why
  - Any manual steps the user needs to take

### 3. Update `references/plugin-registry.md` if you learned something new

If during this run you discovered that a plugin is missing from the public registry, causes Build errored, or has a caveat not already recorded, update `references/plugin-registry.md` in this skill's repo directly — do not wait for a separate evolve run. Add a row to the appropriate table with a `Last confirmed` date. This keeps the reference useful for the next agent.

### 4. Print a configuration rundown

After the PR is open, print a clean summary:

```
## Qlty configuration summary

Plugins enabled: 9

  BLOCK   trufflehog      — secrets detection (all files)
  BLOCK   zizmor          — GitHub Actions security
  COMMENT eslint          — JS/TS linting (extras: @typescript-eslint/eslint-plugin, eslint-plugin-react)
  COMMENT tsc             — TypeScript type checking
  COMMENT prettier        — formatting (JS/TS/CSS/Markdown)
  BLOCK   osv-scanner     — dependency vulnerability scanning
  BLOCK   checkov         — infrastructure security (IaC files)
  COMMENT markdownlint    — markdown quality
  COMMENT yamllint        — YAML correctness
  COMMENT actionlint      — GitHub Actions syntax

Smells: enabled (comment mode), function_parameters threshold = 5

Exclude patterns: 7 (node_modules, dist, generated, *.min.*, *.d.ts, fixtures, .yarn)
```

For any plugin with a non-default mode or config, add a one-line explanation:
- "eslint is set to `comment` — it will post review comments but won't block PRs until the team is comfortable with the rules."
- "trufflehog is set to `block` — any detected secret will fail the build."

---

## Phase 5: Qlty Cloud Guidance

After the PR is ready, tell the user what to verify in Qlty Cloud (this cannot be done via `qlty.toml` — it requires the web UI). Substitute `<org>` and `<repo>` from the actual GitHub repo:

**1. Verify the project is on Qlty Cloud**
Visit `https://qlty.sh/gh/<org>` and confirm the repo is listed. If not, the user may need to install the Qlty GitHub App or add the project manually.

**2. Quality Gate settings**
Visit `https://qlty.sh/gh/<org>/projects/<repo>/settings/review` to:
- Confirm which check types are set to fail the Quality Gate (should match their `block`-mode plugins)
- Enable or disable the Qlty bot as a required PR check

**3. After merging**
After the PR merges to the default branch, Qlty Cloud will run its first full analysis. Visit `https://qlty.sh/gh/<org>/projects/<repo>` to see results.

**4. Suggested follow-up**
- Review `monitor`-mode findings on Qlty Cloud. Once you've triaged the noise, promote any useful plugins to `comment` or `block`.
- If `qlty check` is not already in CI, add it as a step for pre-merge analysis (runs the same checks locally in CI).
- Consider setting up the Qlty quality badge for the README.

**5. Inline issue silencing with `qlty-ignore`**

When a specific line or block legitimately triggers a rule (e.g., an intentional pattern, generated code inline, or a known-safe exception), use `qlty-ignore` inline comments rather than widening `exclude_patterns` or adding `[[triage]]` rules. This is a per-occurrence surgical suppression — it does not affect other instances of the same rule.

Single rule, next line:
```python
# qlty-ignore: ruff:E501
long_generated_string = "..."
```

Multiple rules or tools, comma-separated:
```typescript
// qlty-ignore: eslint:no-console, eslint:no-debugger
console.log("intentional");
```

Ignore all rules from a tool on the next line:
```go
// qlty-ignore: golangci-lint
someWeirdButCorrectPattern()
```

Next-line-only (does not apply to indented block beneath it):
```python
# qlty-ignore>: bandit:B101
assert condition  # only this line
```

Region silencing (start `+`, end `-`):
```python
# qlty-ignore+: bandit
eval(trusted_config_string)
process(trusted_config_string)
# qlty-ignore-: bandit
```

Tell the user: "If you encounter specific false positives after the PR merges, use `qlty-ignore` comments rather than broadening the global config — it keeps suppressions visible and local to the affected code."
