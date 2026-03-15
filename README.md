# feast-stack

This repository contains the high-level details of the FEAST stack and it's various components as well as example implementations and deployments of the stack.

## Dev Setup

In general, for getting started with development on a new Linux machine:

```sh
sudo apt update

sudo apt install -y \
build-essential \
cmake \
ninja-build \
pkg-config \
git \
curl \
zip \
unzip \
tar

cd ~
git clone https://github.com/microsoft/vcpkg ~/vcpkg
~/vcpkg/bootstrap-vcpkg.sh
export VCPKG_ROOT=$HOME/vcpkg
```
