# wasm-microservice
A project repository to investigate running WebAssembly microservices in a heterogeneous infrastructure


# Journal

I am running this on the VisionFive 2 board from [StarFive](https://doc-en.rvspace.org/Doc_Center/visionfive_2.html).

This board is running [Ubuntu 24.04](https://ubuntu.com/download/risc-v) following instructions [here](https://wiki.ubuntu.com/RISC-V/StarFive%20VisionFive%202).

## Rust and WebAssembly

Install rust and WebAssembly target:

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
rustup target install wasm32-wasi
```

Install some other pre-requisites

```bash
sudo apt install build-essential cmake python3 clang-18 ninja-build

```

Install wasi-sdk from https://github.com/WebAssembly/wasi-sdk/releases/tag/wasi-sdk-22

I have compiled it from source, but you should be able to use the regular clang compiler and just use the wasi sysroot. But building wasi-sdk on the VisionFive 2 takes a _very_ long time. 

After compilation has finished. Move the wasi-sdk to `/opt`

```bash
sudo mv build/install/opt/wasi-sdk /opt
export WASI_SDK_PATH=/opt/wasi-sdk
```


On RISCV you also need to explicitly install `rust-lld`

```bash
sudo apt install lld-18
cargo install cargo-binutils
mkdir -p ~/.rustup/toolchains/stable-riscv64gc-unknown-linux-gnu/lib/rustlib/riscv64gc-unknown-linux-gnu/bin
ln -s /usr/lib/llvm-18/bin/lld ~/.rustup/toolchains/stable-riscv64gc-unknown-linux-gnu/lib/rustlib/riscv64gc-unknown-linux-gnu/bin/rust-lld
```

## Install Wasmtime

## Install Spin

Eventually, we want to run our wasm microservices in docker or kubernetes. For that we need containerd, docker and kubernetes. Containerd can run Wasm-modules with Wasmtime using the shim in the [`runwasi` repository](https://github.com/containerd/runwasi). Unfortunately, WASI 2.0 (aka the component cmodel) is not yet supported  on RISCV64. Instead, we can run Spin through the shim provided by [spinkube](https://github.com/spinkube/containerd-shim-spin). Again, this doesn't compile right out of the box, so we first need to build a local Spin for RISCV64.

Download Spin:

```bash
git clone https://github.com/fermyon/spin
```

This does not compile right out of the box, as `tokio-rustls:9.23..2` depends on `ring:0:16:20` which does not compile on RISCV64. Change the version of `tokio-rustls` to 0.24.1 (0.26.0 doesn't work) and it will build:

```diff
diff --git a/crates/trigger-http/Cargo.toml b/crates/trigger-http/Cargo.toml
index 3104f537..8e0cea92 100644
--- a/crates/trigger-http/Cargo.toml
+++ b/crates/trigger-http/Cargo.toml
@@ -33,7 +33,7 @@ spin-world = { path = "../world" }
 terminal = { path = "../terminal" }
 tls-listener = { version = "0.10.0", features = ["rustls"] }
 tokio = { version = "1.23", features = ["full"] }
-tokio-rustls = { version = "0.23.2" }
+tokio-rustls = { version = "0.24.1" }
 url = "2.4.1"
 tracing = { workspace = true }
 wasmtime = { workspace = true }
 ```


## Build support for docker and containerd 

The main source of information for this work comes from [this page](https://forum.rvspace.org/t/docker-engine-and-docker-cli-on-riscv64/267). 

These are the steps I needed to do:

Install dependenves (some which should already be there).
```bash
sudo apt install build-essential cmake pkg-config libseccomp2 libseccomp-dev libdevmapper-dev libbtrfs-dev
```

You need go language for the next step. Unfortunately, the packaged golang on my Ubuntu 24.04 for RISCV64 is too new for runc so I need to download the source for version 21.20 and build it:

```bash
wget https://go.dev/dl/go1.21.10.src.tar.gz
tar xvfz go1.21.10.src.tar.gz
cd go/src
sudo apt-get install gccgo-5
sudo ln -s /usr/bin/go-13 /usr/bin/go
GOROOT_BOOTSTRAP=/usr ./make.bash
# this will take a while
```

Then install runc and containerd:

```bash
#runc
git clone https://github.com/opencontainers/runc
pushd runc
make
sudo make install
popd

#containerd
git clone https://github.com/containerd/containerd
pushd containerd
make BUILDTAGS="no_btrfs"  # apparently this downloads go 1.22 !?
sudo make install
popd

#docker-cli
mkdir -p $GOPATH/src/github.com/docker/
pushd $GOPATH/src/github.com/docker/
git clone https://github.com/docker/cli
pushd cli
git checkout v20.10.13
VERSION=20.10.13 DISABLE_WARN_OUTSIDE_CONTAINER=1 GO111MODULE=off GOOS=linux GOARCH=riscv64 make
sudo cp build/docker-linux-riscv64 /usr/local/bin
sudo ln -sf /usr/local/bin/docker-linux-riscv64 /usr/local/bin/docker
popd
popd


#docker-init
git clone https://github.com/krallin/tini
pushd tini
export CFLAGS="-DPR_SET_CHILD_SUBREAPER=36 -DPR_GET_CHILD_SUBREAPER=37"
cmake . && make
sudo cp tini-static /usr/local/bin/docker-init
popd

#docker-proxy
mkdir -p $GOPATH/src/github.com/docker
pushd $GOPATH/src/github.com/docker
git clone https://github.com/docker/libnetwork/
pushd libnetwork
sudo apt install golang-github-ishidawataru-sctp-dev
# go get github.com/ishidawataru/sctp # old instructions that no longer work
GO111MODULE=off go build ./cmd/proxy
sudo cp proxy /usr/local/bin/docker-proxy
popd
popd

#rootlesskit
mkdir -p $GOPATH/src/github.com/rootless-containers/
pushd $GOPATH/src/github.com/rootless-containers/
git clone https://github.com/rootless-containers/rootlesskit.git
pushd rootlesskit
make
sudo make install
popd
popd


#dockerd
mkdir -p $GOPATH/src/github.com/docker/
pushd $GOPATH/src/github.com/docker/
git clone https://github.com/moby/moby docker
pushd docker
git checkout v20.10.13
sudo cp contrib/dockerd-rootless.sh /usr/local/bin
VERSION=20.10.13 DISABLE_WARN_OUTSIDE_CONTAINER=1 GO111MODULE=off GOOS=linux GOARCH=riscv64 ./hack/make.sh binary
sudo cp bundles/binary-daemon/dockerd-20.10.13 /usr/local/bin/dockerd
popd
popd
``` 

## Try it out

Execute the following commands in different terminal shells.

### Execute containerd
sudo containerd

### Execute dockerd
sudo dockerd #or with the proxy parameter

### Run docker client
sudo docker version


## Systemd config

To enable Systemd support, create the following files:

`/etc/systemd/system/containerd.service`

```bash
cat << EOF | sudo tee -a /etc/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd
KillMode=process
Delegate=yes
LimitNOFILE=1048576
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity
TasksMax=infinity

[Install]
WantedBy=multi-user.target
EOF
```

`/etc/systemd/system/docker.service`

```bash
cat << EOF | sudo tee -a /etc/systemd/system/docker.service
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
BindsTo=containerd.service
After=network-online.target firewalld.service containerd.service
Wants=network-online.target
Requires=docker.socket

[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
ExecStart=/usr/local/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutSec=0
RestartSec=2
Restart=always

# Note that StartLimit* options were moved from "Service" to "Unit" in systemd 229.
# Both the old, and new location are accepted by systemd 229 and up, so using the old location
# to make them work for either version of systemd.
StartLimitBurst=3

# Note that StartLimitInterval was renamed to StartLimitIntervalSec in systemd 230.
# Both the old, and new name are accepted by systemd 230 and up, so using the old name to make
# this option work for either version of systemd.
StartLimitInterval=60s

# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity

# Comment TasksMax if your systemd version does not supports it.
# Only systemd 226 and above support this option.
TasksMax=infinity

# set delegate yes so that systemd does not reset the cgroups of docker containers
Delegate=yes

# kill only the docker process, not all processes in the cgroup
KillMode=process

[Install]
WantedBy=multi-user.target
EOF
```

`/etc/systemd/system/docker.socket`

```bash
cat << EOF | sudo tee -a /etc/systemd/system/docker.socket
[Unit]
Description=Docker Socket for the API

[Socket]
# If /var/run is not implemented as a symlink to /run, you may need to
# specify ListenStream=/var/run/docker.sock instead.
ListenStream=/run/docker.sock
SocketMode=0660
SocketUser=root
SocketGroup=docker

[Install]
WantedBy=sockets.target
EOF
```

`/etc/docker/daemon.json` (optional)

```bash
cat << EOF | sudo tee -a /etc/docker/daemon.json
{
  "debug": true,
  "max-concurrent-uploads": 1
}
EOF
```

### Post-installation steps

Create docker group, add user to it and create the services:

```bash
sudo groupadd docker
sudo usermod -aG docker $USER
sudo systemctl enable docker.service
sudo systemctl enable containerd.service
sudo systemctl start docker.service
sudo systemctl start containerd.service
```

Log out and in again to enable the group settings to take effect.

Try it out, e.g. with the following container:

```bash
docker run -d -p 8080:8080 carlosedp/echo_on_riscv
```

and test by accessing `http://localhost:8080"

# Now install runwasi (wasmtime) and spin containerd shims

Containerd can run webassembly images from a OCI-compliant container registry (such as docker-hub) through something called "shims". We are going to install two shims, `runwasi` which can run Wasmtime modules and `containerd-shim-spin` which can run spin modules. Both use Wasmtime as the underlying WebAssembly runtime. 

## runwasi

We only care about wasmtime, so we restrict the runtime to that.

```bash
sudo apt install  protobuf-compiler
git clone https://github.com/containerd/runwasi.git
cd runwasi
rm Cargo.lock # without removing this, there I experienced a compiler error
git checkout containerd-shim-wasmtime/v0.4.0
OPT_PROFILE=release RUNTIMES=wasmtime make build
```

## containerd-shim-spin



# Issues building on Jetson Nano Aarch64

## runwasi

Build just like for RISCV64:

```bash
sudo apt install  protobuf-compiler
git clone https://github.com/containerd/runwasi.git
cd runwasi
OPT_PROFILE=release RUNTIMES=wasmtime make build
```

## containerd-shim-spin

The default makefile assumes building in a cross-compilation environment. The instructions below is for when you are building them directly on the target host.

The regular cc compiler on the Jetson Nano on Ubuntu 18.04 is too old so we had to install a new gcc version 12.3 which is going to be used in the building process.

```bash
git clone git@github.com:spinkube/containerd-shim-spin.git
cd containerd-shim-spin
CC=gcc cargo build --release --manifest-path=containerd-shim-spin/Cargo.toml

```



