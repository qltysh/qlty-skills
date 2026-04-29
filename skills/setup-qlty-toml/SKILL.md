---
name: setup-qlty-toml
description: Configure and fine-tune qlty.toml for static analysis using the Qlty CLI and Qlty Cloud. Use this skill when a user wants to set up Qlty for the first time, improve an existing qlty.toml configuration, choose which linters and security scanners to enable, configure plugin modes (block/comment/monitor), set code smell thresholds, or tune exclude patterns. Handles single repos and monorepos. Supports all Qlty-supported languages: JavaScript, TypeScript, Python, Ruby, Go, Java, Kotlin, PHP, Rust, Swift, Shell, CSS, SQL, Terraform, Docker, and more. Do NOT use for code coverage setup (use setup-coverage instead), writing or running tests, or general CI/CD changes unrelated to Qlty analysis.
metadata:
  stage: production
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
Scan the repo for significant file extensions and note the primary language(s), presence of Dockerfiles, Terraform, YAML, SQL, OpenAPI specs, and GitHub Actions workflows. Multiple language manifests (`package.json`, `go.mod`, `Cargo.toml`, `pyproject.toml`, etc.) in different subdirectories signal a monorepo.

**Existing tooling:**
Scan for linter and formatter config files — any tool-specific config (`.rubocop.yml`, `pyproject.toml`, `.eslintrc*`, `biome.json`, `.golangci.yml`, etc.) tells you which tools the team already uses. To find the matching Qlty plugin for a given tool, check the plugin registry at https://github.com/qltysh/qlty/tree/main/qlty-plugins/plugins — each plugin's `plugin.toml` lists the config files it recognizes.

Read any existing tool configs to understand what extensions, parsers, or plugins are in use — those typically become `extra_packages` in the plugin block.

For npm-based tools, read `package.json` devDependencies. For Python-based tools, check `pyproject.toml` or `requirements*.txt`. These tell you what the team has already pinned.

**Generated and vendored directories:**
Look for directories likely containing generated, vendored, or third-party code: `vendor/`, `node_modules/`, `dist/`, `build/`, `target/`, `generated/`, `.yarn/`, minified files (`*.min.*`), TypeScript declaration files (`**/*.d.ts`). Also check `.gitignore`.

**CI configuration:**
Check `.github/workflows/` and note whether `qlty check` is already in CI.

Produce a brief summary of findings, then move on to Phase 2.

---

## Phase 2: Fetch Documentation

Fetch only what you need — don't read everything upfront.

**1. Plugin availability (always do this first):**
Fetch https://github.com/qltysh/qlty/tree/main/qlty-plugins/plugins to confirm which plugins exist. Each subdirectory name is a valid plugin name. Do not add a plugin you haven't confirmed here.

