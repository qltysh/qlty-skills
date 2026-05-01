---
name: setup-qlty-toml
description: Configure and fine-tune qlty.toml for static analysis using the Qlty CLI and Qlty Cloud. Use this skill when a user wants to set up Qlty for the first time, improve an existing qlty.toml configuration, choose which linters and security scanners to enable, configure plugin modes (block/comment/monitor), set code smell thresholds, or tune exclude patterns. Handles single repos and monorepos. Supports all Qlty-supported languages: JavaScript, TypeScript, Python, Ruby, Go, Java, Kotlin, PHP, Rust, Swift, Shell, CSS, SQL, Terraform, Docker, and more. Do NOT use for code coverage setup (use setup-coverage instead), writing or running tests, or general CI/CD changes unrelated to Qlty analysis.
metadata:
  stage: production
---

# Set Up qlty.toml

You are a senior software engineer configuring Qlty static analysis for this repository. Work through these phases in order.

---

## Phase 1: Analyze the Repository

**Check Qlty state — this determines your mode for the entire run:**

- **No `.qlty/qlty.toml`** → run `qlty init` to generate the baseline. Plugin selection is `qlty init`'s job — do not second-guess which plugins it enables. Your job is to fine-tune the generated config (Phases 2–4).
- **Existing `.qlty/qlty.toml`** → read it carefully. You are refining, not replacing. After fine-tuning, scan for plugins that are clearly absent but strongly signaled by the repo (e.g., a `.golangci.yml` with no `golangci-lint` block). Surface at most 2–3 recommendations in the PR description — do not add them automatically.

**Catalog the repo:**
- Languages and file types: primary language(s), Dockerfiles, Terraform, YAML, SQL, OpenAPI, GitHub Actions
- Existing tooling: scan for linter/formatter configs (`.rubocop.yml`, `pyproject.toml`, `.eslintrc*`, `biome.json`, `.golangci.yml`, etc.). For each tool config found, locate its Qlty plugin via the registry at https://github.com/qltysh/qlty/tree/main/qlty-plugins/plugins — each plugin's `plugin.toml` lists the config files it recognizes
- Read existing tool configs to identify extensions, parsers, or plugins in use — these become `extra_packages`
- Generated/vendored directories: `vendor/`, `node_modules/`, `dist/`, `build/`, `target/`, `generated/`, minified files, `.d.ts` files. Also check `.gitignore`
- Multiple language manifests in subdirectories → monorepo
- CI config: note whether `qlty check` is already in `.github/workflows/`

Produce a brief summary of findings before moving on.

---

## Phase 2: Fetch Documentation

Fetch only what you need — do not read everything upfront.

1. **Plugin availability** (always): https://github.com/qltysh/qlty/tree/main/qlty-plugins/plugins — each subdirectory is a valid plugin name. Do not add a plugin not listed here.

