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

## Spin

Eventually, we want to 