**2. Per-plugin README (fetch for each plugin you're about to enable):**
Every plugin has a README at:
```
https://raw.githubusercontent.com/qltysh/qlty/main/qlty-plugins/plugins/{plugin-name}/README.md
```
Read it before writing the plugin block. The README is the authoritative source for:
- What `extra_packages` or `config_files` the plugin needs
- Valid field values (e.g. `drivers` options for `trivy`)
- Version constraints and known limitations
- Config file format requirements

Fetch only for plugins on your proposed list — not all of them upfront.

**3. qlty.toml format (fetch only if you encounter an unfamiliar field):**
- https://docs.qlty.sh/qlty-toml — field reference
- https://docs.qlty.sh/cli/linter-extensions — `extra_packages` vs `package_file` vs `package_filters`

**4. Qlty-internal behavioral caveats (always read — it's small):**
Read `references/plugin-registry.md`. This file captures caveats about how Qlty itself behaves that you won't find in plugin READMEs — config file handling, cache behavior, cloud vs. local differences, and plugins with known issues in the current CLI version.

---

## Phase 3: Interactive Configuration Decisions

Work through each decision below. For each one: **briefly explain what it does, state your recommendation with reasoning, and ask the user to confirm or change it before moving on.** Group related decisions in one message to reduce back-and-forth.

**If running autonomously** (e.g., in an eval or non-interactive context): apply all recommended defaults from each decision below and proceed without asking for confirmation.

### Decision 1: Plugin Selection

Based on detected languages and existing config files, propose a full plugin list organized by category. Present as a table:

| Plugin | Category | Why recommended | Enabled? |
|---|---|---|---|
| ... | ... | ... | ... |

**How to select plugins:**

1. **Only propose plugins you confirmed exist** in the GitHub registry fetched in Phase 2. If a plugin name doesn't appear as a subdirectory there, don't add it — check `references/plugin-registry.md` for any known caveats on why.

2. **Coverage categories to consider:** language linters, formatters, security scanners (secrets, dependency vulnerabilities, IaC, CI pipeline), and cross-language tools matching what's in the repo (Markdown, YAML, GitHub Actions, Docker, OpenAPI, etc.). For each category, find the relevant plugins in the registry and check their READMEs to confirm they apply.

Ask the user to confirm the list or add/remove plugins before proceeding.

### Decision 2: Plugin Modes

Explain the four modes once, clearly:

- **`block`**: Fails the Qlty Quality Gate — the PR cannot be merged until issues are resolved. Use for issues you always want fixed.
- **`comment`**: Posts a code review comment on the PR, but does not block merging. Use for important feedback without hard enforcement.
- **`monitor`**: Issues are visible on Qlty Cloud but not posted to GitHub at all. Use for noisy tools while you build trust in their signal.
- **`disabled`**: Plugin does not run.

Present your mode recommendations using these principles by plugin category:

| Category | Default mode | Reasoning |
|---|---|---|
| Secrets detection | `block` | Always urgent and high-signal — block immediately |
| Security scanners (vulnerabilities, IaC, CI pipeline) | `block` | High-value, low-noise — block by default |
| Language linters | `comment` | Important feedback, non-blocking by default |
| Formatters | `comment` | Style, not correctness — visible but non-blocking |
| Config/docs quality (Markdown, YAML, Actions, Docker) | `comment` | Useful but not blocking |
| Deep code quality / broad pattern scanners | `monitor` | Can be noisy on first run — observe before enforcing |

For each plugin on your list, identify its category and apply the matching default. Adjust based on the plugin's README signal-to-noise guidance and the team's risk tolerance.

Ask: "Do any of these feel too strict or too lenient for your team? If you're starting fresh and want to ease in gradually, we can move security scanners to `monitor`. If you want stricter enforcement, we can promote linters to `block`."

### Decision 3: Extra Packages and Config Files

For plugins that require external packages or have config files the team has already written, propose the right `extra_packages`, `config_files`, and `package_file` entries.

**How to determine what a plugin needs:**
You read each plugin's README in Phase 2 — use that as your source. The README specifies required packages, supported config file formats, and version constraints. Do not guess or rely on memory.

For npm-based plugins, also read the existing project config file (e.g. `.eslintrc`, `stylelint.config.js`) to identify `extends`, `plugins`, and `parser` entries — each one maps to an npm package that needs to be in `extra_packages`.

**Version pinning:** Use versions already pinned in `package.json` devDependencies wherever possible. Fall back to the latest stable version only if the package is not already in devDependencies. Cross-check package major versions against the plugin's bundled tool version (found in the README) — mismatches can pass locally but fail in Qlty Cloud.

**IMPORTANT: Always use `config_files = ["path"]` when referencing a plugin's config file. Never use `config = "path"` — that is not a valid TOML field.**

**Config file references** — if a standalone config file exists, add `config_files`:
```toml
[[plugin]]
name = "{plugin-name}"
config_files = ["{path/to/config}"]
extra_packages = ["{package}@{version}"]
```

If the config is embedded in `package.json`, use `package_file = "package.json"` instead of `config_files`.

**`package_file` + `package_filters`** — When the project's `package.json` (or `Gemfile`, `composer.json`) already declares all needed linter plugins, prefer referencing it over duplicating versions in `extra_packages`. Use `package_filters` to limit which packages get installed from that file:

```toml
[[plugin]]
name = "{plugin-name}"
package_file = "package.json"
package_filters = ["{unscoped-package-prefix}"]
```

**Warning: `package_filters` is a prefix filter and does NOT match scoped npm packages** (those starting with `@`). Use `extra_packages` with explicit versions when any required package is scoped.

Choose `package_file` when: the project's package manager file already lists all needed plugins and you want versions to stay in sync, AND all needed packages are unscoped (no `@scope/` prefix). Choose `extra_packages` when: the project has no package file, you need specific versions independent of the project, or any required packages are scoped npm packages.

**Note on lock files:** With `package_file`, Qlty respects locked versions. With `package_file` + `package_filters`, lock files are ignored. With `extra_packages`, lock files are not used at all.

**`version`** — Pin the tool version (the linter itself, not its extensions). Use when a specific tool version is needed for compatibility with the project's config:

```toml
[[plugin]]
name = "{plugin-name}"
version = "{x.y.z}"
```

**`affects_cache`** — Additional files whose changes should invalidate the plugin's analysis cache. Use when a plugin reads config from files not automatically detected:

```toml
[[plugin]]
name = "{plugin-name}"
affects_cache = ["{config-file}"]
```

**`skip_upstream`** — Set to `true` to skip analysis of files outside the current diff. Useful for type-checkers that would otherwise analyze all imported files:

```toml
[[plugin]]
name = "{plugin-name}"
skip_upstream = true
```

**`.qlty/configs/` directory** — Store a plugin's config file inside `.qlty/` to keep it out of the repo root. Qlty will automatically provision it during analysis:

```toml
[[plugin]]
name = "{plugin-name}"
config_files = [".qlty/configs/{config-file}"]
```

**Plugin-specific caveats and known issues:** Check `references/plugin-registry.md` before writing any plugin block — it records Qlty-internal behaviors that the plugin README won't tell you (config file handling, version pinning requirements, plugins with known issues in the current CLI).

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
plugins = ["{plugin-name}"]
file_patterns = ["{glob}"]
```

Ask: "Are there directories with generated code, third-party code you don't own, or files with intentional violations that should be excluded?"

### Decision 5: Test Patterns

Show the current `test_patterns`. These help Qlty Cloud distinguish test code from production code for more accurate quality metrics.

Defaults cover most frameworks (`**/test/**`, `**/spec/**`, `**/*.test.*`, `**/*.spec.*`, etc.).

Ask if they need additions for custom test directories (e.g., `**/e2e/**`, `**/integration_tests/**`, `**/features/**` for Cucumber).

### Decision 6: Code Smells and Complexity Thresholds

Explain: "Qlty has built-in maintainability analysis — separate from any linter plugin. It flags things like overly complex functions, too many parameters, deeply nested code, and duplicate logic. These are called 'smells'."

Fetch https://docs.qlty.sh/analysis-configuration for the current list of configurable smells, their field names, and default thresholds. The config supports per-smell enable/disable, duplication sensitivity tuning, and per-language threshold overrides:

```toml
[smells]
mode = "comment"

# Disable a specific smell
[smells.{smell-name}]
enabled = false

# Tune duplication sensitivity
[smells.duplication]
nodes_threshold = 100

# Per-language override
[language.{lang}.smells]
{smell-name}.threshold = {value}
```

Ask:
1. Do you want to enable smells? (Recommended: yes, `mode = "comment"`)
2. Do any thresholds need adjustment for this codebase?
3. Are there specific smells to disable entirely?
4. Do you want per-language overrides?

### Decision 7: Monorepo Configuration (skip if not a monorepo)

If the repo has distinct services or packages in subdirectories with different language stacks:

**Same-language workspaces** (e.g., a Cargo workspace with multiple Rust crates, a Lerna monorepo with all-JS packages): do NOT scope by `prefix` — a single config covers all packages. Use `prefix` only when different subdirectories use different languages or need different plugin configs.

Explain: "The `prefix` field on a plugin scopes it to run only within a specific subdirectory. This is useful when different parts of the repo use different languages or have separate linter configs."

```toml
[[plugin]]
name = "{plugin-name}"
prefix = "{subdirectory}"
config_files = ["{subdirectory/config-file}"]
```

Also explain `[[exclude]]` for suppressing a plugin on specific paths:
```toml
[[exclude]]
plugins = ["{plugin-name}"]
file_patterns = ["{subdirectory}/**"]
```

Ask: "Should any plugins be scoped to specific subdirectories? Do you want to exclude any plugins from running in specific parts of the repo?"

### Decision 8: Triage Rules (advanced — skip if not needed)

`[[triage]]` blocks let you modify how specific issues are reported without changing the entire plugin's mode. They are evaluated in order — first match wins. Use them when:

- A specific rule is too noisy in general, but you don't want to move the whole plugin to `monitor`
- You want to promote a specific finding from `comment` → `block` without promoting the whole plugin
- A rule should be silenced only in certain file paths (e.g., ignore `no-console` in test files)
- You want to recategorize findings (change `level` from warning → error, or adjust `category`)

```toml
# Downgrade a noisy rule to monitor-only
[[triage]]
plugins = ["{plugin-name}"]
rules = ["{rule-id}"]
set.mode = "monitor"

# Promote a high-severity finding to block
[[triage]]
plugins = ["{plugin-name}"]
rules = ["{rule-id}"]
set.mode = "block"
set.level = "error"

# Silence a rule in specific paths only
[[triage]]
plugins = ["{plugin-name}"]
rules = ["{rule-id}"]
file_patterns = ["**/*.test.*"]
set.ignored = true
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

### 2. Verify locally with `qlty check --sample`

Before opening the PR, run:

```bash
qlty check --sample
```

This runs all enabled plugins on a random sample of real files and catches runtime crashes — failures that TOML syntax validation and `qlty plugins list` would both miss. A config can be structurally valid and still cause "Build errored" in Qlty Cloud if a plugin crashes at runtime (wrong config format, missing binary, incompatible version, missing styles directory, etc.).

**For each plugin that errors:**

1. **Check the debug output:** Qlty prints a path to a debug `.yaml` file alongside the error message. Read it — it contains the exact command that was run and stderr output.
2. **Check the troubleshooting docs:** https://qlty.sh/d/lint-error
3. **Attempt a fix:** common fixes include correcting `config_files` paths, adjusting `extra_packages`, removing an unsupported field, or updating the plugin's config format (e.g., ESLint flat config).
4. **If not fixable:** comment out the entire `[[plugin]]` block. Put a one-line explanation on the line immediately above `# [[plugin]]`:

```toml
# disabled: {reason} — re-enable when {condition}
# [[plugin]]
# name = "{plugin-name}"
# mode = "{mode}"
```

After resolving or disabling all errors, run `qlty check --sample` again and confirm it completes without any `Error` results before proceeding.

**Do not open the PR while any plugin is in an error state.** An erroring plugin causes "Build errored" for the entire Qlty Cloud build — blocking all PR feedback, not just the failing plugin.

### 3. Create a branch and open a PR

- Branch: `qlty-setup` (or `qlty-config` if that already exists)
- Commit: `Add .qlty/qlty.toml configuration`
- PR description should include:
  - Every plugin enabled and its mode
  - Any `extra_packages` added and why
  - Any plugins that were disabled during local verification, and why
  - Any manual steps the user needs to take

### 4. Update `references/plugin-registry.md` if you learned something new

If during this run you discovered that a plugin is missing from the public registry, causes Build errored, or has a caveat not already recorded, update `references/plugin-registry.md` in this skill's repo directly — do not wait for a separate evolve run. Add a row to the appropriate table with a `Last confirmed` date. This keeps the reference useful for the next agent.

### 5. Print a configuration rundown

After the PR is open, print a clean summary:

```
## Qlty configuration summary

Plugins enabled: N

  BLOCK   {plugin}   — {what it covers}
  COMMENT {plugin}   — {what it covers}
  MONITOR {plugin}   — {what it covers}

Smells: enabled (comment mode)[, {any non-default threshold overrides}]

Exclude patterns: N ({list key patterns})

Disabled during local verification: N
  DISABLED  {plugin}  — {reason; how to re-enable}
```

For each plugin, add a one-line explanation of its mode and what it covers.

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
```
# qlty-ignore: {plugin}:{rule-id}
```

Multiple rules, comma-separated:
```
# qlty-ignore: {plugin}:{rule}, {plugin}:{rule}
```

All rules from a plugin:
```
# qlty-ignore: {plugin}
```

Region silencing (start `+`, end `-`):
```
# qlty-ignore+: {plugin}
...affected lines...
# qlty-ignore-: {plugin}
```

Tell the user: "If you encounter specific false positives after the PR merges, use `qlty-ignore` comments rather than broadening the global config — it keeps suppressions visible and local to the affected code."
