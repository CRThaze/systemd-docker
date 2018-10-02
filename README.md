# Introduction
This repository provides a wrapper which improves the integration of Docker containers run as `systemd` services. 

Usually, a Docker container is launched with `docker run ...` where `docker` is the *Docker client* - a command 
line utility connected to the *Docker engine running in another process*, which executes the *image builds, running 
containers, etc. in yet other processes*. If a Docker container is started as a `systemd` service using 
an instruction like `ExecStart=docker run ...`, **`systemd` interacts with the Docker client process instead of the 
actual container process, which can lead to a bunch of odd situations**:
- the client can detach or crash while the container is doing fine, yet `systemd` would trigger failure handling 
- worse, the container crashes and should be taken care of, but the client stalled - `systemd` is blind and won't do  
  anything
- when a container is stopped with `docker stop ...`, attached client processes exit with an error code instead of 
  0/success. Unless specially configued, this also triggers `systemd`'s failure handling in an inappropriate 
  situation

The **key thing that this wrapper does is** that it moves the container process from the *cgroups set up by Docker* 
to the *service unit's cgroup* **to give `systemd` the supervision of the actual Docker container process**.  
It's written in Golang and allows to *leverage all the cgroup functionality of `systemd` and `systemd-notify`*.

# Repository history and credits
- the code was written by [@ibuildthecloud](https://github.com/ibuildthecloud) and his co-contributors in this [repository](https://github.com/ibuildthecloud/systemd-docker). 
The motivation is explained in this [Docker issue #6791](https://github.com/docker/docker/issues/6791) and this [mailing list thread](https://groups.google.com/d/topic/coreos-dev/wf7G6rA7Bf4/discussion).
- [@agend07](https://github.com/agend07) and co-contributors fixed outdated dependancies and did a first clean-up
- I removed all outdated and broken elements and created a new compilation docker container which can be found [here]()

# Installation
Supposing that a Go environment is available, the build instruction is `go get github.com/dontsetse/systemd-docker`. The 
executable can then be found in the Go binary directory, usually something like `$GO_ROOT/bin` and it's called 
`systemd-docker`.

It can also be build using a stand-alone docker image, see [here]()

# Use
Both
- `systemctl` to manage `systemd` services, and
- the `docker` CLI

can be used and everything should stay in sync.

In the `systemd` unit files, the instruction to launch the Docker container takes the form 

`ExecStart=systemd-docker [<sysd-dkr_opts>] run [<dkr-run_opts>] <img_name> [<cnt_params>]`

where
- `<sysd-dkr_opts>` are the systemd-docker flags explained in the [systemd-docker Options](#systemd-docker-options)
- `<dkr-run_opts>` are the usual flags defined by `docker run` - a few restrictions apply, see section 
  [Docker restrictions](#docker-restrictions)
- `<img_name>` is the name of the Docker image to run
- `<cnt_params>` are the parameters provided to the container when it's started  

Note: like any executable, `systemd-docker` should be in folder that is part of `$PATH` to be able to use it globally, 
      otherwise use a absolute path in the instruction like f.ex. `ExecStart=/opt/bin/systemd-docker ...` 

Here's an unit file example to run a Nginx container:
```ini
[Unit]
Description=Nginx
After=docker.service
Requires=docker.service

[Service]
ExecStart=systemd-docker run --rm --name %n nginx
Restart=always
RestartSec=10s
Type=notify
NotifyAccess=all
TimeoutStartSec=120
TimeoutStopSec=15

[Install]
WantedBy=multi-user.target
```
The use of `%n` is a `systemd` feature explained in the [automtic container naming section](#automatic-container-naming)
Supposing that the example given above is stored under the likely path `/etc/systemd/system/nginx.service`, the 
container is named nginx. 
 
Note: `Type=notify` and `NotifyAccess=all` are important

## Container names
Container names are compulsory to make sure that a `systemd` service always relates to/acts upon the same container(s). 
While it may seem as if that could be omitted as long as the `--rm` flag used, that's misleading: the deletion process 
triggered by this flag is actually part of the Docker client logic and if the client detaches for whatever reason from 
the running container, the information is lost (even if another client is re-attached later) and *the container will 
**not** be deleted*.
 
`systemd-docker` adds an additional check and looks for the named container when `run` is called - if one exists and 
is stopped, it will be deleted.

# Systemd integration details
## Automatic container naming
`systemd` populates a range of variables among which `%n` stands for the name of service, derived from it's filename. 
This  allows to write a self-configuring `ExecStart` instructions using the parameters
 
`ExecStart=systemd-docker ... run ... --name %n --rm ...`

## Use of `systemd` environment variables
`systemd` handles environment variables with the instructions `Environment=...` and `EnvironmentFile=...`. To inject
variables into other instructions, the pattern is *${variable_name}*. With the flag -e they can be passed to the 
container (and not to `systemd-docker`)

Example: `ExecStart=systemd-docker ... run -e ABC=${ABC} -e XYZ=${XYZ} ...`

`systemd-docker` has an option to pass on all defined environment variables using the `--env` flag, see the 
[environment variables section](#environment-variables) in the `systemd-docker` options

# Systemd-docker options
## Logging
By default the container's stdout/stderr is written to the system journal. This may be disables with `--logs=false`.

Example: `ExecStart=systemd-docker ... --logs=false ... run ...`

## Environment Variables
`systemd` handles environment variables with the instructions `Environment=...` and `EnvironmentFile=...`. To inject 
variables into other instructions, the pattern is *${variable_name}*

Example: `ExecStart=systemd-docker ... run -e ABC=${ABC} -e XYZ=${XYZ} ...` 

The systemd environment variables are automatically passed through to the docker container if the `--env` flag is provided.  
This will essentially read all the current environment variables and add the appropriate `-e ...` flags to the docker run 
command.  For example:

```
EnvironmentFile=/etc/environment
ExecStart=systemd-docker --env run ...
```
The contents of `/etc/environment` will be added to the docker run command.

## Cgroups
By default all application cgroups are moved to systemd. This implies that the `docker run` flags  `--cpuset` and/or `-m` 
are incompatible. It's also possible to control which cgroups are transfered using individual  `--cgroups` flags for 
each cgroup to transfer. **`-cgroups name=systemd` is the strict minimum, if it's not specified, `systemd` will lose track 
of the container**. 
This implies that the `docker run` flags  `--cpuset` and/or `-m` are incompatible.

`ExecStart=/opt/bin/systemd-docker --cgroups name=systemd --cgroups=cpu run --rm --name %n nginx`

The above command will use the `name=systemd` and `cpu` cgroups of systemd but then use Docker's cgroups for all the others, like the freezer cgroup.

## PID File
To create a PID file for the container, just add `--pid-file=<path/to/pid_file>`.

Example: `ExecStart=systemd-docker ... --pid-file=/var/run/%n.pid ... run ...`

## systemd-notify support

By default `systemd-docker` will send READY=1 to the `systemd` notification socket.  With the `systemd-docker` `--notify` flag the READY=1 call is 
delegated to the container itself. To achieve this, `systemd-docker` bind mounts the `systemd` notification socket into the container and sets the 
NOTIFY_SOCKET environment variable. 

Example: `ExecStart=systemd-docker ... --notify ... run ...`

Please be aware that `systemd-notify` comes with its own quirks - more info can be found in this 
[mailing list thread](http://comments.gmane.org/gmane.comp.sysutils.systemd.devel/18649).  In short, `systemd-notify` is not reliable because often 
the child dies before `systemd` has time to determine which cgroup it is a member of.

## Container removal behavior

To disable `systemd-docker`'s "stopped container removal" procedure, the flag `... --rm=false ...` can be used.

Example: `ExecStart=systemd-docker ... --rm=false ... run ...`

# Docker restrictions
## --cpuset and/or -m
These flags can't be used because they are incompatible with the cgroup migration(s) inherent to `systemd-docker`. 

## -d (detaching the Docker client)
The `-d` flag provided to `docker run` has no effect under `systemd-docker`. To cause the Docker client to detach after the container is running, use 
the `systemd-docker` options `--logs=false --rm=false`. If either `--logs` or `--rm` is true, the Docker client instance used by `systemd-docker` is kept 
alive until the `systemd` service is stopped or the container exits.

# Known issues
## Inconsistent cgroup
CentOS 7 is inconsistent in the way it handles some cgroups. It has `3:cpuacct,cpu:/user.slice` in `/proc/[pid]/cgroups` but the corresponding path 
`/sys/fs/cgroup/cpu,cpuacct/` doesn't exist. This causes `systemd-docker` to fail when it tries to move the PIDs there. To solve this the systemd
cgroup must be explicitely mentioned: 

`systemd-docker ... --cgroups name=systemd ... run ...`

See https://github.com/ibuildthecloud/systemd-docker/issues/15 for details.

# License
[Apache License, Version 2.0](http://www.apache.org/licenses/LICENSE-2.0)
