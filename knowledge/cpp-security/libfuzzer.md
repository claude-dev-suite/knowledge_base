# C++ Security - LibFuzzer

> **Source**: https://llvm.org/docs/LibFuzzer.html
> **Skill**: dev-suite skill `security/cpp-security` — see SKILL.md for the always-loaded quick reference.

## What this covers

Writing a fuzz harness for a real parser, structuring corpora and dictionaries for high coverage,
combining LibFuzzer with sanitizers for triage, and the OSS-Fuzz integration pattern for
continuous fuzzing.

## Deep dive

### The fuzz harness

LibFuzzer drives a single function:

```cpp
// fuzz_parser.cpp
#include <cstdint>
#include <cstddef>
#include <span>
#include "parser.hpp"

extern "C" int LLVMFuzzerTestOneInput(const uint8_t* data, size_t size) {
    if (size > 1024 * 1024) return 0;     // size limit to keep iterations fast

    try {
        Document doc = parse(std::span<const uint8_t>{data, size});
        // Optional: round-trip and compare
        auto serialized = doc.serialize();
        Document re_parsed = parse(serialized);
        if (re_parsed != doc) {
            __builtin_trap();              // signals a logic bug to the fuzzer
        }
    } catch (const ParseError&) {
        // expected for malformed input
    }
    return 0;
}
```

Build:

```bash
clang++ -fsanitize=fuzzer,address,undefined \
        -fno-omit-frame-pointer -O1 -g \
        -std=c++20 \
        fuzz_parser.cpp parser.cpp -o fuzz_parser
```

`-fsanitize=fuzzer` provides the `main()` and the coverage-guided engine. ASan + UBSan turn
exploitable bugs into immediate aborts the fuzzer can record.

### Corpus layout

```
corpus/                   # input directory; each file is one test case
  empty.bin               # zero bytes
  minimal.json
  edge_unicode.json
  prior_crashes/          # add fixed crashes to prevent regression
    crash-abc123
```

Run:

```bash
./fuzz_parser corpus/             # corpus directory (read-write — fuzzer adds new findings)
./fuzz_parser -max_total_time=600 corpus/
./fuzz_parser -max_len=4096 -dict=parser.dict corpus/
```

The fuzzer mutates inputs from the corpus and writes interesting new ones (those that hit new
coverage) back into the directory.

### Dictionaries — prime the mutator with structural tokens

`parser.dict`:
```
"true"
"false"
"null"
"\"key\":"
"\\u0000"
"\\xff\\xfe"
```

Each line is one token to be inserted/replaced during mutation. A good dictionary can improve
coverage by 10x for structured formats (JSON, HTTP, ASN.1).

### Persistent vs in-process modes

LibFuzzer is in-process by default — fastest iteration, but a crash kills the run. To maximize
coverage discovery, run in-process; for stress/longevity use a wrapper script:

```bash
while ./fuzz_parser corpus/; do : ; done   # restart on crash
```

Persistent fork mode (`-fork=N`) runs N child processes and respawns crashed ones automatically.
Useful for parallelism on a single machine:

```bash
./fuzz_parser -fork=$(nproc) -ignore_crashes=1 -max_total_time=3600 corpus/
```

### Crash triage

A crash is written to `./crash-<sha1>`. Reproduce:

```bash
./fuzz_parser ./crash-abc123              # rerun on that single input
```

Convert to a permanent regression test:

```cpp
TEST(ParserRegression, CrashAbc123) {
    std::vector<uint8_t> input = read_file("test_data/crash-abc123");
    EXPECT_NO_THROW(parse(std::span{input.data(), input.size()}));
}
```

### Coverage measurement

```bash
clang++ -fsanitize=fuzzer,address -fprofile-instr-generate -fcoverage-mapping ...
LLVM_PROFILE_FILE=fuzz.profraw ./fuzz_parser -runs=10000 corpus/
llvm-profdata merge -sparse fuzz.profraw -o fuzz.profdata
llvm-cov show ./fuzz_parser -instr-profile=fuzz.profdata -format=html > coverage.html
```

Aim for >80% coverage of parser code paths before declaring "well fuzzed."

### Combining with sanitizers — pick wisely

| Combo | Use case |
|-------|----------|
| `fuzzer,address,undefined` | Default — finds memory bugs and UB |
| `fuzzer,memory` | Uninitialized reads (need MSan-built libc++) |
| `fuzzer,thread` | Race conditions in parser if multithreaded |
| `fuzzer` alone | Coverage-guided exploration; logic bugs only |

You can't combine ASan + MSan + TSan, so pick the right pair for your bug class.

### OSS-Fuzz integration

For OSS projects, OSS-Fuzz runs your harnesses continuously on Google infrastructure for free.
Integration is a `Dockerfile`, `build.sh`, and `project.yaml`:

```dockerfile
# Dockerfile
FROM gcr.io/oss-fuzz-base/base-builder
RUN apt-get update && apt-get install -y cmake ninja-build
COPY . $SRC/myproject
WORKDIR $SRC/myproject
COPY build.sh $SRC/
```

```bash
# build.sh
cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=Release \
      -DCMAKE_CXX_FLAGS="$CXXFLAGS" -DCMAKE_C_FLAGS="$CFLAGS" \
      -DCMAKE_EXE_LINKER_FLAGS="$LIB_FUZZING_ENGINE"
cmake --build build --target fuzz_parser
cp build/fuzz_parser $OUT/
[ -d corpus ] && zip -j $OUT/fuzz_parser_seed_corpus.zip corpus/*
[ -f parser.dict ] && cp parser.dict $OUT/fuzz_parser.dict
```

OSS-Fuzz runs ~10 sanitizer variants daily and files private security issues for crashes.

### Best practices for harnesses

1. **One harness per format / API surface.** Don't multiplex inputs through a single dispatcher.
2. **Guard expensive ops** behind a size threshold (return early on >1 MB inputs).
3. **Disable logging** during fuzzing — crashes amplified by syslog volume kill performance.
4. **Mark expected exceptions as success.** Only crashes / hangs / timeouts are bugs.
5. **Add seed corpus from real-world inputs** — reduces the time to first interesting coverage by
   orders of magnitude.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Harness returns non-zero | LibFuzzer treats it as a crash | Always `return 0` (the only valid value) |
| Static state across `LLVMFuzzerTestOneInput` calls | Cross-input state poisons | Use process state sparingly; reset per call |
| No size cap | One huge mutation eats the budget | Early `return 0` on absurd sizes |
| Catching `...` in the harness when SUT genuinely crashes | Hides bugs | Catch only the parser's expected exception type |
| Logging on stderr during fuzzing | Massive log → slow → low iter/s | Quiet mode for the fuzz binary |
| Building fuzzer with `-O0` | 10x slower exploration | `-O1` minimum (LibFuzzer needs frame pointers) |
| Forgetting the dictionary | 10x worse coverage on structured inputs | Always craft a dict for the format |

## See also

- https://github.com/google/fuzzing/blob/master/tutorial/libFuzzerTutorial.md
- https://google.github.io/oss-fuzz/getting-started/new-project-guide/
