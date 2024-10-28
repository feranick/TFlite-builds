# TFlite-builds

TFlite cross-platform builds. Current stable version: `2.17.1`.

## Why this repo?

The information provided in the [official tensorflow lite page for building `whl` packages for ARM](https://www.tensorflow.org/lite/guide/build_cmake_pip) to cross-compile TFlite for different architectures is a good start. However there seem to be several issues. This repo aims at providing more info towards successful compilation. And some binaries as well. 

The provided builds are fully compatible with  [Coral.ai EdgeTPU through the updated libedgetpu drivers](https://github.com/feranick/libedgetpu).

## Building - Docker

- Install docker:

##
    sudo apt install docker.io

- Download tensorflow and checkout the relevant version.

##
    git clone https://github.com/tensorflow/tensorflow.git
##
    cd tensorflow
##
    git checkout vX.XX.XX


If compiling against `python 3.11` or newer, you have to allow installation of whl packages from external sources. Add the following line in `tensorflow/lite/tools/pip_package/Dockerfile.py3`:
```
    RUN ln -sf /usr/bin/python$PYTHON_VERSION /usr/bin/python3
    RUN curl -OL https://bootstrap.pypa.io/get-pip.py
->  RUN rm /usr/lib/python$PYTHON_VERSION/EXTERNALLY-MANAGED
    RUN python3 get-pip.py
    RUN rm get-pip.py
```
Optional: update `cmake` by editing `tensorflow/lite/tools/pip_package/Dockerfile.py3` to replace:
```
RUN curl -OL https://github.com/Kitware/CMake/releases/download/v3.16.8/cmake-3.16.8-Linux-x86_64.sh
RUN mkdir /opt/cmake
RUN sh cmake-3.16.8-Linux-x86_64.sh --prefix=/opt/cmake --skip-license
```
with:
```
RUN curl -OL https://github.com/Kitware/CMake/releases/download/v3.29.6/cmake-3.29.6-linux-x86_64.sh
RUN mkdir /opt/cmake
RUN sh cmake-3.29.6-linux-x86_64.sh --prefix=/opt/cmake --skip-license
```

- Edit Makefile to your system `tensorflow/lite/tools/pip_package/Makefile`:

For `Python 3.11` and earlier:
```
# Values: debian:<version>, ubuntu:<version>
BASE_IMAGE ?= ubuntu:22.04
PYTHON_VERSION ?= 3.11
NUMPY_VERSION ?= 1.24.4
```
For `Python 3.12` and later:
```
# Values: debian:<version>, ubuntu:<version>
BASE_IMAGE ?= ubuntu:22.04
PYTHON_VERSION ?= 3.12
NUMPY_VERSION ?= 1.26.4
```

### Changes specific to Ubuntu 24.04 and newer

- Replace the file `tensorflow/lite/tools/pip_package/update_sources.sh` with the one provided in this repository.
- In the file `tensorflow/lite/tools/pip_package/Makefile` replace: 

```
docker-build: docker-image
	mkdir -p $(TENSORFLOW_DIR)/bazel-ci_build-cache
	docker run \
		--rm --interactive $(shell tty -s && echo --tty) \
		$(DOCKER_PARAMS) \
		$(TAG_IMAGE) \
		/with_the_same_user /bin/bash -C /tensorflow/tensorflow/lite/tools/pip_package/build_pip_package_with_cmake.sh $(TENSORFLOW_TARGET)
  ```
  with 
  ```
  docker-build: docker-image
	mkdir -p $(TENSORFLOW_DIR)/bazel-ci_build-cache
	docker run \
		--rm --interactive $(shell tty -s && echo --tty) \
		$(DOCKER_PARAMS) \
		$(TAG_IMAGE) \
        /bin/bash -C /tensorflow/tensorflow/lite/tools/pip_package/build_pip_package_with_cmake.sh $(TENSORFLOW_TARGET)
  ```    
  
- In the file `tensorflow/lite/tools/pip_package/Dockerfile.py3` remove:
```
python$PYTHON_VERSION \
      python$PYTHON_VERSION-dev \
      python$PYTHON_VERSION-venv \
-->   python$PYTHON_VERSION-distutils \
      libpython$PYTHON_VERSION-dev \
      libpython$PYTHON_VERSION-dev:armhf \
      libpython$PYTHON_VERSION-dev:arm64
```

### Building using Debian
If you are building for `debian:bookworm` or `debian:bullseye`, you need to remove/comment the following line from `tensorflow/lite/tools/pip_package/Dockerfile.py3`, since the added ppa repository is specific only to ubuntu.

```
RUN yes | add-apt-repository ppa:deadsnakes/ppa
```

### Compilation

- Run compilation (adjust the values for `TENSORFLOW_TARGET` and `PYTHON_VERSION` to fit your needs:

```
BUILD_NUM_JOBS=4 make -C tensorflow/lite/tools/pip_package docker-build TENSORFLOW_TARGET=aarch64 PYTHON_VERSION=3.11
```
Note: You can change the variable `BUILD_NUM_JOBS` from 4 to `$(nproc)` to use the max number of cores in your CPU for fastest compilation.

These are the supported targets.

- `armhf`:  ARMv7 VFP with Neon Compatible with Raspberry Pi 3 and 4
- `rpi0`: ARMv6 Compatible with Raspberry Pi Zero
- `aarch64`: aarch64 (ARM 64-bit) Coral Mendel Linux 4.0 or Raspberry Pi with Ubuntu Server 20.04.01 LTS 64-bit of 22.04.x LTS 64-bit
- `native`: 	Your workstation 	It builds with "-mnative" optimization

## Building - Native
### Using Bazel

##
    PYTHON=python3 tensorflow/lite/tools/pip_package/build_pip_package_with_bazel.sh native


### Using CMake
##
    BUILD_NUM_JOBS=4 PYTHON=python3 tensorflow/lite/tools/pip_package/build_pip_package_with_cmake.sh native

Note: You can change the variable `BUILD_NUM_JOBS` from 4 to `$(nproc)` to use the max number of cores in your CPU for fastest compilation.

## GPU support for native builds
When compiling with either `docker` or native using `cmake` GPU support is disableed by default. To enable it, edit this file:

##
    nano tensorflow/lite/tools/pip_package/build_pip_package_with_cmake.sh

and add to line 110:
```
     native)
       BUILD_FLAGS=${BUILD_FLAGS:-"-march=native -I${PYTHON_INCLUDE} -I${PYBIND11_INCLUDE} -I${NUMPY_INCLUDE}"}
       cmake \
         -DCMAKE_C_FLAGS="${BUILD_FLAGS}" \
         -DCMAKE_CXX_FLAGS="${BUILD_FLAGS}" \
  ==>    -DTFLITE_ENABLE_GPU=ON \
         "${TENSORFLOW_LITE_DIR}"
```
