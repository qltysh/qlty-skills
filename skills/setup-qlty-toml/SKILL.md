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
| `biome.json`, `biome.jsonc` | biome | Replaces eslint+prettier for some TS/JS projects — don't enable both biome and eslint/prettier |
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
| `zizmor.yml` | zizmor | GitHub Actions security scanner (more thorough than actionlint for security) |
| `coffeelint.json` | coffeelint | CoffeeScript linter |
| `.editorconfig` | editorconfig-checker | Checks files conform to .editorconfig rules |
| `brakeman.ignore` | brakeman | Ruby on Rails security scanner |
| `.pmd/` or `pmd.xml` | pmd | Java/Kotlin static analysis — if using pmd standalone |
| `radarlint.properties` | radarlint-{java,kotlin,js,python,ruby,php,go} | Deep code quality for respective language |
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

Before making decisions, fetch these reference pages so your recommendations are grounded in current docs:

- https://docs.qlty.sh/qlty-toml
- https://docs.qlty.sh/analysis-configuration
- https://docs.qlty.sh/plugins

IMPORTANT: Only access URLs from the https://docs.qlty.sh domain.

---

## Phase 3: Interactive Configuration Decisions

Work through each decision below. For each one: **briefly explain what it does, state your recommendation with reasoning, and ask the user to confirm or change it before moving on.** Group related decisions in one message to reduce back-and-forth.

### Decision 1: Plugin Selection

Based on detected languages and existing config files, propose a full plugin list organized by category. Present as a table:

| Plugin | Category | Why recommended | Enabled? |
|---|---|---|---|
| trufflehog | Security | Secrets detection — all repos benefit | Yes |
| eslint | Linter | TypeScript/JavaScript found | Yes |
| ... | ... | ... | ... |

**Selection rules:**

- **Security baseline (always recommend):** `trufflehog` for secrets. Add `gitleaks` only if the team specifically wants git-history scanning (it's slower). Add `osv-scanner` or `trivy` if dependency lockfiles are present.
- **Language linters:** Pick the best tool for each detected language. Don't double-up competing tools:
  - Python: prefer `ruff` (covers flake8+isort+pyupgrade). Add `mypy` for type-checked projects, `bandit` for security-focused ones.
  - JS/TS: prefer `eslint`. Add `tsc` if `tsconfig.json` exists and the project uses TypeScript compilation. Consider `biome` only if the project already uses it (don't introduce it alongside eslint).
  - Ruby: prefer `rubocop`. Add `reek` for smell detection, `brakeman` for Rails apps, `haml-lint` if HAML files exist.
  - Go: prefer `golangci-lint` if `.golangci.*` config exists (it wraps gofmt and more). Otherwise use `gofmt`.
  - Java: `google-java-format` for formatting, `pmd` for static analysis. Add `radarlint-java` for deep quality analysis.
  - Kotlin: `ktlint` for formatting/linting. Add `radarlint-kotlin` for deep quality analysis.
  - PHP: `phpstan` for static analysis, `php-codesniffer` for style. Use `php-cs-fixer` instead of php-codesniffer if `.php-cs-fixer.*` exists.
  - Rust: `clippy` for linting, `rustfmt` for formatting.
  - Swift: `swiftlint` for linting, `swiftformat` for formatting.
  - Shell: `shellcheck` for linting, `shfmt` for formatting.
  - CSS/Sass: `stylelint`.
  - SQL: `sqlfluff`.
  - Terraform: `terraform` (format) + `tflint` (linting) + `checkov` (security).
- **Cross-language tools (recommend unless repo is too small/simple):**
  - `markdownlint` — if any `.md` files exist
  - `actionlint` — if `.github/workflows/` exists
  - `yamllint` — if any `.yml`/`.yaml` files exist
  - `prettier` — if JS/TS/CSS/markdown/HTML present (skip if biome is used)
  - `checkov` — if Docker, Terraform, Kubernetes, or CloudFormation files exist
  - `semgrep` — optional, for teams that want semantic code scanning
  - `zizmor` — if GitHub Actions workflows exist and security is a priority (more thorough than actionlint for security)
  - `kube-linter` — if Kubernetes manifests exist
  - `spectral` — if OpenAPI/AsyncAPI specs exist
  - `vale` — if the repo has substantial documentation prose

**Trivy `drivers`:** By default trivy scans containers and lockfiles. Add `drivers = ["config"]` to also scan IaC (Terraform, Docker, K8s). Add `drivers = ["secret"]` for secret scanning (though trufflehog is better for this).

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
| `osv-scanner`, `trivy`, `checkov`, `tflint`, `semgrep`, `bandit`, `pmd`, `php-codesniffer`, `radarlint-*`, `kube-linter`, `ripgrep` | `monitor` | Can be noisy on first run — observe before enforcing |

Ask: "Do any of these feel too strict or too lenient for your team? If you're starting fresh and want to ease in gradually, we can move more to `monitor`. If you're enforcing from day one, we can promote security scanners to `block`."

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

Ask:
1. Do you want to enable smells? (Recommended: yes, `mode = "comment"`)
2. Do any thresholds need adjustment for your codebase? (e.g., lower `function_parameters` to 3 for strict teams, or raise `function_complexity` to 20 for data-heavy code)
3. Do you want per-language overrides? (e.g., Python is often written with more parameters in data science code)

### Decision 7: Monorepo Configuration (skip if not a monorepo)

If the repo has distinct services or packages in subdirectories with different language stacks:

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

---

## Phase 4: Implement and Open PR

Once all decisions are confirmed:

### 1. Write `.qlty/qlty.toml`

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
[language.python.checks]
function_parameters.threshold = X

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

Only include sections with non-default values or explicit user decisions. Don't add empty blocks or `[[triage]]` blocks unless requested.

### 2. Create a branch and open a PR

- Branch: `qlty-setup` (or `qlty-config` if that already exists)
- Commit: `Add .qlty/qlty.toml configuration`
- PR description should include:
  - Every plugin enabled and its mode
  - Any `extra_packages` added and why
  - Any manual steps the user needs to take

### 3. Print a configuration rundown

After the PR is open, print a clean summary:

```
## Qlty configuration summary

Plugins enabled: 9

  BLOCK   trufflehog      — secrets detection (all files)
  BLOCK   zizmor          — GitHub Actions security
  COMMENT eslint          — JS/TS linting (extras: @typescript-eslint/eslint-plugin, eslint-plugin-react)
  COMMENT tsc             — TypeScript type checking
  COMMENT prettier        — formatting (JS/TS/CSS/Markdown)
  COMMENT markdownlint    — markdown quality
  COMMENT yamllint        — YAML correctness
  COMMENT actionlint      — GitHub Actions syntax
  MONITOR osv-scanner     — dependency vulnerability scanning
  MONITOR checkov         — infrastructure security (IaC files)

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
