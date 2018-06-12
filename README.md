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

The target system must have `python2.7` installed. The pre-built package targets Linux 64 bits only.

## Installation

Download the latest stable version [here](https://downloads.dataiku.com/public/dkumonitor/download_latest.html). Or use `curl` and `jq`:

```
latest_version=$(curl -s https://downloads.dataiku.com/latest_dkumonitor.json|jq -r '.version')
curl -O https://downloads.dataiku.com/public/dkumonitor/$latest_version/dkumonitor-$latest_version.tar.gz
```

Unpack and run the installer:

```
tar xf dkumonitor-x.x.x.tar.gz
./dkumonitor-x.x.x/installer -d DATA_DIR -p PORT 
```

`DATA_DIR` is the directory into which you want the *dkumonitor* to be installed. `PORT` is the lower bound of a reserved range of 10 ports. *Grafana* will be exposed on `PORT`, *carbon* will be exposed on `PORT+1` and *carbonapi* on `PORT+2`.

And thats it. Use `--help` on the installer to see the possible flags:

- `--directory`,`-d`: The directory in which the monitor will be installed.
- `--port`,`-p`: The lower bound of a 10 port range in which the services will be exposed by default. Grafana will be exposed at `port`, the carbon metrics server will be exposed on `port+1` and the carbonapi endpoint will be exposed on `port+2`.
- `--type`,`-t` can be either
    - `all`: installs all the services.
    - `agent`: installs only collectd.
    - `server`: installs the metrics servers and Grafana.
    - `backend`: installs the metrics servers only.
    - `frontend`: installs Grafana only.
- `--update`,`-u`: Runs in update mode. Update mode takes only `--directory` as additional parameter and updates the links and environment to the new binaries and static assets. Configuration files are not modified by the update mode.
- `--carbon-listen`: Overrides the default listening IP of carbon. Default is `0.0.0.0`.
- `--carbonapi-listen`: Overrides the default listening IP of carbonapi. Default is `0.0.0.0`.
- `--hostname`: Overrides the default hostname used by collectd. The default behavior is to use the reversed FQDN obtained with `hostname -f` (`host.example.domain` gives `domain.example.host`).
- `--system-collectd`: Use the `collectd`executable found in the `PATH` instead of the one packaged.


## Result

This generates the following files and directories:

- `conf` contains the configuration files. Those can be customized and wont be modified by the update mode.
- `bin` contains simlinks and configuration to link back to the packaged binaries.
- `storage` is where every persistent data is kept. That is whisper files, the Grafana sqlite3 database as well as its sessions and plugins (if you install some). You might want to backup this one.
- `run` contains pid files, unix domain sockets and log files.
- `static` is a symlink to the Grafana fronted static assets.


## Usage

Launch the daemons:

```
DATA_DIR/bin/dkm start
```

You can use `stop`, `restart` and `status`. Each of this commands can be executed onto a single service. For instance, to restart only `collectd`, use `DATA_DIR/bin/dkm restart collectd`.

You can then use the various services. Grafana has a default admin login that you can change interactively or in its config file before launch.

## Install the boot service

Launch this command as `root`:

```
DATA_DIR/bin/dkmadmin install-boot
```

Optionally, the `--name` flag can be used in case you want to install several instances on the same host. Both init scripts and `systemd` are supported and automatically detected.

## Update to new version of dkumonitor

The binaries can be updated by running the installer in update mode:

```
./dkumonitor-x.x.x/installer -d DATA_DIR --update
```

No configuration files will be modified this way.

## Additional modules

The *dkumonitor* agent is *collectd* and is packaged with a wide variety of modules. Only a few of them are used by default. To use additional modules, add files with their configuration in `DATA_DIR/conf/collectd.d/`. You must ensure that the modules dependencies are installed. Please refer to the collectd documentation and use the `ldd` command onto the module file `./dkumonitor-x.x.x/lib/collectd/MODULE_NAME.so` to check if its dependencies are all available.

## Configure DSS

### Full system monitoring

Stop DSS, then Run the DSS monitoring integration:

```
DSS_DATA_DIR/bin/dssadmin install-monitoring-integration \
    -graphiteServer DKU_MONITOR_HOST:DKU_MONITOR_PORT+1
```

Additional flags are available, use `DSS_DATA_DIR/bin/dssadmin install-monitoring-integration -h` to see them. Restart DSS when you are done.

### API nodes QPS for API Deployer 

If you dont want a full system monitoring but still want API nodes monitoring within the API Deployer, add the following keys in `DSS_DATA_DIR/config/server.json`:

```
"graphiteCarbonServerURL":"DKU_MONITOR_HOST:DKU_MONITOR_PORT+1",
"graphiteCarbonPrefix": "domain.example.hostname"
```

The value of the environment variable `DKU_GRAPHITE_ADDITIONAL_PREFIX` will be appended to the prefix if the variable is not empty. 

### API Deployer

Go to *API Deployer* `->` *Infrastructures* `->`*YOUR_INFRASTRUCTURE* `->` *Settings* `->` *API nodes*, and add the prefix of each API node. Go to *API Deployer* `->` *Infrastructures* `->`*YOUR_INFRASTRUCTURE* `->` *Settings* `->` *Monitoring*, and set the URL of the *carbonapi* endpoint which is `http://DKUMONITOR_HOST:DKUMONITOR_PORT+2`. You can then monitor the API nodes activity right from each service homepage in the API Deployer.


# Build

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

It takes some time, especially for Grafana frontend. A tarball is generated into the `dist` directory.  Optionally, when you need to develop on the package script, you can use the `--skip-build` and `--skip-archive` flags to skip the compilation and compression passes. You can follow more details about the compilation process by tailing `build/build.log`. The actual list of collectd modules built depends on which dependencies for them are available at compile time. The `udev` support in the `disk` module is disabled for portability reasons.
