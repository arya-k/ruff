# Contributing to Ruff

Welcome! We're happy to have you here. Thank you in advance for your contribution to Ruff.

- [The Basics](#the-basics)
  - [Prerequisites](#prerequisites)
  - [Development](#development)
  - [Project Structure](#project-structure)
  - [Example: Adding a new lint rule](#example-adding-a-new-lint-rule)
    - [Rule naming convention](#rule-naming-convention)
  - [Example: Adding a new configuration option](#example-adding-a-new-configuration-option)
- [MkDocs](#mkdocs)
- [Release Process](#release-process)
- [Benchmarks](#benchmarks)

## The Basics

Ruff welcomes contributions in the form of Pull Requests.

For small changes (e.g., bug fixes), feel free to submit a PR.

For larger changes (e.g., new lint rules, new functionality, new configuration options), consider
creating an [**issue**](https://github.com/charliermarsh/ruff/issues) outlining your proposed
change. You can also join us on [**Discord**](https://discord.gg/c9MhzV8aU5) to discuss your idea with
the community.

If you're looking for a place to start, we recommend implementing a new lint rule (see:
[_Adding a new lint rule_](#example-adding-a-new-lint-rule), which will allow you to learn from and
pattern-match against the examples in the existing codebase. Many lint rules are inspired by
existing Python plugins, which can be used as a reference implementation.

As a concrete example: consider taking on one of the rules from the [`flake8-pyi`](https://github.com/charliermarsh/ruff/issues/848)
plugin, and looking to the originating [Python source](https://github.com/PyCQA/flake8-pyi)
for guidance.

### Prerequisites

Ruff is written in Rust. You'll need to install the
[Rust toolchain](https://www.rust-lang.org/tools/install) for development.

You'll also need [Insta](https://insta.rs/docs/) to update snapshot tests:

```shell
cargo install cargo-insta
```

### Development

After cloning the repository, run Ruff locally with:

```shell
cargo run -p ruff_cli -- check /path/to/file.py --no-cache
```

Prior to opening a pull request, ensure that your code has been auto-formatted,
and that it passes both the lint and test validation checks:

```shell
cargo fmt  # Auto-formatting...
cargo clippy --fix --workspace --all-targets --all-features  # Linting...
cargo test  # Testing...
```

These checks will run on GitHub Actions when you open your Pull Request, but running them locally
will save you time and expedite the merge process.

Note that many code changes also require updating the snapshot tests, which is done interactively
after running `cargo test` like so:

```shell
cargo insta review
```

If you have `pre-commit` [installed](https://pre-commit.com/#installation) then you can use it to
assist with formatting and linting. The following command will run the `pre-commit` hooks:

```shell
pre-commit run --all-files
```

Your Pull Request will be reviewed by a maintainer, which may involve a few rounds of iteration
prior to merging.

### Project Structure

Ruff is structured as a monorepo with a [flat crate structure](https://matklad.github.io/2021/08/22/large-rust-workspaces.html),
such that all crates are contained in a flat `crates` directory.

The vast majority of the code, including all lint rules, lives in the `ruff` crate (located at
`crates/ruff`). As a contributor, that's the crate that'll be most relevant to you.

At time of writing, the repository includes the following crates:

- `crates/ruff`: library crate containing all lint rules and the core logic for running them.
- `crates/ruff_cli`: binary crate containing Ruff's command-line interface.
- `crates/ruff_dev`: binary crate containing utilities used in the development of Ruff itself (e.g., `cargo dev generate-all`).
- `crates/ruff_macros`: library crate containing macros used by Ruff.
- `crates/ruff_python`: library crate implementing Python-specific functionality (e.g., lists of standard library modules by versionb).
- `crates/flake8_to_ruff`: binary crate for generating Ruff configuration from Flake8 configuration.

### Example: Adding a new lint rule

At a high level, the steps involved in adding a new lint rule are as follows:

1. Determine a name for the new rule as per our [rule naming convention](#rule-naming-convention).
1. Create a file for your rule (e.g., `crates/ruff/src/rules/flake8_bugbear/rules/abstract_base_class.rs`).
1. In that file, define a violation struct. You can grep for `#[violation]` to see examples.
1. Map the violation struct to a rule code in `crates/ruff/src/registry.rs` (e.g., `E402`).
1. Define the logic for triggering the violation in `crates/ruff/src/checkers/ast.rs` (for AST-based
   checks), `crates/ruff/src/checkers/tokens.rs` (for token-based checks), `crates/ruff/src/checkers/lines.rs`
   (for text-based checks), or `crates/ruff/src/checkers/filesystem.rs` (for filesystem-based
   checks).
1. Add a test fixture.
1. Update the generated files (documentation and generated code).

To define the violation, start by creating a dedicated file for your rule under the appropriate
rule linter (e.g., `crates/ruff/src/rules/flake8_bugbear/rules/abstract_base_class.rs`). That file should
contain a struct defined via `#[violation]`, along with a function that creates the violation
based on any required inputs.

To trigger the violation, you'll likely want to augment the logic in `crates/ruff/src/checkers/ast.rs`,
which defines the Python AST visitor, responsible for iterating over the abstract syntax tree and
collecting diagnostics as it goes.

If you need to inspect the AST, you can run `cargo dev print-ast` with a Python file. Grep
for the `Check::new` invocations to understand how other, similar rules are implemented.

To add a test fixture, create a file under `crates/ruff/resources/test/fixtures/[linter]`, named to match
the code you defined earlier (e.g., `crates/ruff/resources/test/fixtures/pycodestyle/E402.py`). This file should
contain a variety of violations and non-violations designed to evaluate and demonstrate the behavior
of your lint rule.

Run `cargo dev generate-all` to generate the code for your new fixture. Then run Ruff
locally with (e.g.) `cargo run -p ruff_cli -- check crates/ruff/resources/test/fixtures/pycodestyle/E402.py --no-cache --select E402`.

Once you're satisfied with the output, codify the behavior as a snapshot test by adding a new
`test_case` macro in the relevant `crates/ruff/src/[linter]/mod.rs` file. Then, run `cargo test`.
Your test will fail, but you'll be prompted to follow-up with `cargo insta review`. Accept the
generated snapshot, then commit the snapshot file alongside the rest of your changes.

Finally, regenerate the documentation and generated code with `cargo dev generate-all`.

#### Rule naming convention

The rule name should make sense when read as "allow _rule-name_" or "allow _rule-name_ items".

This implies that rule names:

- should state the bad thing being checked for

- should not contain instructions on what you what you should use instead
  (these belong in the rule documentation and the `autofix_title` for rules that have autofix)

When re-implementing rules from other linters, this convention is given more importance than
preserving the original rule name.

### Example: Adding a new configuration option

Ruff's user-facing settings live in a few different places.

First, the command-line options are defined via the `Cli` struct in `crates/ruff/src/cli.rs`.

Second, the `pyproject.toml` options are defined in `crates/ruff/src/settings/options.rs` (via the
`Options` struct), `crates/ruff/src/settings/configuration.rs` (via the `Configuration` struct), and
`crates/ruff/src/settings/mod.rs` (via the `Settings` struct). These represent, respectively: the
schema used to parse the `pyproject.toml` file; an internal, intermediate representation; and the
final, internal representation used to power Ruff.

To add a new configuration option, you'll likely want to modify these latter few files (along with
`cli.rs`, if appropriate). If you want to pattern-match against an existing example, grep for
`dummy_variable_rgx`, which defines a regular expression to match against acceptable unused
variables (e.g., `_`).

Note that plugin-specific configuration options are defined in their own modules (e.g.,
`crates/ruff/src/flake8_unused_arguments/settings.rs`).

You may also want to add the new configuration option to the `flake8-to-ruff` tool, which is
responsible for converting `flake8` configuration files to Ruff's TOML format. This logic
lives in `crates/ruff/src/flake8_to_ruff/converter.rs`.

Finally, regenerate the documentation and generated code with `cargo dev generate-all`.

## MkDocs

To preview any changes to the documentation locally:

1. Install MkDocs and Material for MkDocs with:

   ```shell
   pip install -r docs/requirements.txt
   ```

1. Generate the MkDocs site with:

   ```shell
   python scripts/generate_mkdocs.py
   ```

1. Run the development server with:

   ```shell
   mkdocs serve
   ```

The documentation should then be available locally at
[http://127.0.0.1:8000/docs/](http://127.0.0.1:8000/docs/).

## Release Process

As of now, Ruff has an ad hoc release process: releases are cut with high frequency via GitHub
Actions, which automatically generates the appropriate wheels across architectures and publishes
them to [PyPI](https://pypi.org/project/ruff/).

Ruff follows the [semver](https://semver.org/) versioning standard. However, as pre-1.0 software,
even patch releases may contain [non-backwards-compatible changes](https://semver.org/#spec-item-4).

## Ecosystem CI

GitHub Actions will run your changes against a number of real-world projects from GitHub and
report on any diagnostic differences. You can also run those checks locally via:

```shell
python scripts/check_ecosystem.py path/to/your/ruff path/to/older/ruff
```

You can also run the Ecosystem CI check in a Docker container across a larger set of projects by
downloading the [`known-github-tomls.json`](https://github.com/akx/ruff-usage-aggregate/blob/master/data/known-github-tomls.jsonl)
as `github_search.jsonl` and following the instructions in [scripts/Dockerfile.ecosystem](https://github.com/charliermarsh/ruff/blob/main/scripts/Dockerfile.ecosystem).
Note that this check will take a while to run.

## Benchmarks

First, clone [CPython](https://github.com/python/cpython). It's a large and diverse Python codebase,
which makes it a good target for benchmarking.

```shell
git clone --branch 3.10 https://github.com/python/cpython.git crates/ruff/resources/test/cpython
```

To benchmark the release build:

```shell
cargo build --release && hyperfine --ignore-failure --warmup 10 \
  "./target/release/ruff ./crates/ruff/resources/test/cpython/ --no-cache" \
  "./target/release/ruff ./crates/ruff/resources/test/cpython/"

Benchmark 1: ./target/release/ruff ./crates/ruff/resources/test/cpython/ --no-cache
  Time (mean ± σ):     293.8 ms ±   3.2 ms    [User: 2384.6 ms, System: 90.3 ms]
  Range (min … max):   289.9 ms … 301.6 ms    10 runs

  Warning: Ignoring non-zero exit code.

Benchmark 2: ./target/release/ruff ./crates/ruff/resources/test/cpython/
  Time (mean ± σ):      48.0 ms ±   3.1 ms    [User: 65.2 ms, System: 124.7 ms]
  Range (min … max):    45.0 ms …  66.7 ms    62 runs

  Warning: Ignoring non-zero exit code.

Summary
  './target/release/ruff ./crates/ruff/resources/test/cpython/' ran
    6.12 ± 0.41 times faster than './target/release/ruff ./crates/ruff/resources/test/cpython/ --no-cache'
```

To benchmark against the ecosystem's existing tools:

```shell
hyperfine --ignore-failure --warmup 5 \
  "./target/release/ruff ./crates/ruff/resources/test/cpython/ --no-cache" \
  "pyflakes crates/ruff/resources/test/cpython" \
  "autoflake --recursive --expand-star-imports --remove-all-unused-imports --remove-unused-variables --remove-duplicate-keys resources/test/cpython" \
  "pycodestyle crates/ruff/resources/test/cpython" \
  "flake8 crates/ruff/resources/test/cpython"

Benchmark 1: ./target/release/ruff ./crates/ruff/resources/test/cpython/ --no-cache
  Time (mean ± σ):     294.3 ms ±   3.3 ms    [User: 2467.5 ms, System: 89.6 ms]
  Range (min … max):   291.1 ms … 302.8 ms    10 runs

  Warning: Ignoring non-zero exit code.

Benchmark 2: pyflakes crates/ruff/resources/test/cpython
  Time (mean ± σ):     15.786 s ±  0.143 s    [User: 15.560 s, System: 0.214 s]
  Range (min … max):   15.640 s … 16.157 s    10 runs

  Warning: Ignoring non-zero exit code.

Benchmark 3: autoflake --recursive --expand-star-imports --remove-all-unused-imports --remove-unused-variables --remove-duplicate-keys resources/test/cpython
  Time (mean ± σ):      6.175 s ±  0.169 s    [User: 54.102 s, System: 1.057 s]
  Range (min … max):    5.950 s …  6.391 s    10 runs

Benchmark 4: pycodestyle crates/ruff/resources/test/cpython
  Time (mean ± σ):     46.921 s ±  0.508 s    [User: 46.699 s, System: 0.202 s]
  Range (min … max):   46.171 s … 47.863 s    10 runs

  Warning: Ignoring non-zero exit code.

Benchmark 5: flake8 crates/ruff/resources/test/cpython
  Time (mean ± σ):     12.260 s ±  0.321 s    [User: 102.934 s, System: 1.230 s]
  Range (min … max):   11.848 s … 12.933 s    10 runs

  Warning: Ignoring non-zero exit code.

Summary
  './target/release/ruff ./crates/ruff/resources/test/cpython/ --no-cache' ran
   20.98 ± 0.62 times faster than 'autoflake --recursive --expand-star-imports --remove-all-unused-imports --remove-unused-variables --remove-duplicate-keys resources/test/cpython'
   41.66 ± 1.18 times faster than 'flake8 crates/ruff/resources/test/cpython'
   53.64 ± 0.77 times faster than 'pyflakes crates/ruff/resources/test/cpython'
  159.43 ± 2.48 times faster than 'pycodestyle crates/ruff/resources/test/cpython'
```

You can run `poetry install` from `./scripts` to create a working environment for the above. All
reported benchmarks were computed using the versions specified by `./scripts/pyproject.toml`
on Python 3.11.

To benchmark Pylint, remove the following files from the CPython repository:

```shell
rm Lib/test/bad_coding.py \
  Lib/test/bad_coding2.py \
  Lib/test/bad_getattr.py \
  Lib/test/bad_getattr2.py \
  Lib/test/bad_getattr3.py \
  Lib/test/badcert.pem \
  Lib/test/badkey.pem \
  Lib/test/badsyntax_3131.py \
  Lib/test/badsyntax_future10.py \
  Lib/test/badsyntax_future3.py \
  Lib/test/badsyntax_future4.py \
  Lib/test/badsyntax_future5.py \
  Lib/test/badsyntax_future6.py \
  Lib/test/badsyntax_future7.py \
  Lib/test/badsyntax_future8.py \
  Lib/test/badsyntax_future9.py \
  Lib/test/badsyntax_pep3120.py \
  Lib/test/test_asyncio/test_runners.py \
  Lib/test/test_copy.py \
  Lib/test/test_inspect.py \
  Lib/test/test_typing.py
```

Then, from `crates/ruff/resources/test/cpython`, run: `time pylint -j 0 -E $(git ls-files '*.py')`. This
will execute Pylint with maximum parallelism and only report errors.

To benchmark Pyupgrade, run the following from `crates/ruff/resources/test/cpython`:

```shell
hyperfine --ignore-failure --warmup 5 --prepare "git reset --hard HEAD" \
  "find . -type f -name \"*.py\" | xargs -P 0 pyupgrade --py311-plus"

Benchmark 1: find . -type f -name "*.py" | xargs -P 0 pyupgrade --py311-plus
  Time (mean ± σ):     30.119 s ±  0.195 s    [User: 28.638 s, System: 0.390 s]
  Range (min … max):   29.813 s … 30.356 s    10 runs
```
