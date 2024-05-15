# Simple spin project 
Reference from docker conference: https://github.com/nigelpoulton/dockercon2023-wasm-lab/tree/main

# Getting spin
```
curl -fsSL https://developer.fermyon.com/downloads/install.sh | bash -s -- -v v2.4.2
```
# Creating the project
```
spin new
```

# Choose from the templates
Choose http-rust, name the app "anyname", and set the following options:

HTTP base: /
HTTP path: /yo


Save  changes and run spin build to build the app as a Wasm app.

```
spin build
spin up
```

# With docker

Dockerfile
```
FROM scratch
COPY /target/wasm32-wasi/release/wasmdocker.wasm .
COPY spin.toml .
```

The wasm module is available in the src directory after wasm build.
Therefore, we can edit spin.toml and replace 
```
source = "wasmdocker.wasm"
```

# Build 
```
docker buildx build \
  --platform wasi/wasm \
  -t wasmdocker .
```

# Run

```
docker run -d \
  --runtime=io.containerd.spin.v2 \
  --platform=wasi/wasm \
  -p 3000:80 \
  wasmdocker /
```
http://localhost:3000/yo


# wrk usage
```
sudo apt-get install build-essential libssl-dev git -y
git clone https://github.com/wg/wrk.git wrk
cd wrk
make
# move the executable to somewhere in PATH, ex:
sudo cp wrk /usr/local/bin
```

# To test our server

./wrk -t12 -c400 -d30s http://localhost:3000/yo

-t12 specifies 12 threads.
-c400 specifies 400 connections.
-d30s specifies the duration of 30 seconds.

# Using the pre-built WRK Docker image as an alternative:
```
docker run --rm -it williamyeh/wrk -t12 -c400 -d30s http://host.docker.internal:3000/yo
```
# For more runtimes

https://docs.docker.com/desktop/wasm/


