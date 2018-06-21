# browserperfrunner

This repository contains a set of script and tools that automate running
benchmarks with different browsers.

The focus is on benchmarking the browser itself.

Most of the code from this repository is also available on the WebKit repository.

The idea of having this also on a separate repository is to allow running
this scripts without having to clone the whole WebKit repository (which is huge)

This is specially useful for running benchmarks on embedded boards.

Currently there are two main tools here:

  * `run-benchmark`: allows to run benchmarks and saves the perf data to a json
    file
  * `browserperfdash-benchmark`: its a wrapper over `run-benchmark` that allows
    to upload the perf data to a dashboard like https://github.com/Igalia/browserperfdash.
    The idea is to run this continously on a bot with different browser versions.


## Running browserperfdash-benchmark

* This is an example of how to run `browserperfdash-benchmark`

```
./browserperfdash-benchmark --plan sunspider \
	--config-file /etc/browserperfdash-benchmark-config.txt \
	--browser chrome --browser-version m67
```

Check the [config-file-example.txt](webkitpy/browserperfdash/config-file-example.txt) to see the format of the config file.

* To see the available plans:

```
./run-benchmark --list-plans
```

And to run all plans one after the other simply pass ```--allplans```
