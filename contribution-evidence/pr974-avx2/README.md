# AVX2 simple-escape validation evidence

Source: [57b98f2](https://github.com/WanSuiMm/sonic/commit/57b98f2678d5ee897525942edfd53b3922b48c51),
based on [Sonic #974](https://github.com/bytedance/sonic/pull/974) at
`0c260f4d13d748274c030e153ff0398657fd57be`.
The [optimization-only comparison](https://github.com/WanSuiMm/sonic/compare/0c260f4d13d748274c030e153ff0398657fd57be...57b98f2678d5ee897525942edfd53b3922b48c51)
contains 14 files: a C header, one test/benchmark file and twelve generated
AVX2 files. This evidence branch is separate from the source PR branch.

Read this page, then [validation](validation.json) and
[generation replay](generation_replay.json). The complete
[benchmark aggregates](benchmark_summary.json) retain all rounds/ranges and
are secondary. [Source hashes](source_freeze.json) bind the exact tested
content; [evidence_manifest.json](evidence_manifest.json) records derivative
file hashes. These compact derivatives preserve the frozen results; raw logs,
binaries and machine-specific receipts are retained privately.

## Question and intervention

Can #974's validating scanner avoid enumerating every simple escape in a
dense block? The candidate reuses quote/backslash masks and uses four AVX2
comparisons for the other simple escape letters. A block takes this path only
without an unescaped quote, raw control, outgoing escape carry or pending
Unicode continuation. Unicode, malformed input and boundary cases retain the
existing validation loop. Unicode-first blocks avoid a redundant classifier
attempt. SSE/ARM generated files, public APIs, ABI bridges and CPU requirements
are unchanged.

## Controlled measurements

One pinned Intel Core Ultra 9 285K core, Linux amd64, Go 1.27.2,
GOMAXPROCS=1, ConfigStd JIT, auto/noavx2. Four variants are measured: shipped
#974, unchanged #974 regenerated locally (control), earlier AVX2 prototype,
and the final candidate. Both regenerated controls and candidates use clang
13.0.1, asm2asm `7551cd4109d65d83475ccf98ac3a6844c774e44f`, the same target
and flags. No native caller/ABI patch is applied.

Each of 20 input families has unknown-field and known-field destinations,
three paired 100ms rounds, with separate 20ms warmups excluded. Mode/variant
order rotates. Published aggregates retain medians, ranges, paired ratios,
ratios of medians, bytes/op and allocations/op. Binaries were built from the
same focused issue-test sources for every variant; this avoids unrelated
issue-module fixture initialization in the deployed timing binaries.

The primary denominator is the same-toolchain regenerated control, because
regeneration alone changes timing. Shipped-parent comparisons remain separate.
All bodies below contain 4,096 encoded bytes; values are median ns/op.

| Simple-escape body | Unknown control -> final | Speedup | Known control -> final | Speedup |
|---|---:|---:|---:|---:|
| Newline | 2751 -> 1076 | 2.56x | 4790 -> 2921 | 1.64x |
| Quote | 2758 -> 1074 | 2.57x | 4813 -> 2919 | 1.65x |
| Backslash | 3871 -> 1078 | 3.59x | 5942 -> 2884 | 2.06x |
| Eight simple escapes mixed | 2859 -> 1069 | 2.67x | 4928 -> 2890 | 1.71x |

Unicode-dense unknown fields: control / earlier prototype / final are
2657 / 2792 / 2646 ns/op. The former 5.1% regression is absent in this run;
the final 0.4% difference is not a significance claim. Mixed Unicode is
2.0% slower unknown and 2.7% slower known than control. Surrogate pairs remain
near control. The largest ordinary-case median increase is 4.37% for known
Plain256; this is not a claim that all inputs improve.

The 20 families cover plain lengths 16/32/64/256/4096; sparse lengths
32/64/256/4096; four dense simple-escape patterns; Unicode, surrogate and
mixed Unicode; and mixed simple escapes at lengths 16/32/64/128.
All 72 predefined gates pass: dense-simple AVX2 cases require at least 20%
lower median time than both controls; remaining cases allow up to 5% higher
median time. SSE dense-simple rows are descriptive; no SSE speedup is claimed.

## Validation and replay

- The finite catalog has 6,440 unique inputs and 19,320 ConfigStd destination
  comparisons per setting. Four settings (default/optimized x auto/noavx2)
  pass for the final Windows candidate and all four Linux variants, with
  zero ConfigStd acceptance/value mismatches. The same corpus is reused;
  these are not independent fuzz samples.
- Go 1.27.3 Linux full root and full nested issue-module suites pass.
  Focused issue/native/decoder suites pass under Go 1.27.3 and 1.26.10 with
  default and optimized configurations, ten invocations total.
- All twelve AVX2 generated files replay byte-for-byte using the pinned
  generator and trailing-whitespace normalization. Source freeze and
  downloaded archive/binary hashes were independently checked.

The added public tests cover dense escapes, Unicode/surrogate boundaries,
escaped backslash followed by literal x, raw controls and malformed Unicode
after a dense simple prefix. The benchmark is in
[issue938_dense_test.go](https://github.com/WanSuiMm/sonic/blob/57b98f2678d5ee897525942edfd53b3922b48c51/issue_test/issue938_dense_test.go).
For an ordinary checkout with the required Go SDK/dependencies:

```sh
cd issue_test
go test -ldflags=-checklinkname=0 -run '^TestIssue938_' -count=1 .
SONIC_USE_OPTDEC=0 SONIC_USE_FASTMAP=0 SONIC_ENCODER_USE_VM=0 \
  SONIC_MODE=auto GOMAXPROCS=1 go test -ldflags=-checklinkname=0 -run '^$' \
  -bench '^BenchmarkIssue938_DenseEscape$' -benchtime=100ms -count=3 -benchmem .
```

The table uses the separately regenerated control; comparing the shipped
parent alone includes generation drift. The parent control must receive the
same benchmark file, SDK and generation toolchain for that comparison.

ARM runtime, full Windows/macOS suites, the entire upstream Go/race matrix,
production JSON distributions and statistical generalization beyond these
three microbenchmark rounds are unverified. An earlier dual-ISA prototype
failed SSE/X15 qualification and was excluded before timing; the current
candidate leaves SSE untouched. Existing #974 error-position behavior is
unchanged.
