# C++ Quality - clang-format

> **Source**: https://clang.llvm.org/docs/ClangFormat.html
> **Skill**: dev-suite skill `quality/cpp-quality` — see SKILL.md for the always-loaded quick reference.

## What this covers

How clang-format discovers and merges `.clang-format` files, the `--lines=` flag for formatting
only changed regions, IDE integration, pre-commit hook patterns, and the `git-blame-ignore-revs`
file that lets you reformat the whole repo without poisoning blame.

## Deep dive

### File discovery

clang-format walks **upward** from the file being formatted looking for `.clang-format` (or
`_clang-format` on Windows-hostile filesystems). The first match wins entirely — there is no
inheritance. To cascade, copy the parent and modify.

If no file is found and no `--style=file` was given, the `LLVM` style is used silently.
This is a frequent source of confusion in monorepos: forgetting a `.clang-format` in a subtree
silently formats it differently.

### Format only changed lines (preserves untouched code)

```bash
# Format the lines you actually changed in the working tree
git clang-format -- src/foo.cpp

# Format only diff lines vs another ref
git diff -U0 --no-color HEAD^ -- '*.cpp' '*.hpp' \
  | clang-format-diff -p1 -i

# Manually: format lines 100-150 only
clang-format -i --lines=100:150 src/foo.cpp
```

`git clang-format` (ships with LLVM) is the most ergonomic: stage your changes, run it, re-add.

### Pre-commit hook

Using the `pre-commit` framework:

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/mirrors-clang-format
    rev: v18.1.8
    hooks:
      - id: clang-format
        types_or: [c++, c]
        args: [--style=file]
```

Or as a raw git hook:

```bash
#!/usr/bin/env bash
# .git/hooks/pre-commit
files=$(git diff --cached --name-only --diff-filter=ACMR | grep -E '\.(cpp|hpp|h|cc)$')
[ -z "$files" ] && exit 0
echo "$files" | xargs clang-format --dry-run --Werror || {
    echo "Run: git diff --cached --name-only ... | xargs clang-format -i"
    exit 1
}
```

Reject commits that aren't formatted; don't auto-format on commit (silent rewrites confuse the
author).

### IDE integration

| Editor | Setup |
|--------|-------|
| VS Code | C/C++ extension, set `"C_Cpp.formatting": "clangFormat"` |
| CLion | Built-in, `Settings → Editor → Code Style → C/C++ → Enable ClangFormat` |
| Vim | `clang-format.py` plugin or `Plug 'rhysd/vim-clang-format'` |
| Emacs | `clang-format.el` from LLVM |
| Visual Studio | Built-in since VS 2017 15.7 |

All of them call out to the `clang-format` binary and respect `.clang-format` discovery.

### Reformat the whole repo without destroying `git blame`

Running `clang-format -i` across the whole tree then committing makes `git blame` point at the
reformat commit for every line. The fix:

```bash
git ls-files '*.cpp' '*.hpp' | xargs clang-format -i
git commit -am "style: reformat all sources with clang-format"

# Capture the commit SHA
git rev-parse HEAD >> .git-blame-ignore-revs

# Tell future blame to ignore it
git config blame.ignoreRevsFile .git-blame-ignore-revs
git add .git-blame-ignore-revs
git commit -am "chore: ignore reformat commit in git blame"
```

GitHub respects `.git-blame-ignore-revs` automatically in its blame view. New contributors get
correct blame as soon as they `git config blame.ignoreRevsFile`.

### `// clang-format off` regions

```cpp
// clang-format off
const Matrix3 kIdentity{
    1, 0, 0,
    0, 1, 0,
    0, 0, 1,
};
// clang-format on
```

Use sparingly — for tabular literals, generated `extern "C"` blocks, or aligned bitfield
declarations. Each off region adds maintenance cost.

### Verifying formatted-ness in CI

```bash
# Fails the job if anything would change
git ls-files '*.cpp' '*.hpp' '*.cc' '*.h' \
  | xargs clang-format --dry-run --Werror
```

`--dry-run` doesn't write; `--Werror` returns non-zero if any reformatting was needed.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Auto-formatting on commit hook | Rewrites without author seeing the diff | Reject in pre-commit, format manually |
| `// clang-format off` for whole files | Maintenance bomb | Use `DisableFormat: true` in a per-directory `.clang-format` |
| Reformat commit in normal history | Destroys blame for every line | Capture SHA in `.git-blame-ignore-revs` |
| Mixing clang-format versions across team | Tiny diff drift on every commit | Pin via pre-commit `rev:`, fail CI on version mismatch |
| Forgetting `Standard: c++20` | Defaults change between LLVM versions | Always pin `Standard:` explicitly |

## See also

- https://clang.llvm.org/docs/ClangFormatStyleOptions.html
- https://docs.github.com/en/repositories/working-with-files/using-files/viewing-a-file#ignore-commits-in-the-blame-view
