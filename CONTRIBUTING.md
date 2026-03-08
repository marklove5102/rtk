# Contributing to RTK

## Two ways to add a filter

**For yourself or your project** → use `.rtk/filters.toml` (project-local) or `~/.config/rtk/filters.toml` (user-global). No PR needed. See the [Custom Filters](README.md#custom-filters) section in the README.

**For everyone** → add a built-in filter to `src/filters/`. That's what this guide covers.

---

## Adding a built-in filter

A built-in filter is a single `.toml` file in `src/filters/`. At compile time, `build.rs` concatenates all files alphabetically and embeds the result in the binary.

### Step 1 — Create the file

```
src/filters/my-tool.toml
```

Name the file after the command: `rsync.toml`, `gradle.toml`, `flutter-build.toml`. For commands with subcommands, use `cmd-subcmd.toml` (e.g. `mix-compile.toml`).

### Step 2 — Write the filter

Minimum required fields: `description`, `match_command`, and at least one action field.

```toml
[filters.my-tool]
description = "Compact my-tool output"
match_command = "^my-tool\\b"
strip_ansi = true
strip_lines_matching = [
  "^\\s*$",
  "^Downloading ",
  "^Installing ",
]
max_lines = 40
on_empty = "my-tool: ok"
```

### Step 3 — Add inline tests

Every filter must have at least one `[[tests.<name>]]` entry. A build guard enforces this.

```toml
[[tests.my-tool]]
name = "strips noise, keeps errors"
input = """
Downloading dep v1.0
Installing dep v1.0

ERROR: build failed at line 42
"""
expected = "ERROR: build failed at line 42"

[[tests.my-tool]]
name = "empty output returns on_empty message"
input = "Downloading dep v1.0\nInstalling dep v1.0\n"
expected = "my-tool: ok"
```

Use real command output as `input`, not synthetic data. Run the command once and paste the output.

### Step 4 — Build

```bash
cargo build
```

`build.rs` validates the combined TOML and fails with a clear error if your file has a syntax problem or a duplicate filter name.

### Step 5 — Run the tests

Two test suites to run:

```bash
# Unit tests (Rust)
cargo test

# Inline TOML tests (your [[tests.*]] entries)
cargo run -- verify
```

`cargo test` will fail on two guards — this is expected and required:

**Guard 1 — exact filter count:**
```
assertion failed: Expected exactly 21 built-in filters, got 22.
Update this count when adding/removing filters in src/filters/.
```
Fix: open `src/toml_filter.rs`, find `test_builtin_filter_count`, increment the number.

**Guard 2 — expected filter names:**
```
assertion failed: Built-in filter 'my-tool' is missing
```
Fix: in the same file, find `test_builtin_all_expected_filters_present`, add `"my-tool"` to the list.

After both fixes, all tests should pass:

```bash
cargo test        # all unit tests green
cargo run -- verify   # all inline TOML tests green
```

### Step 6 — Open a PR

```bash
git checkout -b feat/filter-my-tool
git add src/filters/my-tool.toml src/toml_filter.rs
git commit -m "feat(filters): add my-tool built-in filter"
gh pr create --base master
```

---

## Filter field reference

| Field | Type | Description |
|-------|------|-------------|
| `description` | string | One-line description shown in `rtk verify` output |
| `match_command` | regex | Matched against the full command string (e.g. `"^docker\\s+inspect"`) |
| `strip_ansi` | bool | Strip ANSI escape codes before all other stages |
| `strip_lines_matching` | regex[] | Drop lines matching any of these patterns |
| `keep_lines_matching` | regex[] | Keep only lines matching at least one pattern (mutually exclusive with `strip_lines_matching`) |
| `replace` | array | Regex substitutions applied line-by-line: `{ pattern = "...", replacement = "..." }` |
| `match_output` | array | Short-circuit rules: if the full output blob matches `pattern`, emit `message` and stop. First match wins. |
| `truncate_lines_at` | int | Truncate lines longer than N characters (unicode-safe) |
| `max_lines` | int | Keep only the first N lines after all other stages |
| `tail_lines` | int | Keep only the last N lines (applied after `strip_*` / `keep_*`, before `max_lines`) |
| `on_empty` | string | Emitted when filtered output is empty (e.g. `"make: ok"`) |

### Pipeline order

Stages run in this order on every execution:

```
strip_ansi → replace → match_output → strip/keep_lines → truncate_lines_at → tail_lines → max_lines → on_empty
```

`match_output` is a short-circuit: if any rule matches, the remaining stages are skipped.

### `match_command` tips

- Always anchor with `^` — RTK matches the full command string including arguments
- Use `\\s+` for subcommands: `"^docker\\s+compose\\s+ps"`
- Use `\\b` for word boundaries: `"^make\\b"` (matches `make all` but not `makefile`)
- Use `(a|b)` for variants: `"^mvn\\s+(compile|package|clean|install)\\b"`

---

## What makes a good built-in filter

**Good candidates:**
- Commands with noisy but structured output (progress bars, download logs, verbose build steps)
- Output where 80%+ of lines are noise (the 60% token savings target)
- Tools used broadly across the ecosystem (not project-specific)

**Not a good fit:**
- Commands with unstructured or unpredictable output (filtering would silently drop useful content)
- Commands already handled by a dedicated Rust module (`git`, `cargo`, `gh`, `docker`, `kubectl`, etc.) — TOML filters run first but Rust modules have more control
- Commands where the full output is always short (<20 lines) — passthrough is fine

**Token savings target:** at least 60% reduction on realistic input. Add a third test case that verifies this if your filter is aggressive:

```toml
[[tests.my-tool]]
name = "60pct token savings on realistic output"
input = """
[paste 30+ lines of real noisy output here]
"""
expected = "[the 5-line summary you kept]"
```

---

## Checklist before opening a PR

- [ ] `src/filters/my-tool.toml` created
- [ ] At least 2 `[[tests.my-tool]]` entries with real command output
- [ ] `cargo build` passes (TOML validation)
- [ ] `test_builtin_filter_count` updated in `src/toml_filter.rs`
- [ ] `test_builtin_all_expected_filters_present` updated in `src/toml_filter.rs`
- [ ] `cargo test` — all tests pass
- [ ] `cargo run -- verify` — all inline tests pass
- [ ] `cargo clippy --all-targets` — no errors
