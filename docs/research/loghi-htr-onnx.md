# Loghi-HTR to ONNX: feasibility and the processing around the network

*Research for [#6](https://github.com/pslits/htr-tooling/issues/6). Written 2026-11-20. Everything here comes from reading sources. No conversion was run and no model was loaded.*

## Sources and versions read

| What | Version / commit | Link prefix used below |
|---|---|---|
| Loghi-HTR | tag `2.3.2` = HEAD `c24c745e9aeb287bedeb6b4edee670ae5c81a3cd` (2026-01-27) | `LH:` = https://github.com/knaw-huc/loghi-htr/blob/c24c745e9aeb287bedeb6b4edee670ae5c81a3cd/ |
| Loghi (pipeline + Dockerfiles) | `main` = tag `2.3.2` = `e74351383ec64fd03689c8550d3123f0eb92c037` (2026-10-28) | `LO:` = https://github.com/knaw-huc/loghi/blob/e74351383ec64fd03689c8550d3123f0eb92c037/ |
| Keras | tag `v3.13.2` = `e29d0efbefe21b80c041849bd73cc3e54c872de3` (the version in the Loghi HTR Docker image) | `K:` = https://github.com/keras-team/keras/blob/e29d0efbefe21b80c041849bd73cc3e54c872de3/ |
| tf2onnx | release `v1.17.0` = `e4962b280d9bfa75ea1217cfc59e785a25fa1c4b` (2026-03-04); `main` = `1f665ad97f833ed6f795ef7110e3ea559274df66` (2026-11-03) | `T:` = https://github.com/onnx/tensorflow-onnx/blob/e4962b280d9bfa75ea1217cfc59e785a25fa1c4b/ |
| TensorFlow | tag `v2.20.0` | `TF:` = https://github.com/tensorflow/tensorflow/blob/v2.20.0/ |
| Published models | SURFdrive share `YA8HJuukIUKznSP`, folder `loghi-htr/`, listed over its public WebDAV on 2026-11-20 | https://surfdrive.surf.nl/files/index.php/s/YA8HJuukIUKznSP?path=%2Floghi-htr |

**Which TensorFlow / Keras.** The image `loghi/docker.htr:2.3.2` builds TensorFlow **2.20.0** from source and installs **Keras 3.13.2** and numpy 2.4.2 (`LO:docker/docker.htr/Dockerfile` lines 36–44, 66–70). It clones loghi-htr at whatever HEAD is current at build time, with no pinned tag (line 59). The repo's own `requirements.txt` says TF 2.19.0 / Keras 3.9.2, and the Dockerfile notes "The requirements.txt is not used" (line 64). So the brief's "TF 2.20" is correct for the Docker reference.

## 1. Short answer

- **Conversion is feasible in principle, but no released tool supports it.** The most promising route is Keras 3's own `model.export(path, format="onnx")` on the TensorFlow backend. It calls `tf2onnx.convert.from_function` on a traced `tf.function` (`K:keras/src/export/onnx.py` L88–103). Keras's own test suite exports a model of **stacked `Bidirectional(LSTM)` layers** this way and checks the ONNX Runtime output against Keras (`K:keras/src/export/onnx_test.py` L49–70, L92–112). However, tf2onnx itself is tested only on TF 2.13 and 2.15, i.e. Keras 2 (`T:README.md` L35–38; `.github/workflows/unit_test_ci.yml` L35 on `main`). Its maintainers say TF 2.16+/Keras 3 is not yet supported ([onnx/tensorflow-onnx#2329](https://github.com/onnx/tensorflow-onnx/issues/2329), [#2348](https://github.com/onnx/tensorflow-onnx/issues/2348), [#2486](https://github.com/onnx/tensorflow-onnx/issues/2486)). Whether a full Loghi-HTR model converts can only be shown by trying it.
- **Known blockers, and how to avoid them:**
  1. Tracing on a machine that can see a GPU produces `CudnnRNNV3`, which tf2onnx cannot convert. The fix is to convert on the CPU.
  2. `tf2onnx.convert.from_keras` crashes on Keras 3 models (`KeyError: 'keras_tensor_N'`). The fix is to use `from_function`, Keras's `export`, or a SavedModel instead.
  3. Keras skips its ONNX export tests under numpy 2. A conversion environment with numpy < 2 is safest; TF 2.20 allows numpy ≥ 1.26.
  4. The model must have a float32 dtype policy.
  5. The torch backend cannot export dynamic shapes, so the line width would be fixed.
- **Second route:** every published model also exists as a legacy **TF 2.x SavedModel** in `loghi-legacy/`. `python -m tf2onnx.convert --saved-model` on TF 2.15 stays inside tf2onnx's tested range. This route needs no Keras 3 at all.
- **Custom layers:** the only custom layer is `ResidualBlock` (`LH:src/model/custom_layers.py` L11). The recommended published model (`generic_new_17_2023_05_25_4channel`) uses only standard layers: Conv2D, ELU, BatchNormalization, Add, Reshape, 5 × Bidirectional LSTM (512 units per direction), Dropout, Dense(457) and Activation. This comes from the model summary stored in its `config.json`.
- **`skip_mismatch=True`** appears in exactly one place: the fallback in `LH:src/model/management.py` L294–297. That fallback runs when `keras.models.load_model` raises `TypeError`/`ValueError` on a `.keras` file. It loads the weights with `skip_mismatch=True` **inside `warnings.simplefilter("ignore")`**, so any layer that fails to load keeps its random initial weights and **nobody sees a message**. It then **overwrites `model.keras`** (the original is renamed to `model.keras.old`). One published model folder already shows the traces of this fallback (§4.3).
- **Outside the network** (this moves into htr-tooling):
  - decode the PNG to the model's channel count (1, 3 or 4);
  - resize to height 64 with bilinear interpolation, keeping the aspect ratio;
  - divide by 255;
  - pad 25 px left and right with 0;
  - compute `0.5 − x`;
  - transpose to width-major;
  - batch with padding value −10;
  - run softmax output through TF CTC greedy decoding (or beam search) with the blank as the last class;
  - shift indices by +1 into the tokenizer (`[PAD]`, `[UNK]`, then the characters);
  - compute one confidence per line, `exp(sum log p / number of decoded characters)`.

  **Loghi-HTR produces no per-character confidence and no alternatives.**

## 2. Conversion routes

### 2.1 Keras 3 `model.export(format="onnx")`

- **TF and JAX backends:** the model is wrapped in an `ExportArchive` endpoint, traced, and passed to `tf2onnx.convert.from_function(decorated_fn, input_signature, opset=opset_version, output_path=filepath)` (`K:keras/src/export/onnx.py` L88–103, L209–219). Before that, Keras monkey-patches tf2onnx for numpy 2 (`patch_tf2onnx()`, L97; `K:keras/src/export/tf2onnx_lib.py`, "Patches `tf2onnx` to ensure compatibility with numpy>=2.0.0").
- **Input signature:** this comes from the model if none is given (L76–82). For the TF backend, `None` dimensions stay dynamic. The test exports, then runs ORT again with **twice the batch size** (`onnx_test.py` L113–124).
- **Documented constraints** (docstring, L46–56): "The dtype policy must be "float32" for the model". On the torch backend, "dynamic shape feature is not yet supported … all instances of `None` … will be replaced with `1`".
- **BiLSTM coverage:** `get_model("lstm")` builds `Bidirectional(LSTM(10, return_sequences=True))` twice, with merge modes `sum` and `concat`, and compares ORT to Keras with `assertAllClose` (`onnx_test.py` L49–70, L92–112). This test was added for [keras-team/keras#21390](https://github.com/keras-team/keras/issues/21390) (LSTM export failing on the torch backend; fixed by PR #21434, confirmed with Keras 3.12.0 in that issue).
- **Test limits** that matter here:
  - The whole ONNX test class is skipped when **numpy 2.x** is installed ("ONNX export is currently incompatible with NumPy 2.0", L86–90).
  - It is also skipped on GPU ("Fails on GPU", L80–84).
  - The LSTM test uses a fixed sequence length of 4 (`Input((4, 10))`). It does **not** test a variable time axis or a 4-D image input.
- **OpenVINO export:** Keras 3.13 also has `model.export(path.xml, format="openvino")`. On the TF backend it calls `openvino.convert_model` on the traced function (`K:keras/src/export/openvino.py`, TF/JAX branch). This bypasses tf2onnx and produces OpenVINO IR, not ONNX. It is noted as an alternative for the Arc and was not examined further.

### 2.2 tf2onnx directly

- **Supported versions:**
  - TF 2.13–2.15 tested (Keras 2), Python 3.10–3.12, opsets 14–18 tested, default opset **15** (`T:README.md` L35–48).
  - CI on `main` still runs only `tf_version: ['2.13.0', '2.15.0']` (`.github/workflows/unit_test_ci.yml` L35).
  - Issue #2486 (2026-08-19, open) says so plainly: "nothing newer is exercised". It links #2329 ("TF 2.16 / Keras 3 support", open since 2024-04) and #2348 ("`from_keras` broken on TF >= 2.16"). In #2348 a collaborator wrote "Didn't upgrade tf2onnx to support 2.17 yet, it's in the plan" and "Not yet, there are couple of code issues need to be figured out."
- **`from_keras` on Keras 3:**
  - v1.17.0 added a fallback for "TF 2.16+ removed trace_model_call" (`T:tf2onnx/convert.py` L441–474).
  - The output-name lookup still fails: `output_names = [reverse_lookup[out] for out in model_out_names]` raises `KeyError: 'keras_tensor_65'` (L529; [#2448](https://github.com/onnx/tensorflow-onnx/issues/2448), open). A fix is open as [PR #2487](https://github.com/onnx/tensorflow-onnx/pull/2487), validated by its author on TF 2.16.1 / Keras 3.15.1 / ORT 1.29.0.
  - Users report that passing `input_signature` explicitly works around it (comment in #2348). **Use `from_function`, Keras `export`, or `--saved-model` rather than `from_keras`.**
- **cuDNN:**
  - Keras 3 on TF uses the cuDNN kernel `tf.raw_ops.CudnnRNNV3` whenever a GPU is visible and the LSTM arguments allow it (`K:keras/src/backend/tensorflow/rnn.py` L785–807 `cudnn_ok` → `_is_gpu_available()`; L810–859; `CudnnRNNV3` at L933).
  - tf2onnx has a handler for `CudnnRNN` only (`tf2onnx/onnx_opset/rnn.py` L171 on `main`). `CudnnRNNV3` is reported unsupported ([#2359](https://github.com/onnx/tensorflow-onnx/issues/2359), open).
  - **Trace with no GPU visible** (e.g. `CUDA_VISIBLE_DEVICES=-1`). Then Keras uses its generic `rnn()` while-loop.
- **LSTM fusion:**
  - tf2onnx has rewriters that turn TF/Keras LSTM loops into the ONNX `LSTM` op, and Bi-LSTM pairs into a bidirectional `LSTM` (`tf2onnx/rewriter/lstm_tf2_rewriter.py`, `bilstm_rewriter.py`). Their patterns were written for TF 2.x/Keras 2 graphs (variants "TF LSTM", "keras LSTM", "keras LSTM with bias").
  - **Not established:** whether Keras 3's while-loop matches these patterns. If it does not, the loop is emitted as a generic ONNX `Loop`. Such a loop should still compute correctly, but more slowly and with less support across execution providers. Older reports of exactly this fallback: [#1851](https://github.com/onnx/tensorflow-onnx/issues/1851) "Single layer Keras LSTM converted to loop instead of ONNX LSTM op" and [#1948](https://github.com/onnx/tensorflow-onnx/issues/1948).
- **Unsupported ops do not stop the conversion:**
  - `from_keras` and `from_function` call `_convert_common(..., continue_on_error=True)` (`T:tf2onnx/convert.py` L543, L610).
  - An op with no mapping is logged (`logger.error("Tensorflow op [%s: %s] is not supported")`, `tf2onnx/tfonnx.py` L263; "Unsupported ops", L625) and stays in the graph.
  - *Expected, not verified:* such a model then fails when ONNX Runtime creates the session. So this is loud, not silent, but a log line during conversion is easy to miss.
- **Opset:** the network needs nothing beyond Conv, BatchNormalization, Elu, Add, Reshape, MatMul/Gemm, Softmax and LSTM (opset ≥ 7) or Loop, so opsets 15–18 are enough. Keras passes `opset_version=None` → tf2onnx default 15.

### 2.3 Via a SavedModel

- **Fresh export:** Keras 3 can write a TF SavedModel (`model.export(path, format="tf_saved_model")`, `K:keras/src/export/saved_model.py`). That can go through `python -m tf2onnx.convert --saved-model`, which avoids `from_keras`. It still runs tf2onnx on a TF 2.20 graph, outside the tested range.
- **Published legacy SavedModels:** every published model also exists under `loghi-htr/loghi-legacy/<name>/` with `saved_model.pb`, `variables/`, `keras_metadata.pb` and `fingerprint.pb` (WebDAV listing). These were written by TF ≤ 2.15 (Keras 2). Converting them with tf2onnx on TF 2.15 is **inside tf2onnx's tested matrix**. One limit: Loghi-HTR 2.3.2 itself can no longer load them. `convert_model` calls `tf.keras.models.load_model(directory)` and on failure tells the user to run `utils/convert-v2-to-v3`, then `exit(1)` (`LH:src/model/conversion.py` L41–48). That utility pins `tensorflow==2.15.0`, `keras==2.15.0` (`LH:utils/convert-v2-to-v3/requirements.txt`).
- **A check from two sources:** converting both the legacy SavedModel and the current `model.keras` of the same model gives two ONNX files that must agree.

### 2.4 Via the torch backend

- **Possible in principle, poor fit for variable-width lines.** The torch backend cannot export dynamic shapes (§2.1). Every line would have to be padded to one or a few fixed widths, and padding changes BiLSTM output (§5.5).
- **LSTM export bugs:** LSTM export on torch was broken before Keras 3.12, and a reporter says it still fails when a GPU is available (#21390, last comment).
- **`ResidualBlock`** builds `tf.keras.layers` inside `build`/`call` (`LH:src/model/custom_layers.py` L100–160), so models that use it would need checking on a non-TF backend. The recommended model does not use it.

## 3. Model input and output contract (from the code)

| | value | source |
|---|---|---|
| input layout | `[batch, width, height, channels]` (the image is transposed: width is the time axis) | `LH:src/data/loader.py` L109–110; `LH:src/api/batch_predictor.py` L241 |
| height | taken from the model's input layer in the CLI; **hard-coded 64** in the API | `LH:src/data/data_handling.py` L39–41; `batch_predictor.py` L227–234 |
| channels | taken from the model's input layer in the CLI; from `config.json` (`model_channels` or `args.channels`) in the API | `data_handling.py` L40; `batch_predictor.py` L69–110 |
| dynamic dimensions | batch and width are `None`; height and channels are fixed | model summary in the `generic_new_17` `config.json`: `image (InputLayer) [(None, None, 64, 4)]` |
| output | softmax `[batch, T, n_classes]`, float32 (the last Activation is forced to float32) | `LH:src/model/replacing.py` L127–138 |
| classes | `n_classes = len(tokenizer)` = `[PAD]` + `[UNK]` + characters; output index *j* ↔ tokenizer index *j+1*; **the last output index is the CTC blank** | `LH:src/model/losses.py` L33 (`y_true − 1`), L96–100 (TF `ctc_loss`, whose blank is the last class); `LH:src/utils/decoding.py` L104 (`+= 1`) |

Check against a real model: `generic_new_17_2023_05_25_4channel` has 455 characters in `charlist.txt` and `dense3 (Dense) … 457` outputs. 457 = `[UNK]` + 455 characters + blank, which matches the mapping above **if all 455 code points are kept**. Loghi-HTR 2.3.2 drops one of them (U+0007) when it reads `charlist.txt`; see §5.7.

**Mixed precision.** A model trained on GPU runs under the `mixed_float16` policy (`LH:src/setup/environment.py` L183–196). The training callback saves a clone with the same config (`LH:src/model/custom_callback.py` L102–120). Loghi itself rebuilds the model in float32 when running on the CPU (`gpu == "-1"` or `--use_float32` → `adjust_model_for_float32`, `LH:src/model/management.py` L25–62, L120–124). *Not established:* the dtype policy stored in each published `model.keras` (this needs opening the zip's `config.json`). A converter must force float32 the same way, because Keras export requires it (§2.1).

## 4. How Loghi-HTR loads models, and where an untrained network can come from

### 4.1 Normal path

`main.py` → `load_or_create_model` → `load_model_from_directory(dir)` (`LH:src/main.py` L77–79; `LH:src/model/management.py` L305–337, L129–199). The API calls the same function with `compile=False` (`LH:src/api/batch_predictor.py` L169–172). It works like this:

1. If any `*.pb` file is in the folder → `convert_model` (legacy SavedModel; fails under Keras 3 with the hint above) (L165–166).
2. Otherwise take the first `*.keras` file and call `tf.keras.models.load_model(model_file, custom_objects, compile)` (L169–176).

In Keras 3, a full `.keras` load that cannot load some object **raises** `ValueError("A total of N objects could not be loaded…")` (`K:keras/src/saving/saving_lib.py` L417–418, L500–507 → `_raise_loading_failure` L631–645 with `warn_only=False`). So the normal path fails loudly.

### 4.2 The `skip_mismatch=True` fallback

If step 2 raises `TypeError` or `ValueError`, Loghi logs the error and calls `_convert_old_model_to_new` (`management.py` L177–183). That function:

1. Unzips the `.keras` file to the fixed path `/tmp/keras_model_extraction`, rewrites `config.json`, and rebuilds the model with `model_from_json` (L223–291). The rewrites:
   - the module path;
   - BatchNormalization `axis` from a list to an int;
   - Bidirectional LSTM `recurrent_initializer` → `OrthogonalInitializer`, and `time_major` dropped;
   - `Policy` → `DTypePolicy`.
2. **Loads the weights silently** (L294–297):
   ```python
   with warnings.catch_warnings():
       warnings.simplefilter("ignore")
       model.load_weights(os.path.join(temp_dir, 'model.weights.h5'),
                          skip_mismatch=True)
   ```
3. **Persists the result:** renames the original to `model.keras.old` (only if that name does not exist yet) and saves the rebuilt model as `model.keras` (L185–195).

This was introduced in commit `e5f5f88` (2024-08-23, "Fix training + loading legacy models").

**Why this can give an untrained network without any error.** This is how Keras 3.13.2 behaves:

- `load_weights_only` collects every object whose `load_own_variables` raises. With `skip_mismatch=True` it only calls `warnings.warn(msg)` instead of raising (`saving_lib.py` L569–628, L624–625 `_raise_loading_failure(error_msgs, warn_only=skip_mismatch)`, L642–643).
- Loghi's `simplefilter("ignore")` swallows that warning. Each affected layer keeps the values it was **just initialised with** (Glorot/orthogonal random weights, zero biases, BatchNorm γ=1/β=0/mean 0/var 1).
- Mismatches that end up here:
  - **Missing path.** When the weight path for a layer is not in the H5 file (e.g. a layer or sublayer was renamed by the config rewrite or by a different Keras version), `H5IOStore.get` returns an **empty dict** ("Defaults to an empty dict if not found", L1114–1131). `_check_load_own_variables` then raises "expected N variables, but received 0" (`K:keras/src/layers/layer.py` L1383–1415).
  - **Wrong shape.** `v.assign(store[...])` raises (`layer.py` L1428–1429).
- **Further sources of silent wrong weights** (no error at any `skip_mismatch` setting):
  - weights loaded into the right shapes but in the wrong order;
  - an ONNX export of a network that was *already* broken by this fallback (the converter faithfully freezes whatever is in memory);
  - forgetting that `customize_model` replaces the final layer when `--replace_final_layer` is set or when `config["model"]` is not a directory (`management.py` L95–99). That is a training path, not used by the inference pipeline.

**Side effects to keep in mind.**

- The fallback writes inside the model folder (`model.keras` replaced, `.old` created).
- Loading a `charlist.txt` also writes a `tokenizer.json` next to it (`LH:src/utils/text.py` L81–103).
- So running Loghi-HTR can change a model folder. A read-only mount would make the fallback fail instead.

### 4.3 A published model has been through this fallback

The SURFdrive folder `loghi-htr/republic-2023-01-02-base-generic_new14-2022-12-20/` holds:

- `model.keras.old` (127,366,337 B, 2025-05-14 12:59);
- `model.keras` (127,365,795 B, 13:35:19);
- `tokenizer.json` (13:35:12);
- `charlist.txt`.

That is exactly the set of files `_convert_old_model_to_new` plus `Tokenizer.load_from_file` leave behind. File size cannot show whether every layer loaded, because an untrained copy has the same number of parameters. **Not established:** whether that `model.keras` is fully trained. It must be checked as in §4.4.

### 4.4 How to detect a wrong network

All of these work without trusting the conversion:

1. **Load strictly.** Load with `keras.models.load_model` only, never with `skip_mismatch`. Refuse (or flag) a model folder that contains `model.keras.old`.
2. **Compare weights with the source.** A `.keras` file is a zip with `config.json`, `metadata.json` and `model.weights.h5` (`saving_lib.py`, `_VARS_FNAME_H5`). The H5 can be read with `h5py` without TensorFlow. Compare a digest of each layer's variables in `model.keras.old` and `model.keras`, or in the legacy SavedModel's `variables/`.
   - Untrained layers are also visible by statistics: BatchNorm moving mean exactly 0 and variance exactly 1, biases exactly 0, LSTM recurrent kernels exactly orthogonal.
   - ONNX initializers cannot be compared byte-for-byte, because tf2onnx re-packs LSTM gates.
3. **Compare outputs with the reference.** On fixed line crops, compare the ONNX Runtime softmax with Keras's (max abs diff, e.g. ≤ 1e-4 in float32) and require identical greedy text. Keras's own export test uses this kind of `assertAllClose` (`onnx_test.py` L112).
4. **Check against the Docker reference end to end.** The text in `results.txt` from `loghi/docker.htr:2.3.2` on the fixtures must equal the htr-tooling text (brief §5).
5. **Plausibility at run time.** An untrained CTC network gives near-uniform or all-blank output: empty or garbage text, and line confidences far below a trained model's. Use CER against known ground truth on one or two bundled fixture lines as a smoke test at model-load time.

## 5. Processing outside the network

### 5.1 How the Loghi pipeline calls Loghi-HTR

- **Line images:** `MinionCutFromImageBasedOnPageXMLNew … -output_type png -channels 4` writes **4-channel (RGBA) PNG** line crops. Their paths go into `lines.txt` (`LO:scripts/inference-pipeline.sh` L183–200).
- **The HTR call** (L221–231):
  ```
  python3 /src/loghi-htr/src/main.py --model $HTRLOGHIMODEL --batch_size 64 \
      --inference_list lines.txt --results_file results.txt --gpu $GPU \
      --output output/ --beam_width $BEAMWIDTH
  ```
  with `BEAMWIDTH=1` by default (L49). `--greedy` is not passed. The argparse default `beam_width` is 10 (`LH:src/setup/arg_parser.py` L208–215), but the pipeline's 1 selects the greedy decoder (`decoding.py` L52).
- **Merge step:** `MinionLoghiHTRMergePageXML` reads `results.txt`, the model's `config.json` and the run's `output/config.json` (L240–250).

### 5.2 Pre-processing in the CLI path, in order

This is the path used by `inference-pipeline.sh`: `LH:src/data/manager.py` `_create_dataset` → `LH:src/data/loader.py` `DataLoader.load_image`.

1. **Decode** with `tf.io.decode_image(bytes, channels=C)`, where C is the model's channel count (`loader.py` L13–32, L79–83).
   - For PNG, TF 2.20 handles alpha and gray like this (`TF:tensorflow/core/lib/png/png_io.cc`):
     - Alpha is **stripped, not composited**, when fewer channels are asked for (`png_set_strip_alpha`, L273).
     - RGB → gray uses `png_set_rgb_to_gray(…, 0.299, 0.587)` (L305).
     - Gray → RGB is replicated (L308).
   - So a 1-channel model reading the 4-channel crops sees the raw RGB of the masked-out area. A 4-channel model gets the alpha mask as its 4th input channel.
   - Unreadable images are dropped when the dataset is built (`manager.py` `_process_line`).
2. **Resize to the model height**, keeping the aspect ratio: `tf.image.resize(image, [H, 99999], preserve_aspect_ratio=True)` (`loader.py` L86–87). Default method **bilinear, `antialias=False`, half-pixel centres** (`TF:tensorflow/python/ops/image_ops_impl.py` L1622–1627, L1773). The new size uses `round(scale × dim)`, and `tf.round` rounds half to even (L1493–1500).
3. **Scale:** `/ 255.0` → [0, 1] (`loader.py` L88).
4. **Pad the width by 50 px:**
   - The "augmentation model" always contains `ResizeWithPadLayer(target_height=64, additional_width=50, name="extra_resize_with_pad")` ("For some reason, the original adds a 50px pad to the width here", `LH:src/data/augmentation.py` L207–211).
   - `load_image` calls this layer with `training=True` **even at inference** (`loader.py` L95–99).
   - It pads `floor(50/2)=25` columns left and 25 right with **0.0** (`LH:src/data/augment_layers.py` `ResizeWithPadLayer.call`, after L337; `binary=False` → constant 0.0).
   - Note: `target_height=64` is hard-coded, so a model with another height (e.g. the legacy `…height48…`) would also be resized to 64 here. *Not checked further.*
   - Other augmentation layers run with `training=False` at inference.
5. **Normalise:** `image = 0.5 − image`. White paper → −0.5, black ink → +0.5, the 25-px pads → +0.5 (`loader.py` L106–107).
6. **Transpose** to `[W, H, C]`, float32 (`loader.py` L109–111).
7. **CTC width guard:** `_ensure_width_for_ctc` pads to 16 px × label length. At inference the label is empty, so in practice it does nothing (`loader.py` L145–178).
8. **Batch** with `padded_batch(batch_size, padded_shapes=[None, None, C], padding_value=-10.0)`, in file order with no sorting or bucketing by width, `deterministic=True` (`manager.py` L536–556).

### 5.3 Pre-processing in the API path (`loghiwebservice` backend)

`LH:src/api/batch_predictor.py` `process_sample` L216–243 gives the same numbers, written differently:

- decode with the channels from `config.json`; invalid images become a 64×64 zero image;
- `resize(…, [64, 99999], preserve_aspect_ratio=True)`, `/255`;
- `tf.image.resize_with_pad(image, 64, width + 50)`, which gives the same 25/25 zero pad;
- `0.5 − x`, transpose;
- `padded_batch` with −10 (L364–381);
- batches are formed by arrival time (`patience`), not by width (L246–312).

### 5.4 Running the network

`model.predict_on_batch(images)` (`LH:src/modes/utils.py` L114; API `batch_predictor.py` L439).

### 5.5 Batching affects the result

- The network has no Masking layer (the stored summary of `generic_new_17` shows none). Batch padding of −10 columns therefore goes through the convolutions *and* the five **bidirectional** LSTMs.
- The backward direction reads the padding before the real line. So **a line's output depends on the widest line in its batch** (and in the API, on arrival timing).
- Decoding also runs over the full padded length: `input_len = pred.shape[1]` for every sample (`decoding.py` L92).
- For bit-for-bit equivalence with the Docker reference, htr-tooling must reproduce the same batch composition: batch size 64, `lines.txt` order (as written by `find`), padding value −10. The alternative is to accept and record a deviation, for example by running each line alone or bucketing by width.

### 5.6 Decoding

`LH:src/utils/decoding.py`, one `DecodingWorker` thread (`LH:src/modes/utils.py` L45–56; `LH:src/utils/threading.py` L288–326).

1. `log(transpose(y, [1,0,2]) + 1e-7)`, i.e. Keras `epsilon()` (`decoding.py` L44–46).
2. **Greedy** (when `--greedy` is set or `beam_width == 1`): `tf.nn.ctc_greedy_decoder(merge_repeated=True)`, then `log_prob = −log_prob`. **Beam search** otherwise: `tf.nn.ctc_beam_search_decoder(beam_width, top_paths=1)` (L52–63). In both, the blank is the **last** class (TF convention, matching the training loss).
3. Dense indices are padded with −1, then `+1`, then mapped through the tokenizer's inverse `StringLookup`. Index 0 = `[PAD]` → removed; output 0 → `[UNK]`, which is *not* removed. The text is then `.strip()`-ed (L65, L103–107).
4. **Confidence, one per line:** `exp(log_prob / n)`, where `n` = the number of decoded characters (at least 1), clipped to [0, 1] (L109–131). For greedy decoding, `log_prob` is the sum over **all** T frames (padding included) of the best class's log-probability.
5. **Optional word beam search** runs only when `--corpus_file` is given (`LH:src/modes/evaluation.py` L39–43, `LH:src/utils/wbs.py`). It uses the external `CTCWordBeamSearch` and is not used by the pipeline.
6. **Output:** `results.txt` lines `filename \t confidence \t text` (`LH:src/utils/threading.py`, `ResultWriter`, L45). There is **no per-character confidence, no character positions, and no alternatives**.

### 5.7 Tokenizer and charlist

- **Which file:** `tokenizer.json` is preferred, then `charlist.txt` (`LH:src/main.py` L54–70; API `LH:src/api/batch_decoder.py` L54–98).
- **`charlist.txt`:** one line, each Unicode code point is one token in file order (no sorting). `Tokenizer` puts `[PAD]` (mask) and `[UNK]` (OOV) in front (`LH:src/utils/text.py` L38–56, L81–103).
- **One character is dropped when reading:** **U+0007 (BEL)** (`unwanted_chars = ["\x07"]`, L87; the literal byte is invisible in the source). This rule has changed over time:
  - commit `1a284fe` (2024-08-28) dropped U+0007 *and* U+001B (ESC);
  - commit `4ea432e` (2025-02-06, "Fix legacy charlist special token conversion") reduced it to U+0007 only.
- **`tokenizer.json`:** `{"0": "[PAD]", "1": "[UNK]", "2": …}`. For the republic model, the generated `tokenizer.json` from index 2 onward equals `charlist.txt` character by character. This was checked on the downloaded files; that charlist has no BEL.
- **Open point for the recommended model:**
  - The `charlist.txt` of `generic_new_17_2023_05_25_4channel` has 455 code points and **starts with U+0007 U+001B**. With BEL dropped, the tokenizer has 456 entries against 457 network outputs (457 would be needed for the mapping in §3).
  - The 2023 training code (`git_hash` 331d8a8 in its `config.json`) kept BEL as an ordinary class: `char_list = list(file.read())` (`src/main.py` L250 at 331d8a8) and `StringLookup(vocabulary=chars, mask_token='')` (`src/DataGeneratorNew.py` L433–447 at 331d8a8).
  - So the correct index mapping for this model under 2.3.2 was **not established**. It may be off by one. Characterisation tests against the Docker reference will show what Loghi actually does, and htr-tooling must copy that behaviour (or fix it and record a deviation).

## 6. What a published Loghi-HTR model folder contains

From the training callback (`LH:src/model/custom_callback.py` L102–130):

- `model.keras`;
- `tokenizer.json`;
- `config.json` (the full `Config`: `git_hash`, `args`, the model summary, `model_channels`);
- `normalization.json`, if one was used;
- plus `log.csv` and the loss/CER plots from training.

In the SURFdrive share (`loghi-htr/`, listed 2026-11-20):

| folder | files |
|---|---|
| `generic_new_17_2023_05_25_4channel` (the "if you are unsure" choice in the share's README) | `model.keras` (402 MB), `charlist.txt`, `config.json` (old format: `args.channels: 4`, `args.height: 64`, model summary; no `model_channels`), `README.txt`, `file.txt`, `LICENSE` |
| `generic-2023-02-15` | `model.keras` (373 MB), `charlist.txt`, `file.txt`, `LICENSE`, **no `config.json`** (the loghi-htr README says it lacks one and is 1-channel, `LH:README.md` L205) |
| `float32-generic-2023-02-15` | `model.keras` (124 MB), `charlist.txt`, `LICENSE` |
| `republic-2023-01-02-base-generic_new14-2022-12-20` | `model.keras`, **`model.keras.old`**, `tokenizer.json`, `charlist.txt`, `LICENSE` (§4.3) |
| `republic-recommended-batch-8-2024-06-16` | `model.keras`, `charlist.txt`, `config.json`, plots, `LICENSE` |
| `mibudera-fcr500-2025-07-22-augmented-valcer-0.0379` | `model.keras`, `tokenizer.json`, `config.json`, `log.csv`, plot, `LICENSE` |
| `loghi-legacy/<same names>` | TF SavedModels: `saved_model.pb`, `variables/`, `assets/`, `keras_metadata.pb`, `fingerprint.pb`, `charlist.txt`, sometimes `config.json`; usable by "Loghi version of 2.2.15 or lower" (share README) |

What this means:

- A folder can have **`charlist.txt` or `tokenizer.json` or both**, and **may lack `config.json`**. A loader must handle all of these, and must take the channel count from the model input rather than from `config.json`.
- **New fact:** every model folder now has a `LICENSE`, added 2026-11-02: "MIT License, Copyright (c) 2026 Rutger van Koert". Brief §6 says no licence was found.
- *Not established:* the Keras version that wrote each `model.keras`, and its dtype policy. Both are in the zip's `metadata.json`/`config.json`, which were not downloaded.

## 7. What could not be established by reading

- Whether a full Loghi-HTR `model.keras` converts with Keras 3.13 `export(format="onnx")` / tf2onnx 1.17.0 on TF 2.20 without errors, and whether its BiLSTMs become ONNX `LSTM` or `Loop` nodes.
- Whether a dynamic width survives conversion (the Keras test covers only a dynamic batch, with a fixed sequence length).
- Which ONNX Runtime execution providers (OpenVINO, DirectML) run the resulting `LSTM`/`Loop` nodes on the Arc. The ONNX Runtime repository could not be reached through the GitHub API from here; this belongs to the runtime ticket.
- Whether the published `republic-2023-01-02…/model.keras` lost weights in the fallback.
- Whether dropping U+0007 from the `generic_new_17` charlist gives the right class mapping (§5.7).
- The width downsampling factor of each model (T versus W). The width axis is `None` in the stored summaries, so this needs the model.
- The pixel values that `MinionCutFromImageBasedOnPageXMLNew` writes outside the polygon in RGBA crops. That matters for 1-channel models (alpha is stripped, not composited) and belongs to the cut-step ticket.

## Implications for htr-tooling

1. **Conversion recipe to try first (phase-1 ONNX exploration):**
   - TF 2.20 + Keras 3.13 on the CPU with no GPU visible, numpy 1.26.
   - Load strictly with `keras.models.load_model(…, compile=False)`.
   - Rebuild with float32 dtype policies.
   - `model.export("model.onnx", format="onnx", input_signature=[InputSpec((None, None, H, C))])`, opset 15–18.

   **Fallback:** tf2onnx `--saved-model` on the `loghi-legacy` SavedModel under TF 2.15. Do not use `tf2onnx.convert.from_keras`.
2. **Conversion is a one-off, outside htr-tooling's runtime.** It needs TensorFlow, which the brief keeps inside Docker. It can run in the Loghi HTR Docker image or a pinned conversion environment. htr-tooling's runtime needs only ONNX Runtime plus the pre- and post-processing below.
3. **Every converted model needs a verification step before use** (see brief §11, "Model loading and verification"):
   - reject `skip_mismatch` loads and folders that contain `model.keras.old`;
   - compare softmax outputs ONNX vs Keras on fixture lines within a tolerance;
   - require identical greedy text;
   - store the result (source file hash, Keras/TF/tf2onnx versions, opset, max abs diff) next to the `.onnx`.

   A cheap smoke test when the model loads (CER on bundled lines) catches untrained networks at run time.
4. **Never write into a model folder.** Loghi-HTR does (`model.keras` → `.old`, a generated `tokenizer.json`). htr-tooling should treat model folders as read-only and keep converted artefacts elsewhere.
5. **The pre-processing port must match TF exactly where it can:**
   - PNG decode with alpha stripped and the 0.299/0.587/0.114 gray conversion;
   - bilinear, half-pixel, non-antialiased resize to height H with round-half-even width;
   - `/255`, 25+25 px pad of 0, `0.5 − x`, transpose to `[W, H, C]`;
   - batch padding −10.

   Java vs Python and OpenCV vs TF resize differences (brief §8) apply here too.
6. **Batch composition is part of the reference behaviour.** Bidirectional LSTMs and full-length decoding make each line's text and confidence depend on the widest line in its batch. To match `results.txt` exactly, htr-tooling needs the same batching: 64 lines in `lines.txt` order. Any smarter batching (sorting or bucketing by width, which is faster) is a deviation to measure and record under brief §5.
7. **Decoding to port:**
   - TF-compatible CTC greedy decoding (blank = last class, merge repeats);
   - beam search for `--beam_width > 1`;
   - output index *j* → token *j+1*, with `[PAD]` removed and `[UNK]` kept as the literal string;
   - `strip()`;
   - line confidence `exp(Σ max log(p+1e-7) / n_chars)`.

   Beam-search equivalence with `tf.nn.ctc_beam_search_decoder` will need its own characterisation test.
8. **Per-character confidence and alternatives (brief §4.5) are not produced by Loghi-HTR.** The CLI writes one confidence per line. Producing `Glyph` confidences and alternatives from the softmax would be a new capability. Check this against the no-go "nothing Loghi cannot do today" and against what `MinionLoghiHTRMergePageXML` / `SplitPageXMLTextLineIntoWords` write today.
9. **The model folder adapter** must handle:
   - `tokenizer.json` or `charlist.txt`;
   - `config.json` present or missing, in old (`args.channels`) or new (`model_channels`) form;
   - height and channels taken from the model input as the source of truth;
   - Loghi's exact `charlist.txt` rule (drop U+0007, prepend `[PAD]`, `[UNK]`);
   - a load-time check that `len(tokenizer) == number of model outputs` (§5.7 shows a published model where this is off by one).
10. **The licence situation has changed:** the published models now carry an MIT `LICENSE` (© 2026 Rutger van Koert). The rule "no weights in the repository" is still the owner's decision, but the brief's reason ("no licence file was found") is out of date.
