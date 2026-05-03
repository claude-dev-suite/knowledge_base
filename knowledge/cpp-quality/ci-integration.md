# C++ Quality - CI Integration for Linters

> **Source**: https://clang.llvm.org/extra/clang-tidy/Integrations.html
> **Skill**: dev-suite skill `quality/cpp-quality` — see SKILL.md for the always-loaded quick reference.

## What this covers

End-to-end CI configurations (GitHub Actions, GitLab CI, Jenkins) that run clang-format check,
clang-tidy, and cppcheck efficiently. Includes caching strategies, PR-only diffs vs full scans,
and converting tool output to inline PR review comments.

## Deep dive

### GitHub Actions — full pipeline

```yaml
# .github/workflows/cpp-quality.yml
name: C++ Quality
on: [push, pull_request]

jobs:
  format:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - name: Install clang-format
        run: sudo apt-get install -y clang-format-18
      - name: Check formatting
        run: |
          git ls-files '*.cpp' '*.hpp' '*.cc' '*.h' \
            | xargs clang-format-18 --dry-run --Werror

  tidy:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0          # need full history for diff vs main
      - uses: actions/cache@v4
        with:
          path: build/
          key: tidy-${{ hashFiles('CMakeLists.txt', '**/CMakeLists.txt') }}
      - name: Install
        run: sudo apt-get install -y clang-tidy-18 ninja-build cmake
      - name: Configure
        run: |
          cmake -B build -G Ninja \
                -DCMAKE_EXPORT_COMPILE_COMMANDS=ON \
                -DCMAKE_BUILD_TYPE=Debug
      - name: Build (for generated headers)
        run: cmake --build build --target generated_headers
      - name: Tidy changed files
        if: github.event_name == 'pull_request'
        run: |
          git diff --name-only --diff-filter=AM \
                   origin/${{ github.base_ref }}...HEAD -- '*.cpp' '*.hpp' \
            | xargs -r clang-tidy-18 -p build/ --quiet
      - name: Tidy whole tree (push to main)
        if: github.event_name == 'push'
        run: run-clang-tidy-18 -p build/ -quiet -j$(nproc)

  cppcheck:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - name: Install
        run: sudo apt-get install -y cppcheck
      - name: Configure (for compile_commands.json)
        run: cmake -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
      - name: Run
        run: |
          cppcheck --enable=warning,performance,portability \
                   --suppress=missingIncludeSystem \
                   --inline-suppr \
                   --error-exitcode=2 \
                   --xml --xml-version=2 \
                   -j$(nproc) \
                   --project=build/compile_commands.json \
                   2> cppcheck.xml
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: cppcheck-report
          path: cppcheck.xml
```

### Inline PR review comments via reviewdog

```yaml
- name: Tidy with reviewdog
  uses: reviewdog/action-cpp-tidy@v1
  with:
    reporter: github-pr-review
    level: warning
    flags: '-quiet'
```

`reviewdog` posts a comment on each affected line, only for lines changed in the PR. Avoids the
"50 unrelated warnings on a 5-line PR" problem.

### GitLab CI

```yaml
# .gitlab-ci.yml
stages: [check]

format:
  stage: check
  image: silkeh/clang:18
  script:
    - git ls-files '*.cpp' '*.hpp' | xargs clang-format --dry-run --Werror

clang-tidy:
  stage: check
  image: silkeh/clang:18
  cache:
    key: build
    paths: [build/]
  script:
    - apt-get update && apt-get install -y cmake ninja-build
    - cmake -B build -G Ninja -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
    - run-clang-tidy -p build/ -quiet -j$(nproc)
  artifacts:
    when: on_failure
    paths: [build/CMakeFiles/CMakeError.log]
```

### Jenkins (declarative pipeline)

```groovy
pipeline {
    agent { docker { image 'silkeh/clang:18' } }
    stages {
        stage('Format') {
            steps { sh 'git ls-files "*.cpp" "*.hpp" | xargs clang-format --dry-run --Werror' }
        }
        stage('Tidy') {
            steps {
                sh 'cmake -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON'
                sh 'run-clang-tidy -p build/ -quiet > tidy.log || true'
                recordIssues tools: [clangTidy(pattern: 'tidy.log')]
            }
        }
    }
}
```

`recordIssues` from the Warnings Next Generation plugin parses tool output into the Jenkins UI.

### Caching strategies that actually work

- **CMake build directory**: cache `build/` with a key derived from `CMakeLists.txt` content.
  Reuses object files when only sources changed.
- **ccache**: install and set `CMAKE_C_COMPILER_LAUNCHER=ccache` /
  `CMAKE_CXX_COMPILER_LAUNCHER=ccache`. Cache `~/.cache/ccache`.
- **Conan/vcpkg packages**: cache the package cache (`~/.conan2`, `vcpkg/installed/`).

Don't cache `compile_commands.json` itself — it's cheap to regenerate and a stale one is worse than
none.

### PR-only diff strategy

Running tools only on changed lines is dramatically faster on large codebases:

```bash
# Get changed C++ files vs base branch
CHANGED=$(git diff --name-only --diff-filter=AM \
            origin/${BASE_REF}...HEAD -- '*.cpp' '*.hpp' '*.cc' '*.h')

# clang-tidy only those
[ -n "$CHANGED" ] && echo "$CHANGED" | xargs clang-tidy -p build --quiet

# clang-format only those
[ -n "$CHANGED" ] && echo "$CHANGED" | xargs clang-format --dry-run --Werror
```

Run the *full* scan only on `push` to `main` (catches issues from PRs that were merged with stale
diff bases).

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Running clang-tidy with no build artifacts | Generated headers missing → "file not found" | `cmake --build build --target generated_headers` first |
| `fetch-depth: 1` then trying diff vs main | Shallow clone has no main | `fetch-depth: 0` for diff jobs |
| Caching build/ across compiler versions | Mismatched object files | Include compiler version in cache key |
| Running every linter on every commit | Slow CI | Tier: format on every commit, tidy on PR, cppcheck nightly |
| Inlining all warnings as PR comments | Reviewer drowning | Use reviewdog with `level: warning` and per-line filtering |
| `--quiet` not set | Massive logs hide failures | Always `--quiet` in CI |

## See also

- https://github.com/reviewdog/reviewdog
- https://www.jenkins.io/doc/pipeline/steps/warnings-ng/
