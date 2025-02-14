# Capturing Results

ReBench is designed to be used with existing benchmarking harnesses,
which means we need to capture the measurements created by them.
For this purpose, ReBench uses what we call 'gauge adapters'.
They typically parse the output generated by harnesses.

## Available Harness Support

ReBench currently provides builtin support for the following benchmark harnesses:

- `JMH`: [JMH](http://openjdk.java.net/projects/code-tools/jmh/), Java's microbenchmark harness
- `PlainSecondsLog`: a plain seconds log, i.e., a floating point number per line
- `ReBenchLog`: the ReBench log format, which indicates benchmark name and run time in milliseconds or microseconds
- `SavinaLog`: the harness of the [Savina](https://github.com/shamsimam/savina) benchmarks
- `ValidationLog`: the format used by [SOMns](https://github.com/smarr/SOMns)'s ImpactHarness
- `Time`: a harness that uses `/usr/bin/time` automatically

### `PlainSecondsLog`

This adapter attempts to read every line of program output as a millisecond
measurement. Lines which cannot be parsed as floats are skipped, e.g. `1.0` and
`  2 ` are valid, while `out: 1` and `1, 2, 3` are not and would be ignored.

Example output from a harness or benchmark:

```
342
543
100.23
54.12
```

Implementation Notes:

 - Python's `float()` function is used for parsing

### `ReBenchLog`

The ReBenchLog parser is the most commonly used and has most features.
It supports parsing of microseconds and milliseconds values.
Though, internally, ReBench stores all values as milliseconds using Python's
floating point numbers, i.e., as 64-bit values.

Furthermore, it supports capturing other criteria in addition to the overall
run time, which can be useful to measure the time of subtasks or metrics such
as memory use. When other criteria a provided, the `total` time is expected to
be the last in the output, concluding the overall data point.

The approximate format that `ReBenchLog` parses is as follows:

    optional_prefix benchmark_name: iterations=123 runtime: 1000[ms|us]

Example output from a harness or benchmark, each a different value for total
run time:

```
Dispatch: iterations=1 runtime: 557ms
LanguageFeatures.Dispatch: iterations=1 runtime: 309557us
LanguageFeatures.Dispatch total: iterations=2342 runtime: 557ms
```

Example output with additional criteria, that indicate memory use:

```
Savina.Chameneos: trace size:    3903398byte
Savina.Chameneos: external data: 40byte
Savina.Chameneos: iterations=1 runtime: 64208us
Savina.Chameneos: trace size:    3903414byte
Savina.Chameneos: external data: 40byte
Savina.Chameneos: iterations=1 runtime: 48581us
```

Implementation Notes:

 - For parsing of the total run time, the following regular expression is used:
   `r"^(?:.*: )?([^\s]+)( [\w\.]+)?: iterations=([0-9]+) runtime: ([0-9]+)([mu])s")`

 - For arbitrary criteria, which may also be used for the `total` criteria,
   the following regular expression should match
   `r"^(?:.*: )?([^\s]+): ([^:]{1,30}):\s*([0-9]+)([a-zA-Z]+)")`

## Supporting other Benchmark Harnesses

To add support for your own harness, check the `rebench.interop` module.
In there, the `adapter` module contains the `GaugeAdapter` base class.

The key method to implement is `parse_data(self, data, run_id, invocation)`.
The method is expected to return a list of `DataPoint` objects.
Each data point can contain a number of `Measurement` objects, where one of
them needs to be indicated as the `total` value.
The idea here is that a harness can measure different phases of a benchmark
or different properties, for instance memory usage.
These can be encoded as different measurements. The overall run time is
assumed to be the final measurement to conclude the information for a single
iteration of a benchmark.
A good example to study is the `rebench_log_adapter` implementation.
