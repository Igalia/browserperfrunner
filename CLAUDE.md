# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

`browserperfrunner` automates running browser performance benchmarks (Speedometer, JetStream, MotionMark,
etc.) and optionally uploading the results to a dashboard
([browserperfdash](https://github.com/Igalia/browserperfdash)). It exists as a standalone repo so it can be
run on embedded/constrained boards without cloning the full WebKit repository. It is packaged for Yocto via
the `browserperfrunner` recipe in
[meta-webkit](https://github.com/Igalia/meta-webkit/blob/master/recipes-benchmark/browserperfrunner/browserperfrunner_git.bb).

There are two entry-point scripts:

- **`run-benchmark`** — runs a single benchmark plan locally and writes results to a JSON file.
- **`browserperfdash-benchmark`** — a wrapper around the same benchmark-running machinery that additionally
  uploads results (via HTTP POST) to one or more `browserperfdash` server instances. Intended to run
  continuously on a bot across different browser versions.

## Where the code actually comes from

Almost all of the Python code under `webkitpy/` and `libraries/` is **vendored from the WebKit repository's
`Tools/Scripts/`** — this repo does not maintain it independently. `devtools/resetcodefromWKrepo` is the
tool used to refresh it:

```
devtools/resetcodefromWKrepo /path/to/local/webkit/checkout
```

It wipes `run-benchmark`, `browserperfdash-benchmark`, `libraries/`, and `webkitpy/`, then copies back the
transitive import closure of the entry points from `${WKREPO}/Tools/Scripts/`: the `benchmark_runner` and
`browserperfdash` packages wholesale, the four local libraries that `webkitpy/__init__.py` registers as
local AutoInstall packages (`webkitcorepy`, `webkitscmpy`, `webkitbugspy`, `webkitexpectationspy` — all
required just to `import webkitpy`), the individual `webkitpy` modules they import, and the `gtk`/`wpe`
port modules that `PortFactory` loads dynamically on Linux (needed by the `browser-binary-size` plan
plugin). `webkitpy/common/net/bugzilla` and `.../buildbot` are deliberately excluded: they're only reachable
through lazy `host.bugs`/`host.buildbot` properties, and bugzilla can't even be imported outside a WebKit
checkout (it reads `metadata/contributors.json` at import time — and `unittest discover` imports every
package `__init__` it walks, so copying it breaks `runtests`).

When a re-sync adds new upstream imports, the script's copy list must grow with them. Read that script
before assuming any given `webkitpy` file is meant to be hand-edited here — changes to vendored code are
generally better made upstream in WebKit and re-synced, not patched locally.

After resetting, run `./devtools/runtests` to verify all dependencies were copied, plus
`./run-benchmark --list-plans` and `./browserperfdash-benchmark --config-file <cfg> --browser-version x
--list-plans` as smoke tests (the latter exercises the plugin loader and its `port`/`host` import chain).
First import triggers AutoInstall downloads into `libraries/autoinstalled` (gitignored).

## Commands

```bash
# Run the test suite (discovers all *_unittest.py files)
./devtools/runtests
# equivalent to:
python -m unittest discover -p '*_unittest.py'

# List available benchmark plans
./run-benchmark --list-plans

# Run one plan
./run-benchmark --plan speedometer2 --browser minibrowser-wpe --platform linux

# Run every plan in sequence
./run-benchmark --allplans

# Run a benchmark and upload the result to a browserperfdash server
./browserperfdash-benchmark --plan speedometer2 \
    --config-file /etc/browserperfdash-benchmark-config.txt \
    --browser minibrowser-wpe --browser-version <version>
```

Test config lives in `webkitpy/pytest.ini` / `webkitpy/tox.ini` (pytest-based; also runnable via
`python -m unittest`). `webkitpy/pylintrc` configures linting.

Linux runtime dependencies (see `README.md`):
```
sudo apt-get install curl make patch perl libarchive-zip-perl procps psmisc ruby \
     subversion python2.7 python-psutil python-setuptools python-gobject gir1.2-gtk-3.0
```

## Architecture

### Benchmark execution (`webkitpy/benchmark_runner/`)

This is the core engine, shared by both `run-benchmark` and `browserperfdash-benchmark`:

- **`run_benchmark.py`** — CLI arg parsing and top-level `main()`/`start()` used directly by `run-benchmark`.
- **`benchmark_runner.py`** (`BenchmarkRunner`) — abstract base class. Loads a `.plan` file (JSON, under
  `data/plans/*.plan`), builds the benchmark web content via `BenchmarkBuilder`, drives iterations through a
  `BrowserDriver`, merges/dumps results to JSON, and prints a formatted summary
  (`BenchmarkResults`/`benchmark_results.py`). Plans can `import_plan_file` another plan and can declare
  `subtests`.
- Two concrete runners subclass it, selected via `--driver`:
  - **`webserver_benchmark_runner.py`** (`WebServerBenchmarkRunner`, driver name `webserver`, the default) —
    serves the benchmark over a local HTTP server (`http_server_driver/`, builtin or twisted) and has the
    browser page POST results back to `/report`.
  - **`webdriver_benchmark_runner.py`** (`WebDriverBenchmarkRunner`, driver name `webdriver`) — drives the
    browser via Selenium WebDriver and pulls `window.webdriver_results` directly.
- **`browser_driver/`** — one `BrowserDriver` subclass per (platform, browser) pair (e.g.
  `linux_minibrowserwpe_driver.py`, `linux_cog_driver.py`, `linux_chrome_driver.py`,
  `linux_epiphany_driver.py`, `linux_servo_driver.py`, `osx_safari_driver.py`, ...). Drivers **self-register**
  on import: `browser_driver/__init__.py` calls `load_subclasses(...)` to discover every `BrowserDriver`
  subclass in the directory and registers it into `BrowserDriverFactory` (`browser_driver_factory.py`) keyed
  by `(platform, browser_name)` class attributes. To add support for a new browser/platform, add a new driver
  module here — no other registration step needed. `browser_driver_manual.py` is the "manual" driver (prints
  the URL for a human to open, useful with `--http-server-port` when testing devices you can't automate).
- **`data/plans/*.plan`** — JSON benchmark plan definitions (Speedometer, JetStream, MotionMark, Sunspider,
  etc., including versioned variants). `data/plans/Skipped` lists plan names to skip when running
  `--allplans`. `data/patches/` holds patches applied to upstream benchmark sources; `data/firefox_profile/`
  holds a Firefox profile for that driver.
- **`generic_factory.py`** / `utils.py` — small shared factory/helper utilities used by the driver factories.

### Dashboard upload (`webkitpy/browserperfdash/`)

- **`browserperfdash_runner.py`** (`BrowserPerfDashRunner`) — used by the `browserperfdash-benchmark` entry
  point. Parses an INI-style config file (`config-file-example.txt` documents the format) describing one or
  more target dashboard servers (`bot_id`/`bot_password`/`post_url`) plus optional `[settings]` /
  `[settings:<name>]` sections for default CLI arg values and named plan lists (used with
  `--plans-from-config`). It runs a plan through the same `BenchmarkRunner` subclasses as `run-benchmark`,
  then POSTs the resulting JSON to each configured server, handling the long-poll `202` streaming-response
  protocol that the `browserperfdash` Django app uses to avoid timeouts.
  Plan version is a hash of the plan file plus any patches it references, used to detect drift on the
  dashboard side. Supports splitting a plan's subtests into individual runs (`<plan>-split-subtests`).
- **`plans/`** — pluggable "extra" plans that aren't `benchmark_runner` `.plan` files. Any `*.py` in this
  directory exposing a module-level `register()` function returning `{'name': ..., 'run': ...}` is picked up
  automatically as an additional plan `browserperfdash-benchmark` can run (see
  `_load_and_list_plan_plugins` in `browserperfdash_runner.py`). Example: `browser_binary_size.py`, which
  measures installed binary size rather than running a web benchmark, and pulls in
  `webkitpy.binary_bundling.ldd`, `webkitpy.common.host`, and `webkitpy.port` for that.

### Vendored WebKit infrastructure

Most of the rest of `webkitpy/` (`common/`, `port/`, `tool/`, `style/`, `layout_tests/`, `w3c/`, `xcode/`,
etc.) is generic WebKit `Tools/Scripts` infrastructure pulled in wholesale by past syncs. Only a subset is
actually exercised by this repo's own code (see above); the rest is present because it was easier to sync
everything than to prune it, and because some browserperfdash plan plugins reach into arbitrary parts of it
(e.g. `webkitpy.port`, `webkitpy.common.host`). Don't assume every file under `webkitpy/` is relevant to
benchmark running — check whether `run-benchmark`/`browserperfdash-benchmark`/`webkitpy/benchmark_runner`/
`webkitpy/browserperfdash` actually import it before spending time on it.

`libraries/webkitcorepy/` provides small shared utilities (e.g. `NullContext`, `Timeout`) imported as
`webkitcorepy` by `webserver_benchmark_runner.py`.
