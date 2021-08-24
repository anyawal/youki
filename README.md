# youki: A container runtime in Rust

[![Discord](https://img.shields.io/discord/849943000770412575.svg?logo=discord)](https://discord.gg/zHnyXKSQFD)
[![GitHub commit activity](https://img.shields.io/github/commit-activity/m/containers/youki)](https://github.com/containers/youki/graphs/commit-activity)
[![GitHub contributors](https://img.shields.io/github/contributors/containers/youki)](https://github.com/containers/youki/graphs/contributors)
[![Github CI](https://github.com/containers/youki/actions/workflows/main.yml/badge.svg?branch=main)](https://github.com/containers/youki/actions)

<p align="center">
  <img src="docs/youki.png" width="230" height="230">
</p>

youki is an implementation of [runtime-spec](https://github.com/opencontainers/runtime-spec) in Rust, referring to [runc](https://github.com/opencontainers/runc).

# About the name

youki is pronounced as /joʊki/ or yoh-key.
youki is named after a Japanese word 'youki', which means 'a container'. In Japanese language, youki also means 'cheerful', 'merry', or 'hilarious'.

# Motivation

Here is why I am rewriting a new container runtime in Rust.

- Rust is one of the best languages to implement oci-runtime. Many container tools are written in Go. It's all very nice products. However, the container runtime requires the use of system calls, which requires a bit of special handling when implemented in Go. This is too tricky(e.g. _namespaces(7)_, _fork(2)_); with Rust, it's not that tricky and you can use system calls. Also, unlike C, Rust provides the benefit of memory management. Rust is not yet a major player in the container field, and Rust has the potential to contribute more to this field. I hope to be one of the examples of how Rust can be used in this field.
- youki has the potential to be faster and use less memory than runc. This means that it can work in environments with tight memory usage. I don't have any benchmarks, etc., as it is not yet fully operational, but I expect that it will probably perform better when implemented in Rust. In fact, [crun](https://github.com/containers/crun#performance), a container runtime implemented in C, is quite high performance. For example, it may be possible to experiment with asynchronous processing using async/await in some parts.
- The development of [railcar](https://github.com/oracle/railcar) has been suspended. This project was very nice but is no longer being developed. This project is inspired by it.
- I have fun implementing this. In fact, this may be the most important.

# Related project

- [containers/oci-spec-rs](https://github.com/containers/oci-spec-rs) - OCI Runtime and Image Spec in Rust

# Status of youki

youki is not at the practical stage yet. However, it is getting closer to practical use, running with docker and passing all the default tests provided by [opencontainers/runtime-tools](https://github.com/opencontainers/runtime-tools).
![youki demo](docs/demo.gif)

|    Feature     |                   Description                   |                                                State                                                |
| :------------: | :---------------------------------------------: | :-------------------------------------------------------------------------------------------------: |
|     Docker     |               Running via Docker                |                                                  ✅                                                  |
|     Podman     |               Running via Podman                | It works, but cgroups isn't supported. WIP on [#24](https://github.com/containers/youki/issues/24)  |
|   pivot_root   |            Change the root directory            |                                                  ✅                                                  |
|     Mounts     |    Mount files and directories to container     |                                                  ✅                                                  |
|   Namespaces   |         Isolation of various resources          |                                                  ✅                                                  |
|  Capabilities  |            Limiting root privileges             |                                                  ✅                                                  |
|   Cgroups v1   |            Resource limitations, etc            |                                                  ✅                                                  |
|   Cgroups v2   |             Improved version of v1              | Support is complete except for devices. WIP on [#78](https://github.com/containers/youki/issues/78) |
|    Seccomp     |             Filtering system calls              |                     WIP on [#25](https://github.com/containers/youki/issues/25)                     |
|     Hooks      | Add custom processing during container creation |                                                  ✅                                                  |
|    Rootless    |   Running a container without root privileges   | It works, but cgroups isn't supported. WIP on [#77](https://github.com/containers/youki/issues/77)  |
| OCI Compliance |        Compliance with OCI Runtime Spec         |                                   39 out of 55 test cases passing                                   |

# Getting Started

Local build is only supported on linux.
For other platforms, please use [Vagrantfile](#setting-up-vagrant) that we prepared.

## Requires

- Rust(See [here](https://www.rust-lang.org/tools/install))
- Docker(See [here](https://docs.docker.com/engine/install))

## Dependencies

### Debian, Ubuntu and related distributions

```sh
$ sudo apt-get install    \
      pkg-config          \
      libsystemd-dev      \
      libdbus-glib-1-dev
```

### Fedora, Centos, RHEL and related distributions

```sh
$ sudo dnf install   \
      pkg-config     \
      systemd-devel  \
      dbus-devel
```

## Build

```sh
$ git clone git@github.com:containers/youki.git
$ cd youki
$ ./build.sh
$ ./youki -h # you can get information about youki command
```

## Tutorial

Let's try to run a container that executes `sleep 5` using youki.
Maybe this tutorial is need permission as root.

```sh
$ git clone git@github.com:containers/youki.git
$ cd youki
$ ./build.sh
$ mkdir tutorial
$ cd tutorial
$ mkdir rootfs
$ docker export $(docker create busybox) | tar -C rootfs -xvf -
```

Prepare a configuration file for the container that will run `sleep 5`.

```sh
$ curl https://gist.githubusercontent.com/utam0k/8ab419996633066eaf53ac9c66d962e7/raw/e81548f591f26ec03d85ce38b0443144573b4cf6/config.json -o config.json
$ cd ../
$ ./youki create -b tutorial tutorial_container
$ ./youki state tutorial_container # You can see the state the container is in as it is being generate.
$ ./youki start tutorial_container
$ ./youki state tutorial_container # Run it within 5 seconds to see the running container.
$ ./youki delete tutorial_container # Run it after the container is finished running.
```

Change the command to be executed in config.json and try something other than `sleep 5`.

## Usage

Starting the docker daemon.

```
$ dockerd --experimental --add-runtime="youki=$(pwd)/target/x86_64-unknown-linux-gnu/debug/youki"
```

In case you get an error like :

```
failed to start daemon: pid file found, ensure docker is not running or delete /var/run/docker.pid
```

That means your normal Docker daemon is running, and it needs to be stopped. For that, open a new shell in same directory and run :

```
$ systemctl stop docker # might need root permission
```

Now in the same shell run the first command, which should start the docker daemon.

You can use youki in a different terminal to start the container.

```
$ docker run -it --rm --runtime youki busybox
```

Afterwards, you can close the docker daemon process in other the other terminal. To restart normal docker daemon (if you had stopped it before), run :

```
$ systemctl start docker # might need root permission
```

### Integration test

Go and node-tap are required to run integration test. See the [opencontainers/runtime-tools](https://github.com/opencontainers/runtime-tools) README for details.

```
$ git submodule update --init --recursive
$ ./integration_test.sh
```

### Setting up Vagrant

You can try youki on platforms other than linux by using the Vagrantfile we have prepared.

```
$ git clone git@github.com:containers/youki.git
$ cd youki
$ vagrant up
$ vagrant ssh
# in virtual machine
$ cd youki # in virtual machine
$ ./build.sh
```

# Community

We also have an active [Discord](https://discord.gg/h7R3HgWUct) if you'd like to come and chat with us.

# Design and implementation of youki

TBD(WIP on [#14](https://github.com/containers/youki/issues/14))

# Contribution

This project welcomes your PR and issues.
For example, refactoring, adding features, correcting English, etc.
If you need any help, you can contact me on [Twitter](https://twitter.com/utam0k).

Thanks to all the people who already contributed!

<a href="https://github.com/containers/youki/graphs/contributors">
  <img src="https://contributors-img.web.app/image?repo=containers/youki" />
</a>
