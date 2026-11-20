# How Laypa and Loghi-HTR training works in Docker

*Research for [#8](https://github.com/pslits/htr-tooling/issues/8). Written 2026-11-20. Primary sources only; no upstream code was run.*

## Sources and versions

Every claim below cites a file at a fixed commit. All three upstream repositories are at release **2.3.2**, and HEAD equals the tag:

| repo | commit (= tag 2.3.2) | date | link prefix used below |
|---|---|---|---|
| knaw-huc/loghi | `e74351383ec64fd03689c8550d3123f0eb92c037` | 2026-10-28 | `L/` = https://github.com/knaw-huc/loghi/blob/e74351383ec64fd03689c8550d3123f0eb92c037/ |
| knaw-huc/laypa (fork of stefanklut/laypa) | `d0ed632ab6ca4e595d7224f0bc046d337b1bcb0e` | 2026-03-24 | `P/` = https://github.com/knaw-huc/laypa/blob/d0ed632ab6ca4e595d7224f0bc046d337b1bcb0e/ |
| knaw-huc/loghi-htr (fork of rvankoert/loghi-htr) | `c24c745e9aeb287bedeb6b4edee670ae5c81a3cd` | 2026-01-27 | `H/` = https://github.com/knaw-huc/loghi-htr/blob/c24c745e9aeb287bedeb6b4edee670ae5c81a3cd/ |
| knaw-huc/loghi-tooling | `361dd36a3b557944cdffaba1344a735e67aea613` | — | `T/` = https://github.com/knaw-huc/loghi-tooling/blob/361dd36a3b557944cdffaba1344a735e67aea613/ |

Loghi 2.3.2 pins exactly these commits as submodules (`laypa`, `loghi-htr`, `loghi-tooling`). The scripts use the images `loghi/docker.laypa:2.3.2`, `loghi/docker.htr:2.3.2` and `loghi/docker.loghi-tooling:2.3.2`, published on Docker Hub on 2026-10-28. The Docker Hub tag `latest` is older (2026-04-08, i.e. 2.3.0). Issues are disabled on the knaw-huc forks of Laypa and Loghi-HTR. Their issues live on `stefanklut/laypa` and `rvankoert/loghi-htr`. Loghi's own issues are on `knaw-huc/loghi`.

Detectron2 and fvcore behaviour is cited at `facebookresearch/detectron2@247691a1c292654867deb35cabb27f18d75a1363` (`D/`) and `facebookresearch/fvcore@8363d8d9e048e04a528112a15a50696fe125f97f` (`F/`). Laypa installs detectron2 from git HEAD without a pin (`P/environment.yml`), so the version inside a given image cannot be read from the source tree.

## Loghi's training scripts at a glance

| script | what it does | status at 2.3.2 |
|---|---|---|
| `scripts/create-train-data.sh` | PageXML + images → line crops + `image<TAB>text` lists, 90/10 split | works; uses `MinionCutFromImageBasedOnPageXMLNew` |
| `scripts/htr-train-pipeline.sh` | trains or fine-tunes Loghi-HTR in `loghi/docker.htr` | works; settings are edited inside the script |
| `scripts/laypa-train-pipeline.sh` | — | **empty file (0 bytes)**, added in commit `31cd82b` "WIP: update docs" (2025-09-15) |
| `scripts/generate-synthetic-images.sh` | `MinionGeneratePageImages` from fonts + texts | optional augmentation; uses the untagged `loghi/docker.loghi-tooling` image |
| `scripts/convert-v2-to-v3.sh` | converts old Keras-2 HTR models to Keras-3 | pinned to `loghi/docker.htr:2.2.22`; reported broken in [loghi#53](https://github.com/knaw-huc/loghi/issues/53) |
| `scripts/inference-pipeline.sh` | how a trained model is used | see "Using a trained model" in each section |

`L/docs/training.md` has a "Laypa" heading with no content, and a short prose description of the HTR script. **Loghi has no official way to train Laypa in Docker.** Users call Laypa's `train.py` inside `loghi/docker.laypa` themselves.

---

## 1. Loghi-HTR

### 1.1 Preparing the data: `create-train-data.sh` + `MinionCutFromImageBasedOnPageXMLNew`

Usage: `create-train-data.sh <input_path> <output_path> [numthreads=4]` (`L/scripts/create-train-data.sh` L21–41).

**Input.** Images in `<input_path>` with PageXML in `<input_path>/page/<image>.xml` (`L/scripts/README.md` L19–28). The PageXML must hold TextLines with Baseline, Coords and TextEquiv.

**Settings at the top of the script** (edited by hand, L9–15): `trainsplit=90`, `include_text_styles=1`, `skip_unclear=1`.

**Command** (L84–98):

```
docker run -u $(id -u):$(id -g) --rm -v $inputdir:$inputdir -v $outputdir:$outputdir \
  loghi/docker.loghi-tooling:2.3.2 \
  /src/loghi-tooling/minions/target/appassembler/bin/MinionCutFromImageBasedOnPageXMLNew \
  -input_path $inputdir -outputbase $outputdir -channels 4 -output_type png \
  -write_text_contents -threads $numthreads -include_text_styles -no_page_update \
  -skip_unclear -use_2013_namespace
```

What the minion does with these flags (`T/minions/src/main/java/nl/knaw/huc/di/images/minions/MinionCutFromImageBasedOnPageXMLNew.java`):

- **The line polygons are recalculated from the baselines before cutting.** This is the default unless `-no_text_line_contour_recalculation` is given (L240, L415, L582–585). The training crops therefore come from the same `LayoutProc` heuristic as the inference crops, not from the ground-truth `Coords`.
- `-channels 4` writes RGBA PNGs with the line mask in the alpha channel (L224, L617 `includeMask = channels == 4`).
- `-write_text_contents` writes, per line, `<outputbase>/<image>/<image>-<lineId>.png` and `.txt`. It also writes one `loghi-all-lines.txt` per page, with `path<TAB>conf<TAB>TextEquiv/Unicode` (L554, L618, L653–668, L680–720). Lines with empty text are skipped (L655–660).
- `-include_text_styles` turns `TextLine/@custom` `textStyle {offset; length; …}` into special marker characters in the `.txt` (option text L238; `T/pagexmlutils/src/main/java/nl/knaw/huc/di/images/pagexmlutils/GroundTruthTextLineFormatter.java` L39–86). `-use_tags` would write HTML-like tags instead. **These markers become part of the charlist (tokenizer) of the trained model.**
- `-skip_unclear` drops lines tagged `unclear` (option text L241: "In general set this when training, but not for inferencing").
- `-no_page_update` leaves the PageXML untouched (L381), so `-use_2013_namespace` has no effect here.
- Hard-coded size filters: `minWidth=5`, `minHeight=5`, `minWidthToHeight=0`. Interline distance 35 (L46, L304–306).
- Options the script does not use: `-minimum_confidence`/`-maximum_confidence` (filter lines by `TextEquiv@conf`), `-rescaleheight`, `-xheight`, `-minimum_xheight`, `-page_path` (L223–251). The Loghi maintainer recommends `-minimum_confidence 0.8` for pseudo-labelling ([loghi#38](https://github.com/knaw-huc/loghi/issues/38)).

**Lists** (L102–119). The script finds every `*.png` under the output and writes `training_all.txt` as `<abs path to png><TAB><contents of .txt>`. It then shuffles that file with `shuf` (unseeded) and uses `split` to write `training_all_train.txt` (90 %) and `training_all_val.txt` (the rest). The split is **by line, random and not reproducible**. `split` writes `xaa`/`xab` to the current directory. The maintainer: "random splitting, either by page if you have a lot of data. Or by line (the default) if you have little data" ([loghi#38](https://github.com/knaw-huc/loghi/issues/38)).

**List format accepted by Loghi-HTR** (`H/src/data/manager.py` `_process_line`/`_get_ground_truth`/`_get_sample_weight`, L300–475). Fields are split on TAB. Field 0 is the image path and the last field is the text. With three or more fields, field 1 is a float **sample weight**. Blank lines and lines starting with `#` are skipped, as are missing files (unless `--no_verify_image_exists`). Train and validation lines with empty text are skipped. Several list files can be passed as one space-separated string.

### 1.2 Training: `htr-train-pipeline.sh` → `src/main.py`

All settings are shell variables edited inside the script (`L/scripts/htr-train-pipeline.sh` L6–48):

| variable | default | becomes |
|---|---|---|
| `USEBASEMODEL` / `HTRBASEMODEL` | `0` / placeholder | `--model <dir>` when 1 (fine-tune) |
| `HTRNEWMODEL` | `recommended` | `--model recommended` when not fine-tuning |
| `GPU` | `0` | `--gpu`; `-1` = CPU (then no `--gpus` for Docker) |
| `listdir` → `trainlist`, `validationlist` | `$listdir/training_all_{train,val}.txt` | `--train_list`, `--validation_list` |
| `datadir` | `/tmp/path_to_training_images` | extra mount only |
| `epochs` | **1** | `--epochs` |
| `multiply` | 1 | `--aug_multiply` |
| `REPLACEFINALLAYER` | 0 | `--replace_final_layer` |
| `batch_size` | 40 | `--batch_size` |
| `model_name` | `myfirstmodel` | `--model_name` |
| `learning_rate` | 0.0003 | `--learning_rate` |
| `outputdir` | `$(mktemp -d)/output` | `--output` |

Command (L90–111):

```
docker run [--gpus device=$GPU] --rm -u uid:gid -m 32000m --shm-size 10240m -ti \
  [-v $(dirname $HTRBASEMODEL):…] -v $tmpdir -v $listdir -v $datadir -v $outputdir \
  loghi/docker.htr:2.3.2 bash -c "TF_USE_LEGACY_KERAS=0 python3 /src/loghi-htr/src/main.py \
    --train_list … --do_validate --validation_list … --learning_rate … --batch_size … \
    --epochs … --gpu … --seed 1 --beam_width 1 --model … --aug_multiply … \
    --model_name … --output … [--replace_final_layer]"
```

Notes on the script:
- By default the output goes to a fresh `mktemp -d` under `/tmp`, which users lose ([loghi#33](https://github.com/knaw-huc/loghi/issues/33)).
- `-ti` needs a TTY.
- `--seed 1` does nothing unless `--deterministic` is also passed. The seed is only applied inside `set_deterministic` (`H/src/setup/environment.py` L85–86).
- The script exposes no augmentation flags, so all augmentations are off apart from `--aug_multiply`.
- Users add environment variables such as `TF_GPU_ALLOCATOR=cuda_malloc_async` to avoid out-of-memory errors ([loghi#55](https://github.com/knaw-huc/loghi/issues/55), open).

**All `main.py` options and defaults** (`H/src/setup/arg_parser.py` L5–245). Only `--model` is required (L265).

- *General:* `--gpu "0"` (`-1` CPU, `all`, or `0,1`), `--output output`, `--config_file`, `--batch_size 4`, `--seed 42`, `--tokenizer`, `--test_list`, `--test_images`, `--no_verify_image_exists`.
- *Training:* `--epochs 40`, `--width 65536`, `--train_list`, `--steps_per_epoch None` (marked FIXME in `H/src/modes/training.py` L81), `--output_checkpoints`, `--early_stopping_patience 20`, `--do_validate`, `--validation_list`, `--training_verbosity_mode auto`.
- *Inference:* `--inference_list`, `--results_file output/results.txt`.
- *LR:* `--optimizer adamw`, `--learning_rate 0.0003`, `--decay_rate 0.99`, `--decay_steps -1`, `--warmup_ratio 0.0`, `--decay_per_epoch`, `--linear_decay`.
- *Model:* `--model` (library key, VGSL string, or a model directory), `--use_float32`, `--model_name`, `--replace_final_layer`, `--replace_recurrent_layer <VGSL>`, `--freeze_conv_layers`, `--freeze_recurrent_layers`, `--freeze_dense_layers`.
- *Augmentation (all off):* `--aug_multiply 1`, `--aug_elastic_transform`, `--aug_random_crop`, `--aug_random_width`, `--aug_distort_jpeg`, `--aug_random_shear`, `--aug_blur`, `--aug_invert`, `--aug_binarize_otsu`, `--aug_binarize_sauvola`, `--visualize_augments`, `--aug_mask` (experimental).
- *Decoding:* `--greedy`, `--beam_width 10`, `--corpus_file`, `--wbs_smoothing 0.1`.
- *Misc:* `--normalization_file` (JSON `{orig: repl}`, applied to train and evaluation text), `--deterministic`, `--decoding_threads 1`, `--bidirectional`, `--safe_bidi`.

**Config files.** `--config_file x.json` with `{"args": {"general": {…}, "training": {…}, …}}`. Options given on the command line override the file (`H/src/setup/config.py` L250–313). There are examples in `H/configs/` (`training.json`, `finetuning.json`, …). These differ from the script: rmsprop, 100 or 50 epochs, elastic/random-crop/random-width on, `output_checkpoints` on, `freeze_conv_layers` + `replace_final_layer` in `finetuning.json`. The README recommends config files (`H/README.md` L75–118).

**Model library** (`H/src/model/management.py` L373–425). `recommended` = `None,None,64,1 Cr3,3,24 Mp2,2,2,2 Bn Cr3,3,48 Bn Cr3,3,96 Mp2,2,2,2 Bn Cr3,3,96 Mp2,2,2,2 Bn Rc3 Bl512 D50 ×5 Fs92`. That is height 64, **1 input channel**, and five BiLSTM-512 layers. The 4-channel crops are decoded to the model's channel count (`H/src/data/loader.py` L82; channel count from `H/src/data/data_handling.py` L40). An image that fails to decode silently becomes a 64×64 zero image (`loader.py` L24).

**Precision.** On GPU without `--use_float32` the model trains in `mixed_float16`. On `--gpu -1` it is float32 (`H/src/setup/environment.py` L184–203). When a model is loaded with `--gpu -1` it is converted to float32 (`H/src/model/management.py` L121–124). This is why Loghi publishes separate "float32-" models for CPU (`L/README.md` L112).

### 1.3 Fine-tuning from a base model

- `--model <dir>` loads the first `*.pb` in the directory (old SavedModel, converted on load) or the first `*.keras` file (`management.py` L164–199, L325–329). A Keras-2 `.keras` file that fails to load is converted. **The converter renames the original to `model.keras.old` and writes the new file into the base-model directory** (L186–195). It loads weights with `skip_mismatch=True` (L296–297), which is the trap noted in the brief §8.
- The tokenizer comes from `--tokenizer`, or else from `<model dir>/tokenizer.json`, or else from a legacy `charlist.txt` (`H/src/main.py` L54–70). **Loading a `charlist.txt` writes a new `tokenizer.json` next to it** (`H/src/utils/text.py` L80–101). So the model directory must be writable, both for fine-tuning and for inference.
- **Without `--replace_final_layer`** the base tokenizer is kept. Training lines with characters outside it are **dropped** as "Unsupported characters", and only a warning is logged (`H/src/data/manager.py` L128–131, L435–449). Validation, test and inference lines are kept.
- **With `--replace_final_layer`** a new tokenizer is built from the characters in the *training* list. The final Dense layer is replaced, and the other weights are kept (`manager.py` L128–163; `management.py` L95–99). A model built from scratch always gets a final layer sized to the tokenizer (same line). The maintainer: use it "only … if you are adding/removing characters" ([loghi#38](https://github.com/knaw-huc/loghi/issues/38)).
- Freezing (`--freeze_*`) works by layer name prefix (`conv`/`residual`, `bidirectional`, `dense`) (`management.py` L102–118). `--replace_recurrent_layer` swaps the BiLSTM stack (README FAQ L374–390).
- The base model's own `config.json` is **not read** during training. No link to the base model is recorded except the path in `args.model.model` (see 1.5).

### 1.4 Output

Directory `--output` (`H/src/main.py`, `H/src/modes/training.py`, `H/src/model/custom_callback.py`):

```
<output>/config.json            # written once, before training (main.py L130-134)
<output>/tokenizer.json         # main.py L99-101
<output>/log.csv                # Keras CSVLogger per epoch: loss, CER_metric, WER_metric, val_*, lr (training.py L51-54)
<output>/loss_plot.png, cer_plot.png   # after training (training.py L152-161)
<output>/validation.csv         # with --do_validate: cer,cer_lower,cer_simple[,normalized_*][,wbs_*] (modes/utils.py L184-199)
<output>/<model.name>/best_val/          # each time val CER improves (custom_callback.py L175-180)
    model.keras  tokenizer.json  config.json  [normalization.json]
<output>/<model.name>/epoch_<n>_CER_<x>[_val_<y>]/   # only with --output_checkpoints (L183-188)
```

- **The model is saved only by this callback.** `best_val/` needs a validation list. Without a validation list and without `--output_checkpoints`, **no model is written at all** (`custom_callback.py` L175–188; `main.py` has no final save).
- Saves run in background threads (L74–84), and the saved model is a clone with every layer set to trainable again (L114–120).
- `EarlyStopping` on `val_CER_metric` with `restore_best_weights=True` (`training.py` L69–76).
- A loadable **model directory** is `model.keras` + `tokenizer.json` + `config.json`. Inference also accepts a legacy `charlist.txt` instead of `tokenizer.json`.

### 1.5 Provenance recorded

`config.json` (`H/src/setup/config.py` L45–72, L140–156, L158–248; `main.py` L131–134):

- `args`: every option, grouped by category. This includes `train_list`, `validation_list`, the base model path in `args.model.model`, the learning rate, augmentation and so on.
- `git_hash`: from the file `version_info` in the working directory, else `git log`, else `"Unavailable"` (L330–359). In `loghi/docker.htr`, `version_info` is written when the image is built from **loghi-htr HEAD at build time, not from the submodule pin**, and the `.git` directory is then deleted (`L/docker/docker.htr/Dockerfile` L59–62). The image's WORKDIR is `/src/loghi-htr/src`, so the file is found.
- `uuid`: a fresh `uuid4` **per run**. The copy in `best_val/` has the same uuid as the top-level copy.
- `url_code`: the constant `https://github.com/knaw-huc/loghi`. `notes`: `""`.
- `model`: the `model.summary()` lines. `model_name`. `model_channels`.

What is **not** recorded:
- a timestamp;
- hashes or contents of the training data (only list paths, valid inside the container);
- the Docker image tag or digest;
- the uuid of the base model;
- the final metrics (only in `log.csv` and `validation.csv`);
- the split seed (`shuf` is unseeded).

### 1.6 Using a trained model

`inference-pipeline.sh` (`L/scripts/inference-pipeline.sh` L182–255) has four steps:

1. `MinionCutFromImageBasedOnPageXMLNew -channels 4 -output_type png` (no `-write_text_contents`).
2. `find … -name '*.png' > lines.txt`.
3. `main.py --model $HTRLOGHIMODEL --batch_size 64 --inference_list lines.txt --results_file results.txt --gpu $GPU --output $tmp/output --beam_width $BEAMWIDTH`. Note that the script runs this `docker run` **twice**: L206–217 is prefixed with `echo` and only prints the command, and L220–231 runs it.
4. `MinionLoghiHTRMergePageXML -config_file $HTRLOGHIMODEL/config.json -htr_code_config_file $tmp/output/config.json`.

The merge minion turns the model's `config.json` into a PageXML `MetadataItem type="processingStep" name="htr-0" value="loghi-htr"`. Its labels are `githash`, `model_name`, `uuid`, plus whitelisted config keys (default whitelist: `batch_size`) (`T/minions/src/main/java/nl/knaw/huc/di/images/minions/MinionLoghiHTRMergePageXML.java` L211–270, L301–330, L516–526). Two defects matter for provenance:
- The `-htr_code_config_file` value (the inference run's git hash) is read but **never used**. The constructor ignores its `gitHash` argument, and the `githash` written is the *training* code's hash from the model config (L51–67, L214).
- The minion reads the key `"url-code"`, while Loghi-HTR writes `"url_code"`, so `url-code` is never written (L320–322 vs `H/src/setup/config.py` L72).
- `readHTRConfigFile` calls `.toString()` on `git_hash` and `model` without a null check (L308–309), so a `config.json` without these keys fails.

So a PageXML produced by Loghi names the HTR model by its **training-run uuid, model_name and training git hash**.

---

## 2. Laypa

### 2.1 Command and options

There is no Loghi script for this (see above). Laypa's README describes it (`P/README.md` L211–274):

```
python train.py -c CONFIG -t TRAIN [TRAIN ...] -v VAL [VAL ...] \
  [--tmp_dir DIR] [--keep_tmp_dir] [--num-gpus 1] [--num-machines 1] \
  [--machine-rank 0] [--dist-url tcp://127.0.0.1:<port>] [--opts KEY VALUE ...]
```

(`P/train.py` L27–68). Inside Docker it would be `docker run --gpus … -v … loghi/docker.laypa:2.3.2 python train.py …`. The image's WORKDIR is `/src/laypa`, and its ENTRYPOINT activates the conda env `/env` (`L/docker/docker.laypa/Dockerfile`, `_entrypoint.sh`). The image runs as `USER 1000` unless `-u` is given. Users run Laypa scripts this way with their own `docker run` lines. One example, for `eval.py`, is in [stefanklut/laypa#34](https://github.com/stefanklut/laypa/issues/34).

- `-t`/`-v` each accept directories, image paths or `.txt` files of image paths, can be repeated, and can be mixed (`P/README.md` L247–257).
- `--opts` overrides any config key (`P/core/setup.py` L95, `merge_from_list`; new keys are not allowed there).
- `train.py` asserts `num_gpus <= torch.cuda.device_count()` (L131–133). **With the default `--num-gpus 1` it fails on a machine without CUDA.**
- `--num-gpus 0` probably falls through to a single-process run, because detectron2 `launch` only spawns when `world_size > 1` (`D/detectron2/engine/launch.py` L54–84). This was not tested.
- `MODEL.DEVICE` falls back to `cpu` when no CUDA is found (`P/core/setup.py` L140–150). But `AMPTrainer` asserts that CUDA is present (`D/detectron2/engine/train_loop.py` L481), and `baseline_general.yaml` turns AMP on. CPU training therefore also needs `MODEL.AMP_TRAIN.ENABLED False`. The maintainer was unsure whether CPU training is multi-threaded ([stefanklut/laypa#18](https://github.com/stefanklut/laypa/issues/18)).

### 2.2 Input

Images + `page/<stem>.xml` with matching stems, in the same layout as for HTR training (`P/README.md` L178–200). The PageXML needs baselines and/or regions, depending on the mode (`P/tutorial/README.md`).

Before training, `preprocess_datasets` (`P/core/preprocess.py` L12–91; `P/data/preprocess.py` L730–792) turns PageXML into label masks in a temporary directory (`--tmp_dir`, else a system temp directory, removed unless `--keep_tmp_dir`; `train.py` L109–114). It writes `mode.txt` and `info.json`, and uses a `multiprocessing.Pool(os.cpu_count())`.

What is learned is set by `MODEL.MODE` (`P/page_xml/xml_regions.py` L234–255): `baseline`, `start`, `end`, `separator`, `baseline_separator`, `text_line`, `top_bottom`, `region`, `class_baseline`. For regions it also depends on `PREPROCESS.REGION.REGIONS`/`MERGE_REGIONS`/`REGION_TYPE`. Baselines are drawn `PREPROCESS.BASELINE.LINE_WIDTH` px wide (`P/configs/extra_defaults.py` L83–86).

### 2.3 Configuration

The config is a detectron2 YAML that inherits through `_BASE_`. Defaults come in this order: detectron2 defaults (`P/configs/defaults.py`), then Laypa's extras (`P/configs/extra_defaults.py`), then the file, then `--opts` (`P/core/setup.py` L80–95).

The example configs live in `P/configs/segmentation/{baseline,region,top_bottom}/` and `P/configs/panoptic/`. The base config `segmentation_base.yaml` uses:
- `SemanticSegmentor` on R-50-FPN;
- `SOLVER.IMS_PER_BATCH 8`, `BASE_LR 0.0002`, `MAX_ITER 250000`, `CHECKPOINT_PERIOD 25000`;
- `TEST.EVAL_PERIOD 10000`, `OUTPUT_DIR ./output`, `SEED 42`, `DATALOADER.NUM_WORKERS 16`.

`baseline_general.yaml` (`NAME: general_model`) changes:
- `IMS_PER_BATCH 32`, `MAX_ITER 750000`, `NUM_WORKERS 64`;
- AMP on, crop 1024², heavy augmentation;
- `LINE_WIDTH 10`, `SQUARE_LINES True`, `NUM_CLASSES 2`.

Laypa's own defaults (`extra_defaults.py`) include optimizer `ADAMW`, `BACKBONE_MULTIPLIER 0.1`, `AMSGRAD True` and `RUN_DIR True`. The maintainer's tuning advice is in [stefanklut/laypa#53](https://github.com/stefanklut/laypa/issues/53): `IMS_PER_BATCH`, `NUM_WORKERS`, `INPUT.CROP`, `MAX_SIZE_TRAIN`, AMP, and scaling `MAX_ITER` against batch size.

### 2.4 Three weight keys, three meanings

| key | used by | effect |
|---|---|---|
| `MODEL.WEIGHTS` | `train.py` via `resume_or_load(resume=MODEL.RESUME)` | **fine-tune**: load the model weights only, start at iteration 0 with a new optimizer (`P/train.py` L117–120; `D/detectron2/engine/defaults.py` L432–449). It can be a `detectron2://ImageNetPretrained/...` URL (tutorial config), which needs network access. |
| `TRAIN.WEIGHTS` | `train.py` | **resume from a file**: `checkpointer.load(path)` restores model, optimizer, scheduler and iteration. Training continues at `iter+1` (L121–123). If the checkpoint is already at `MAX_ITER`, training ends at once ([stefanklut/laypa#44](https://github.com/stefanklut/laypa/issues/44)). |
| `MODEL.RESUME True` | `setup_cfg` + `train.py` | resume from `last_checkpoint`. `OUTPUT_DIR` is set to the directory of the `-c` config, so you point `-c` at a run's saved `config.yaml` (`P/core/setup.py` L137–138; `train.py` L118–120). |
| `TEST.WEIGHTS` | `inference.py` | the weights used for inference. Inference refuses to run without them (`P/inference.py` L95–99). |

The README tells users to set `MODEL.WEIGHTS` for inference (`P/README.md` L309, L316, L465). **The code reads `TEST.WEIGHTS`.** Loghi's pipeline passes `--opts MODEL.WEIGHTS "" TEST.WEIGHTS <pth>` (`L/scripts/inference-pipeline.sh` L124–128).

**Fine-tuning recipe** (from the maintainer in [stefanklut/laypa#44](https://github.com/stefanklut/laypa/issues/44) and `P/README.md` L263–265):
- set `MODEL.WEIGHTS` to the base `.pth` (for example baseline2's `model_best_mIoU.pth`);
- leave `TRAIN.WEIGHTS` empty;
- lower `SOLVER.MAX_ITER` a lot.

The config must match the base model's architecture (`MODE`, `NUM_CLASSES`). In practice that means starting from the base model's own `config.yaml`. That file may still carry stale `TRAIN.WEIGHTS`, `OUTPUT_DIR` and so on from its original training, so a driver should override them.

### 2.5 Output

`OUTPUT_DIR` is relative by default (`./output/...`, relative to `/src/laypa` in the image), so **it is lost with `docker run --rm` unless overridden**. With `RUN_DIR True` (the default) and/or `NAME`, the real directory is `OUTPUT_DIR/RUN_<YYYY-mm-dd_HH-MM-SS>_<NAME>` (`P/core/setup.py` L128–135). A driver has to discover it, for example from the log line "Full config saved to …".

```
<run dir>/config.yaml                     # full merged config (setup.py L180-200; unique_path → config_1.yaml etc. if it exists)
<run dir>/log.txt                         # logger output (P/utils/logging_utils.py L92-137)
<run dir>/training_images.txt, validation_images.txt   # resolved image paths (P/core/preprocess.py L53-84)
<run dir>/metrics.json, events.out.tfevents.*          # detectron2 JSONWriter / TensorboardXWriter (D/detectron2/engine/defaults.py L264-281)
<run dir>/checkpoints/model_XXXXXXX.pth   # every SOLVER.CHECKPOINT_PERIOD (F/fvcore/common/checkpoint.py L428)
<run dir>/checkpoints/model_final.pth     # at MAX_ITER (L442)
<run dir>/checkpoints/model_best_{mIoU,fwIoU,mACC,pACC}.pth  # BestCheckpointer at each TEST.EVAL_PERIOD (P/core/trainer.py L278-312)
<run dir>/checkpoints/last_checkpoint
```

The evaluation metrics are mIoU, fwIoU, mACC and pACC on `sem_seg` (or `binary_seg`) (`P/core/trainer.py` L213–229, L278–310). They appear in `metrics.json` and `log.txt`. A loadable Laypa **model** is a `config.yaml` plus one `.pth`, usually `model_best_mIoU.pth`. Loghi's pipeline only mounts `dirname(config.yaml)`, so the `.pth` must lie in the same directory tree (`L/scripts/inference-pipeline.sh` L118–128).

### 2.6 Provenance recorded

`setup_cfg` stamps these keys into the config, and `config.yaml` saves them (`P/core/setup.py` L97–126):
- `SETUP_TIME` (timestamp);
- `LAYPA_UUID` (fresh `uuid4`);
- `LAYPA_GIT_HASH` (from `version_info` in the working directory, else `git rev-parse HEAD`; `P/core/setup.py` L55–63). `L/docker/docker.laypa/buildImage.sh` L133–135 writes `version_info` from the submodule;
- `CONFIG_PATH`;
- `TRAINING_PATHS` and `VALIDATION_PATHS` (paths inside the container);
- `SEED`.

There is no image tag, no data hash and no base-model identity beyond the `MODEL.WEIGHTS` path.

**At inference these values are regenerated.** `setup_cfg` runs again and overwrites `LAYPA_UUID`, `LAYPA_GIT_HASH` and `SETUP_TIME` with values for the *inference* run (L97–105). The PageXML `MetadataItem type="processingStep" name="layout-analysis" value="laypa"` has labels `githash`, `uuid`, `confidence`, plus optional `--whitelist` config keys (`P/page_xml/page_xml_editor.py` L166–210, L446–457; `P/page_xml/output_page_xml.py` L247–254). **It therefore identifies the inference run, not the trained model.** This differs from Loghi-HTR. Note also that in baseline mode the mask goes on to `MinionExtractBaselines`, which rewrites the PageXML.

### 2.7 Using a trained model

```
docker run … loghi/docker.laypa:2.3.2 python run.py -c <model>/config.yaml -i <images> -o <images> \
    --opts MODEL.WEIGHTS "" TEST.WEIGHTS <model>/model_best_mIoU.pth
```

(`L/scripts/inference-pipeline.sh` L114–158). `run.py` is deprecated in favour of `inference.py`, with the same arguments (`P/run.py` L1–20).

In baseline mode the output is a mask `page/<stem>.png` plus a placeholder PageXML. `MinionExtractBaselines -input_path_png page/` then turns the mask into baselines. Region mode writes polygons into the PageXML directly (`P/README.md` L276–281; `P/tutorial/README.md`).

---

## Implications for htr-tooling

Facts that the training-driving and provenance decisions will depend on.

1. **There is no upstream Laypa training script to stay compatible with.** `laypa-train-pipeline.sh` is empty and `docs/training.md` is blank there. The only stable interface is `python train.py -c … -t … -v … --opts …` in `loghi/docker.laypa`. For Loghi-HTR the interface is `python3 /src/loghi-htr/src/main.py` with flags or `--config_file` in `loghi/docker.htr`. The shell scripts are templates that users edit, not CLIs, so "drop-in compatibility" does not apply to training in the way it does to the minions.
2. **Training data creation is a minion job.** HTR training data comes from `MinionCutFromImageBasedOnPageXMLNew` with the training-only flags `-write_text_contents -include_text_styles -skip_unclear -no_page_update` (and optionally `-minimum_confidence`).
   - The phase-1 Python replacement of that minion must cover these flags, or training stays on the Java minion.
   - The crops are cut from contours **recalculated from baselines**, and the style markers enter the model's alphabet. If the Python cutter's crops or text formatting differ from Java's, a model trained on Java crops is tested on Python crops (and vice versa). Train/inference consistency is part of the equivalence question.
3. **The train/val split is not reproducible upstream.** It uses unseeded `shuf` and splits by line. htr-tooling can do better (seeded, by line or by page, recorded) without changing what Loghi-HTR consumes, because the list format is just `path<TAB>[weight<TAB>]text`.
4. **What makes up a trained model:**
   - **HTR model:** `model.keras` + `tokenizer.json` (or legacy `charlist.txt`) + `config.json`, taken from `<output>/<name>/best_val/`.
   - **Laypa model:** `config.yaml` + one `.pth` (by convention `model_best_mIoU.pth`), taken from `<run dir>/` and `<run dir>/checkpoints/`.
   - htr-tooling must find these. The Laypa run directory has a timestamped name, and both tools write to default paths inside the container unless output paths are given.
5. **Fine-tuning is a different mechanism in each tool:**
   - HTR: `--model <dir>`, plus `--replace_final_layer` if the alphabet changes. Without that flag, out-of-alphabet training lines are **silently dropped**, which a driver should report.
   - Laypa: `MODEL.WEIGHTS=<pth>` with `TRAIN.WEIGHTS` empty, plus a much smaller `SOLVER.MAX_ITER`. `TRAIN.WEIGHTS` *resumes* instead, which is a known trap.
6. **Model directories must be writable.** Loghi-HTR may write `tokenizer.json` (from `charlist.txt`) and rename `model.keras` → `model.keras.old` during a Keras-2→3 conversion *inside the base model directory*. A read-only mount breaks loading. A writable mount silently changes the user's model files. A driver should copy the model to a work directory first.
7. **Hardware.**
   - Laypa training needs CUDA by default (the `--num-gpus` assert, and AMP needs CUDA).
   - CPU training needs `--num-gpus 0` (inferred, untested) and AMP off, and is likely impractically slow.
   - Loghi-HTR trains on CPU with `--gpu -1` (float32).
   - Docker Desktop on Windows cannot reach the owner's Arc GPU (brief §5), and the brief rules out promising NVIDIA testing (§9). **Phase-2 training can realistically be tested on the owner's machine only on CPU with tiny fixtures**, and real GPU training would go untested. This should be settled in the specification.
8. **Precision tag.** GPU-trained HTR models are `mixed_float16` unless `--use_float32` is used. This matters for the ONNX ticket and for CPU inference, where Loghi converts to float32 on load.
9. **Upstream provenance is thin and inconsistent:**
   - **Loghi-HTR:** `config.json` has a training-run `uuid`, a `git_hash` (of loghi-htr HEAD at *image build time*, not the submodule pin), all args and `model_name`. It has no timestamp, image digest, data hash, base-model uuid or metrics.
   - **Laypa:** `config.yaml` has `SETUP_TIME`, `LAYPA_UUID`, `LAYPA_GIT_HASH`, the config and the data paths. These are **overwritten at inference**, so Laypa's PageXML metadata identifies the inference run, not the model.
   - **Merge minion:** `MinionLoghiHTRMergePageXML` writes the HTR model's uuid, name and training githash into PageXML. It drops the inference code's githash (an unused argument) and never writes `url-code` (a key mismatch).

   If htr-tooling wants provenance, it must record its own sidecar. Candidate fields: image name and digest; upstream git hashes; data list hashes; split seed and method; base model identity (uuid or hash); final and best metrics; start/end times; the full command. Whether the deviations above are fixed in the PageXML it writes falls under the brief's "known bugs are fixed, not copied" rule.
10. **Docker invocation details a driver must handle:**
    - Same-path bind mounts: every path in a list file must resolve inside the container, so lists hold absolute host paths, mounted at the same path.
    - `-u uid:gid`: this has no direct Windows equivalent. Docker Desktop and the Laypa image's `USER 1000` need checking.
    - `-m 32000m --shm-size 10240m`.
    - `--gpus device=N` only when using a GPU ([loghi#18](https://github.com/knaw-huc/loghi/issues/18)).
    - No `-t` when not attached to a TTY.
    - Optionally `TF_GPU_ALLOCATOR=cuda_malloc_async` ([loghi#55](https://github.com/knaw-huc/loghi/issues/55)).
    - Pin images by `2.3.2` or by digest, never `latest`, because `latest` is older than 2.3.2.

### Could not be established

- The exact contents of the published SURFdrive model folders (whether Laypa folders hold `config.yaml` + `model_best_mIoU.pth`, and whether HTR folders hold `config.json`/`charlist.txt`/`tokenizer.json`). The SURFdrive share was not fetched.
- Whether Laypa CPU training works with `--num-gpus 0` and AMP off. This is inferred from the source only.
- Which detectron2 commit is inside `loghi/docker.laypa:2.3.2` (unpinned `git+https` install).
- Whether Keras 3's `EarlyStopping(restore_best_weights=True)` changes the in-memory model used by the post-training `--do_validate` pass. It does not affect the saved `best_val/` model.
