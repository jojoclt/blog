---
title: "Deploying Models to Android (Arm64) [PART-2]"
summary: "Despite the old models, some still have requirements to use this model"
categories: ["Post","Blog",]
tags: ["models","mxnet","tvm"]
#externalUrl: ""
showSummary: true
date: 2025-12-10
series: ["tvm"]
series_order: 2
---


# **Android TVM Demo: Running a Full Inference Pipeline with TVM + Chaquopy**


Before wiring everything together, there are a few important requirements and project-structure rules that you must follow when running a real model inside an Android TVM demo.

This section explains:

* Where to put your compiled TVM models
* How TVMRunner loads asset files
* What versions of Apache TVM work
* Gradle / NDK requirements
* Chaquopy build quirks
* How to structure your inference pipeline into **Android ↔ Python stages**

---

# 0. Code for compiling MXNet Model to TVM Models. (Recommend using NDK29 as it supports 16 KB Page Size which require for Android 16)

```py
import os

# ======================================================
# 🔧 CONFIG
# ======================================================
# TODO Set to True if compiling for local machine and set the target accordingly
local = False
ndk_path = os.environ.get("ANDROID_NDK")
if ndk_path is None and not local:
    raise EnvironmentError("Please set ANDROID_NDK environment variable for cross-compilation.")
toolchain = f"{ndk_path}/toolchains/llvm/prebuilt/linux-x86_64/bin"
cross_cc = f"{toolchain}/aarch64-linux-android21-clang"

import mxnet as mx
import tvm
from tvm import relay
from tvm.contrib import cc

# ======================================================
# 🧠 1. Load your MXNet model
# ======================================================
# Replace these with your own model filenames (without extensions)

# TODO ./path/to/file/my-model
prefix = "./ecg12_v2/ECG12Net_v2 (MI-1)"
# TODO epoch 0000
epoch = 0  # or whatever number matches your params filename

output_prefix = "./a_very_new_ecg_AMI/"
os.makedirs(output_prefix, exist_ok=True)

# This loads: my_model-symbol.json and my_model-0000.params
sym, arg_params, aux_params = mx.model.load_checkpoint(prefix, epoch)

# ======================================================
# 🧩 2. Convert to Relay IR
# ======================================================
# TODO change shape as needed
shape_dict = {"data": (11,1,12,1024)}  # change if your model expects different input
mod, params = relay.frontend.from_mxnet(
    sym,
    shape_dict,
    arg_params=arg_params,
    aux_params=aux_params
)

# ======================================================
# ⚙️ 3. Build with TVM (CPU target example)
# ======================================================

if local:
    target = "llvm"  # or "llvm -mtriple=aarch64-linux-android" for Android
else:
    target = "llvm -mtriple=aarch64-linux-android"

with tvm.transform.PassContext(opt_level=3):
    lib = relay.build(mod, target=target, params=params)

# Optional cross-compile for Android
if local:
    compile = None
else:
    compile = cc.cross_compiler(
    cross_cc,
    options=[
        "-lm",
        # "-Wl,-z,max-page-size=0x4000"
    ],
)

lib.export_library(os.path.join(output_prefix, "deploy_lib_cpu.so"), fcompile=compile)

# ======================================================
# 📦 4. Export artifacts
# ======================================================
with open(os.path.join(output_prefix, "deploy_graph.json"), "w") as f:
    f.write(lib.get_graph_json())

with open(os.path.join(output_prefix, "deploy_param.params"), "wb") as f:
    f.write(relay.save_param_dict(lib.get_params()))

print("✅ Export done: deploy_lib_cpu.so, deploy_graph.json, deploy_param.params")


```

# **1. Project Requirements**

### ✅ **1.1 Pre-built TVM models**

This demo assumes you already compiled models using TVM (Relay → Graph Executor → .so + JSON + params).
Place all compiled runtime artifacts inside:

Note that this guide only works on Android Physical Device, if you want to try it on emulator Android x86 on Windows, you need to rebuild the model to support it.

```
app/src/main/assets/
```

Example:

```
assets/
 └── standard/ami
        └── deploy_graph.json
        └── deploy_param.params
 └── deploy_lib_cpu.so
```

Your `TVMRunner` should point to these exact filenames:

```kotlin
val jsonPath = "deploy_graph.json"
val paramsPath = "deploy_param.params"
val libPath = "deploy_lib_cpu.so"
```

Assets are copied into the APK automatically.

---

# **2. Apache TVM Version Compatibility**

Tested and working versions:

* **TVM 0.11.0**

Older versions use slightly different graph executor APIs and may require patching.
Newer builds (compared to upstream main) still work as long as the JNI runtime symbols match.

---

# **3. Android Project Folder Layout**

To ensure Gradle can find runtime sources, JNI paths, and tvm4j JAR files:

```
tvm/
└── apps/
    └── android_deploy/   ← your Android Studio project should be placed here
```

Why?
Some Gradle tasks depend on relative paths such as:

```
../../../jvm/core/target/
../../../jvm/native/src/main/native/
```

If you move the Android folder elsewhere, everything breaks.

---

# **4. NDK Path Requirement (Very Important)**

In your `build.gradle.kts`, you hard-point to your local NDK path:

```kotlin
val ndkBuild = "C:\\Users\\Jojo\\AppData\\Local\\Android\\Sdk\\ndk\\29.0.13846066\\build\\ndk-build.cmd"
```

or on linux with the ndk downloaded it will be at /android-ndk-r29/build/ndk-build.cmd or ndk-build

✔ Make sure the version number matches the NDK version installed in:
**Android Studio → Settings → SDK Manager → Android SDK → SDK Tools → NDK**

If you upgrade the NDK, this path must be updated.

---

# **5. Chaquopy Build Quirks**

Chaquopy is powerful, but sometimes Android Studio behaves… uniquely.

Common issue:

```
Cannot Run app on device, sdk not found.
```

This happens **even if the SDK is installed**.

### ✔ Fixes:

* Click **Sync Project with Gradle Files**
* Run **Build → Make Project**
* If still broken → close Android Studio → reopen
* After reopening, Chaquopy usually resolves and the build succeeds

This is a known Chaquopy behavior because it performs its own internal Python environment bootstrap.

---

# **6. Designing a Real Inference Pipeline**

Instead of calling a single big function, it’s cleaner and more maintainable to split the workflow across **Android** and **Python (Chaquopy)**.

separate into

| **Android**       | **Chaquopy (Python)** | **Android**       | **Chaquopy (Python)**                            | **Android**         |
| ----------------- | --------------------- | ----------------- | ------------------------------------------------ | ------------------- |
| Read JSON         | Preprocess Data       | Run TVM Inference | Postprocess Data                                 | Show Result to User |
| Input: (12, 5000) | (11, 1, 12, 1024)     | Output: (11, 3)   | AMI classification (e.g., STEMI, NSTEMI, PSTEMI) | Final UI Display    |



# **7. Summary**

Your Android TVM demo must include:

### ✔ Prebuilt TVM models in **assets/**

### ✔ Android folder placed **inside `/apps`** of TVM

### ✔ Correct NDK path in `build.gradle.kts`

### ✔ Chaquopy sync issues (rebuild / restart)

### ✔ Clean multi-stage inference pipeline:

1. Android → Read raw data
2. Chaquopy → Preprocess
3. Android → Run TVM
4. Chaquopy → Postprocess
5. Android → Display result

This separation makes the demo reliable, maintainable, and easy to verify.