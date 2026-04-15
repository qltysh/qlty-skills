# [[triage]] Block Reference

Source: https://docs.qlty.sh/qlty-toml

`[[triage]]` blocks modify how specific issues are reported. They are evaluated in order — the **first matching block** wins. Use triage when you need surgical control over individual rules without changing the plugin's overall mode.

## Match Conditions (all optional, combined as AND)

| Field | Type | Description |
|---|---|---|
| `plugins` | `["name"]` | Match by plugin name |
| `rules` | `["rule-id"]` | Match by rule identifier (as reported by the tool) |
| `levels` | `["error"\|"warning"\|"note"]` | Match by current issue severity level |
| `file_patterns` | `["glob"]` | Match by file path glob |

## Set Values (what to change on match)

| Field | Values | Description |
|---|---|---|
| `set.mode` | `"block"\|"comment"\|"monitor"\|"disabled"` | Override the issue's effective mode — **preferred for suppression** |
| `set.level` | `"error"\|"warning"\|"note"` | Override the issue's severity level |
| `set.ignored` | `true` | Suppress entirely — use `set.mode = "disabled"` as a safer alternative; `set.ignored` behavior may not be consistent across Qlty versions |
| `set.category` | string | Recategorize the finding — **use with caution**, may cause "Build errored" in Qlty Cloud; avoid until confirmed supported in target environment |

## Common Patterns

### Downgrade one noisy rule to monitor

```toml
[[triage]]
plugins = ["rubocop"]
rules = ["Style/StringLiterals"]
set.mode = "monitor"
```

### Promote a high-value finding to block

```toml
[[triage]]
plugins = ["semgrep"]
rules = ["python.django.security.injection.sql"]
set.mode = "block"
set.level = "error"
```

### Ignore a rule in test files only

```toml
[[triage]]
plugins = ["eslint"]
rules = ["no-console"]
file_patterns = ["**/*.test.*", "**/*.spec.*", "**/test/**"]
set.ignored = true
```

### Silence all findings from a noisy plugin (first-run strategy)

```toml
[[triage]]
plugins = ["pmd"]
set.mode = "monitor"
```

### Downgrade warnings from a specific plugin in generated files

```toml
[[triage]]
plugins = ["markdownlint"]
file_patterns = ["**/CHANGELOG.md", "**/CHANGES.md"]
set.ignored = true
```

### Promote all error-level findings from bandit to block

```toml
[[triage]]
plugins = ["bandit"]
levels = ["error"]
set.mode = "block"
```

## Rule ID Format

Rule IDs are what the tool reports. To find them:
- Run `qlty check` locally and look at the `rule` field in the output
- Check the plugin's own documentation
- Examples: `rubocop:Style/StringLiterals`, `eslint:no-console`, `golangci-lint:errcheck`, `semgrep:python.django.security.injection.sql`

## When to use [[triage]] vs [[exclude]] vs qlty-ignore

| Situation | Best tool |
|---|---|
| Skip a directory for all plugins | `exclude_patterns = ["**/dir/**"]` |
| Skip a directory for one plugin | `[[exclude]]` with `plugins` and `file_patterns` |
| Adjust a specific rule's mode/level globally | `[[triage]]` |
| Suppress one specific occurrence in code | `qlty-ignore` inline comment |
