# ONNX Runtime and OpenVINO support matrix for Windows and Linux

Research for [issue #7](https://github.com/pslits/htr-tooling/issues/7). State as of **2026-11-20**. All PyPI data comes from the JSON API (`https://pypi.org/pypi/<name>/json`) and from the wheel files themselves (file lists and `METADATA` read from the published wheels). "Win" means `win_amd64`, "Linux" means `manylinux_*_x86_64`.

## Short answer

- **Python floor is 3.11.** `onnxruntime` dropped Python 3.10 wheels with 1.24.1 (2026-02-05). Every current ONNX Runtime package ships wheels for 3.11–3.14, except the legacy `onnxruntime-openvino` (3.11–3.13 only, frozen at 1.24.1).
- **OpenVINO now comes as a plugin.** The OpenVINO EP that was built into ONNX Runtime is deprecated. Its replacement is `onnxruntime-ep-openvino` (Intel, 1.8.1). This is one `py3-none` wheel per OS that bundles the OpenVINO runtime and is loaded into a normal `onnxruntime` install. It reaches the Arc 140T GPU and the NPU of the Core Ultra 7 255H.
- **The `onnxruntime*` "flavour" packages cannot be installed side by side.** `onnxruntime`, `-gpu`, `-openvino`, `-directml` and `-windowsml` all install the same `onnxruntime/` package directory and none declares `Provides-Dist`. If two are installed, the last one wins without any warning. Plugin EPs (`onnxruntime-ep-openvino`) install into their own module and do coexist.
- **DirectML has stalled.** It is in "sustained engineering", and `onnxruntime-directml` has had no release since 1.24.4 (2026-03-17). New Windows GPU/NPU work goes to Windows ML (`onnxruntime-windowsml`, current at 1.30.1).

## 1. Wheel matrix

| PyPI package | Latest (upload date) | Win x64 | Linux x64 | What it is |
|---|---|---|---|---|
| `onnxruntime` | 1.32.0 (2026-11-13) | cp311–cp314 | cp311–cp314 (`manylinux_2_28`) | CPU EP. Also hosts plugin EPs. |
| `onnxruntime-gpu` | 1.32.0 (2026-11-13) | cp311–cp314 | cp311–cp314 | CUDA EP (+ TensorRT EP DLL). Built for CUDA 13.0 / cuDNN 9 since 1.27. Extras `[cuda,cudnn]` pull the NVIDIA runtime wheels. |
| `onnxruntime-directml` | 1.24.4 (2026-03-17) | cp311–cp314 | — | DirectML EP. No release for 8 months. |
| `onnxruntime-openvino` | 1.24.1 (2026-02-26) | cp311–cp313 | cp311–cp313 | Legacy built-in OpenVINO EP, OpenVINO 2025.4.1. Deprecated. |
| `onnxruntime-ep-openvino` | 1.8.1 (2026-10-26) | `py3-none` (any CPython ≥3.10) | `py3-none` (`manylinux_2_28`) | OpenVINO **plugin** EP. Bundles OpenVINO + TBB. Needs a separate `onnxruntime` ≥1.23. |
| `onnxruntime-windowsml` | 1.30.1.202610302006 (2026-11-13) | cp311–cp314 (also win_arm64) | — | Windows ML's copy of ORT. Ships `DirectML.dll`. |
| `windowsml` | 2.5.96 (2026-10-26) | `py3-none` | — | ctypes bindings for the Windows ML EP catalog. Extra `[with-ort]` pins `onnxruntime-windowsml==1.28.3.202610130701`. |
| `openvino` | 2026.4.1 (2026-10-01) | cp310–cp314 | cp310–cp314 | OpenVINO's own runtime. Reads `.onnx` directly. |

Sources: [onnxruntime](https://pypi.org/pypi/onnxruntime/json), [onnxruntime-gpu](https://pypi.org/pypi/onnxruntime-gpu/json), [onnxruntime-directml](https://pypi.org/pypi/onnxruntime-directml/json), [onnxruntime-openvino](https://pypi.org/pypi/onnxruntime-openvino/json), [onnxruntime-ep-openvino](https://pypi.org/pypi/onnxruntime-ep-openvino/json), [onnxruntime-windowsml](https://pypi.org/pypi/onnxruntime-windowsml/json), [windowsml](https://pypi.org/pypi/windowsml/json), [openvino](https://pypi.org/pypi/openvino/json).

### By Python version (latest release that has a wheel)

| Package | 3.10 | 3.11 | 3.12 | 3.13 | 3.14 |
|---|---|---|---|---|---|
| `onnxruntime` / `-gpu` | ≤1.23.2 only (2025-10-22) | 1.32.0 | 1.32.0 | 1.32.0 | 1.32.0 |
| `onnxruntime-directml` | ≤1.23.0 only | 1.24.4 | 1.24.4 | 1.24.4 | 1.24.4 |
| `onnxruntime-openvino` | ≤1.23.0 only | 1.24.1 | 1.24.1 | 1.24.1 | **none** |
| `onnxruntime-ep-openvino` | 1.8.1 (but no current ORT) | 1.8.1 | 1.8.1 | 1.8.1 | 1.8.1 |
| `onnxruntime-windowsml` | ≤1.27.0 only | 1.30.1 | 1.30.1 | 1.30.1 | 1.30.1 |
| `openvino` | 2026.4.1 (last; dropped in 2026.5) | 2026.4.1 | 2026.4.1 | 2026.4.1 | 2026.4.1 |

Notes on the matrix:

- **3.10 is gone.** `onnxruntime` 1.23.2 is the last release with cp310 wheels, and 1.24.1 (2026-02-05) is the first without ([PyPI](https://pypi.org/pypi/onnxruntime/json)). The Intel release notes for the OpenVINO EP say "Python 3.10 wheels are no longer published" ([intel/onnxruntime v5.9, 2026-02-25](https://github.com/intel/onnxruntime/releases/tag/v5.9)). OpenVINO announces: "Support for Python 3.10 will be discontinued in OpenVINO 2026.5" ([OpenVINO 2026.4.0 release notes](https://github.com/openvinotoolkit/openvino/releases/tag/2026.4.0)).
- **3.14 is supported** by `onnxruntime` since 1.24.1. No package ships free-threaded (`cp314t`) wheels ([PyPI](https://pypi.org/pypi/onnxruntime/json)).
- **`onnxruntime-openvino` 1.24.1 metadata is inconsistent.** It says `Requires-Python >=3.10` and its description claims "Python 3.11, 3.12, 3.13 and 3.14", but only cp311–cp313 wheels exist ([PyPI](https://pypi.org/pypi/onnxruntime-openvino/1.24.1/json)).
- **Windows ML docs lag the wheels.** The docs say "Python 3.10 to 3.13" ([get started, ms.date 2026-04-28](https://learn.microsoft.com/en-us/windows/ai/new-windows-ml/get-started)), while `onnxruntime-windowsml` has shipped cp311–cp314 since 1.27.1 (2026-07-15) ([PyPI](https://pypi.org/pypi/onnxruntime-windowsml/json)).
- **CUDA 13 is the default.** "Starting with version 1.27, GPU packages published to PyPI (`onnxruntime-gpu`) … are built with CUDA 13.0 by default" ([CUDA EP docs](https://onnxruntime.ai/docs/execution-providers/CUDA-ExecutionProvider.html)). The `onnxruntime-gpu` 1.32.0 extras pin `nvidia-cuda-runtime~=13.0` and `nvidia-cudnn-cu13~=9.0` (wheel `METADATA`). The [install page](https://onnxruntime.ai/docs/install/) still says "CUDA 12.x (default)", so it is out of date.
- **Linux wheels** are `manylinux_2_28`, which needs glibc ≥2.28 (`onnxruntime`, `-gpu`, `-openvino`, `-ep-openvino`).

## 2. Intel Core Ultra 7 255H: Arc 140T GPU and NPU

**The hardware.** The Core Ultra 7 255H is "Intel Core Ultra Processors (Series 2)", formerly Arrow Lake, launched Q1'25. It has an "Intel Arc 140T GPU" and an "Intel AI Boost" NPU rated at 13 TOPS (Int8) ([Intel ARK](https://www.intel.com/content/www/us/en/products/sku/241751/intel-core-ultra-7-processor-255h-24m-cache-up-to-5-10-ghz/specifications.html)).

**Which OpenVINO version.** OpenVINO 2025.0 (2025-02-05) states: "Intel Core Ultra 200H processors (formerly code named Arrow Lake-H) are now fully supported" ([release notes](https://www.intel.com/content/www/us/en/developer/articles/release-notes/openvino/2025-0.html)). OpenVINO 2025.2 added "Enhanced Linux support with the latest GPU driver for built-in GPUs on Intel Core Ultra Processor Series 2 (formerly codenamed Arrow Lake H)" ([release notes](https://www.intel.com/content/www/us/en/developer/articles/release-notes/openvino/2025-2.html)). The current system requirements list Intel Arc GPUs and Core Ultra Series 1–3. The NPU is supported on "Windows 11, 64-bit (22H2 and later)" and Ubuntu 22.04/24.04 ([system requirements](https://docs.openvino.ai/2026/about-openvino/release-notes-openvino/system-requirements.html)). So any OpenVINO ≥2025.0 covers this machine. That includes the 2025.4.1 inside `onnxruntime-openvino` 1.24.1 and the 2026.4+ inside `onnxruntime-ep-openvino` 1.8.1.

### Routes to the Arc and the NPU from Python

| Route | Package(s) | Status | Notes |
|---|---|---|---|
| **A. OpenVINO plugin EP** | `onnxruntime` + `onnxruntime-ep-openvino` | Current, recommended by ORT docs | Wheel bundles `openvino.dll`, the CPU/GPU/NPU plugins, `openvino_intel_npu_compiler.dll` and TBB (verified from the 1.8.1 `win_amd64` wheel file list). Built against OpenVINO ≥2026.4 (`find_package(OpenVINO 2026.4 REQUIRED …)` in [cmake/openvino.cmake @ v1.8.1](https://github.com/onnxruntime/onnxruntime-ep-openvino/blob/v1.8.1/cmake/openvino.cmake)). The exact bundled patch version was not established. |
| B. Legacy built-in EP | `onnxruntime-openvino` (+ `openvino==2025.4.1` on Windows) | Deprecated, frozen at 1.24.1 | "The built-in OpenVINO Execution Provider in the ONNX Runtime repository is deprecated. We strongly recommend using the standalone Plugin EP instead" ([OpenVINO EP docs](https://onnxruntime.ai/docs/execution-providers/OpenVINO-ExecutionProvider.html)). Linux wheels bundle OpenVINO 2025.4.1. Windows needs `pip install openvino==2025.4.1` plus `onnxruntime.tools.add_openvino_win_libs.add_openvino_libs_to_path()` ([v5.9 release](https://github.com/intel/onnxruntime/releases/tag/v5.9)). |
| C. OpenVINO directly | `openvino` | Current (2026.4.1) | `read_model`/`compile_model` take `.onnx` directly. The docs warn this "offers lower performance and stability, as well as fewer optimization options" than converting to IR ([model preparation](https://docs.openvino.ai/2026/openvino-workflow/model-preparation.html)). It is a second, non-ORT runtime API. |
| D. Windows ML catalog | `onnxruntime-windowsml` + `wasdk-*` packages | Current | Intel's `OpenVINOExecutionProvider` is downloaded on demand through `ExecutionProviderCatalog` on Windows 11 24H2 (build 26100)+ ([Windows ML EPs](https://learn.microsoft.com/en-us/windows/ai/new-windows-ml/supported-execution-providers)). The plugin repo also has a `packaging/winml/msix` target ([tree @ v1.8.1](https://github.com/onnxruntime/onnxruntime-ep-openvino/tree/v1.8.1/packaging/winml)), so this is most likely the same plugin delivered as an MSIX (not verified). |

**How route A is used.** The plugin uses ORT's newer EP-device API, not `providers=["OpenVINOExecutionProvider"]`:

```python
ort.register_execution_provider_library("openvino_ep", onnxruntime_ep_openvino.get_library_path())
devs = [d for d in ort.get_ep_devices() if d.ep_name == "OpenVINOExecutionProvider" and d.ep_metadata.get("ov_device") == "GPU"]
so = ort.SessionOptions(); so.add_provider_for_devices(devs, {})
```

The devices are `CPU`, `GPU`, `GPU.1`, … and `NPU`. AUTO is exposed as a separate EP name, `OpenVINOExecutionProvider.AUTO`. For the NPU, "static shapes preferred — see `reshape_input`" ([QuickStart @ v1.8.1](https://github.com/onnxruntime/onnxruntime-ep-openvino/blob/v1.8.1/docs/QuickStart.md); [PyPI README](https://pypi.org/project/onnxruntime-ep-openvino/)).

**Version coupling between the plugin and ORT.**

- The PyPI README says "Requires ONNX Runtime 1.23.0 or later", but the build requirements say ORT 1.30.0+ ([README](https://github.com/onnxruntime/onnxruntime-ep-openvino/blob/v1.8.1/README.md)).
- ORT 1.31 broke plugin 1.8.0. The fix shipped in 1.8.1 ([issue #4](https://github.com/onnxruntime/onnxruntime-ep-openvino/issues/4); [v1.8.1 release, 2026-10-23](https://github.com/onnxruntime/onnxruntime-ep-openvino/releases/tag/v1.8.1)).
- ORT 1.32.0 support is announced for "the next release of OpenVINO EP" ([issue #9 comment, 2026-11-19](https://github.com/onnxruntime/onnxruntime-ep-openvino/issues/9#issuecomment-7012303577)). Whether 1.8.1 works with ORT 1.32.0 was **not established**.
- The plugin wheel declares **no** `Requires-Dist` on `onnxruntime` (wheel `METADATA`), so the application has to pin ORT itself.

**Known costs.** Registering the plugin loads every OpenVINO device plugin by default, about 180 MB with GPU/OpenCL, and these cannot be unloaded. The env-level config `ep.openvinoexecutionprovider.hardware_device_type` limits this ([issue #3](https://github.com/onnxruntime/onnxruntime-ep-openvino/issues/3), open).

**Driver requirements on Windows.**

- **GPU (Arc 140T):** the Intel graphics driver. "The use of GPU requires drivers that are not included in the Intel Distribution of OpenVINO toolkit package" ([system requirements](https://docs.openvino.ai/2026/about-openvino/release-notes-openvino/system-requirements.html)). OpenVINO gives no minimum version for native Windows. The only stated minimum is 30.0.100.9955 for WSL ([GPU configuration](https://docs.openvino.ai/2026/get-started/install-openvino/configurations/configurations-intel-gpu.html)). Intel's Windows ML page says to "install the latest GPU and NPU drivers using Intel Driver Support Assistant" ([Intel Windows ML support](https://www.intel.com/content/www/us/en/developer/articles/guide/windows-ml-support.html)).
- **NPU:** the Intel NPU driver for Windows, "available through Windows Update but it may also be installed manually" ([NPU configuration](https://docs.openvino.ai/2026/get-started/install-openvino/configurations/configurations-intel-npu.html)). The latest is **32.0.100.5600 (2026-11-19)**. It supports Core Ultra Series 2 (Arrow Lake) on Windows 11 23H2–26H1 and "includes support for OpenVINO 2026.4.1" ([Intel download page](https://www.intel.com/content/www/us/en/download/794734/intel-npu-driver-windows.html)).
- **NPU compiler:** OpenVINO 2026.0 previewed "NPU compiler integration with the NPU plugin … without relying on OEM driver updates" ([2026.0.0 release](https://github.com/openvinotoolkit/openvino/releases/tag/2026.0.0)). The plugin wheel ships `openvino_intel_npu_compiler.dll`.
- **Linux, for reference:** the NPU needs [intel/linux-npu-driver](https://github.com/intel/linux-npu-driver). v1.39.1 (2026-11-19) lists Arrow Lake on Ubuntu 24.04/26.04 with OpenVINO 2026.4.1 ([release](https://github.com/intel/linux-npu-driver/releases/tag/v1.39.1)).

## 3. Can several EP packages coexist?

**Flavour packages: no.** I checked the cp312 `win_amd64` wheels of `onnxruntime` 1.32.0, `onnxruntime-gpu` 1.32.0, `onnxruntime-openvino` 1.24.1, `onnxruntime-directml` 1.24.4 and `onnxruntime-windowsml` 1.30.1. Each one installs the top-level package `onnxruntime/` with its own `capi/onnxruntime_pybind11_state` and provider DLLs. None declares `Provides-Dist: onnxruntime` (wheel `METADATA`). pip and uv therefore treat them as unrelated distributions, so both get installed and the files of the last one overwrite the other. The failure is silent: `get_available_providers()` simply loses the EP. This is documented with a reproduction in [qdrant/fastembed#608](https://github.com/qdrant/fastembed/issues/608) (2026-02-25). It explains that `onnxruntime-gpu` stopped declaring `Provides-Dist: onnxruntime` after ~1.19, so any dependency on `onnxruntime` by name pulls in the CPU wheel next to `-gpu`. The ORT docs do not warn about this anywhere I could find ([install page](https://onnxruntime.ai/docs/install/)).

**Plugin EPs: yes.**

- `onnxruntime-ep-openvino` installs the separate module `onnxruntime_ep_openvino`. It only provides a library path that is registered at run time, and "can be loaded by any ONNX Runtime 1.23.0+ application" ([README](https://github.com/onnxruntime/onnxruntime-ep-openvino/blob/v1.8.1/README.md)). So `onnxruntime` + `onnxruntime-ep-openvino` coexist by design. `onnxruntime-gpu` + `onnxruntime-ep-openvino` should coexist too; this follows from the plugin API but was not tested.
- Counter-example of a plugin that does declare the dependency: `onnxruntime-qnn` 2.8.0 is also a plugin (own module `onnxruntime_qnn`) but declares `Requires-Dist: onnxruntime>=1.24.2` (wheel `METADATA`). Installed next to `onnxruntime-gpu`, it would trigger the clash above.

**GitHub caveat.** `https://github.com/microsoft/onnxruntime` returns **HTTP 404** as of 2026-11-20, both through `gh api` and the web. This means the ORT repo's issues (for coexistence reports) and the release notes linked from PyPI could not be consulted. PyPI and onnxruntime.ai still serve. The plugin EPs now live under the [`onnxruntime` GitHub organisation](https://github.com/onnxruntime) (`onnxruntime-ep-openvino`, `onnxruntime-qnn`, `onnxruntime-ep-tensorrt`, …).

## 4. DirectML and Windows ML

- **DirectML.** "DirectML is in sustained engineering. DirectML continues to be supported, but new feature development has moved to WinML". It needs `DisableMemPattern` and `ORT_SEQUENTIAL`, and only one thread may call `Run` per session ([DirectML EP docs](https://onnxruntime.ai/docs/execution-providers/DirectML-ExecutionProvider.html)). `onnxruntime-directml` stopped at 1.24.4 (2026-03-17), so a DirectML extra built on it pins ORT to 1.24.x.
- **What Windows ML is.** "the Windows-supported and maintained copy of ONNX Runtime", with CPU and DirectML ("legacy") included. Vendor EPs (OpenVINO, QNN, VitisAI, NvTensorRtRtx, AMD GPU) are downloaded on demand on Windows 11 24H2 (build 26100)+ ([overview](https://learn.microsoft.com/en-us/windows/ai/new-windows-ml/overview); [EPs](https://learn.microsoft.com/en-us/windows/ai/new-windows-ml/supported-execution-providers)).
- **Windows ML from Python.** It is framework-dependent only: `pip install wasdk-Microsoft.Windows.AI.MachineLearning[all] wasdk-Microsoft.Windows.ApplicationModel.DynamicDependency.Bootstrap onnxruntime-windowsml`, *plus* a separately installed Windows App SDK Runtime of the matching version. Python must be "unpackaged … not the Microsoft Store" ([install and deploy](https://learn.microsoft.com/en-us/windows/ai/new-windows-ml/distributing-your-app)). The newer `windowsml` package offers a ctypes wrapper for the EP catalog ([PyPI](https://pypi.org/project/windowsml/)).
- **Assessment.** This is a viable route for "DirectML/NPU on Windows" with an up-to-date ORT (1.30.1). It is heavier to install (system runtime, Store-Python exclusion) and is Windows-only. I did not verify at run time whether `onnxruntime-windowsml` exposes `DmlExecutionProvider` through the plain Python `providers=` list; the wheel does ship `DirectML.dll`.

## 5. Implications for htr-tooling

These are the facts the Python-version and packaging decision (#13) depends on:

1. **`requires-python = ">=3.11"` is forced by ONNX Runtime.** No current ORT wheel exists for 3.10. 3.11–3.14 are all covered by `onnxruntime`, `-gpu`, `-windowsml`, `onnxruntime-ep-openvino` and `openvino`. Only the deprecated `onnxruntime-openvino` lacks 3.14.
2. **Use the plugin EP for OpenVINO**, not `onnxruntime-openvino`. It is Python-version independent (`py3-none`), tracks current ORT, bundles OpenVINO (no `setupvars` and no `add_openvino_libs_to_path` on Windows), and is additive on top of `onnxruntime`. An extra like `openvino = ["onnxruntime-ep-openvino>=1.8.1"]` does not clash.
3. **Pin ORT and the plugin as a pair.** The plugin does not declare its ORT dependency, and ORT minor releases have broken it (1.31 vs plugin 1.8.0). Plugin 1.8.1 "Supports ORT 1.30 feature set" and fixes ORT 1.31. ORT 1.32.0 support is pending. A safe pin today is `onnxruntime>=1.30,<1.32` with `onnxruntime-ep-openvino==1.8.1`.
4. **CUDA and DirectML cannot be additive extras** if the base depends on `onnxruntime`. `onnxruntime-gpu` and `onnxruntime-directml` overwrite the same `onnxruntime/` directory, and the loss of the EP is silent. Options:
   - keep ORT out of the base and offer **mutually exclusive** extras (`cpu`/`cuda`/`directml`), declared with `[tool.uv] conflicts` ([uv docs](https://docs.astral.sh/uv/concepts/projects/config/#conflicting-dependencies)); pip has no equivalent;
   - or document "replace `onnxruntime` with `onnxruntime-gpu`" as a manual step;
   - in either case, add a runtime check that the requested EP is in `ort.get_available_providers()` / `ort.get_ep_devices()`, and fail loudly if it is not.
5. **DirectML is a weak "supported, not tested" target.** `onnxruntime-directml` is frozen at 1.24.4 and would cap ORT below what the OpenVINO plugin is built against (1.30+). Windows ML (`onnxruntime-windowsml`) is the maintained replacement but needs the Windows App SDK runtime. The brief's §5 table may need a revision here.
6. **The runtime port needs two EP-selection mechanisms.**
   - The classic `providers=[...]` / `provider_options` for CPU, CUDA and DirectML.
   - The EP-device API (`register_execution_provider_library`, `get_ep_devices`, `add_provider_for_devices`) for the OpenVINO plugin and for Windows ML catalog EPs.
7. **Driver prerequisites belong in the docs, not in the package.** The Windows GPU driver and the Windows NPU driver (Windows Update or Intel; ≥32.0.100.5600 pairs with OpenVINO 2026.4.1) are not installable via pip.

## Not established / open

- Whether `onnxruntime-ep-openvino` 1.8.1 works with `onnxruntime` 1.32.0 (released 2026-11-13).
- The exact OpenVINO patch version bundled in `onnxruntime-ep-openvino` 1.8.1 (the build minimum is 2026.4).
- Whether the OpenVINO plugin enumerates a CPU device on **non-Intel** CPUs. A commenter on [issue #9](https://github.com/onnxruntime/onnxruntime-ep-openvino/issues/9#issuecomment-7012303577) (not marked as a repo collaborator) says "there is no plan to support non-intel CPUs with this EP". This matters if Linux CI runs on AMD hosts.
- Whether `onnxruntime-windowsml` exposes DirectML via the classic `providers=` list in Python.
- Why `github.com/microsoft/onnxruntime` returns 404, and whether it is coming back. The ORT issue tracker could not be searched for coexistence reports.
- Laypa/Loghi-HTR operator coverage and the effect of dynamic shapes on the NPU (`reshape_input`). These belong to the ONNX exploration, not to this ticket.
