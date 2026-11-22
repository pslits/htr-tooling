# ONNX Runtime only through mutually exclusive extras, never in the base install

`pip install htr-tooling` installs no ONNX Runtime. That install runs every step through the Docker runtime. ONNX Runtime comes only from one of three extras: `cpu` (`onnxruntime`), `openvino` (`onnxruntime` plus `onnxruntime-ep-openvino`, pinned as a pair) and `cuda` (`onnxruntime-gpu[cuda,cudnn]`). These are declared as conflicts in `[tool.uv]`. The reason is that `onnxruntime`, `onnxruntime-gpu`, `-directml` and `-windowsml` all install the same `onnxruntime/` module and declare no `Provides-Dist`, so whichever is installed last silently overwrites the other, and `get_available_providers()` simply loses the EP. A base dependency on `onnxruntime` would therefore break every CUDA install without an error. pip cannot enforce the exclusivity, so a run stops with an error that names the extra to install whenever the requested runtime or execution provider is not really there. See [the ONNX Runtime support matrix](https://github.com/pslits/htr-tooling/issues/7) and [Python version, dependencies, packaging and releases](https://github.com/pslits/htr-tooling/issues/13).

## Considered Options

- **`onnxruntime` in the base, with "swap in `onnxruntime-gpu` by hand" documented for CUDA.** Rejected. A plain install would work on the CPU, but every CUDA user hits the silent overwrite, and every upgrade can bring the CPU wheel back.
- **A `directml` extra.** Rejected. `onnxruntime-directml` is frozen at 1.24.4, while the OpenVINO plugin needs ORT 1.30 or later, so ORT versions would fork permanently. Windows ML, the successor, needs a separately installed Windows App SDK runtime and is out of scope for now.

## Consequences

- "Install exactly one extra" is a rule that only uv enforces. The README states it for pip users, and the run-time check catches the case where it is broken.
- The brief's §5 platform table loses its DirectML row. Windows is tested on CPU and OpenVINO; CUDA is supported but not tested on Windows and Linux.
- The ORT range and the OpenVINO plugin version are bumped together, by hand, after a local run on the Arc and NPU. Dependabot ignores them.
