+++
title = "Building Envoy on Apple Silicon"
date = "2025-11-03"
author = "keithfz"
description = "ARM wrestling with Envoy"
+++

I finally got around to setting up an Envoy dev env on a Mac (previously I was using x86 and Ubuntu). I banged my head against the wall for a couple hours so maybe this will save someone else some time.

I'm on an M1 Macbook Pro on MacOS 26.0.1 -- so relatively up to date at the time of writing.

I managed to get it working through Lima. My Lima config was 

```yaml
arch: "aarch64"
images:
  - location: "https://cloud-images.ubuntu.com/releases/22.04/release/ubuntu-22.04-server-cloudimg-arm64.img"
    arch: "aarch64"
cpus: 8
memory: "16GiB"
disk: "30GiB"
mounts:
  - location: "~/envoy"
    writable: true
containerd:
  system: false
  user: false
provision:
  - mode: system
    script: |
      #!/bin/bash
      set -eux -o pipefail
      export DEBIAN_FRONTEND=noninteractive
      apt-get update
      apt-get install -y \
        build-essential \
        curl \
        git \
        autoconf \
        libtool \
        cmake \
        ninja-build \
        pkg-config \
        python3-pip \
        unzip \
        virtualenv \
        wget \
        docker.io
      wget -O /usr/local/bin/bazel https://github.com/bazelbuild/bazelisk/releases/latest/download/bazelisk-linux-arm64
      chmod +x /usr/local/bin/bazel
```


This provisions a VM mounting `~/envoy` into the VM, and installs a bunch of the envoy dependencies.

I install clang as a seperate step once everything else was ready:
```bash
set -eux -o pipefail

echo "Installing Clang 19..."

# Add LLVM apt repository
wget -qO- https://apt.llvm.org/llvm-snapshot.gpg.key | sudo tee /etc/apt/trusted.gpg.d/apt.llvm.org.asc
echo 'deb http://apt.llvm.org/jammy/ llvm-toolchain-jammy-19 main' | sudo tee /etc/apt/sources.list.d/llvm-19.list

# Update and install Clang 19
sudo apt-get update
sudo apt-get install -y \
  clang-19 \
  lld-19 \
  libc++-19-dev \
  libc++abi-19-dev

# Verify installation
clang-19 --version
echo "Clang 19 installation complete!"
```

Mainly because it initially caused a timeout error on VM provisioning when it was also installed in the Lima YAML.

From there I was able to run `limactl shell envoy-dev bash -c "./ci/do_ci.sh debug.server_only"` from my host machine to run the build. I also ran into some weird CEL errors when compiling. I honestly just patched it to ignore them for now, so we'd have:

```
  def _com_github_google_quiche():
      external_http_archive(
          name = "com_github_google_quiche",
          patch_cmds = [
              "find quiche/ -type f -name \"*.bazel\" -delete",
              "find quiche/ -type f -name '*.h' -exec sed -i '1 i\\#if defined(__clang__)\\n#pragma clang diagnostic
  ignored \"-Wnullability-completeness\"\\n#endif\\n' {} +",
          ],
          build_file = "@envoy//bazel/external:quiche.BUILD",
      )

```

which would result in the following being added:
```c++
  #if defined(__clang__)
  #pragma clang diagnostic ignored "-Wnullability-completeness"
  #endif
```

Now we let it build...

<img src="/images/spongebob-hours-later.png" alt="Spongebob Hours Later" class="center">

You should now be able to run the static binary from within the Lima VM. Unfortunately, by default, it won't work on the Mac host, since we built it for Linux, but we have `limactl ssh` for that.

It's also worth noting that I ran into issues with the `.dwp` portion of the build, but the envoy-static binary was already built, so that's good enough for me!


