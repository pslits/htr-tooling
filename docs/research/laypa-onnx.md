# Laypa to ONNX: feasibility and the processing around the network

*Research for [issue #5](https://github.com/pslits/htr-tooling/issues/5). Written 2026-11-20. Reading only: no model was loaded, converted or run.*

## Question

Can a Laypa model (detectron2 `SemanticSegmentor`, ResNet-50-FPN) be exported to ONNX, and how? What do detectron2's export paths support for semantic segmentation, and what are the known blockers? Which pre- and post-processing does Laypa do around the network, and where in its code? Which Laypa versions and configs do the published models use? Has anyone exported Laypa or a detectron2 semantic segmentation model to ONNX before?

## Short answer

- **The network itself looks exportable.** Every published Laypa model is a *stock* detectron2 `SemanticSegmentor`: ResNet-50 (FrozenBN) + FPN + `SemSegFPNHead` (GroupNorm). None of them uses Laypa's own heads. The graph has only standard ops (Conv, BatchNorm, ReLU, MaxPool, Add, nearest and bilinear Resize, GroupNorm). It has no ROIAlign, no NMS and no deformable convolution. This is a conclusion from reading the code. Nobody has shown it in practice (see §8).
- **detectron2 does not give a ready-made path.** `caffe2_tracing`/`Caffe2Tracer` only supports `GeneralizedRCNN` and `RetinaNet`, needs Caffe2 and produces ONNX that onnxruntime cannot run. `export_model.py --export-method tracing --format onnx` is generic (`TracingAdapter` + `torch.onnx.export`, opset 11), but detectron2 does not test it on semantic segmentation. `scripting` only produces TorchScript. The simplest route is to export only `backbone` + `sem_seg_head` on a tensor that has already been normalised and padded to a multiple of 32, and to keep everything else in htr-tooling.
- **Pre-processing (moves to htr-tooling):** Pillow load with EXIF transpose → RGB uint8 → resize with `cv2.resize(INTER_LINEAR)`. The size comes from the model's config, either `scaling` (×0.5 for `general/baseline2`) or `shortest_edge` 1024 / max 2048 for the older models. Then float32 CHW → `(x − PIXEL_MEAN) / PIXEL_STD` → zero-pad bottom/right to a multiple of 32.
- **Post-processing (moves to htr-tooling):** crop the logits to the resized size → softmax over classes → bilinear upscaling of the probabilities to the original size (`align_corners=False`) → argmax → for 2 classes, `class × 255` written as `page/<stem>.png`, plus a PageXML skeleton with a confidence MetadataItem. `MinionExtractBaselines` then reads that PNG.
- **The models:** Loghi's README recommends `laypa/general/baseline2`, trained with Laypa commit `ad6da58` (2024-04-24). Loghi 2.3.2 runs it in `loghi/docker.laypa:2.3.2` = Laypa `2.3.2` (`d0ed632`) with PyTorch 2.9.1 and an **unpinned** detectron2.
- **Prior art:** none found. No issue, fork or code in the Laypa or Loghi repositories mentions an ONNX export of Laypa. The detectron2 issues have one failed panoptic export (#4354) and no reported success for `SemanticSegmentor`.

## Sources and versions read

| Source | Version |
|---|---|
| Laypa ([knaw-huc/laypa](https://github.com/knaw-huc/laypa), a fork of [stefanklut/laypa](https://github.com/stefanklut/laypa); both `main` branches are identical) | tag `2.3.2` = HEAD = [`d0ed632`](https://github.com/knaw-huc/laypa/tree/d0ed632ab6ca4e595d7224f0bc046d337b1bcb0e) (2026-03-24) |
| Loghi ([knaw-huc/loghi](https://github.com/knaw-huc/loghi)) | HEAD [`e743513`](https://github.com/knaw-huc/loghi/tree/e74351383ec64fd03689c8550d3123f0eb92c037) (2026-10-28); the `laypa` submodule is pinned to `d0ed632` at HEAD and at tag `2.3.2` |
| detectron2 ([facebookresearch/detectron2](https://github.com/facebookresearch/detectron2)) | HEAD [`247691a`](https://github.com/facebookresearch/detectron2/tree/247691a1c292654867deb35cabb27f18d75a1363) (2026-10-27); last release v0.6 (2021-11-15) |
| fvcore ([facebookresearch/fvcore](https://github.com/facebookresearch/fvcore)) | HEAD `8363d8d` |
| PyTorch ([pytorch/pytorch](https://github.com/pytorch/pytorch)) | tag `v2.9.1`, the version Laypa's `environment.yml` pins |
| ONNX operator docs ([onnx/onnx](https://github.com/onnx/onnx) `docs/Operators.md`, `docs/Changelog.md`) | HEAD `37799e6` |
| Published models (SURFdrive share `YA8HJuukIUKznSP`, linked from Loghi `README.md:80-83`) | `config.yaml` and `LICENSE` of all 8 Laypa models, fetched 2026-11-20 over WebDAV. The weights were not downloaded. |

In what follows, `laypa:` means Laypa @ `d0ed632`, `d2:` means detectron2 @ `247691a`, and `torch:` means PyTorch @ `v2.9.1`.

## 1. What the Loghi pipeline actually runs

- `scripts/inference-pipeline.sh` (Loghi @ `e743513`) sets `VERSION=2.3.2` (line 2) and `DOCKERLAYPA=loghi/docker.laypa:$VERSION` (line 77). It runs `python run.py -c $LAYPABASELINEMODEL -i $IMAGES_PATH -o $IMAGES_PATH --opts MODEL.WEIGHTS "" TEST.WEIGHTS $LAYPABASELINEMODELWEIGHTS` (lines 114–128). The optional region model runs the same way (lines 138–150). Afterwards `MinionExtractBaselines` reads `-input_path_png "$IMAGES_PATH"/page/` (lines 166–168).
- `laypa:run.py` is a deprecated alias that calls `inference.main` (lines 1–21).
- The image is built from `laypa:docker/Dockerfile` with `laypa:environment.yml`: `pytorch-gpu=2.9.1`, `cuda-toolkit=12.9`, and detectron2 installed as `git+https://github.com/facebookresearch/detectron2.git`, which is **not pinned to a commit**. So the detectron2 commit inside `docker.laypa:2.3.2` cannot be read from the source. It depends on when the image was built.
- `laypa` contains no ONNX, TorchScript, `jit.trace` or OpenVINO code at all (searched the whole tree with grep).

## 2. The published models and their configs

All eight Laypa models on the share have a `config.yaml` (a dump of the full training config). They all agree on:

`MODEL.META_ARCHITECTURE: SemanticSegmentor`, `BACKBONE.NAME: build_resnet_fpn_backbone`, `RESNETS.DEPTH: 50`, `RESNETS.NORM: FrozenBN`, `SEM_SEG_HEAD.NAME: SemSegFPNHead`, `SEM_SEG_HEAD.NORM: GN`, `CONVS_DIM: 128`, `COMMON_STRIDE: 4`, `SEM_SEG_HEAD.IN_FEATURES: [p2..p5]`, `INPUT.FORMAT: RGB`.

So none of them uses Laypa's custom `WeightedSemSegFPNHead` (`laypa:models/head/sem_seg_head.py`, which only changes the training loss) or `BinarySegmentor` (`laypa:models/meta_arch/binary_seg.py`). Keys such as `RESNETS.STEM_TYPE: deeplab` and `RES5_MULTI_GRID` in `baseline2` are only read by detectron2's DeepLab project (`d2:projects/DeepLab/deeplab/resnet.py`). `build_resnet_fpn_backbone` ignores them.

| model | trained (`SETUP_TIME`) | mode / classes | `PIXEL_MEAN` / `PIXEL_STD` | test-time resize (as Laypa 2.3.2 resolves it) | `.pth` size |
|---|---|---|---|---|---|
| `general/baseline2` (README's suggestion) | 2024-04-24, Laypa `ad6da58` | baseline / 2 | `[123.675,116.28,103.53]` / `[58.395,57.12,57.375]` | `scaling`, `SCALING_TEST 0.5`, `MAX_SIZE_TEST -1` (no cap), DPI auto-detect off → ×0.5 | 455 MB |
| `general/baseline` | 2023-04-15 | baseline / 2 | `[103.53,116.28,123.675]` / `[1,1,1]` | no `INPUT.RESIZE_MODE` key → default `shortest_edge`, `MIN_SIZE_TEST 1024`, `MAX_SIZE_TEST 2048` | 228 MB |
| `republic/baseline_old` | 2022-11-09 | baseline / 2 | as above | as above | 228 MB |
| `republic/region_old` | 2023-04-18 | region / 7 | as above | as above | 228 MB |
| `republic/baseline` | 2023-06-23 | baseline / 2 | as above | as above | 455 MB |
| `republic/region` | 2023-06-23 | region / 7 | as above | as above | 455 MB |
| `cBAD/baseline` | 2023-06-30 | baseline / 2 | as above | as above | 455 MB |
| `republicprint/baseline` | 2023-06-30 | baseline / 2 | as above | as above | 455 MB |

Notes:

- `LAYPA_GIT_HASH` is present only in `baseline2` (`ad6da58e1fad…`, "noise in general model", 2024-04-24; first contained in Laypa tag `2.0.3`). The older configs predate that key.
- **Two normalisation families.** The older models use detectron2's default Caffe2 BGR means with std 1 (`laypa:configs/defaults.py:42,46`), while `INPUT.FORMAT` is `RGB`. So the "B" mean (103.53) is subtracted from the R channel. The model was trained that way, so it is consistent, but mean and std must be read from each config and never hard-coded.
- The "test-time resize" column applies Laypa 2.3.2's defaults to keys that are absent: `INPUT.RESIZE_MODE = "shortest_edge"` (`laypa:configs/extra_defaults.py:118`), merged in `laypa:core/setup.py:86-95`.
- The checkpoints are 228 MB or 455 MB. A ResNet-50-FPN with this head has roughly 28 M parameters (≈110 MB in fp32; my estimate, not measured), so the files probably also hold optimizer state. Laypa loads them through `DetectionCheckpointer(...).load` (`laypa:inference.py:95-99`), which takes only the model weights. I did not verify this because the weights were not downloaded.
- **New fact:** every model folder now has a `LICENSE` (MIT, "Copyright (c) 2026 Stefan Klut", modified 2026-11-02). The Loghi-HTR model `loghi-htr/generic-2023-02-15` also has one (MIT, "Copyright (c) 2026 Rutger van Koert", modified 2026-11-02). Brief §6 says no licence file was found. That is no longer true.

## 3. Pre-processing (Laypa 2.3.2)

In order, for one image:

1. **Load.** `laypa:utils/image_utils.py:84-124` → `image_to_array_dpi` (lines 49–81) does `PIL.Image.open`, then `ImageOps.exif_transpose` (unless `ignore_exif`), reads `info["dpi"]` (and asserts that it is square), then `convert("RGB")` → `np.asarray` → H×W×3 uint8. The height and width written to PageXML are those of the EXIF-transposed image (`laypa:inference.py:167`, `output_page_xml.py:594`).
2. **DPI.** `laypa:data/mapper.py:101-107`: if `INPUT.DPI.AUTO_DETECT_TEST` is false (the default, and the value in `baseline2`), `dpi = MANUAL_DPI_TEST` (300). Otherwise the file's DPI is used, falling back to `DEFAULT_DPI_TEST`.
3. **Resize.** `laypa:inference.py:101` builds `build_augmentation(cfg, "test")` (`laypa:data/augmentations.py:1677-1719`). For test mode this returns only the resize:
   - `scaling` → `ResizeScaling(SCALING_TEST, MAX_SIZE_TEST, TARGET_DPI_TEST)` (lines 132–178): `scale = SCALING_TEST × TARGET_DPI / dpi`. If `max(h,w)` exceeds `MAX_SIZE_TEST`, it is scaled down to that size; `MAX_SIZE_TEST ≤ 0` means no cap. The new size is `int(x + 0.5)`. If `SCALING_TEST ≤ 0`, `laypa:core/setup.py:169-174` replaces it with `SCALING_TRAIN × PREPROCESS.RESIZE.SCALING`.
   - `shortest_edge` → `ResizeShortestEdge(MIN_SIZE_TEST, MAX_SIZE_TEST, "choice")` (lines 206–334): the short side becomes `MIN_SIZE_TEST`, and if the long side then exceeds `MAX_SIZE_TEST` the image is scaled down; the size is rounded with `int(x + 0.5)`. With a single `MIN_SIZE_TEST` the random "choice" is deterministic. In this mode the DPI is not used.
   - `longest_edge` (lines 337–372) gives **square** output for any aspect ratio (both branches set height and width to `edge_length`). That is a Laypa bug. No published model uses this mode.
   - The resize itself is `NT.ResizeTransform.apply_image` (`laypa:data/numpy_transforms.py:72-88`): `img.astype(np.uint8)` then `cv2.resize(..., interpolation=cv2.INTER_LINEAR)`. If the size does not change it is a no-op (`augmentations.py:183-184`).
4. **To tensor.** `laypa:inference.py:171-177`: `torch.as_tensor(image, float32).permute(2,0,1)`. Channels are flipped only if `INPUT.FORMAT == "BGR"` (all published models use `RGB`, so there is no flip). `height`/`width` passed to the model are the **resized** size, so the model returns logits at the resized resolution.
5. **Inside the model** (`d2:detectron2/modeling/meta_arch/semantic_seg.py:100-106`): `(x − pixel_mean) / pixel_std`, then `ImageList.from_tensors(images, backbone.size_divisibility, padding_constraints=...)`. That pads bottom and right with `pad_value = 0.0`, which is zero *after* normalisation, i.e. the mean colour (`d2:detectron2/structures/image_list.py:60-122`). For the FPN, `size_divisibility = strides[-1]` of the input features res2–res5 = **32**, and `square_pad` is 0 (`d2:detectron2/modeling/backbone/fpn.py:65,113-124`).
6. **Precision.** `MODEL.AMP_TEST.ENABLED` defaults to false (`laypa:configs/extra_defaults.py:47-49`), and on CPU Laypa forces float32 anyway (`laypa:inference.py:179-180`).

## 4. The network

- Backbone: ResNet-50 `BasicStem` (7×7 conv, then `max_pool2d(3, stride 2, pad 1)`, `d2:detectron2/modeling/backbone/resnet.py:330-375`). `FrozenBatchNorm2d` runs as `F.batch_norm(training=False)` when no gradient is needed (`d2:detectron2/layers/batch_norm.py:45-66`).
- FPN: lateral 1×1 and output 3×3 convs, with top-down `F.interpolate(scale_factor=2, mode="nearest")` added to the lateral features (`d2:fpn.py:139-160`). `LastLevelMaxPool` adds p6 = `max_pool2d(k=1, s=2)` (line 200), which the semantic head does not use.
- `SemSegFPNHead` (`d2:semantic_seg.py:142-253`): per level, 3×3 conv + `GroupNorm(32, 128)` (`d2:batch_norm.py:189`) + ReLU, followed by `nn.Upsample(×2, bilinear, align_corners=False)` until stride 4. The levels are summed, then a 1×1 predictor gives `NUM_CLASSES` logits, and at inference a final `F.interpolate(×4, bilinear, align_corners=False)` (lines 241–244).
- `sem_seg_postprocess` (`d2:detectron2/modeling/postprocessing.py:77-100`) crops `[:, :h, :w]` to remove the padding, then interpolates to (`height`, `width`). Laypa passes the resized size, so this interpolation does not change the size.
- Laypa reads `outputs[0]["sem_seg"]` (K × h × w logits) (`laypa:inference.py:356-362`).

**Shape constraint.** The FPN adds a ×2-upsampled map to the next lateral map, so H and W must be divisible by 32. The padding in step 5 guarantees this. An exported graph that leaves padding out must receive inputs that are already padded.

## 5. Post-processing (Laypa 2.3.2)

`SavePredictor.save_prediction` → `OutputPageXML.generate_single_page(sem_seg, path, old_height, old_width)` (`laypa:inference.py:330-362`, `laypa:page_xml/output_page_xml.py:564-641`).

**External processing** (`POSTPROCESS.EXTERNAL.ENABLED = True` by default, `laypa:configs/extra_defaults.py:232-235`; Laypa made this the default in commit `15897e6`, 2025-04-30). It runs for every mode except `region` (lines 598–618):

1. `sem_seg_to_classes_and_confidence(sem_seg, old_height, old_width)` (lines 534–562):
   - `softmax(dim=-3)` on the logits at resized resolution;
   - `F.interpolate(probs, size=(old_h, old_w), mode="bilinear", align_corners=False)`;
   - `argmax(dim=-3)` → class map at **original** resolution;
   - confidence = `max(softmax)` at resized resolution, rescaled linearly from `[1/K, 1]` to `[0, 1]`.
2. `generate_image_from_sem_seg` (lines 156–189): with 2 classes the output is `classes × 255`. With more classes each class gets a distinct grey (or colour) from `classes_to_colors` (`laypa:data/dataset.py:108-137`; background is 0).
3. The result is saved as `page/<image stem>.png` with Pillow (`laypa:utils/image_utils.py:171-192`). The PageXML is `page/<image stem>.xml`, holding a `Page` (imageFilename, original height and width) and a `Metadata/MetadataItem` "LaypaProcessingStep" with labels for git hash, UUID, whitelisted config values and the mean confidence (`laypa:page_xml/page_xml_editor.py:446-460`, class at lines 166-205). It contains no lines and no regions. Laypa also links the image into the output folder (`output_page_xml.py:140-154`).

**Region mode** (`MODEL.MODE: region`, `add_regions_to_page`, lines 291–400): classes come from the logits at the **resized** resolution (no upscaling). Per class it runs `cv2.findContours(RETR_EXTERNAL, CHAIN_APPROX_SIMPLE)` and drops contours with fewer than 4 points or with area below `MIN_REGION_SIZE` (10). Each contour is simplified with `approxPolyDP(ε = 0.0005 × arcLength)`, or replaced by `minAreaRect` for `RECTANGLE_REGIONS`. The coordinates are then scaled by original/resized. In Loghi's pipeline the region run writes `page/<stem>.xml` again, after the baseline run and before `MinionExtractBaselines` (Loghi `inference-pipeline.sh:138-170`).

**Class mapping** (`laypa:page_xml/xml_regions.py:221-256`): class 0 is always `background`. `baseline` mode gives `[background, baseline]`. `region` mode gives `background` followed by `PREPROCESS.REGION.REGIONS` in config order, minus the children named in `MERGE_REGIONS` (which map to their parent's class). For the Republic region models this gives 7 classes, matching `NUM_CLASSES: 7`: background, marginalia, page-number, resolution, date, index, attendance. I derived that list from the code and the config; I did not observe it. Region types default to `TextRegion` (lines 179–207).

Laypa's own baseline extractor (`add_baselines_to_page`, lines 191–289, which calls `page_xml/baseline_extractor.py`) is **not** used while external processing is on, which is the default. In Loghi, `MinionExtractBaselines` turns the PNG into baselines.

## 6. detectron2's export paths and what they support

| path | where | semantic segmentation? |
|---|---|---|
| `caffe2_tracing` / `Caffe2Tracer` → `export_caffe2`, `export_onnx`, `export_torchscript` | `d2:detectron2/export/api.py:22-94`, `caffe2_modeling.py`, `caffe2_export.py:33-66` | **No.** `META_ARCH_CAFFE2_EXPORT_TYPE_MAP` holds only `GeneralizedRCNN` and `RetinaNet` (`caffe2_modeling.py:417-420`), so `SemanticSegmentor` fails the lookup at `api.py:60`. The API is only imported when `caffe2` can be imported (`export/__init__.py:8-16`). Its ONNX "contains custom ops only available in caffe2, therefore it cannot be directly executed by other runtime (such as onnxruntime…)" (`api.py:84-86`) and is exported with `ONNX_ATEN_FALLBACK` (`caffe2_export.py:56-63`). The tutorial calls the path "going to be deprecated" (`docs/tutorials/deployment.md:50`). The tutorial also lists `PanopticFPN` as covered (line 97), but the code's map does not include it. |
| `tracing` + `TracingAdapter` → `torch.onnx.export` | `d2:tools/deploy/export_model.py:108-138`, `d2:detectron2/export/flatten.py:186-` | **Plausible, untested upstream.** For a model that is not `GeneralizedRCNN`, the adapter just calls `model([{"image": img}])` and flattens the dict output (lines 113–123), then calls `torch.onnx.export(..., opset_version=STABLE_ONNX_OPSET_VERSION)` = **11** (`export/__init__.py:19-20`). Normalisation, padding, crop and the final interpolate would end up in the graph. `get_sample_inputs` prepares the sample with `ResizeShortestEdge(MIN_SIZE_TEST, MAX_SIZE_TEST)` (lines 154–176), which is *not* Laypa's resize. `--run-eval` only works for torchscript R-CNN/RetinaNet (lines 136–151). detectron2's ONNX tests cover only Mask R-CNN, Cascade R-CNN, RetinaNet and keypoint models (`d2:tests/test_export_onnx.py:37-104`). The deployment table lists only TorchScript as the format for tracing (`deployment.md:21-48`). |
| `scripting` (`scripting_with_instances`) | `export_model.py:62-104` | TorchScript only ("Scripting only supports torchscript format", line 75). |

The tutorial also says: "We don't plan to work on additional support for other formats/runtime" (`deployment.md:51`). The detectron2 repository still gets small fixes (for example `8a9d885`, 2026-03-21, "Forward fix onnx.optimizer imports"), but its last release is v0.6 from 2021.

**The alternative that needs no detectron2 export code:** wrap `model.backbone` and `model.sem_seg_head` in a small module whose input is an already normalised, already padded float tensor `[1, 3, 32·m, 32·n]` and whose output is the logits at the same size. Export that with `torch.onnx.export`, and keep steps 1–5 of §3 and all of §5 in htr-tooling. All the logic this would need is listed above. Whether the export works, and how close it is numerically, is for the conversion spike to show.

## 7. PyTorch ONNX exporter facts that matter (torch 2.9.1)

- `torch.onnx.export(..., dynamo: bool = True, ...)`: the `torch.export`-based exporter is the default (`torch:torch/onnx/__init__.py:59-104`). The TorchScript-based exporter "is *deprecated* since PyTorch 2.9 and should be removed along with TorchScript" (`torch:torch/onnx/_internal/torchscript_exporter/README.md`).
- The dynamo exporter implements ops at **opset 18** (`TORCHLIB_OPSET = 18`, `torch:torch/onnx/_internal/exporter/_constants.py`). If you ask for a lower opset (for example detectron2's 11), it exports at 18 and then tries an automatic down-conversion that "may not be successful" (`torch:torch/onnx/_internal/exporter/_compat.py:113-131,207-209`). The default `opset_version` is 20 (`torch:torch/onnx/_constants.py`: `ONNX_DEFAULT_OPSET = 20`; TorchScript exporter max 20).
- Dynamic H/W: with dynamo, use `dynamic_shapes` (`torch.export` dims); `dynamic_axes` is for the TorchScript exporter (docstring, `__init__.py`). The 32-divisibility in §4 probably has to be stated in the dims (for example `32·k`), or the tracer will find conflicting shape guards at the FPN add. This is my reading of the code, not an observed failure.
- **GroupNorm.** ONNX `GroupNormalization` exists at opset 18 and 21 (`onnx:docs/Operators.md`, function op). Version 18 is **deprecated** and takes scale/bias *per group*, which is not PyTorch's per-channel affine (`onnx:docs/Changelog.md`, "GroupNormalization-18 (deprecated)"). Version 21 takes them per channel. The TorchScript exporter lowers `aten::group_norm` to `InstanceNormalization` plus reshapes (`torch:torch/onnx/_internal/torchscript_exporter/symbolic_opset9.py:5678-5715`). I could not establish how the dynamo exporter's torchlib lowers GroupNorm at opset 18–20, because the onnxscript source could not be reached through the GitHub API on 2026-11-20. The spike should check the op in the exported graph.
- Weights: `external_data=True` is the default for the dynamo exporter (`__init__.py`). The ~28 M-parameter model is far below the 2 GB protobuf limit either way.

## 8. Known blockers, risks and prior art

**Blockers and risks:**

1. **There is no supported path from detectron2.** The export has to be our own (§6). Running it needs torch and detectron2, which is natural in the Laypa Docker image; that keeps detectron2 (Apache-2.0) out of htr-tooling.
2. **Dynamic shape.** The input must be divisible by 32 (§4). If normalisation, padding and crop stay outside the graph, there is no shape logic inside it. If they go inside (the `TracingAdapter` route), the trace contains `F.pad` sizes computed from tensor shapes, plus asserts that turn into constants. #4354 shows such `TracerWarning`s for `image_list.py`.
3. **Large inputs.** `baseline2` runs at ×0.5 with **no size cap** (`MAX_SIZE_TEST: -1`), so a 6000×4000 scan becomes a 3000×2000 input. Input sizes therefore vary per scan. How OpenVINO and DirectML handle a different shape for every image (recompilation cost) is not established here.
4. **Silent weight mismatch.** fvcore's `Checkpointer._load_model` drops tensors whose shapes do not match and calls `load_state_dict(..., strict=False)`, then only *logs* missing, unexpected and skipped keys (`fvcore:fvcore/common/checkpoint.py:254-353`, `strict=False` at line 325). A config that does not match the weights (wrong `NUM_CLASSES`, `NORM`, depth) therefore gives a partly random network with no error. This is the same trap the brief describes for Loghi-HTR's `skip_mismatch=True`.
5. **Numerical differences around the network.** The resize uses `cv2.INTER_LINEAR` on uint8 (OpenCV version dependent). Post-processing interpolates *probabilities* bilinearly at full size and then takes the argmax, which decides the mask boundary pixel by pixel. If htr-tooling does this with OpenCV or NumPy instead of `torch.nn.functional.interpolate`, the boundaries may move by a pixel. On ties `torch.argmax` returns the first index (background).
6. **Precision.** Laypa runs fp32 on CPU. An fp16 or INT8 graph for OpenVINO or DirectML would be a deviation and has to be measured.

**Prior art (searched 2026-11-20):**

- `knaw-huc/laypa` and `stefanklut/laypa` have no issues matching "onnx" or "export" (`gh search issues`).
- `knaw-huc/loghi` has one hit: #52, where a maintainer suggests ONNX only as an idea for converting Loghi-HTR to kraken, not Laypa.
- 9 forks of `stefanklut/laypa`. Only `eeeevieb/laypa` (41 commits ahead) and `cconzen/laypa` (1 ahead) carry their own commits, and none of their commit subjects mention onnx, export, openvino, tensorrt or torchscript.
- detectron2 issues: [#4354](https://github.com/facebookresearch/detectron2/issues/4354) "Panoptic Segmentation Can Not be Exported to ONNX" (open). The tracing → ONNX export of `panoptic_fpn_R_50_1x` fails inside the R-CNN mask branch (`KeyError: 'UNKNOWN_SCALAR'` in a `Cast`, older torch), not in the semantic branch. I found no issue that reports a successful `SemanticSegmentor`/`SemSegFPNHead` ONNX export.
- **Could not be established:** whether anyone has done this outside GitHub, or in repositories that GitHub code search does not index. GitHub's issue search ranks by relevance and may miss items, so finding nothing is not proof that nothing exists.

## Implications for htr-tooling

Facts that the runtime-port and model-verification decisions depend on:

1. **Where the ONNX seam goes.** The ONNX graph should be the network only: input `float32[1,3,32m,32n]` (normalised and padded), output `float32[1,K,32m,32n]` logits. Load, EXIF, DPI, resize, normalise, pad, crop, softmax, upscale, argmax and PNG writing all belong to htr-tooling. They are small, deterministic and listed in §3 and §5 with exact code locations. The same code then also serves the Docker reference path when it is fed Laypa's logits.
2. **The model descriptor is the Laypa `config.yaml`.** Each model needs `INPUT.FORMAT`, `MODEL.PIXEL_MEAN/STD`, `INPUT.RESIZE_MODE`, `SCALING_TEST`, `MIN/MAX_SIZE_TEST`, `INPUT.DPI.*`, `MODEL.MODE`, `SEM_SEG_HEAD.NUM_CLASSES`, `PREPROCESS.REGION.REGIONS/MERGE_REGIONS/REGION_TYPE` and `POSTPROCESS.EXTERNAL.*`. Absent keys must take Laypa 2.3.2's defaults (`configs/defaults.py` + `configs/extra_defaults.py`) and its derivation rules (`core/setup.py:157-174`). Example: the pre-2024 configs have no `RESIZE_MODE` and resolve to `shortest_edge` 1024/2048. There are two normalisation families (§2).
3. **The equivalence contract is the PNG.** In Loghi, Laypa's only output that matters for the baseline chain is `page/<stem>.png` (0/255, original size) plus the Metadata item. `ExtractBaselines` does the rest. The model-verification check can compare (a) logits or probabilities against the Docker reference on the fixtures and (b) the binary mask (pixel agreement or IoU), before any baseline tolerance is applied.
4. **Verification must not trust the weight loader.** The exporter must fail when detectron2's checkpointer reports any missing, unexpected or shape-skipped key, and should assert that the output channel count equals `NUM_CLASSES` from the config (§8.4).
5. **Conversion runs offline, inside Docker.** It needs torch 2.9.1 and a detectron2 commit. The Laypa image does not pin detectron2, so the actual commit in `loghi/docker.laypa:2.3.2` should be recorded once (from the image) and stored with the fixtures.
6. **Opset choice is open.** With the dynamo exporter the floor is 18. GroupNorm is the op to watch: `GroupNormalization-21` is per channel, the deprecated 18 version is per group, and the opset 9 TorchScript exporter uses `InstanceNormalization` (§7). Support in the OpenVINO and DirectML execution providers was not checked here.
7. **Target model for phase 1:** `laypa/general/baseline2` (README's suggestion: 2 classes, ×0.5 scaling, ImageNet mean/std). The region models (7 classes) use a different post-processing path (contours at resized resolution, §5) and would need their own fixtures.
8. **Licence correction for the brief:** the published Laypa and Loghi-HTR model folders now carry MIT `LICENSE` files (dated 2026-11-02). This affects brief §6 ("No licence file was found"), although "no weights in the repository" can stay a choice.
