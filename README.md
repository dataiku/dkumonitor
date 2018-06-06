Dataiku monitor
------------------

This projects bundles all the needed software to deploy system monitoring onto a cluster of machines. It is specifically designed to be easy to use with *Dataiku Data Science Studio*, including the API deployer feature.

# Architecture

The resulting bundle contains all the needed binaries to run a full monitoring system. It has 5 components:

- `go-carbon`: a carbon implementation written in go. 
- `carbonapi`: a go implementation of `graphite-api`. It is a small layer needed by Grafana and Dataiku Data Science Studio to query data. It is built without `libcairo` support.
- `grafana`: backend and fronted of the famous tool.
- `collectd`: A packaged version of collecd. Optionally, the user can choose to use the collectd installed in its own system
- `supervisord`: Is used to maintain the deamonized execution of each of those.

It is designed to run on the monitored machine, but it is possible to run only the server part or the agent part, with a switch at installation type.

# Usage

## Requirements

The target system must have `python2.7` installed.

## Installation

Download the latest stable version here.

```
tar xf dkumonitor-x.x.x.tar.gz
./dkumonitor-x.x.x/installer -d data_dir -p 15000
```

And thats it. Use `--help` on the installer to see the possible flags. Addtionnal flag are `--type` to choose a kind if installation (either `all`, `server` or `agent`). `--system-collectd` forces the installer to use the `collectd` executable from your `PATH`.

## Result

- `conf` contains the configuration files
- `bin` contains a single symlink to the starter script
- `storage` is were every persistent data is kept. That is whisper files, the Grafana sqlite3 database as well as its sessions and plugins (if you install some).
- `run` contains pid files, unix domain sockets and log files
- `static` is a symlink to the Grafana fronted static assets

## Usage

Launch the daemons:

```
data_dir/bin/dkm start
```

The command line usage is that of `supervisorctl`. Arguments are directly forwarded to `supervisorctl`. The `supervisord` is started if its pid file is not there, but it does not start any service by default. On MacOS, `dkumonitor` must be launched as root if you want disks and cpu metrics. This is a requirement of `collectd`. `telegraf` is able to get these metrics without privileged access, but it does not have he `processes` plugin used to track specifically JEKs and FEKs. Since they both have the usage, `telegraf`is not shipped. However, a config file for it is provided, as well as a commented section in the `supervisord` config file.

# Build

The build is supported on both Linux and MacOS.

## Requirements

- `go` version `1.9` or higher. Required for all backend services.
- `npm` to build Grafana backend
- A C compiler, namely `gcc` or `clang`, and the `autotools` suite to compile collectd.
- `python2.7` and `virtualenv`
- `rsync` to speed up the build process
- An internet access

## Run

```
./make-package
```

It takes some time, especially for Grafana frontend. A tarball is generated into the `dist` directory.  Optionally, when you need to develop on the package script, you can use the `--skip-build` and `--skip-archive` flags to skip the compilation and compression passes. You can follow more details about the compilation process by tailing `build/build.log`.


## Configure DSS

In the general settings, put in the address of the server as well as the `port+1`, which the one given at the installation. It is not needed to restart the studio. Using `dataiku.dss.an_instance_name` prefix is highly recommended as well.

## Have fun

Connect with you browser the port `port` of the server and play with Grafana. Add a data source of type `Graphite`, with URL `http://localhost:port+2`, and access mode `proxy`. Name it whatever you want, though `carbon-server` makes sens. After saving, a tip at the bottom of the page should indicate the data source is OK. Wait a little bit for metrics to come in (a minute or so) then you can create dashboards.

# TODO list

- Harden the configuration for auhtentication
- Write a systemd service file template and a script to install it

