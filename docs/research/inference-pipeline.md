# Anatomy of Loghi's `inference-pipeline.sh`

*Research for [#3](https://github.com/pslits/htr-tooling/issues/3). Written 2026-11-20. Primary sources only; the script was read, not run.*

## Sources and versions

| what | where | commit |
|---|---|---|
| the script | [`knaw-huc/loghi` `scripts/inference-pipeline.sh`][S] | tag `2.3.2` = `main` HEAD = `e743513` (2026-10-28) |
| Loghi docs | [`README.md`][README], [`docs/index.md`][INDEX], [`scripts/README.md`][SREADME] | same |
| image build | [`docker/buildAll.sh`][BUILD] | same |
| minions | `knaw-huc/loghi-tooling` (`minions/…/Minion*.java`), pom version `2.1.0` | `361dd36` (submodule pointer at tag 2.3.2) |
| Laypa | `knaw-huc/laypa` (`run.py`, `inference.py`, `page_xml/output_page_xml.py`) | `d0ed632` (submodule pointer) |
| Loghi-HTR | `knaw-huc/loghi-htr` (`src/main.py`, `src/setup/*`, `src/utils/threading.py`) | `c24c745` (submodule pointer) |
| image tags | Docker Hub API `hub.docker.com/v2/repositories/loghi/<repo>/tags` | queried 2026-11-20 |

Between tag `2.3.0` (the version measured in the brief) and `2.3.2`, the only change to the script is `VERSION=2.3.0` → `VERSION=2.3.2` (GitHub compare `90305b9...e743513`). So everything below also holds for 2.3.0.

**Caveat on minion, Laypa and HTR line references.** They are cited at the submodule pointers of tag 2.3.2. But [`buildAll.sh`][BUILD] (L14–37) runs `git submodule update --remote` and `git pull` on each submodule's branch before it builds. So an image holds whatever the branch head was at build time, not necessarily the pinned pointer. To confirm the exact code inside `loghi/docker.*:2.3.2`, you have to inspect the image. That was not done here.

## What the script is

- **One positional argument:** `inference-pipeline.sh <path_to_images>`. With any other number of arguments it prints usage and exits 1 ([S] L70–73). It has **no command-line flags**. Every option is a shell variable that you edit in the file, above the line `# DO NOT MODIFY BELOW THIS LINE` ([S] L6–56).
- The image folder is made absolute with `realpath` ([S] L86). Model paths are **not** made absolute.
- `set -e` and `set -o pipefail` are set ([S] L3–4).
- Every step is a separate `docker run --rm` against one of three images. The container runs as the calling user (`-u $(id -u $USER):$(id -g $USER)`). Each host folder is bind-mounted at **the same path** inside the container (`-v X:X`), so host paths and container paths are identical.
- Every step's stdout goes through `tee -a $tmpdir/log.txt`.

## Pinned images

`VERSION=2.3.2` ([S] L2) feeds all three tags ([S] L76–78):

| variable | image |
|---|---|
| `DOCKERLAYPA` | `loghi/docker.laypa:2.3.2` |
| `DOCKERLOGHIHTR` | `loghi/docker.htr:2.3.2` |
| `DOCKERLOGHITOOLING` | `loghi/docker.loghi-tooling:2.3.2` |

All three tags exist on Docker Hub (pushed 2026-10-28). The `latest` tag of each repo has the **same digest as `2.3.0`**, not 2.3.2 (for example `docker.htr:latest` = `2.3.0` = `sha256:414fc89ac574…`, while `2.3.2` = `sha256:787038b25d2e…`). So the README's `docker pull loghi/docker.*`, which pulls `latest`, gives older images than the script uses.

## Script options and defaults

| variable | default | effect |
|---|---|---|
| `STOPONERROR` | `1` | Meant to make step failures non-fatal when set to 0 ([S] L8, [S] L59–67). **It has no effect:** under `set -e` + `pipefail`, a failing `docker … \| tee` pipeline ends the script before `check_error_and_exit` runs. |
| `BASELINELAYPA` | `1` | Run Laypa baselines, then `MinionExtractBaselines` (steps 1 and 3). |
| `REGIONLAYPA` | `0` | Run Laypa regions (step 2). It only takes effect if `BASELINELAYPA=1`, because it is nested inside that `if` ([S] L138). |
| `LAYPABASELINEMODEL` | `INSERT_FULL_PATH_TO_YAML_HERE` | Laypa config `.yaml` (full path). |
| `LAYPABASELINEMODELWEIGHTS` | `INSERT_FULL_PATH_TO_PTH_HERE` | Laypa weights `.pth` (full path). |
| `LAYPAREGIONMODEL` / `…WEIGHTS` | placeholders | The same, for the region model. |
| `HTRLOGHI` | `1` | Run steps 4–6 (cut, HTR, merge). |
| `HTRLOGHIMODEL` | `INSERT_FULL_PATH_TO_LOGHI_HTR_MODEL_HERE` | Loghi-HTR model **folder** (it holds `config.json`, the model, `charlist.txt`/`tokenizer.json`). |
| `RECALCULATEREADINGORDER` | `1` | Step 7. The comment warns: "this will remove regions found by Laypa". |
| `RECALCULATEREADINGORDERBORDERMARGIN` | `50` | Passed as `-border_margin`. The minion only uses it together with `-clean_borders`. |
| `RECALCULATEREADINGORDERCLEANBORDERS` | `0` | Adds `-clean_borders`. |
| `RECALCULATEREADINGORDERTHREADS` | `4` | Passed as `-threads`. |
| `DETECTLANGUAGE` | `1` | Step 8. |
| `SPLITWORDS` | `1` | Step 9. |
| `createpdf` | `0` | Step 10 (marked "experimental"). |
| `createtxt` | `0` | Step 11. |
| `BEAMWIDTH` | `1` | Passed to Loghi-HTR as `--beam_width` (the comment says not to go above 10). |
| `GPU` | `0` | `-1` means CPU. Any other value N adds `--gpus device=N` to the Laypa and HTR containers and passes `--gpu N` to Loghi-HTR ([S] L80–84). |
| `USE2013NAMESPACE` | **`1`** | Adds `-use_2013_namespace` to every PageXML-writing minion except `ConvertToPdf` and `ConvertPageToTxt` ([S] L110–112). **So the default output is PageXML 2013.** |

Hard-coded values that you cannot change without editing below the "do not modify" line: Laypa/HTR memory `-m 32000m --shm-size 10240m`; HTR `--batch_size 64`; cutter `-output_type png -channels 4 -threads 4`; `-recalculate_textline_contours_from_baselines` on ExtractBaselines.

Before any step runs, the script checks for placeholders. It refuses to run if an enabled model variable still contains its placeholder text ([S] L97–108).

## Steps

`$IMG` = the absolute image folder. `$TMP` = `$IMG/tmp.loghi-inference-XXXXXX` (see "Temporary files" below).

| # | step | image | command | flags passed | reads | writes | optional? |
|---|---|---|---|---|---|---|---|
| 0 | housekeeping | (host) | `mktemp -d`, `mkdir`, `find … -name '*.done' -delete` | — | — | creates `$TMP/`, `$TMP/imagesnippets/`, `$TMP/output/`; **deletes every `*.done` file anywhere under `$IMG`** (recursive) ([S] L88–95) | no |
| 1 | Laypa baselines | `docker.laypa` | `python run.py` (a deprecated wrapper for `inference.py`) | `-c $LAYPABASELINEMODEL -i $IMG -o $IMG --opts MODEL.WEIGHTS "" TEST.WEIGHTS $LAYPABASELINEMODELWEIGHTS`; docker `$DOCKERGPUPARAMS -it -m 32000m --shm-size 10240m`; mounts `dirname(yaml)` and `$IMG` ([S] L115–132) | images directly in `$IMG` (non-recursive glob); the yaml and pth | `$IMG/page/<stem>.xml` (an empty Page with a Laypa processing-step metadata item, 2013 namespace) and `$IMG/page/<stem>.png` (grayscale baseline mask). External post-processing is on by default (`POSTPROCESS.EXTERNAL.ENABLED=True`), so Laypa writes a mask, not baselines. | `BASELINELAYPA` |
| 2 | Laypa regions | `docker.laypa` | `python run.py` | as step 1, with `$LAYPAREGIONMODEL`/`…WEIGHTS`; mounts `dirname(region yaml)` ([S] L138–158) | images in `$IMG` | **overwrites** `$IMG/page/<stem>.xml` with a new PageXML that holds TextRegions. Region mode skips the mask, so the step-1 PNG stays. Turns off `-as_single_region` for step 3. | `REGIONLAYPA` (and only if step 1 runs) |
| 3 | extract baselines | `docker.loghi-tooling` | `MinionExtractBaselines` | `-input_path_image $IMG -input_path_png $IMG/page/ -input_path_page $IMG/page/ -output_path_page $IMG/page/ -recalculate_textline_contours_from_baselines [-as_single_region] [-use_2013_namespace]`; mounts `$IMG` only ([S] L160–179) | for each image file in `page/` (the masks): `page/<stem>.png`, `page/<stem>.xml`, and the page image `$IMG/<imageFilename from PageXML>` (falls back to `<stem>.jpg`) | `page/<stem>.xml` in place, with TextLines (baseline + contour polygon, random UUID ids) merged into one region (`-as_single_region`) or into the Laypa regions | runs if and only if step 1 runs |
| 4 | cut line images | `docker.loghi-tooling` | `MinionCutFromImageBasedOnPageXMLNew` | `-input_path $IMG -outputbase $TMP/imagesnippets/ -tmpdir $TMP -output_type png -channels 4 -threads 4 [-use_2013_namespace]` ([S] L183–198) | images in `$IMG`, `$IMG/page/<stem>.xml` (default `-page_path` = `input_path/page`) | `$TMP/imagesnippets/<stem>/<stem>-<lineId>.png` (4-channel PNG, the alpha channel is the line mask). It **also rewrites `page/<stem>.xml`**: line contours are recalculated from the baselines **a second time** (default on, `minimum_interlinedistance` 35), and the page is saved because `-no_page_update` is not passed. | `HTRLOGHI` |
| 4b | list lines | (host) | `find $TMP/imagesnippets -name '*.png' > $TMP/lines.txt` ([S] L201) | — | snippet PNGs | `$TMP/lines.txt` | `HTRLOGHI` |
| 5 | Loghi-HTR | `docker.htr` | `bash -c "TF_USE_LEGACY_KERAS=0 LD_PRELOAD=/usr/lib/$(arch)-linux-gnu/libtcmalloc_minimal.so.4 python3 /src/loghi-htr/src/main.py …"` | `--model $HTRLOGHIMODEL --batch_size 64 --inference_list $TMP/lines.txt --results_file $TMP/results.txt --gpu $GPU --output $TMP/output/ --beam_width $BEAMWIDTH`; docker `$DOCKERGPUPARAMS -ti -m 32000m --shm-size 10240m`; mounts `$TMP` and `dirname($HTRLOGHIMODEL)`. The command is first **echoed** (not run) for the log ([S] L203–235). | `lines.txt`, snippet PNGs, model folder | `$TMP/results.txt`, one line per snippet: `<path>\t<confidence>\t<text>`; `$TMP/output/config.json` (run config with `git_hash`, `uuid`, `model`, …) | `HTRLOGHI` |
| 6 | merge text | `docker.loghi-tooling` | `MinionLoghiHTRMergePageXML` | `-input_path $IMG/page -results_file $TMP/results.txt -config_file $HTRLOGHIMODEL/config.json -htr_code_config_file $TMP/output/config.json [-use_2013_namespace]`; mounts `dirname($HTRLOGHIMODEL)` and `$IMG`, **but not `$TMP`** ([S] L239–254) | every `page/*.xml`; `results.txt`; both config.json files | `page/<stem>.xml` in place: each line's `TextEquiv` (Unicode, an ASCII transliteration as PlainText, and `conf`) and line `custom` (text styles); **Words are cleared**; a `processingStep` MetadataItem `htr-0` with githash, model_name, uuid and a whitelist (default `batch_size`) | `HTRLOGHI` |
| 7 | reading order | `docker.loghi-tooling` | `MinionRecalculateReadingOrderNew` | `-input_dir $IMG/page/ -border_margin 50 -threads 4 [-clean_borders] [-use_2013_namespace]` ([S] L257–278) | `page/*.xml` | `page/*.xml` in place. It **clears all TextRegions** and re-clusters the lines into new regions plus a ReadingOrder (so Laypa regions from step 2 are lost). | `RECALCULATEREADINGORDER` |
| 8 | language | `docker.loghi-tooling` | `MinionDetectLanguageOfPageXml` | `-page $IMG/page/ [-use_2013_namespace]` (built-in training data; default 8 threads) ([S] L281–292) | `page/*.xml` | `page/*.xml` in place: `primaryLanguage` on lines, regions and the page | `DETECTLANGUAGE` |
| 9 | split words | `docker.loghi-tooling` | `MinionSplitPageXMLTextLineIntoWords` | `-input_path $IMG/page/ [-use_2013_namespace]` ([S] L295–306) | `page/*.xml` | `page/*.xml` in place (atomic write): `Word` elements with interpolated coords | `SPLITWORDS` |
| 10 | PDF | `docker.loghi-tooling` | `MinionConvertToPdf` | positional: `$IMG/output.pdf $IMG` ([S] L308–318) | images in `$IMG`, `$IMG/page/<stem>.xml` | `$IMG/output.pdf` | `createpdf` |
| 11 | TXT | `docker.loghi-tooling` | `MinionConvertPageToTxt` | `-pagexmldir $IMG/page/` (no namespace flag) ([S] L320–330) | `page/*.xml` | `page/<stem>.txt` next to each XML | `createtxt` |
| 12 | cleanup | (host) | `rm -rf $TMP` ([S] L333) | — | — | removes `$TMP` | no |

Notes on the step table:

- **Step 3 needs step 1.** Without `BASELINELAYPA=1` no PageXML exists unless the user supplies `page/*.xml` themselves. The pipeline then starts at step 4 on existing PageXML (for example from Transkribus); see the [scripts README][SREADME] and the [docs][INDEX] on mixing tools.
- **The PageXML is rewritten in place up to 7 times** (steps 1/2, 3, 4, 6, 7, 8, 9). Each step reads the file the previous step wrote. Only the snippets, `lines.txt`, `results.txt` and the HTR run config go through `$TMP`.
- **Line polygons are computed twice.** Step 3 computes them (`-recalculate_textline_contours_from_baselines`), and step 4 computes them again by default. The second result is what lands in the final PageXML and what the snippets are cut from.
- **Step 6 joins lines to text by name.** It matches `results.txt` to TextLines by key `<xml stem>-<TextLine id>`. That key is the snippet filename without `.png` and without its folder ([LT merge L152], `getResultLine`). Line ids therefore have to survive steps 4–6 unchanged.
- **Mounts.** Step 6 mounts `dirname($HTRLOGHIMODEL)`, not `$TMP`. This works only because `$TMP` lives inside `$IMG`, which is mounted.

## Models

- **Laypa.** You pass two absolute paths: the config `.yaml` (`-c`) and the weights `.pth` (`--opts MODEL.WEIGHTS "" TEST.WEIGHTS <pth>`). Only `dirname(yaml)` and `$IMG` are mounted. So the `.pth` must sit in the yaml's folder, or under `$IMG`, or the container cannot see it. A relative path also breaks the `-v` mount, because Docker reads it as a named volume.
- **Loghi-HTR.** You pass one absolute **folder** (`--model`). Loghi-HTR loads `charlist.txt`/`tokenizer.json` from it when it is a directory ([HTR main.py L59–62]). The merge step reads `<folder>/config.json`. The README ([README]) says: "the entire folder containing the 'model' folder and other documents".
- **Model files.** The docs point to a SURFdrive share for both kinds of model. On CPU, they advise the `float32-` HTR models over the default `mixed_float16` ones ([README], [INDEX]).

## GPU selection

- **One variable, `GPU`.** If it is greater than -1, the Laypa and HTR containers get `--gpus device=$GPU` ([S] L80–84). The loghi-tooling containers never get a GPU.
- **Laypa** gets no device flag of its own. Its default is `MODEL.DEVICE="cuda"`, and it falls back to CPU when `torch.cuda.device_count()==0` (`core/setup.py` at `d0ed632`).
- **Loghi-HTR** also gets `--gpu $GPU`. `"-1"` means no GPU. Otherwise it keeps the TF physical GPUs *whose index inside the container* matches ([HTR environment.py L89–111]).
- **Likely bug for `GPU` ≥ 1 (inferred from code, not tested).** `--gpus device=N` exposes only GPU N, which shows up as index 0 inside the container. `--gpu N` then matches nothing, so Loghi-HTR would silently run on CPU.
- **Commands need a terminal.** The Laypa and HTR `docker run` calls use `-it`/`-ti`, which need a TTY. Run without one (cron, CI, a non-interactive SSH session) and Docker fails with "the input device is not a TTY". This is known Docker behaviour; it was not reproduced here.
- **`LD_PRELOAD` is computed on the host.** The path uses `` `arch` `` inside a double-quoted string, so the shell evaluates it on the **host**, not in the container.

## Temporary files

- **`$TMP` lives inside the image folder:** `mktemp -d --tmpdir="$IMG" tmp.loghi-inference-XXXXXX` ([S] L88). It holds:
  - `imagesnippets/<stem>/*.png`
  - `lines.txt`, `results.txt`
  - `output/config.json`, plus whatever else Loghi-HTR writes to `--output`
  - `log.txt`
  - the cutter's per-image staging folders `LoghiCutter_<uuid>.<stem>`, which `-tmpdir $TMP` places there and which are moved atomically into `imagesnippets/`
- **Cleanup happens only on success.** `$TMP`, including `log.txt`, is removed only at the very end ([S] L333). Because of `set -e`, any failure leaves `$TMP` (and its log) behind in the image folder.
- **Leftover temp folders are not re-read.** On the next run Laypa ignores them, because its input glob is non-recursive (`utils/input_utils.py` at `d0ed632`). The cutter and the other minions list only the top-level files of `$IMG` or `page/`.
- **Files left in the image folder after a successful run:**
  - `page/<stem>.xml`, the result
  - **`page/<stem>.png`, the Laypa baseline mask, which is never removed**
  - optionally `page/<stem>.txt` and `output.pdf`
- **`*.done` files: origin unknown.** Step 0 deletes them recursively under `$IMG`. Of the minions read, only the cutter writes `.done` files (`<image>.done`), and only with `-write_done`, which the script does not pass. Why the script deletes them could not be established; it may be a leftover from older versions or from the web-service workflow.

## Cross-check against the Loghi docs

| doc says | script does |
|---|---|
| README and docs/index.md give the placeholder `INSERT_FULLPATH_TO_PTH_HERE` | `INSERT_FULL_PATH_TO_PTH_HERE`. Cosmetic, but anyone grepping for the doc's string finds nothing. |
| docs/index.md: `HTRLOGHIMODEL=INSERT_FULL_PATH_TO_FOLDER_CONTAINING_LOGHI_HTR_MODEL_HERE` | `INSERT_FULL_PATH_TO_LOGHI_HTR_MODEL_HERE`. |
| docs/index.md: the HTR zip unzips to "a folder with a .yaml and a .pth file" | That is a copy of the Laypa text. The HTR model is a folder with `config.json` and a Keras model; the README says so correctly. |
| docs/index.md (performance): "change your batch_size in the default inference pipeline"; "change the `threads` parameter" | `--batch_size 64` and the cutter's `-threads 4` are hard-coded below "DO NOT MODIFY". The only exposed threads option is `RECALCULATEREADINGORDERTHREADS`. |
| README: `docker pull loghi/docker.laypa` (and the same for htr and tooling) | That pulls `latest`, which is 2.3.0. The script runs `:2.3.2`, which Docker pulls on first use anyway. |
| scripts/README: "baseline detection, region detection, …, detecting line language, and splitting text lines into words … each can be enabled or disabled" | Matches, except that region detection is off by default and cannot run without baseline detection. PDF and TXT output are not mentioned. |
| README: "a new folder called page should be created … This contains the PageXML output" | Correct. The page folder also keeps the Laypa `.png` masks, which the docs do not mention. |
| Script comment: `STOPONERROR` "if set to 1 will exit program if any of the docker commands fail" | It always exits on failure, whatever its value (see above). |
| README: images are ".jpg … preferred/tested", placed directly in the folder | Consistent: ExtractBaselines falls back to `<stem>.jpg`, and every image listing is non-recursive. |

The `docs/` folder has no dedicated pipeline page. `docs/index.md` repeats the README quick start, `training.md` covers training, and `webservice.md` covers the web service.

## Implications for htr-tooling

1. **The compatibility contract is "one folder in, `page/` out".** The input is a folder of images, read non-recursively. The output is `page/<stem>.xml` for each image, joined to its image by stem. That is the folder layout the pipeline command and the minion layer must keep.
2. **The default output is PageXML 2013.** The script defaults to `USE2013NAMESPACE=1`, and Laypa itself writes 2013. The brief says 2019 is the default for htr-tooling. The pipeline command must state its namespace default explicitly, and the characterisation fixtures should record 2013, because that is what `inference-pipeline.sh` really produces.
3. **The default chain is 1 → 3 → 4 → 5 → 6 → 7 → 8 → 9.** That covers baselines, extract, cut, HTR, merge, reading order, language and word split. So `DetectLanguageOfPageXml` and `SplitPageXMLTextLineIntoWords` are part of the default output, while the brief puts them in phase 2. Either phase 1 accepts a pipeline that stops after reading order (a recorded deviation), or those two steps run through the Docker fallback in phase 1. This needs a decision on the map.
4. **The in-memory model must reproduce several hidden behaviours:**
   - line contours recomputed twice, the cutter's result being the final one
   - Words cleared by the merge step and recreated by the split step
   - all regions discarded and re-clustered by the reading-order step
   - line ids as UUIDs, used as the join key `<stem>-<lineId>` between snippets and HTR results
5. **Configuration is file-editing, not flags.** A drop-in for the *script* itself (as distinct from the minions) has no CLI surface to copy. What a Loghi user knows is the variable set in the options table above. The pipeline command can map those variables one-to-one onto flags or a config file (for example `--no-reading-order`, `--beam-width`, `--gpu`, `--namespace 2013`) and should also expose what the script hard-codes (batch size, threads).
6. **Models are passed as a Laypa `(yaml, pth)` pair and an HTR folder (with `config.json`).** The HTR `config.json` and the run's own config both feed provenance metadata into the PageXML (`processingStep` items with githash, model name and uuid). Equivalence tests should ignore or normalise that metadata. They should also normalise `LastChange` and the random line UUIDs.
7. **Baseline measurements against the script must account for its quirks:**
   - Laypa and HTR need a TTY (`-it`)
   - `GPU` must be `-1` on the owner's laptop, which has no NVIDIA GPU, so Docker cannot use a GPU there
   - only `$IMG` and the model folders are mounted
   - `latest` ≠ `2.3.2`

   Measure with the pinned `:2.3.2` images (or `:2.3.0` as in the brief; the script is otherwise identical).
8. **Side files are part of what users see.** These are the `page/*.png` masks left behind, `$TMP` left behind on failure, and `*.done` files deleted under the image folder. The pipeline command should decide on each of them deliberately and document it, rather than copy them by accident.
9. **Minion flags are only part of the surface.** The script uses a small subset of each minion's flags (listed in the step table). The drop-in layer must still accept every flag (brief §4.3), but the defaults that matter for equivalence are the ones above: `-recalculate_textline_contours_from_baselines`, `-as_single_region`, `-channels 4`, `-output_type png`, the reading-order `-border_margin 50`, and the cutter's default contour recalculation.

## Open questions this research could not settle

- The exact loghi-tooling, Laypa and Loghi-HTR commits inside the `:2.3.2` images (see the caveat about `buildAll.sh`). This needs `docker run … cat`/`git log` inside the images, or the image labels.
- Where `*.done` files in the image folder come from in current workflows.
- Whether the `GPU ≥ 1` mismatch really sends HTR to the CPU (inferred from code).

[S]: https://github.com/knaw-huc/loghi/blob/e74351383ec64fd03689c8550d3123f0eb92c037/scripts/inference-pipeline.sh
[README]: https://github.com/knaw-huc/loghi/blob/e74351383ec64fd03689c8550d3123f0eb92c037/README.md
[INDEX]: https://github.com/knaw-huc/loghi/blob/e74351383ec64fd03689c8550d3123f0eb92c037/docs/index.md
[SREADME]: https://github.com/knaw-huc/loghi/blob/e74351383ec64fd03689c8550d3123f0eb92c037/scripts/README.md
[BUILD]: https://github.com/knaw-huc/loghi/blob/e74351383ec64fd03689c8550d3123f0eb92c037/docker/buildAll.sh
[LT merge L152]: https://github.com/knaw-huc/loghi-tooling/blob/361dd36a3b557944cdffaba1344a735e67aea613/minions/src/main/java/nl/knaw/huc/di/images/minions/MinionLoghiHTRMergePageXML.java#L152
[HTR main.py L59–62]: https://github.com/knaw-huc/loghi-htr/blob/c24c745e9aeb287bedeb6b4edee670ae5c81a3cd/src/main.py#L59-L62
[HTR environment.py L89–111]: https://github.com/knaw-huc/loghi-htr/blob/c24c745e9aeb287bedeb6b4edee670ae5c81a3cd/src/setup/environment.py#L89-L111

### Further source pointers (tag 2.3.2 submodule commits)

- ExtractBaselines options and main loop: [`MinionExtractBaselines.java` L353–560](https://github.com/knaw-huc/loghi-tooling/blob/361dd36a3b557944cdffaba1344a735e67aea613/minions/src/main/java/nl/knaw/huc/di/images/minions/MinionExtractBaselines.java#L353-L560)
- Cutter options, page update and snippet naming: [`MinionCutFromImageBasedOnPageXMLNew.java` L217–251, L375–416, L583, L685, L740](https://github.com/knaw-huc/loghi-tooling/blob/361dd36a3b557944cdffaba1344a735e67aea613/minions/src/main/java/nl/knaw/huc/di/images/minions/MinionCutFromImageBasedOnPageXMLNew.java#L217-L251)
- Merge: TextEquiv and Words cleared [L185](https://github.com/knaw-huc/loghi-tooling/blob/361dd36a3b557944cdffaba1344a735e67aea613/minions/src/main/java/nl/knaw/huc/di/images/minions/MinionLoghiHTRMergePageXML.java#L185); default whitelist [L520](https://github.com/knaw-huc/loghi-tooling/blob/361dd36a3b557944cdffaba1344a735e67aea613/minions/src/main/java/nl/knaw/huc/di/images/minions/MinionLoghiHTRMergePageXML.java#L520)
- Reading order clears regions: [`MinionRecalculateReadingOrderNew.java` L244](https://github.com/knaw-huc/loghi-tooling/blob/361dd36a3b557944cdffaba1344a735e67aea613/minions/src/main/java/nl/knaw/huc/di/images/minions/MinionRecalculateReadingOrderNew.java#L244)
- Laypa output (mask vs. regions, `page/` dir): [`page_xml/output_page_xml.py` L564–641](https://github.com/knaw-huc/laypa/blob/d0ed632ab6ca4e595d7224f0bc046d337b1bcb0e/page_xml/output_page_xml.py#L564-L641); `POSTPROCESS.EXTERNAL.ENABLED = True` in [`configs/extra_defaults.py`](https://github.com/knaw-huc/laypa/blob/d0ed632ab6ca4e595d7224f0bc046d337b1bcb0e/configs/extra_defaults.py); `run.py` deprecated in favour of [`inference.py`](https://github.com/knaw-huc/laypa/blob/d0ed632ab6ca4e595d7224f0bc046d337b1bcb0e/inference.py)
- Loghi-HTR results format: [`src/utils/threading.py`](https://github.com/knaw-huc/loghi-htr/blob/c24c745e9aeb287bedeb6b4edee670ae5c81a3cd/src/utils/threading.py); `config.json` written to `--output`: [`src/setup/config.py` L140–156](https://github.com/knaw-huc/loghi-htr/blob/c24c745e9aeb287bedeb6b4edee670ae5c81a3cd/src/setup/config.py#L140-L156)
