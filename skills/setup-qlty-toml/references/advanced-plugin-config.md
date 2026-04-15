# Advanced Plugin Configuration Reference

Source: https://docs.qlty.sh/qlty-toml, https://docs.qlty.sh/cli/linter-extensions

## Full [[plugin]] Field Reference

| Field | Type | Description |
|---|---|---|
| `name` | string | Plugin identifier (required) |
| `version` | string | Pin the tool version (e.g. `"1.55.2"`) |
| `mode` | string | `"block"`, `"comment"`, `"monitor"`, `"disabled"` |
| `prefix` | string | Scope plugin to a subdirectory (monorepo) |
| `config_files` | `["path"]` | Paths to config files the plugin should use |
| `extra_packages` | `["pkg@version"]` | Additional packages to install (npm, gem, pip, composer) |
| `package_file` | string | Path to package file (package.json, Gemfile, etc.) |
| `package_filters` | `["name"]` | Limit packages installed from `package_file` |
| `drivers` | `["name"]` | Specific drivers to activate (trivy: `"config"`, `"secret"`, `"vuln"`) |
| `affects_cache` | `["path"]` | Additional files that invalidate the plugin cache |
| `skip_upstream` | bool | Skip analysis of files outside the current diff |
| `triggers` | list | Events that activate the plugin |

## Choosing: extra_packages vs package_file

### Use `extra_packages` when:
- Project has no package.json / Gemfile / composer.json
- You need to pin specific versions independently of the project
- The project has many unrelated dependencies and you want a minimal install

```toml
[[plugin]]
name = "eslint"
extra_packages = [
  "@typescript-eslint/parser@5.62.0",
  "@typescript-eslint/eslint-plugin@5.62.0",
  "eslint-plugin-react@7.33.2",
]
```

### Use `package_file` when:
- The project already declares all needed linter plugins in package.json/Gemfile
- You want plugin versions to stay in sync with the project

```toml
[[plugin]]
name = "eslint"
package_file = "package.json"
```

### Use `package_file` + `package_filters` when:
- package.json has many unrelated deps (slow to install all of them)
- You only want specific packages from the file

```toml
[[plugin]]
name = "eslint"
package_file = "package.json"
package_filters = ["eslint"]
```

**Lock file behavior:**
- `package_file` alone → Qlty respects locked versions
- `package_file` + `package_filters` → lock files are ignored
- `extra_packages` → lock files are not used

## Plugin Version Pinning

Pin the tool itself (separate from extension packages):

```toml
[[plugin]]
name = "golangci-lint"
version = "1.55.2"
```

Use when: a specific tool version is required for compatibility with the project's config format, or when upgrading has broken something.

## Cache Invalidation (affects_cache)

Tell Qlty to re-run the plugin when extra files change (beyond the plugin's own config):

```toml
[[plugin]]
name = "eslint"
affects_cache = [".eslintignore", "tsconfig.json", ".prettierrc"]

[[plugin]]
name = "checkov"
affects_cache = [".checkov.yml"]
```

## Skip Upstream Analysis

Useful for TypeScript where `tsc` would otherwise type-check all imported files:

```toml
[[plugin]]
name = "tsc"
skip_upstream = true
```

## .qlty/configs/ for Plugin Config Files

Store plugin configs inside `.qlty/configs/` to keep the repo root clean. Qlty auto-provisions them during analysis:

```toml
[[plugin]]
name = "rubocop"
config_files = [".qlty/configs/.rubocop.yml"]

[[plugin]]
name = "eslint"
config_files = [".qlty/configs/.eslintrc.js"]
```

## Trivy Drivers

Valid driver values (only these three):

| Driver | What it scans |
|---|---|
| `"config"` | IaC misconfigurations (Terraform, Docker, K8s, CloudFormation) |
| `"secret"` | Secrets in code (trufflehog is usually better for this) |
| `"vuln"` | Container image and OS package vulnerabilities |

```toml
[[plugin]]
name = "trivy"
mode = "block"
drivers = ["config", "vuln"]
```

**Note:** `"fs-vuln"` is NOT a valid driver — use `"vuln"` instead.

## Monorepo Prefix Scoping

Scope a plugin to a specific subdirectory. Only use `prefix` when subdirectories have different languages or separate linter configs:

```toml
[[plugin]]
name = "eslint"
prefix = "frontend"
config_files = ["frontend/.eslintrc.js"]

[[plugin]]
name = "ruff"
prefix = "backend"
```

**Same-language workspaces** (e.g., all Rust crates, all JS packages): do NOT use `prefix` — one plugin config covers all.

## Custom RuboCop Cops

Add custom cop definitions to rubocop via config_files:

```toml
[[plugin]]
name = "rubocop"
config_files = [".rubocop.yml", "lib/custom_cops/my_cop.rb"]
```

## Supported Package Managers by Plugin Type

| Plugin type | Package manager | Field |
|---|---|---|
| eslint, prettier, stylelint, biome, tsc, knip | npm | `extra_packages` or `package_file = "package.json"` |
| rubocop, reek, brakeman | RubyGems | `extra_packages` or `package_file = "Gemfile"` |
| ruff, bandit, mypy, flake8, semgrep | pip | `extra_packages` or `package_file = "requirements.txt"` |
| phpstan, php-codesniffer, php-cs-fixer | Composer | `extra_packages` or `package_file = "composer.json"` |
