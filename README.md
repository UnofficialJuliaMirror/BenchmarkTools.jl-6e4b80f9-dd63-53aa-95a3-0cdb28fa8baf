# BenchmarkTools.jl

[![Build Status](https://travis-ci.org/JuliaCI/BenchmarkTools.jl.svg?branch=master)](https://travis-ci.org/JuliaCI/BenchmarkTools.jl)
[![Coverage Status](https://coveralls.io/repos/github/JuliaCI/BenchmarkTools.jl/badge.svg?branch=master)](https://coveralls.io/github/JuliaCI/BenchmarkTools.jl?branch=master)

BenchmarkTools makes **performance tracking of Julia code easy** by supplying a framework for **writing and running groups of benchmarks** as well as **comparing benchmark results**.

This package is used to write and run the benchmarks found in [BaseBenchmarks.jl](https://github.com/JuliaCI/BaseBenchmarks.jl).

The CI infrastructure for automated performance testing of the Julia language is not in this package, but can found in [Nanosoldier.jl](https://github.com/JuliaCI/Nanosoldier.jl).

## Installation

BenchmarkTools isn't yet registered with Julia's package manager. To install it, you can run the following:

```julia
Pkg.clone("https://github.com/JuliaCI/BenchmarkTools.jl")
```

## Why does this package exist?

Our story begins with two packages, Benchmarks and BenchmarkTrackers. Benchmarks implemented an execution strategy for collecting and summarizing individual benchmark results, while BenchmarkTrackers implemented a framework for organizing, running, and determining regressions of groups of benchmarks. Under the hood, BenchmarkTrackers relied on Benchmarks for actual benchmark execution.

For a while, the Benchmarks + BenchmarkTrackers system was used for automated performance testing of Julia's Base library. It soon became apparent that the system suffered from a variety of issues:

1. Individual sample noise could significantly change the execution strategy used to collect further samples.
2. The estimates used to characterize benchmark results and to detect regressions were statistically vulnerable to noise (i.e. not robust).
3. Different benchmarks have different noise tolerances, but there was no way to tune this parameter on a per-benchmark basis.
4. Running benchmarks took a long time - an order of magnitude longer than theoretically necessary for many functions.
5. Using the system in the REPL (for example, to reproduce regressions locally) was often cumbersome.

The BenchmarkTools package is a response to these issues, designed by examining user reports and the benchmark data generated by the old system. BenchmarkTools offers the following solutions to the corresponding issues above:

1. Benchmark execution parameters are configured separately from the execution of the benchmark itself. This means that subsequent experiments are performed more consistently, avoiding branching "substrategies" based on individual samples.
2. A variety of simple estimators are supported, and the user can pick which one to use for regression detection.
3. Noise tolerance has been made a per-benchmark configuration parameter.
4. Benchmark configuration parameters can be easily cached and reloaded, significantly reducing benchmark execution time.
5. The API is simpler, more transparent, and overall easier to use.

## Usage Guide

#### The Benchmarking Strategy

Running a benchmark using BenchmarkTools has two steps:

1. Defining/tuning configuration parameters (e.g. how many seconds to spend benchmarking, number of samples to take, etc.)
2. Executing a "trial" in which multiple samples are gathered according to the configuration parameters.

The reasoning behind step 1 is that fast-running benchmarks need to be executed and measured differently than slow-running ones. Specifically, if the time to execute a benchmark is smaller than the resolution of your timing method, than a single evaluation of the benchmark will generally not produce a valid sample. Thus, BenchmarkTools provides a mechanism (the `tune!` method) to automatically figure out the number of evaluations per sample needed for a given benchmark.

#### `@benchmark`, `@benchmarkable`, and `tune!`

To quickly benchmark a Julia expression, use `@benchmark`:

```julia
julia> @benchmark sin(1)
BenchmarkTools.Trial:
  samples:         300
  evals/sample:    76924
  noise tolerance: 5.0%
  memory:          0.0 bytes
  allocs:          0
  minimum time:    13.0 ns (0.0% GC)
  median time:     13.0 ns (0.0% GC)
  mean time:       13.0 ns (0.0% GC)
  maximum time:    14.0 ns (0.0% GC)
```

The `@benchmark` macro is essentially shorthand for defining a benchmark, tuning the benchmark's configuration parameters, and running the benchmark. These three steps can be done explicitly using `@benchmarkable`, `tune!` and `run`:

```julia
julia> b = @benchmarkable sin(1) # define the benchmark with default parameters
BenchmarkTools.Benchmark{symbol("##benchmark#7341")}(BenchmarkTools.Parameters(5.0,300,1,true,false,0.05))

julia> tune!(b) # find the right evals/sample for this benchmark
BenchmarkTools.Benchmark{symbol("##benchmark#7341")}(BenchmarkTools.Parameters(5.0,300,76924,true,false,0.05))

julia> run(b)
BenchmarkTools.Trial:
  samples:         300
  evals/sample:    76924
  noise tolerance: 5.0%
  memory:          0.0 bytes
  allocs:          0
  minimum time:    13.0 ns (0.0% GC)
  median time:     13.0 ns (0.0% GC)
  mean time:       13.0 ns (0.0% GC)
  maximum time:    14.0 ns (0.0% GC)
```

You can pass the following keyword arguments to `@benchmark`, `@benchmarkable`, and `run` to configure the execution process:

- `samples`: The number of samples to take. Execution will end if this many samples have been collected. Defaults to `300`.
- `seconds`: The number of seconds budgeted for the benchmarking process. The trial will terminate if this time is exceeded (regardless of `samples`). Defaults to `5.0`.
- `evals`: The number of evaluations per sample. It's recommended to set this automatically via `tune!`. `@benchmark` ignores this parameter. Defaults to `1`.
- `gctrial`: If `true`, run `gc()` before executing the trial. Defaults to `true`.
- `gcsample`: If `true`, run `gc()` before each sample. Defaults to `false`.
- `tolerance`: The noise tolerance of the benchmark, as a percentage. Defaults to `0.05` (5%).

#### Examining Results: `Trial` and `TrialEstimate`

Running a benchmark produces an instance of the `Trial` type:

```julia
julia> t = @benchmark eig(rand(10, 10))
BenchmarkTools.Trial:
  samples:         300
  evals/sample:    6
  noise tolerance: 5.0%
  memory:          20.47 kb
  allocs:          83
  minimum time:    181.82 μs (0.0% GC)
  median time:     187.0 μs (0.0% GC)
  mean time:       203.16 μs (0.7% GC)
  maximum time:    776.57 μs (54.66% GC)

julia> dump(t) # let's look at fields of t
BenchmarkTools.Trial
params: BenchmarkTools.Parameters # Trials store the parameters of their parent process
  seconds: Float64 5.0
  samples: Int64 300
  evals: Int64 6
  gctrial: Bool true
  gcsample: Bool false
  tolerance: Float64 0.05
times: Array(Int64,(300,)) [181825,182299,  …  331774,776574] # every sample is stored in the Trial
gctimes: Array(Int64,(300,)) [0,0,  …  0,0,424460]
memory: Int64 20960
allocs: Int64 83
```

As you can see from the above, a couple of different timing estimates are pretty-printed with the `Trial`. You can calculate these estimates yourself using the `minimum`, `median`, `mean`, and `maximum` functions:

```julia
julia> minimum(t)
BenchmarkTools.TrialEstimate:
  time:    181.82 μs
  gctime:  0.0 ns (0.0%)
  memory:  20.47 kb
  allocs:  83
  noise tolerance: 5.0%

julia> median(t)
BenchmarkTools.TrialEstimate:
  time:    187.0 μs
  gctime:  0.0 ns (0.0%)
  memory:  20.47 kb
  allocs:  83
  noise tolerance: 5.0%

julia> mean(t)
BenchmarkTools.TrialEstimate:
  time:    203.16 μs
  gctime:  1.41 μs (0.7%)
  memory:  20.47 kb
  allocs:  83
  noise tolerance: 5.0%

julia> maximum(t)
BenchmarkTools.TrialEstimate:
  time:    776.57 μs
  gctime:  424.46 μs (54.66%)
  memory:  20.47 kb
  allocs:  83
  noise tolerance: 5.0%
```

We've found that, for most benchmarks that we've tested, the time distribution is almost always right-skewed. This phenomena can be justified by considering the machine noise that affects the benchmarking process to be inherently positive. To put it another way, there aren't really sources of noise that would regularly cause your machine to execute a series of instructions *faster* than the theoretical "ideal" time prescribed by your hardware. From this characterization of benchmark noise, we can characterize our estimators:

- The minimum is a robust estimator for the location parameter of the time distribution, and should generally not be considered an outlier
- The median, as a robust measure of central tendency, should be relatively unaffected by outliers
- The mean, as a non-robust measure of central tendency, will usually be skewed positively by outliers
- The maximum should be considered a noise-driven outlier, and can change drastically between benchmark trials.

#### Comparing Results: `TrialRatio`

BenchmarkTools supplies a `ratio` function for comparing two values:

```julia
julia> ratio(3, 2)
1.5

julia> ratio(1, 0)
Inf

julia> ratio(0, 1)
0.0

julia> ratio(0, 0) # a == b is special-cased to 1.0
1.0
```

Calling the `ratio` function on two `TrialEstimate` instances compares their fields:

```julia
julia> using BenchmarkTools

julia> b = @benchmarkable eig(rand(10, 10));

julia> tune!(b);

julia> m1 = median(run(b))
BenchmarkTools.TrialEstimate:
  time:    180.61 μs
  gctime:  0.0 ns (0.0%)
  memory:  20.47 kb
  allocs:  83
  noise tolerance: 5.0%

julia> m2 = median(run(b))
BenchmarkTools.TrialEstimate:
  time:    180.38 μs
  gctime:  0.0 ns (0.0%)
  memory:  20.47 kb
  allocs:  83
  noise tolerance: 5.0%

julia> ratio(m1, m2)
  BenchmarkTools.TrialRatio:
    time:   1.0012751106712903
    gctime: 1.0
    memory: 1.0
    allocs:  1.0
    noise tolerance: 5.0%
```

#### Classifying Regressions/Improvements: `TrialJudgement`

Use the `judge` function to decide if one estimate represents a regression versus another
estimate:

```julia
julia> m1 = median(@benchmark eig(rand(10, 10)))
BenchmarkTools.TrialEstimate:
  time:    182.28 μs
  gctime:  0.0 ns (0.0%)
  memory:  20.27 kb
  allocs:  81
  noise tolerance: 5.0%

julia> m2 = median(@benchmark eig(rand(10, 10)))
BenchmarkTools.TrialEstimate:
  time:    182.36 μs
  gctime:  0.0 ns (0.0%)
  memory:  20.67 kb
  allocs:  85
  noise tolerance: 5.0%

# percent change falls within noise tolerance for all fields
julia> judge(m1, m2)
BenchmarkTools.TrialJudgement:
  time:   -0.04% => invariant
  gctime: +0.0% => N/A
  memory: -1.97% => invariant
  allocs: -4.71% => invariant
  noise tolerance: 5.0%

# use 0.01 for noise tolerance
julia> judge(m1, m2, 0.01)
BenchmarkTools.TrialJudgement:
  time:   -0.04% => invariant
  gctime: +0.0% => N/A
  memory: -1.97% => improvement
  allocs: -4.71% => improvement
  noise tolerance: 1.0%

# switch m1 & m2
julia> judge(m2, m1, 0.01)
BenchmarkTools.TrialJudgement:
  time:   +0.04% => invariant
  gctime: +0.0% => N/A
  memory: +2.0% => regression
  allocs: +4.94% => regression
  noise tolerance: 1.0%

# you can pass in TrialRatios as well
julia> judge(ratio(m1, m2), 0.01) == judge(m1, m2, 0.01)
true
```

Note that GC isn't considered when determining regression status.

#### `BenchmarkGroup`

#### Speeding up the benchmark process by caching parameters

#### Other Info

- Actual time samples are limited to nanosecond resolution, though estimates might report fractions of nanoseconds

## Tips for getting better results

- Mention how this package can't solve low-frequency noise (sources of noise that change between trials rather than between samples), but you can configure your machine to help with this.
- Use cset
- BLAS.set_num_threads
