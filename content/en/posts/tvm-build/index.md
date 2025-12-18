---
title: "Building MxNet with Apache TVM v0.11 for Android (Arm64) [PART-1]"
summary: "Despite the old models, some still have requirements to use this model"
categories: ["Post","Blog",]
tags: ["models","mxnet","tvm"]
#externalUrl: ""
showSummary: true
date: 2025-12-05
series: ["tvm"]
series_order: 1
---


### Ubuntu 22.04 → Relay → Android Runtime

~~Apache TVM v0.15.0 is the last release that supports **Relay**, making it the ideal version for deploying MXNet/PyTorch models on Android. This guide walks through building TVM on Ubuntu, cross-compiling the Android runtime, exporting Relay modules, and verifying everything on device.~~

This guide is tested on Apache TVM v0.11.0

---

## 🎯 Goal

- Build **TVM v0.11 host tools** for Relay model conversion  
- Export models into `deploy_lib.so`, `deploy_graph.json`, `deploy_param.params`  
- Test deployment using TVM’s Android demo app or RPC

You will also need conda installation on your devices.

## Prerequisites
- Conda
- Linux
- Android Studio
- Android NDK [[LINK]](https://developer.android.com/ndk/downloads), Any version is fine, (tested with r25c)
- TVM Binaries (Below)

## 1. Host Build (Linux)

### 1.1. Download TVM v0.11.0

```bash
wget https://dlcdn.apache.org/tvm/tvm-v0.11.0/apache-tvm-src-v0.11.0.tar.gz
tar -xzf apache-tvm-src-v0.11.0.tar.gz
mv apache-tvm-src-v0.11.0 tvm && cd tvm
```

>  v0.15 is the last version supporting Relay (frontend for MXNet)

> (Tested on v0.11) Later versions use Relax which drops MXNet support.

### 1.2. Environment Setup

```bash
conda create -n tvm-build-venv python=3.7 && conda activate tvm-build-venv

conda install -c conda-forge \
    llvmdev=15 clang=15 clangxx=15 cmake ninja make

pip install 'numpy<1.20' decorator scipy==1.6 psutil attrs mxnet==1.5.0

sudo apt install build-essential openjdk-17-jdk maven
```

### 1.3. Building TVM on Machine Host Build
```bash
mkdir build && cp cmake/config.cmake build && cd build
```

Configure `config.cmake` for CPU-only build with Relay + GraphExecutor:

```bash
echo "set(CMAKE_BUILD_TYPE RelWithDebInfo)" >> config.cmake         # reasonable debug info
echo "set(USE_LLVM \"llvm-config --ignore-libllvm --link-static\")" >> config.cmake  # IR→LLVM
echo "set(HIDE_PRIVATE_SYMBOLS ON)" >> config.cmake                 # smaller .so
echo "set(USE_GRAPH_EXECUTOR ON)" >> config.cmake                   # legacy executor
# Disable GPU backends (enable later if needed)
echo "set(USE_CUDA OFF)" >> config.cmake
echo "set(USE_METAL OFF)" >> config.cmake
echo "set(USE_VULKAN OFF)" >> config.cmake
echo "set(USE_OPENCL OFF)" >> config.cmake
```

Then Build for host:

```bash
cmake ..
make -j4 
```

> The number is a total cores range from 1...MAX_CORE 

### 1.4. Python Binding & Environment Variable

Append to `~/.bashrc`:

```bash
export TVM_HOME=~/tvm
export PYTHONPATH=$TVM_HOME/python:${PYTHONPATH}
```

After done building, navigate to `~/tvm` and run `make jvmpkg` and `make jvminstall`

## 2. Deploy & Verify

### 2.1. Android App Setup (Gradle)

- Create `gradle/wrapper/gradle-wrapper.properties`
    
    ```bash
    distributionUrl=https\://services.gradle.org/distributions/gradle-7.5-bin.zip
    ```
    
- Use **Gradle Wrapper** + **Amazon Corretto 17** as Gradle JDK if have any compatibility issues
- Download [NDK](https://developer.android.com/ndk/downloads) and ensure `$ANDROID_NDK` is visible in `$PATH`.

---

### 2.2. Torch → Relay Export Example

```python
import torch, torchvision, tvm
from tvm import relay
from tvm.contrib import ndk

model = torchvision.models.mobilenet_v2(weights="IMAGENET1K_V1").eval()
dummy = torch.randn([1,3,224,224])
scripted = torch.jit.trace(model, dummy).eval()

shape_list = [("input0", (1,3,224,224))]
mod, params = relay.frontend.from_pytorch(scripted, shape_list)

target = "llvm -mtriple=aarch64-linux-android"
with tvm.transform.PassContext(opt_level=3):
    lib = relay.build(mod, target=target, params=params)

lib.export_library("deploy_lib.so", fcompile=ndk.create_shared)
open("deploy_graph.json","w").write(lib.get_graph_json())
open("deploy_param.params","wb").write(relay.save_param_dict(lib.get_params()))

print("✅ Exported artifacts for Android runtime.")
```

---

### 2.3. Deploying Model in `android_deploy` app

After converting the model to Relay (TVM), the model file must be placed in the `assets/` folder in [github](https://github.com/apache/tvm/tree/v0.15.0/apps/android_deploy/app). Then need to fix some gradle and compatibility. Also Need to setup the correct `ndk-build` version in the gradle files. Then to use the custom model, the `download-models.gradle` import must be disabled in the `build.gradle` file.

Then running the task `buildJni` from gradle will compile and build the correct library for TVM.

---

## Common Pitfalls & Fixes

| Issue | Cause | Fix |
| --- | --- | --- |
| `arg2.ndim is expected to equal 4` | MXNet model shape mismatch | Possible Wrong Runtime, re-export runtime |
| `GraphExecutorFactory not found` | Missing`graph_executor_factory.cc`in tvm_runtime.h | Add file + rebuild`tvm_runtime` |
| `undefined reference to 'sin'` | Missing math lib link | Add`-lm`to`fcompile`flags |