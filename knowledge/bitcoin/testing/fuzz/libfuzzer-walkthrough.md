# libFuzzer Walkthrough (Bitcoin Core) - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/testing/fuzz`.
> Canonical source: https://github.com/bitcoin/bitcoin/blob/master/doc/fuzzing.md
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/testing/fuzz/SKILL.md

## Concept

Bitcoin Core ships dozens of libFuzzer harnesses under
`src/test/fuzz/`. Each harness is a single C++ file implementing
`FUZZ_TARGET(name)` plus optional initialisation. A unified driver
binary (`build_fuzz/bin/fuzz`) selects the harness at runtime via the
`FUZZ` environment variable. Coverage instrumentation comes from
clang's `-fsanitize=fuzzer`. Sanitizers (AddressSanitizer, UBSan)
catch crashes the moment they happen.

The combination - real Core code, real wire-format parsing, coverage
guidance, sanitizers, and a public corpus repository - has uncovered
dozens of integer overflows, OOB reads, and DoS vectors in deserialisers
since 2016. Running these locally on a PR you write is the cheapest
hardening step in Bitcoin development.

## Walkthrough / mechanics

A fuzz target receives raw bytes and exercises some parser/algorithm:

```cpp
#include <test/fuzz/fuzz.h>
#include <primitives/transaction.h>

FUZZ_TARGET(transaction_deserialize)
{
    DataStream ds{buffer};
    try {
        CTransaction tx{deserialize, TX_WITH_WITNESS, ds};
    } catch (const std::ios_base::failure&) {
        // expected on malformed input
    }
}
```

Build with fuzzing + sanitizers. Bitcoin Core 29.0 (April 2025)
replaced Autotools with CMake (minimum 3.22), so `./autogen.sh` and
`./configure` no longer exist. The `libfuzzer` preset sets
`BUILD_FOR_FUZZING=ON` and `SANITIZERS=undefined,address,fuzzer`, pins
`clang`/`clang++`, and builds into `build_fuzz/` (current as of
Bitcoin Core 31.1, July 2026):

```bash
cmake --preset=libfuzzer
cmake --build build_fuzz -j$(nproc)
```

`--preset=libfuzzer-nosan` is the same build without the
address/undefined sanitizers, in `build_fuzz_nosan/` - use it for long
coverage runs where throughput matters more than bug detection.

Run a target with a corpus directory:

```bash
mkdir -p corpora/transaction_deserialize
FUZZ=transaction_deserialize build_fuzz/bin/fuzz \
    corpora/transaction_deserialize \
    -max_total_time=600 \
    -print_final_stats=1
```

Common flags:

- `-max_total_time=N`: stop after N seconds.
- `-runs=N`: stop after N iterations.
- `-jobs=8 -workers=8`: parallel.
- `-dict=dict.txt`: token dictionary to bias inputs.
- `-only_ascii=1`: useful for descriptor/miniscript fuzzing.

To reproduce a crash, run with the offending input as a single arg:

```bash
FUZZ=script build_fuzz/bin/fuzz crash-deadbeef.bin
```

Public seed corpus:

```bash
git clone --depth=1 https://github.com/bitcoin-core/qa-assets.git
ln -s $PWD/qa-assets/fuzz_corpora corpora
```

That ships pre-built corpora for every harness, dramatically
accelerating coverage on first run. The per-harness directories live
under `fuzz_corpora/` (as of September 2026), with token dictionaries
alongside them in `fuzz_dicts/`.

## Worked example

Fuzz the script interpreter against arbitrary scripts and sigs:

```cpp
// src/test/fuzz/script_interpreter.cpp (illustrative)
#include <script/interpreter.h>
#include <test/fuzz/fuzz.h>
#include <test/fuzz/FuzzedDataProvider.h>

FUZZ_TARGET(script_interpreter)
{
    FuzzedDataProvider fdp(buffer.data(), buffer.size());
    auto script = ConsumeScript(fdp);
    auto sig    = fdp.ConsumeRandomLengthString(520);
    auto pubkey = fdp.ConsumeRandomLengthString(65);
    BaseSignatureChecker checker;
    ScriptError err;
    EvalScript({}, script, SCRIPT_VERIFY_NONE, checker,
               SigVersion::BASE, &err);
}
```

End-to-end run:

```bash
# 1. Build with fuzzing
cmake --preset=libfuzzer
cmake --build build_fuzz -j$(nproc)

# 2. Pull seed corpus
git clone --depth=1 https://github.com/bitcoin-core/qa-assets.git
mkdir -p run_corpus
cp -r qa-assets/fuzz_corpora/script_interpreter/* run_corpus/

# 3. Fuzz for 10 minutes
FUZZ=script_interpreter build_fuzz/bin/fuzz \
     run_corpus -max_total_time=600 -print_pcs=1

# 4. If a crash drops to disk:
FUZZ=script_interpreter build_fuzz/bin/fuzz crash-1234abcd
# AddressSanitizer prints stack trace pointing at the bug
```

Continuous integration snippet (GitHub Actions):

```yaml
- name: Fuzz script interpreter
  run: |
    cmake --preset=libfuzzer
    cmake --build build_fuzz --target fuzz -j$(nproc)
    FUZZ=script_interpreter build_fuzz/bin/fuzz \
      qa-assets/fuzz_corpora/script_interpreter \
      -runs=200000 -max_total_time=300
```

Triage workflow when a crash is found:

```bash
# Reproduce
FUZZ=foo build_fuzz/bin/fuzz crash-bin > /tmp/repro.log 2>&1

# Minimise
FUZZ=foo build_fuzz/bin/fuzz \
    -minimize_crash=1 \
    -runs=1000000 crash-bin

# Generate regression test from the minimised input
xxd minimised-bin | head
```

## Common pitfalls

- Configure outside the fuzz presets (no `SANITIZERS=...,fuzzer`, no
  `BUILD_FOR_FUZZING=ON`), and the `fuzz` binary links without
  coverage instrumentation; you then run uninstrumented and find
  nothing. `libfuzzer-nosan` still sets `SANITIZERS=fuzzer`, so it is
  the correct way to trade detection for speed.
- libFuzzer mutates inputs; if your target reads `buffer.data()`
  directly without `FuzzedDataProvider`, structured fields like
  varints rarely flip useful bits. Use `FuzzedDataProvider` to consume
  primitive types.
- Globals leak between iterations. Reset caches in the target's init
  hook - `FUZZ_TARGET(name, .init = initialize_name)`. The older
  `FUZZ_TARGET_INIT` macro no longer exists anywhere in the tree (as
  of September 2026).
- ASan + libFuzzer use ~3-4x more RAM; on a 16 GB box, do not run more
  than 4 jobs in parallel.
- New harness files must be listed in `src/test/fuzz/CMakeLists.txt`
  or they are never compiled into the driver. There is no separate
  target registry to edit: `FUZZ_TARGET` self-registers the name at
  static-init time.

## References

- Bitcoin Core fuzzing docs: `doc/fuzzing.md`
- qa-assets corpus: `https://github.com/bitcoin-core/qa-assets`
- libFuzzer: `https://llvm.org/docs/LibFuzzer.html`
- OSS-Fuzz Bitcoin project:
  `https://github.com/google/oss-fuzz/tree/master/projects/bitcoin-core`
