# libFuzzer Walkthrough (Bitcoin Core) - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/testing/fuzz`.
> Canonical source: https://github.com/bitcoin/bitcoin/blob/master/doc/fuzzing.md
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/testing/fuzz/SKILL.md

## Concept

Bitcoin Core ships dozens of libFuzzer harnesses under
`src/test/fuzz/`. Each harness is a single C++ file implementing
`FUZZ_TARGET(name)` plus optional initialisation. A unified driver
binary (`src/test/fuzz/fuzz`) selects the harness at runtime via the
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

Build with fuzzing + sanitizers:

```bash
./autogen.sh
./configure --enable-fuzz \
            --with-sanitizers=fuzzer,address,undefined \
            CC=clang CXX=clang++
make -j$(nproc)
```

Run a target with a corpus directory:

```bash
mkdir -p corpora/transaction_deserialize
FUZZ=transaction_deserialize src/test/fuzz/fuzz \
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
FUZZ=script src/test/fuzz/fuzz crash-deadbeef.bin
```

Public seed corpus:

```bash
git clone https://github.com/bitcoin-core/qa-assets.git
ln -s $PWD/qa-assets/fuzz_seed_corpus corpora
```

That ships pre-built corpora for every harness, dramatically
accelerating coverage on first run.

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
./autogen.sh
./configure --enable-fuzz --with-sanitizers=fuzzer,address,undefined \
            CC=clang CXX=clang++
make -j

# 2. Pull seed corpus
git clone --depth=1 https://github.com/bitcoin-core/qa-assets.git
mkdir -p run_corpus
cp -r qa-assets/fuzz_seed_corpus/script_interpreter/* run_corpus/

# 3. Fuzz for 10 minutes
FUZZ=script_interpreter src/test/fuzz/fuzz \
     run_corpus -max_total_time=600 -print_pcs=1

# 4. If a crash drops to disk:
FUZZ=script_interpreter src/test/fuzz/fuzz crash-1234abcd
# AddressSanitizer prints stack trace pointing at the bug
```

Continuous integration snippet (GitHub Actions):

```yaml
- name: Fuzz script interpreter
  run: |
    ./configure --enable-fuzz \
      --with-sanitizers=fuzzer,address,undefined \
      CC=clang-15 CXX=clang++-15
    make -j$(nproc) src/test/fuzz/fuzz
    FUZZ=script_interpreter src/test/fuzz/fuzz \
      qa-assets/fuzz_seed_corpus/script_interpreter \
      -runs=200000 -max_total_time=300
```

Triage workflow when a crash is found:

```bash
# Reproduce
FUZZ=foo src/test/fuzz/fuzz crash-bin > /tmp/repro.log 2>&1

# Minimise
FUZZ=foo src/test/fuzz/fuzz \
    -minimize_crash=1 \
    -runs=1000000 crash-bin

# Generate regression test from the minimised input
xxd minimised-bin | head
```

## Common pitfalls

- Build without `--with-sanitizers=fuzzer`, and the `fuzz` binary
  silently links without coverage instrumentation; you then run
  uninstrumented and find nothing.
- libFuzzer mutates inputs; if your target reads `buffer.data()`
  directly without `FuzzedDataProvider`, structured fields like
  varints rarely flip useful bits. Use `FuzzedDataProvider` to consume
  primitive types.
- Globals leak between iterations. Reset caches in
  `FUZZ_TARGET_INIT` or use `--reinit-each-iteration` style guards.
- ASan + libFuzzer use ~3-4x more RAM; on a 16 GB box, do not run more
  than 4 jobs in parallel.
- New harnesses must be added to `src/test/fuzz/fuzz_targets.cpp` (or
  CMake list) or the runtime cannot select them.

## References

- Bitcoin Core fuzzing docs: `doc/fuzzing.md`
- qa-assets corpus: `https://github.com/bitcoin-core/qa-assets`
- libFuzzer: `https://llvm.org/docs/LibFuzzer.html`
- OSS-Fuzz Bitcoin project:
  `https://github.com/google/oss-fuzz/tree/master/projects/bitcoin-core`
