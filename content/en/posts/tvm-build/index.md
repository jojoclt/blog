---
title: "Building MxNet with Apache TVM v0.15 for Android (Arm64)"
summary: "Despite the old models, some still have requirements to use this model"
categories: ["Post","Blog",]
tags: ["models","mxnet","tvm"]
#externalUrl: ""
showSummary: true
date: 2025-11-05
---


### Ubuntu 22.04 → Relay → Android Runtime

Apache TVM v0.15.0 is the last release that supports **Relay**, making it the ideal version for deploying MXNet/PyTorch models on Android. This guide walks through building TVM on Ubuntu, cross-compiling the Android runtime, exporting Relay modules, and verifying everything on device.

---

## 🎯 Goal

- Build **TVM v0.15 host tools** for Relay model conversion  
- Build **Android arm64-v8a runtime** (`libtvm_runtime.so`)  
- Export models into `deploy_lib.so`, `deploy_graph.json`, `deploy_param.params`  
- Test deployment using TVM’s Android demo app or RPC

You will also need conda installation on your devices.

---

## 1. Host Build (Linux)

### 1.1. Download TVM v0.15.0

```bash
wget https://dlcdn.apache.org/tvm/tvm-v0.15.0/apache-tvm-src-v0.15.0.tar.gz
tar -xzf apache-tvm-src-v0.15.0.tar.gz
mv apache-tvm-src-v0.15.0 tvm && cd tvm
```

>  v0.15 is the last version supporting Relay (frontend for MXNet).
Later versions use Relax which drops MXNet support.
### 1.2. Environment Setup

```bash
conda create -n tvm-build-venv python=3.7 && conda activate tvm-build-venv

conda install -c conda-forge \
    llvmdev=15 clang=15 clangxx=15 cmake ninja make

pip install numpy==1.19 decorator scipy==1.6 psutil attrs

sudo apt install build-essential openjdk-17-jdk maven
```

### 1.3. Two-Stage Build (Host + Android)

TVM must be **built twice**:

| Stage | Target | Purpose |
| --- | --- | --- |
| **Host build** | x86_64 (default) | Used for model conversion and exporting Relay modules |
| **Android build** | arm64-v8a | Used for running models on the target Android device |

> **Tip:** The host build is used with Python (`relay.build`, `tvmc`, etc.), while the Android build provides `libtvm_runtime.so` for mobile execution.

### 1.4. Host Build
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

### 1.5. Android Runtime Build
Create a separate build folder for Android, such as `build_android64`:

ANDROID_NDK needs to be download separately.

```bash
mkdir build_android64 && cd build_android64
cmake ../   -DCMAKE_TOOLCHAIN_FILE=$ANDROID_NDK/build/cmake/android.toolchain.cmake   -DCMAKE_BUILD_TYPE=Release   -DANDROID_ABI="arm64-v8a"   -DUSE_CPP_RPC=ON   -DANDROID_STL=c++_static   -DANDROID_NATIVE_API_LEVEL=android-28   -DANDROID_TOOLCHAIN=clang++   -DUSE_GRAPH_EXECUTOR=ON
make tvm_runtime tvm_rpc -j4
make jvmpkg
make jvminstall
```

This step is to generate the android runtime to be built and compile to java for android (via `make jvmpkg` and `make jvminstall`)

> ⚠️ In some cases, you may need to swap folder names (e.g., rename build ⇄ build_android64) when compiling models

> The compiling model to relay part (TVM) needs to be used from the build folder.

> For the make for Android part, temporary swap folder name before running

### 1.6. Python Binding & Environment Variable

Append to `~/.bashrc`:

```bash
export TVM_HOME=~/tvm
export PYTHONPATH=$TVM_HOME/python:${PYTHONPATH}
```

Install Python runtime dependencies for RPC later:

## 2. Runtime for Android (Armv8 / NDK)

> Only tvm_runtime and tvm_rpc are needed on device — no LLVM or Python required.

### 2.1. Set NDK Path

```bash
export ANDROID_NDK=~/Android/Sdk/ndk/{YOUR_VERSION}
export PATH=$ANDROID_NDK:$PATH
```

### 2.2. Cross-Compile Runtime

```bash
cmake ../   -DCMAKE_TOOLCHAIN_FILE=$ANDROID_NDK/build/cmake/android.toolchain.cmake   -DCMAKE_BUILD_TYPE=Release   -DANDROID_ABI="arm64-v8a"   -DUSE_CPP_RPC=ON   -DANDROID_STL=c++_static   -DANDROID_NATIVE_API_LEVEL=android-28   -DANDROID_TOOLCHAIN=clang++   -DUSE_GRAPH_EXECUTOR=ON
make tvm_runtime tvm_rpc -j4
```

## 3. Deploy & Verify via RPC or App

For the RPC app part, the app doesn't work so by following this, I am able to run the model via RPC (Android as Host)

https://zhuanlan.zhihu.com/p/599605907

### 3.1. Android App Setup (Gradle)

- Create `gradle/wrapper/gradle-wrapper.properties`
    
    ```bash
    distributionUrl=https\://services.gradle.org/distributions/gradle-7.5-bin.zip
    ```
    
- Use **Gradle Wrapper** + **Amazon Corretto 17** as Gradle JDK.
- Install NDK and ensure `$ANDROID_NDK` is exported.

---

### 3.2. Torch → Relay Export Example

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

### 3.3. Deploying Model in `android_deploy` app

After converting the model to Relay (TVM), the model file must be placed in the `assets/` folder in [github](https://github.com/apache/tvm/tree/v0.15.0/apps/android_deploy/app). Then need to fix some gradle and compatibility. Also Need to setup the correct `ndk-build` version in the gradle files. Then to use the custom model, the `download-models.gradle` import must be disabled in the `build.gradle` file.

Then running the task `buildJni` from gradle will compile and build the correct library for TVM.

---

## Common Pitfalls & Fixes

| Issue | Cause | Fix |
| --- | --- | --- |
| `arg2.ndim is expected to equal 4` | MXNet model shape mismatch | Possible Wrong Runtime, re-export runtime |
| `GraphExecutorFactory not found` | Missing`graph_executor_factory.cc`in tvm_runtime.h | Add file + rebuild`tvm_runtime` |
| `undefined reference to 'sin'` | Missing math lib link | Add`-lm`to`fcompile`flags |