2. **Per-plugin README** (fetch for each plugin you're about to enable):
   `https://raw.githubusercontent.com/qltysh/qlty/main/qlty-plugins/plugins/{plugin-name}/README.md`
   This is the authoritative source for `extra_packages`, `config_files`, valid field values, version constraints, and config format requirements.

3. **qlty.toml field reference** (fetch only if you encounter an unfamiliar field):
   - https://docs.qlty.sh/qlty-toml
   - https://docs.qlty.sh/cli/linter-extensions

4. **Qlty-internal behavioral caveats** (always read — it's small): `references/plugin-registry.md`
   Captures non-obvious Qlty behaviors not in plugin READMEs: config file handling, cache quirks, cloud vs. local differences, plugins with known issues in the current CLI.

---

## Phase 3: Fine-Tuning

Before writing the config, send the user **one compact prompt** with all tunable decisions. Present fixed options and a clear default for each. Do not explain each option at length — the options speak for themselves.

**If running autonomously** (eval or non-interactive context): skip the prompt entirely and apply all defaults.

### Prompt template

Use the `AskUserQuestion` tool to present all choices as interactive buttons in a single call (up to 4 questions). Populate plugin names from Phase 1 findings. Always put the recommended option first and append "(Recommended)" to its label.

Questions to ask:

1. **Security scanners** (`{trivy, osv-scanner, trufflehog, …}`)
   - Block — fails the Quality Gate (Recommended)
   - Comment — posts a review comment, non-blocking
   - Monitor — visible on Qlty Cloud only

2. **Language linters** (`{eslint, golangci-lint, rubocop, …}`)
   - Comment — review feedback, non-blocking (Recommended)
   - Block — fails the Quality Gate
   - Monitor — visible on Qlty Cloud only

3. **Code smells & complexity**
   - Enable in comment mode (Recommended)
   - Enable in monitor mode
   - Disable

4. **Complexity thresholds**
   - Default (Recommended)
   - Strict — tighten thresholds ~30%
   - Relaxed — loosen thresholds ~30%

Add a note in the question descriptions: "You can always fine-tune these later by updating qlty.toml."

Apply the user's choices (or defaults if they dismiss) and proceed directly to Phase 4 — no further confirmation needed.

### What each choice controls

- **Security scanners** → mode applied to secrets, vulnerability, IaC, and CI pipeline plugins
- **Language linters** → mode applied to language-specific linters (not formatters)
- **Code smells** → enables Qlty's built-in maintainability analysis; fetch https://docs.qlty.sh/analysis-configuration for threshold fields. "Strict" tightens thresholds by ~30%; "relaxed" loosens them
- **Monorepo scoping** → if yes, use `prefix` to scope plugins to subdirectories with different language stacks; same-language workspaces do not need it

### Always apply autonomously (not in the prompt)

- **Formatters**: always `comment`
- **Extra packages and config files**: use each plugin's README as source; pin versions from existing `package.json` devDependencies where possible; use `extra_packages` (not `package_filters`) for scoped npm packages (`@scope/pkg`)
- **Exclude patterns**: add exclusions for generated, vendored, minified, and fixture directories found in Phase 1; avoid broad names like `**/config/**`
- **Test patterns**: extend defaults for any custom test directories found in Phase 1
- **Triage rules**: skip on first run unless a plugin is clearly too noisy based on its README signal-to-noise guidance

---

## Phase 4: Implement and Open PR

### 1. Write `.qlty/qlty.toml`

**CRITICAL — these all cause "Build errored":**
- Do NOT include `[cli]` or `[settings]` sections from `qlty init` output
- Do NOT use the old `[[sources.community]]` format — use `[[source]] name = "default" default = true`
- Do NOT use the old `[plugins.enabled]` / `[[plugins.enabled]]` format — use `[[plugin]]` blocks
- Do NOT nest `exclude_patterns` as a section — it must be a top-level array
- The `[[source]]` block is required — without it Qlty cannot resolve plugins. Do NOT set `directory` to a URL

Field reference: https://docs.qlty.sh/qlty-toml

### 2. Verify locally

```bash
qlty check --sample
```

Catches runtime crashes that TOML syntax validation misses. For each plugin that errors: read the debug `.yaml` Qlty prints, check https://qlty.sh/d/lint-error, attempt a fix. If not fixable, comment out the `[[plugin]]` block with a one-line explanation above it. **Do not open the PR while any plugin is erroring** — one erroring plugin causes "Build errored" for the entire cloud build.

### 3. Open PR

- Branch: `qlty-setup`
- PR description: every plugin enabled with its mode, any `extra_packages` and why, any plugins disabled during verification and why, any manual steps needed

### 4. Update `references/plugin-registry.md` if you learned something new

If a plugin is missing from the public registry, causes a build error, or has a caveat not already recorded, add it inline — do not wait for a separate evolve run.

### 5. Print a configuration rundown

```
Plugins enabled: N
  BLOCK   {plugin} — {what it covers}
  COMMENT {plugin} — {what it covers}
  MONITOR {plugin} — {what it covers}

Smells: {enabled/disabled}, {mode}, {any non-default thresholds}
Disabled during verification: {plugin} — {reason}
```

---

## Phase 5: Qlty Cloud Guidance

Tell the user what to verify in Qlty Cloud (requires the web UI, not qlty.toml):

1. **Project visible**: `https://qlty.sh/gh/<org>` — confirm the repo is listed; if not, install the Qlty GitHub App
2. **Quality Gate**: `https://qlty.sh/gh/<org>/projects/<repo>/settings/review` — confirm block-mode plugins are set to fail the gate
3. **After merging**: first full analysis runs automatically; results at `https://qlty.sh/gh/<org>/projects/<repo>`
4. **Follow-up**: review monitor-mode findings, promote useful ones to comment/block; add `qlty check` to CI if not already there
5. **Inline silencing**: for specific false positives use `qlty-ignore` comments (see https://docs.qlty.sh) rather than broadening global config
