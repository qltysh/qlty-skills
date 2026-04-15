# Inline Issue Silencing with qlty-ignore

Source: https://docs.qlty.sh/silencing-issues

`qlty-ignore` is a universal inline comment directive for suppressing specific analysis issues at the source code level. It works in both Qlty CLI and Qlty Cloud. Use it for surgical per-occurrence suppression when a rule is legitimately triggering on correct code.

## Basic Syntax

```
# qlty-ignore: tool:rule-id
```

The directive applies to the **next line** by default (and any indented block beneath it).

## Examples by Language

### Python

```python
# qlty-ignore: ruff:E501
long_generated_string = "this line is intentionally long because it contains a URL or generated value"

# qlty-ignore: bandit:B101
assert user_input == expected  # assertion is intentional in this test helper
```

### TypeScript / JavaScript

```typescript
// qlty-ignore: eslint:no-console
console.log("intentional debug output kept for operator visibility");

// qlty-ignore: eslint:@typescript-eslint/no-explicit-any
const data: any = JSON.parse(rawInput);
```

### Go

```go
// qlty-ignore: golangci-lint:errcheck
file.Close()  // best-effort close, error intentionally ignored
```

### Ruby

```ruby
# qlty-ignore: rubocop:Style/StringLiterals
my_string = 'intentionally using single quotes here'
```

## Multiple Rules on One Line

Comma or space separated:

```python
# qlty-ignore: ruff:E501, ruff:E402
import sys; sys.path.insert(0, ".")  # bootstrap path for script entrypoint
```

## Tool-Level Suppression (all rules from a tool)

```python
# qlty-ignore: bandit
eval(trusted_input)  # suppresses all bandit rules on this line
```

## Scope Control

### Next line only (no indented block)

The `>` prefix restricts the suppression to exactly the next line:

```python
# qlty-ignore>: ruff:E501
long_line = "..."
    still_analyzed = True  # this IS analyzed
```

### Region silencing (`+` start, `-` end)

For larger blocks:

```python
# qlty-ignore+: bandit
eval(trusted_config)
process(trusted_config)
another_call(trusted_config)
# qlty-ignore-: bandit
```

### Unsilence a line within a region

```python
# qlty-ignore+: ruff
some_noisy_function()
# -ruff:E501  ← this line re-enables E501 checking for the next line
actually_check_this_line = long_string
# qlty-ignore-: ruff
```

## Rule Specifier Grammar

```
[prefix] tool-name [: rule-qualifier]
```

- `tool-name`: the plugin name (e.g., `eslint`, `rubocop`, `ruff`, `bandit`, `golangci-lint`)
- `rule-qualifier`: the specific rule (e.g., `no-console`, `Style/StringLiterals`, `E501`)
- Separator between tool and rule: `:` or `/`
- Valid characters: letters, numbers, underscores, hyphens, dots, `/`, `@`

## When to use qlty-ignore vs [[triage]] vs [[exclude]]

| Need | Use |
|---|---|
| This one line is a known-safe exception | `qlty-ignore` inline comment |
| This rule is always noisy across all files | `[[triage]]` with `set.mode = "monitor"` |
| This rule should be silenced in test files | `[[triage]]` with `file_patterns` |
| This directory should skip all analysis | `exclude_patterns` |
| This directory should skip one plugin | `[[exclude]]` block |

## Best Practice

Tell the user: "Use `qlty-ignore` for surgical one-off suppressions — it keeps the suppression visible in code review and co-located with the affected code. If you're adding `qlty-ignore` to the same rule in many places, that's a signal to use `[[triage]]` instead."
