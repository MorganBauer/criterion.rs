## Frequently Asked Questions

### How Should I Run Criterion.rs Benchmarks In A CI Pipeline?

Criterion.rs benchmarks can be run as part of a CI pipeline just as they
normally would on the command line - simply run `cargo bench`.

To compare the master branch to a pull request, you could run the benchmarks on
the master branch to set a baseline, then run them again with the pull request
branch. An example script for Travis-CI might be:

```bash
#!/usr/bin/env bash

if [ "${TRAVIS_PULL_REQUEST_BRANCH:-$TRAVIS_BRANCH}" != "master" ] && [ "$TRAVIS_RUST_VERSION" == "nightly" ]; then
    REMOTE_URL="$(git config --get remote.origin.url)";
    cd ${TRAVIS_BUILD_DIR}/.. && \
    git clone ${REMOTE_URL} "${TRAVIS_REPO_SLUG}-bench" && \
    cd  "${TRAVIS_REPO_SLUG}-bench" && \
    # Bench master
    git checkout master && \
    cargo bench && \
    # Bench pull request
    git checkout ${TRAVIS_COMMIT} && \
    cargo bench;
fi
```

(Thanks to [BeachApe](https://beachape.com/blog/2016/11/02/rust-performance-testing-on-travis-ci/) for the script on which this is based.)

Note that cloud CI providers like Travis-CI and Appveyor introduce a great deal
of noise into the benchmarking process. For example, unpredictable load on the
physical hosts of their build VM's. Benchmarks measured on such services tend
to be unreliable, so you should be skeptical of the results. In particular,
benchmarks that detect performance regressions should not cause the build to
fail, and apparent performance regressions should be verified manually before
rejecting a pull request.

### `cargo bench -- --verbose` Panics

This occurs because the `libtest` benchmark harness implicitly added to your
crate is executing before the Criterion.rs benchmarks, and it panics when
presented with a command-line argument it doesn't expect. There are two ways to
work around this at present:

You could run only your Criterion benchmark, like so:

`cargo bench --bench my_benchmark -- --verbose`

Note that `my_benchmark` here corresponds to the name of your benchmark in your
`Cargo.toml` file.

Another option is to disable benchmarks for your lib or app crate. For example,
for library crates, you could add this to your `Cargo.toml` file:

```toml
[lib]
bench = false
```

Of course, this only works if you define all of your benchmarks in the
`benches` directory.

See [Rust Issue #47241](https://github.com/rust-lang/rust/issues/47241) for
more details.

### How Should I Benchmark Small Functions?

Exactly the same way as you would benchmark any other function.

It is sometimes suggested that benchmarks of small (nanosecond-scale) functions should iterate the
function to be benchmarked many times internally to reduce the impact of measurement overhead.
This is _not_ required with Criterion.rs, and it is not recommended.

To see this, consider the following benchmark:

```rust
fn compare_small(c: &mut Criterion) {
    use criterion::black_box;
    use criterion::ParameterizedBenchmark;

    c.bench(
        "small",
        ParameterizedBenchmark::new("unlooped", |b, i| b.iter(|| i + 10), vec![10])
            .with_function("looped", |b, i| b.iter(|| {
                for _ in 0..10000 {
                    black_box(i + 10);
                }
            }))
    );
}
```

This benchmark simply adds two numbers - just about the smallest function that could be performed.
On my computer, this produces the following output:

```
small/unlooped          time:   [270.00 ps 270.78 ps 271.56 ps]
Found 2 outliers among 100 measurements (2.00%)
  2 (2.00%) high severe
small/looped            time:   [2.7051 us 2.7142 us 2.7238 us]
Found 5 outliers among 100 measurements (5.00%)
  3 (3.00%) high mild
  2 (2.00%) high severe
```

2.714 microseconds/10000 gives 271.4 picoseconds, or pretty much the same
result. Interestingly, this is slightly more than one cycle of my 4th-gen Core
i7's maximum clock frequency of 4.4 GHz, which shows how good the pipelining is
on modern CPU's. Regardless, Criterion.rs is able to accurately measure
functions all the way down to single instructions. See the [Analysis
Process](./analysis.md) page for more details on how Criterion.rs performs its
measurements.
