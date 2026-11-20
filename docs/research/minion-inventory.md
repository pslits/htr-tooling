# Minion inventory: loghi-tooling as shipped in `loghi/docker.loghi-tooling:2.3.0`

*Research for issue #2. Written 2026-11-20. Primary sources only: the Java source of
[knaw-huc/loghi-tooling](https://github.com/knaw-huc/loghi-tooling), its README, the
issues of [knaw-huc/loghi](https://github.com/knaw-huc/loghi) (loghi-tooling has GitHub
issues switched off), the `inference-pipeline.sh` and Docker build files in knaw-huc/loghi,
and the Docker Hub registry metadata and layers of `loghi/docker.loghi-tooling`. No code
was run. Where a behaviour is described as a bug, that comes from reading the code, unless
stated otherwise.*

## How to read the citations

- **LT** = `https://github.com/knaw-huc/loghi-tooling/blob/71217767ed017a0dea6b66b681d8fa4a5729b634/`
  (the commit inside the 2.3.0 image, see §1). Line numbers are for that commit.
- **M/** = `LT minions/src/main/java/nl/knaw/huc/di/images/minions/`
- **WS/** = `LT loghiwebservice/src/main/java/nl/knaw/huc/di/images/loghiwebservice/`
- **PU** = `LT pagexmlutils/src/main/java/nl/knaw/huc/di/images/pagexmlutils/PageUtils.java`
- **LP** = `LT layoutanalyzer/src/main/java/nl/knaw/huc/di/images/layoutanalyzer/layoutlib/LayoutProc.java`
- **PIPE** = `https://github.com/knaw-huc/loghi/blob/2.3.0/scripts/inference-pipeline.sh`
- "loghi#N" = `https://github.com/knaw-huc/loghi/issues/N`

---

## 1. Version pin

| fact | value | source |
|---|---|---|
| loghi-tooling commit in the image | **`71217767ed017a0dea6b66b681d8fa4a5729b634`** (2026-02-24, "handle recursive dir finding…") — this is exactly tag `2.3.0` | see method below |
| image digest (`2.3.0`) | `sha256:828d011d7faa30d3ad8b338cc2759d4378d8fd80c7c948ce701d709091fb8e13`, built 2026-04-08 | Docker Hub tag API |
| `latest` tag | same digest as `2.3.0` (not moved to 2.3.1*/2.3.2) | Docker Hub tag API |
| base / JVM / OpenCV | Ubuntu 20.04, OpenJDK 11 JRE, OpenCV 4.12.0 built from source (`opencv-4120.jar`) | image config `history` |
| other code in the image | prima-core-libs at `6328dd79…` (loghi submodule pin), `rvankoert/langident` cloned at build time (HEAD at build, not pinned) | loghi `2.3.0` tree; image history |
| image labels | only Ubuntu's (`org.opencontainers.image.version=20.04`); no VCS label. `ENV LOGHI_VERSION=2.3.0`, `WORKDIR /src/prima-core-libs/java`, `EXPOSE 9006` | image config |

**Method.** The image has no git metadata (`buildImage.sh` deletes `.git`;
`loghi/docker/docker.loghi-tooling/buildImage.sh` at loghi tag 2.3.0). The layer
`COPY loghi-tooling /src/loghi-tooling` (`sha256:930a8e82…`) still holds the full source tree,
because the later `RUN` that deletes `src/` adds a new layer and does not shrink the earlier one.
I streamed that layer, computed the git blob hash of every non-`target/` file and compared it
with `git ls-tree` of tag 2.3.0: **570 of 570 files match, 0 differ**. The only files in the
tag but not in the image are 23 PNGs in `layoutanalyzer/src/test/resources/in/`, which
`buildImage.sh` removes on purpose. Independently, the loghi repo at tag `2.3.0` pins its
`loghi-tooling` submodule to the same commit `7121776`. The next commit on that lineage is
dated 2026-04-14, after the image build.

**Why the brief says "loghi-tooling 2.1.0".** `2.1.0` is the Maven project version in
`pom.xml`, and it is still `2.1.0` at tags 2.2.33, 2.3.0 and 2.3.2 (LT `pom.xml:9`). The
Dockerfile runs `mvn versions:set -DnewVersion=$LOGHI_VERSION clean package` in one call. In
that call the build has already read the version from the POM before `versions:set` rewrites
it. So the jars in the image are still named `*-2.1.0.jar`; the final layer's file listing
shows, for example, `altoxmlutils/target/altoxmlutils-2.1.0.jar`. The release is 2.3.0, and
the jars carry 2.1.0.

**What the image ships.** It has exactly the 17 `appassembler` launchers listed in the brief,
plus `LoghiWebserviceApplication` (final-layer listing; LT `minions/pom.xml:39-114`).
`MinionExtractBaselinesStartEndNew2` exists in the source but has no launcher.

### What changed between 2.3.0 and 2.3.2

- **The history was rewritten.** Tag `2.3.2` (`361dd36`) is *not* a descendant of `2.3.0`;
  `git merge-base` finds nothing. On the new lineage, commit `1d1e354` has the identical tree
  (`2e18a58…`) as `7121776`. After that there are **34 commits** (2026-04-14 … 2026-10-27).
  The image `2.3.2` (2026-10-28) moves to Ubuntu 26.04, OpenJDK 17 and OpenCV 4.14.0.
- **Changes that matter for compatibility or results:**
  - **Image formats:** one shared `ImageFormats`/`ImageReader` with jp2/j2k support.
    `ExtractBaselines` and `StartEndNew` now take *any* supported image in `-input_path_png`
    as a mask (before: `.png` only). `ConvertToPdf` takes any supported image (before:
    `.jpg` only). Commits `30029d0`, `3576317`, `f329c75`, `1946dda`.
  - **`ExtractBaselines`:** contour recalculation now passes `ignoreBroken=true` (was `false`).
  - **`SplitPageXMLTextLineIntoWords`:** the `-threads` option was **removed** from
    `getOptions()`, so `-threads N` now fails to parse. The minion then prints help and
    returns, still with exit code 0.
  - **Web service:** `/cut-from-image-based-on-page-xml-new` now defaults
    `recalculate_text_line_contours_from_baselines` to **false** (was true; `affba02`). There
    is a new **`POST /extract-and-cut`** endpoint (`88b740e`).
  - **Speed work** by T. Koornstra (`6393049`, `d9ff0cd`, `0386710`, `cd7ae91`): contour
    recalculation memory, `calcSeamImage` bulk Mat access, and the hot paths of reading order,
    word split and merge. These are meant to keep results the same. 2.3.2 adds
    `SplitImageRecalculateCorrectnessTest`.
  - `ARUProcessor` (1334 lines) was deleted. `System.out` was replaced by SLF4J. The
    busy-wait `while(!executor.isTerminated()){}` became `awaitTermination` in several
    minions. The README dropped the stale `removetext` note (§3.11).
  - **Not changed:** CLI flag names of all other minions; the `StartEndNew` NPE (§3.2); the
    `-margin`/`-minimum_interlinedistance` no-ops in `ExtractBaselines`.

---

## 2. Things all minions share

- **CLI parsing** uses Apache commons-cli `DefaultParser`. Options take a single dash
  (`-input_path`); long names given with one dash also work. On a parse error most minions
  print help and **return with exit code 0**. `ConvertPageToTxt` lets the `ParseException`
  escape (non-zero exit), and `FixPageXML`/`ConvertOCRResult` print the message and return 0.
- **Exit status does not show worker failures.** Each file is a `Runnable` on a fixed
  thread pool. Exceptions are caught and printed, or they kill the pool thread, and `main`
  still ends normally. `inference-pipeline.sh`'s `check_error_and_exit` only sees Docker's
  exit code (PIPE `:60-67`), so a page that fails inside a minion goes unnoticed. In 2.3.0,
  `main` waits with a CPU-burning busy loop (`while (!executor.isTerminated()) {}`, e.g.
  M/MinionExtractBaselines.java:564).
- **The PageXML version is a flag on (almost) every command:** `-use_2013_namespace` means
  2013 output, otherwise 2019. 2013 output is made by writing 2019 with Jackson and then
  running an XSLT (`transformpage.xslt`; PU `:249-275`). The XSLT drops
  `TextStyle/@xHeight`. **`inference-pipeline.sh` defaults to `USE2013NAMESPACE=1`** (PIPE
  `:54`, `:110-112`), so the standard pipeline writes PageXML 2013. Reading accepts both
  versions: the parser matches element names and ignores the namespace.
- **Reading and writing PageXML** (PU):
  - Every read calls `rebuildReadingOrder`, which builds or repairs
    `ReadingOrder/OrderedGroup` from region `custom="readingOrder {index:n;}"` (`:342-409`).
  - Most writes go through `convertAndValidate`, which validates against the PAGE schema and
    **throws "Page is not valid"** instead of writing (`:281-299`).
  - `writePageToFile` is not atomic; `writePageToFileAtomic` is used by `ExtractBaselines`,
    `Split` and the web service (`:301-312`).
  - `FixPageXML`, `ConvertOCRResult` and `StartEndNew3`'s intermediate step serialise without
    validation.
  - Region ids are UUIDs. Ids that start with a digit get the prefix `region_` (loghi#35).
- **Folder layout.** Images live in `<dir>`, PageXML in `<dir>/page/<image-basename>.xml`
  (Cut, ShrinkRegions/TextLines, ConvertToPdf). In the pipeline Laypa writes its PNG masks
  into `<dir>/page/` too (PIPE `:166-171`).
- **Line ids link the steps.** Cut names each snippet `<image-basename>-<TextLine@id>.png`;
  LoghiHTRMerge matches `<xml-basename>-<TextLine@id>`. ExtractBaselines keeps old line ids
  when a new baseline overlaps an old one (`BaselinesMapper`).
- **Metadata:** most steps append a `MetadataItem type="processingStep"` (names
  `layout-analysis`, `htr-<i>`, `reading-order`, `word-splitting`, `detect-language`). They
  build up over repeated runs.
- **`BaseMinion`** (M/BaseMinion.java:28-29) has a hard-coded server URI
  `http://localhost:9006/` and a hard-coded API key, left over from an older job server. None
  of the 17 CLI paths uses them.

---

## 3. The minions

Columns: **flag** · **arg** (type) · **default** · **req** (required) · notes. "—" = no argument (boolean switch).

### 3.1 `MinionExtractBaselines` — used by the pipeline (phase 1)

M/MinionExtractBaselines.java. **Reads** the baseline masks `<input_path_png>/*.png`, the
matching `<input_path_page>/<base>.xml`, and the original image (found via
`Page/@imageFilename` in `-input_path_image`, falling back to `<base>.jpg`; `:535-548`).
**Writes** `<output_path_page>/<base>.xml`.

| flag | arg | default | req | notes |
|---|---|---|---|---|
| `-input_path_png` | dir | — | yes | only `*.png` processed (`:519`) |
| `-input_path_page` | dir | — | yes | |
| `-output_path_page` | dir | — | yes | created if missing |
| `-input_path_image` | dir | null | no* | *effectively required: `Path.of(inputPathImage, …)` with null throws NPE in `main` (`:536`); explicitly required with `-recalculate_textline_contours_from_baselines` (`:477-479`) |
| `-as_single_region` | — | false | no | wipes existing regions, puts all lines in one full-page region (`:240-257`) |
| `-p2palaconfig` | file (JSON) | null | no | adds a P2PaLA `MetadataItem` |
| `-laypaconfig` | file (YAML) | null | no | adds a Laypa `MetadataItem`; prints every YAML key to stdout (`:715`) |
| `-config_white_list` | list | [] | no | config keys copied as metadata labels |
| `-margin` | int | 50 | no | **no effect**: `mergeTextLines` ignores it and calls `attachTextLines(…, 0)` (`:237`) |
| `-threads` | int | 4 | no | |
| `-invert_image` | — | false | no | `THRESH_BINARY_INV` (old P2PaLA: 0 = baseline) |
| `-threshold` | int | 32 | no | binarisation of the mask |
| `-region_order_list` | csv | none | no | region types for `reorderRegions`; `null` appended so untyped regions are kept (`:469-472`) |
| `-use_2013_namespace` | — | false | no | |
| `-recalculate_textline_contours_from_baselines` | — | false | no | the pipeline sets it (PIPE `:171`) |
| `-minimum_interlinedistance` | int | 35 | no | **parsed but unused**: the recalculation uses the constant 35 (`:481-484` vs `:633-636`) |
| `-split_baselines` | — | false | no | "experimental": splits vertically merged baselines (`LP.splitBaselines`) |
| `-trigger_line_thickness_multiplier` | double | 1.2 | no | for `-split_baselines` |
| `-max_line_thickness_multiplier` | double | 1.5 | no | for `-split_baselines` |
| `-minimum_baseline_thickness` | int | 1 | no | passed to the contour recalculation |
| `-help` | — | | | |

**What it does** (`:571-654`, `:120-205`):
1. Threshold the mask and take 8-connected components.
2. For each component, walk the columns of its bounding box. A column counts when at least
   3 pixels (`minimumHeight`) carry the label; its y is the mean of the row indices. Keep
   every 50th counted column plus the last one (`:157-163`), then simplify the line with
   tolerance 1 (`:343`).
3. Drop lines with fewer than 2 points or narrower than 15 px (`:579-581`, `:199`).
4. The line `Coords` is first just the component's bounding box. Line ids are mapped to old
   ids by overlap (`BaselinesMapper`).
5. Attach lines to regions with a sweep of overlap percentages 0.51 → 0.01 (`:232-239`).
   Lines left over each get their own new region.
6. Optionally reorder regions, add Laypa/P2PaLA metadata, and recalculate the line polygons
   with `LP.recalculateTextLineContoursFromBaselines(…, SHRINK_FACTOR=4, 35, thickness=10,
   minimumBaselineThickness, false)` (`:632-637`).
7. Write atomically with schema validation.

Note: the pipeline runs it with `-as_single_region` unless a Laypa region model is enabled
(`REGIONLAYPA=0` by default, PIPE `:12`, `:135`).

**Oddities:**
- `-margin` and `-minimum_interlinedistance` are no-ops.
- `imageMat.channels()` is called before the null check (`:595` vs `:600`).
- The PageXML is parsed twice (once in `main` to find the image filename).
- The help text names `MinionExtractBaselines`, but the README shows `MinionExtractBaseLines`
  (capital L), which does not exist as a launcher.
- **Tests:** `MinionExtractBaselinesTest` (3 tests of `mergeTextLines` on in-memory pages)
  and `BaselinesMapperTest` (6 tests, in-memory baselines).

### 3.2 `MinionExtractBaselinesStartEndNew` — not used by the pipeline

M/MinionExtractBaselinesStartEndNew.java. **Reads** three separate masks (baseline, start,
end) and an existing PageXML. **Writes** PageXML (non-atomic, validated). Only pages for
which all four files exist are processed (`:599-603`).

| flag | arg | default | req | notes |
|---|---|---|---|---|
| `-input_path_png` | dir | — | yes | baseline masks `*.png` |
| `-input_path_png_start` | dir | — | yes | |
| `-input_path_png_end` | dir | — | yes | |
| `-input_path_pagexml` | dir | — | yes | `<base>.xml` must exist |
| `-output_path_pagexml` | dir | — | yes | |
| `-as_single_region` | — | false | no | |
| `-remove_empty_regions` | — | false | no | |
| `-margin` | int | 50 | no | passed on, unused (same `mergeTextLines` as 3.1) |
| `-dilation` | int | 5 | no | x offset for the start point |
| `-minimum_height` | int | 5 | no | minimum pixels per column |
| `-region_order` | csv | [] | no | |
| `-use_2013_namespace` | — | false | no | |

There is no `-help` option (passing it gives a parse error, which prints help) and no
`-threads` (the pool size is cores/2+1, `:509`). Masks are thresholded at >0.

**What it does:** traces from start points to end points, then from end points backwards,
then the remaining components (each kept only if ≥50 columns). Duplicate lines are removed,
`mergeTextLines` is applied, then `LP.reorderRegions` and `LP.recalculateTextLinesFromBaselines`.

**Bugs (code reading):**
1. **Every page fails with an NPE in 2.3.0 and 2.3.2.** `zeroMatThresholded` is released and
   set to `null` (`:219`; `OpenCVWrapper.release` returns `null`,
   `layoutanalyzer/…/OpenCVWrapper.java:40-60`). It is then used again as the target of
   `Imgproc.threshold` (`:305`), so the call dereferences `null`. The NPE is not among the
   caught exceptions (`:668`), so the worker dies and no XML is written.
2. Even without that, `LP.reorderRegions(page, regionOrderList)` (`:428`) is called with an
   empty list unless `-region_order` is given. `reorderRegions` keeps only regions whose type
   is in the list (LP `:1538-1644`), so **all regions and lines would be dropped**.

No tests.

### 3.3 `MinionExtractBaselinesStartEndNew3` — not used by the pipeline

M/MinionExtractBaselinesStartEndNew3.java. **Reads** *one* colour PNG per page that holds
three channels: R = baseline, G = start, B = end (split at `:761-769`; the README's
"a png containing baselines and a png containing baseline start and ending" is
misleading). **Writes** PageXML (non-atomic, validated). If `<base>.xml` is missing, it makes
a page from the mask, with `imageFilename=<base>.jpg` (`:128-133`, `:728`).

| flag | arg | default | req | notes |
|---|---|---|---|---|
| `-input_path_png` | dir | — | yes | **must end in `/`**: paths are built by string concatenation (`:724-726`) |
| `-input_path_pagexml` | dir | — | yes | same trailing-slash requirement |
| `-output_path_pagexml` | dir | — | yes | same; created with `mkdir` (not `mkdirs`) |
| `-as_single_region` | — | false | no | **ignored**: `mergeTextLines(…, addLinesWithoutRegion=true, asSingleRegion=true, …)` is hard-coded (`:345-347`, "TODO asSingleRegion parameter") |
| `-remove_empty_regions` | — | false | no | |
| `-margin` | int | 50 | no | used in the 2nd attach pass |
| `-thickness` | int | 15 | no | **unused** |
| `-thickness_start_end` | int | 25 | no | **unused** |
| `-minimum_height` | int | 5 | no | **unused** |
| `-threads` | int | cores/2+1 | no | |
| `-use_2013_namespace` | — | false | no | |
| `-help` | — | | | |

**What it does:**
1. Threshold each channel at 50. Each baseline component is classified by how many start
   and end components it touches.
2. The orientation comes from the start and end centres. The component is rotated, the
   baseline is read with the pixel-weighted centroid per column (`PixelWeight` hard-coded,
   `:808`), and the points are rotated back.
3. Fragments without an end are merged with fragments without a start within 25 px, or with
   lines that have neither within 50 px.
4. **Debug strings are written into the output as `TextLine/TextEquiv`:** `"normal"`,
   `"linesWithMultipleEnd <x> <x>"`, `"only a start, but no end"`,
   `"mergeTextLinesWithoutEndToTextLinesWithoutStart"`, … (`:211`, `:225`, `:233`, `:246`,
   `:253`, `:285`, `:307`).
5. Statistics go to stdout.

**Bugs:**
- For components with no start and no end, the fallback line uses `rect.y + rect.y / 2`
  where `rect.height / 2` was surely meant (`:252`).
- `getCenter` uses `int` accumulators and divides by zero if nothing overlaps (`:580-588`).
- `close()` releases `labeledRemaining` a second time; that throws, the exception is caught,
  and the remaining Mats leak (`:340` vs `:843`).

No tests.

### 3.4 `MinionCutFromImageBasedOnPageXMLNew` — used by the pipeline (phase 1)

M/MinionCutFromImageBasedOnPageXMLNew.java. **Reads** the images in `-input_path`
(`.jpg/.jpeg/.png/.tif/.tiff`, case-insensitive, `:516-521`) and
`<page_path or input_path/page>/<base>.xml`. **Writes:**
- line snippets `<outputbase>/<base>/<base>-<lineId>.<output_type>`, built in a temporary
  directory and then moved into place (`:548-549`, `:700-703`);
- optionally `.txt/.box/.conf` files and `loghi-all-lines.txt`;
- **and, by default, rewrites the input PageXML in place**: with the recalculated line
  polygons and `TextStyle/@xHeight` (`:374`, `:467-475`, `:600-603`, `:705-707`).

| flag | arg | default | req | notes |
|---|---|---|---|---|
| `-input_path` | dir | — | yes | |
| `-outputbase` | dir | — | yes | |
| `-output_type` | str | `png` | no | `png` uses compression level `-png_compressionlevel`; anything else goes to `imwrite` with that extension |
| `-channels` | int | 4 | no | 4 = with a transparency mask outside the line polygon; any other value = no mask |
| `-threads` | int | 1 | no | the pipeline uses 4 |
| `-write_text_contents` | — | false | no | for training data: writes the text files; lines with empty text are skipped |
| `-xheight` | int | null | no | fixed x-height |
| `-minimum_xheight` | int | `LP.MINIMUM_XHEIGHT` | no | |
| `-rescaleheight` | int | null | no | rescale snippet to this height (min width 32) |
| `-min_width` | int | 5 | no | |
| `-difor_names` | — | false | no | names `textline_<base>_<lineId>` |
| `-page_path` | dir | `input_path/page` | no | |
| `-no_page_update` | — | false | no | without it, **the PageXML is rewritten** |
| `-write_done` | **takes an argument** (`hasArg=true`, `:227`) | false | no | used as a boolean (`:376`), so `-write_done` swallows the next token |
| `-ignore_done` | — | false | no | see the done-file bug below |
| `-copy_font_file` | — | false | no | copies `<base>_font.txt` (synthetic data) |
| `-include_text_styles` | — | false | no | style marker characters in the text |
| `-use_tags` | — | false | no | HTML tags instead of markers; needs `-include_text_styles` |
| `-no_text_line_contour_recalculation` | — | false | no | recalculation is **on by default**; the help text ("recalculate textline contours…") reads as the opposite |
| `-skip_unclear` | — | false | no | skips lines whose `custom` contains `unclear` |
| `-use_2013_namespace` | — | false | no | for the rewritten PageXML (2013 drops `xHeight`) |
| `-minimum_confidence` / `-maximum_confidence` | double | null | no | filter on `TextEquiv@conf` |
| `-minimum_interlinedistance` | int | 35 | no | |
| `-output_confidence_file` | — | false | no | |
| `-png_compressionlevel` | int | 1 | no | |
| `-tmpdir` | dir | `java.io.tmpdir` | no | pipeline: `$tmpdir`; should be on the same file system for atomic moves |
| `-minimum_baseline_thickness` | int | 1 | no | |
| `-fix_errors` | — | false | no | passed to `readPageFromFile(…, fixErrors)` |
| `-help` | — | | | |

Not settable from the CLI: `min_height=5`, `min_width_to_height=0`,
`outputBoxFile=outputTxtFile=true` (`:297-306`). The web service can set them.

**What it does:** recalculates the line polygons from the baselines
(`LP.recalculateTextLineContoursFromBaselines`, interline distance and projection profile;
loghi#37), then for each line calls `LP.getBinaryLineStrip` (seam-carved strip around the
baseline using x-height; loghi#30), filters by size, optionally rescales, and writes.

**Bugs and oddities:**
- **Done-file mismatch.** The check looks for `<identifier>.done` relative to the working
  directory (`:791`), but the writer creates `<full image path>.done` (`:479`). So
  `-write_done` files are never seen; in practice every run is `-ignore_done`.
- The image is decoded before the done check.
- `width/height` is integer division.
- On a broken image, it tries `Files.createDirectory(<outputbase>/<base>/<base>)`, which
  throws if the parent is missing (`:543`).
- History (loghi#43): native OpenCV segfaults with many threads in 2.2.4–2.2.7, fixed 2.2.8/2.2.9.

**Tests:** `MinionCutFromImageBasedOnPageXMLNewTest` is entirely commented out.
`SpecialTagReplacementTest` is a `main()` that prints tag replacements; it has no assertions
and no `@Test`.

### 3.5 `MinionLoghiHTRMergePageXML` — used by the pipeline (phase 1)

M/MinionLoghiHTRMergePageXML.java. **Reads** every `*.xml` in `-input_path`, the Loghi-HTR
`results.txt`, and optionally the model `config.json`. **Writes** each PageXML in place
(validated, non-atomic, `:548-557`).

| flag | arg | default | req | notes |
|---|---|---|---|---|
| `-input_path` | dir | — | yes | exits 1 if missing (`:537-540`) |
| `-results_file` | file | — | yes | |
| `-config_file` | file (JSON) | null | no | model config; **must contain `git_hash` and `model`**, or an NPE ends `main` (`:311-312`) |
| `-htr_code_config_file` | file (JSON) | null | no | **no effect**: its `git_hash` goes into the `gitHash` constructor parameter, which is never stored (`:53-69`, `:570`); same NPE risk as above |
| `-threads` | int | 4 | no | |
| `-comment` | str | null | no | appended to `Metadata/Comments` |
| `-use_2013_namespace` | — | false | no | |
| `-config_white_list` | list | `["batch_size"]` | no | config keys copied into metadata labels (searched recursively) |
| `-help` | — | | | |

**Results format** (`:384-421`): `path<TAB>conf<TAB>text` (2 tabs), or
`path<TAB>metadata<TAB>conf<TAB>text` (3 tabs, with a warning). The key is the last path
segment minus `.png`. Any other tab count throws `IllegalArgumentException` and aborts the
whole run, so a tab inside the predicted text breaks it. The README's example (space
separated, `page.xml-line.png`) is out of date.

**What it does for each `TextLine` with a result** (`:151-203`):
1. Turn `<u>/<s>/<sub>/<sup>` tags back into style marker characters.
2. Set `TextEquiv(conf, PlainText = ASCII transliteration, Unicode = text)`.
3. **Delete all `Word` children.**
4. Rebuild `custom` from the old one: drop old `textStyle` and **`readingOrder`** (it is set
   to `""`) and add new `textStyle {offset;length;…}` entries.
5. If there is more than one HTR config, add a `UserDefined` `htrProcessingStep`.

Lines without a result are left untouched. One `MetadataItem htr-<i>` is added per config.

**Bugs:**
- `extractTextLineCustom` splits `custom` on `}` and steps `i += 2` (`:89-91`), so **every
  other custom entry is dropped**: for `a {..} b {..} c {..}` it keeps `a` and `c`. The test
  `extractTextLineCustomTest2` passes only because the skipped entry happens to be
  `readingOrder`.
- See also loghi#36: preserving line `custom` was fixed in 2.1.4.

**Tests:** `MinionLoghiHTRMergePageXMLTest`, 22 `@Test`s on inline strings: result-line parsing,
nested and unclosed style tags, `TextLineCustom` round-trip. In loghiwebservice:
`LoghiHTRMergePageXMLResourceTest` (6 tests, result-line parsing).

### 3.6 `MinionPyLaiaMergePageXML` — not used by the pipeline

M/MinionPyLaiaMergePageXML.java. **Reads** `*.xml` in `-input_path` and a PyLaia results
file. **Writes** each XML in place.

| flag | arg | default | req | notes |
|---|---|---|---|---|
| `-input_path` | dir | — | yes | |
| `-results_file` | file | — | yes | |
| `-config_file` | file (JSON) | "" | no | declared twice (`:119-122`); needs `git_hash` and `model` |
| `-config_white_list` | list | `["batch_size"]` | no | looked up under `args` |
| `-use_2013_namespace` | — | false | no | |
| `-help` | — | | | |

Threads = all cores (`:137`), no flag.

**Results format** (`:195-211`): `path char char <space> char …`, i.e. space-separated
symbols with a `<space>` token. The key is the last path segment minus `.jpg`, and it must
equal `<xml filename including .xml>-<lineId>` (`:55-56`). That key scheme does **not** match
what Cut produces (`<base>-<lineId>.png`), so it only fits an older or external cutter.
Sets `TextEquiv` without confidence and does not touch `Word`s. Static shared map. No tests.

### 3.7 `MinionRecalculateReadingOrderNew` — used by the pipeline (phase 1)

M/MinionRecalculateReadingOrderNew.java. **Reads and writes** `*.xml` in `-input_dir`, in
place.

| flag | arg | default | req | notes |
|---|---|---|---|---|
| `-input_dir` | dir | — | yes | the README says `-input_path_page`, which is wrong |
| `-threads` | int | 2 | no | the pipeline uses 4 |
| `-clean_borders` | — | false | no | drops short baselines touching the left/right edge |
| `-border_margin` | int | 200 | no | the pipeline uses 50 |
| `-as_single_region` | — | false | no | all lines in one bounding-box region; **no readingOrder custom and no metadata** (`:373-384`) |
| `-dubious_size_width` | double | null (= width × multiplier) | no | |
| `-dubious_size_width_multiplier` | double | 0.05 | no | |
| `-interline_clustering_multiplier` | double | 1.5 | no | |
| `-reading_order_list` | csv | `[null]` | no | if given, no `null` is added, so regions of types not listed are **dropped** by `reorderRegions` |
| `-use_2013_namespace` | — | false | no | |
| `-help` | — | | | |

**What it does** (`:198-339`):
1. **Throws away all existing `TextRegion`s** (`:214`) and clusters all lines again into new
   regions. Lines join a cluster if their starts are within `interlineMedian × 1.5`, if they
   lie centred under the current line, or if they fall inside the cluster's box.
2. New regions get `custom="structure {type:Text;}"` and a box `Coords`.
3. Lines are sorted by distance to the region's top-left corner and get
   `readingOrder {index:n;}` (keeping the rest of the old custom).
4. Then `LP.reorderRegions` sets the region order and `ReadingOrder`.

The pipeline itself warns "this will remove regions found by Laypa" (PIPE `:28`); see
also loghi#36. The two-stage logic is described in loghi#26.

**Oddities:**
- With `-clean_borders`, the dropped lines vanish from the page completely.
- Exceptions inside a worker are swallowed **silently** on the CLI: the error writer is
  `Optional.empty()` (`:391-392`).
- An unreadable XML in `main` aborts the whole run.

**Tests:** `MinionRecalculateReadingOrderNewTest`, 3 tests on in-memory pages (region
counts only).

### 3.8 `MinionShrinkRegions` — not used by the pipeline

M/MinionShrinkRegions.java + PU `shrinkRegions` (`:1965-2050`). `-input <dir>` (req),
`-use_2013_namespace`, `-help`. Threads = cores. Only `*.jpg` images trigger work (`:44`), and
the page is read from `<dir>/page/<base>.xml`. **The image itself is not used.** The region
`Coords` is rebuilt from the points of its lines' `Coords` (a walk over corner points),
contrary to the README's "based on image content". No tests.

### 3.9 `MinionShrinkTextLines` — not used by the pipeline

M/MinionShrinkTextLines.java + PU `shrinkTextLines` (`:1735-1879`). Same flags as 3.8.
Images are picked with `javax.activation.MimetypesFileTypeMap` (`:51-56`). Whether `.png` is
recognised depends on that library's built-in MIME table; **not verified**.

What it does: adaptive threshold of the image (block size = width/50). It then trims
**baseline** points at the start and end where no ink is found within
`[y−xHeight(or 10), y+2·xHeight(or 20)]`, looking at most 50 points inward. It does not
change `Coords`, and it removes lines with fewer than 2 baseline points. No tests.

### 3.10 `MinionSplitPageXMLTextLineIntoWords` — used by the pipeline (`SPLITWORDS=1` by default)

M/MinionSplitPageXMLTextLineIntoWords.java + LP `splitLinesIntoWords` (`:3447-3557`).
`-input_path <dir>` (req), `-use_2013_namespace`, `-threads` (default 4; **removed in 2.3.2**),
`-help`. Rewrites each XML in place (atomic, validated).

What it does:
1. Split the line text on spaces. Width per character = baseline length / (characters +
   spaces).
2. Each `Word` gets the baseline segment for its length and a polygon from fixed offsets
   (35 px above, 10 px below; "FIXME TI-541"), plus `TextEquiv` with **no confidence**.
3. **No `Glyph`s.**
4. Lines with a character width < 2 px are **removed from the region** (`:3491-3495`).

Exceptions are silent on the CLI. History: loghi#29 (Transkribus rejected trailing spaces in
`custom`, fixed in 2.1.2). No unit tests. Web-service integration fixture:
`split_text_lines_into_words/`.

### 3.11 `MinionFixPageXML` — not used by the pipeline

M/MinionFixPageXML.java. Recurses through `-input_path` and rewrites every `*.xml` **in
place**. There is no `-output_path`, although the README's first example uses one; that
example fails with "Unrecognized option".

| flag | short | arg | default | notes |
|---|---|---|---|---|
| `-input_path` | **`-h`** | dir | — | required in practice ("no input path given"); `-h` is not help |
| `-removetext` | `-r` | — | false | `TextEquiv=null` on regions and lines; Word text stays |
| `-removewords` | `-w` | — | false | `Word` list = null |
| `-namespace` | `-n` | URI | 2019 | full namespace URI, not a switch |

**The brief's example bug is already fixed in the shipped code.** The README at 2.3.0 still
says "Known issue: `-removetext` sets `removeWords`" (LT `README.md:32`, `:211`). But commit
`083bd3b` (2025-11-21, "fix MinionFixPageXML where removeText caused removedWords to be
enabled") changed `removeWords = true` to `removeText = true` *before* 2.3.0, and the code at
`7121776` is correct (M/MinionFixPageXML.java:94-96). The note is stale; 2.3.2's README
dropped it (`a127ca2`).

Output is not validated (`convertPcGtsToString`).

**Tests:** `MinionFixPageXMLTest`, 4 tests on inline 2013/2019 strings (namespace and
schemaLocation).

### 3.12 `MinionConvertOCRResult` — not used by the pipeline

M/MinionConvertOCRResult.java (`:254-382`).

| flag | long | arg | default |
|---|---|---|---|
| `-i` | `-inputpath` | dir | `/scratch/altotest/` |
| `-o` | `-outputpath` | dir | `/tmp/limited/` |
| `-t` | `-targetformat` | str | `PageXML2013` |

There is no help option and no required flag. Not recursive.

What each target format does:
- `pagexml2019`: converts **ALTO → PageXML 2019** (plain Jackson `XmlMapper`, not
  `PageUtils`' writer; whether the 2019 namespace ends up in the output was not verified).
  Other source formats are skipped.
- `alto`: converts **PageXML 2013 → ALTO**, written only if it validates against `alto.xsd`.
- **The default `pagexml2013` and `hocr` do nothing** (empty `case`), so running with
  defaults is a no-op.

Format detection is by string matching. No tests in minions (`AltoUtilsTest` lives in
altoxmlutils).

### 3.13 `MinionConvertPageToTxt` — optional in the pipeline (`createtxt=0`)

M/MinionConvertPageToTxt.java + PU `convertToTxt` (`:2186-2208`). Writes `<xml-base>.txt`
next to each XML.

| flag | arg | default | notes |
|---|---|---|---|
| `-pagexmldir` | dir | — | required in practice (exit 1) |
| `-no_overwrite` | — | false | |
| `-overwrite_when_newer` | — | false | takes precedence over `-no_overwrite` |
| `-recurse_dir` | — | false | only XMLs whose parent folder is named `page` |
| `-threads` | int | 4 | |
| `-silent` | — | false | |
| `-linebased` | — | false | one line per TextLine, blank line between regions |
| `-plaintext` | — | false | `TextEquiv/PlainText` instead of `Unicode` |

There is no `-help`; an unknown flag throws `ParseException` (exit ≠ 0).

The default is a **merged** mode: lines are joined with spaces, a trailing `-` is
de-hyphenated by joining with the next line, and regions are separated by `\n`. The order is
the document order of the XML, not `ReadingOrder` indices. Reads with `ignoreErrors=true`.
The flags `-recurse_dir`, `-overwrite_when_newer`, `-threads` and `-silent` arrived in the
2.3.0 commit itself (`7121776`). No tests.

### 3.14 `MinionConvertToPdf` — optional in the pipeline (`createpdf=0`, "experimental")

M/MinionConvertToPdf.java. **Positional arguments:** `<output.pdf> <image-dir>`; no flags.
Lowercase `*.jpg` only in 2.3.0 (`:52`). Page from `<dir>/page/<base>.xml`. One PDF page per
image (PDFBox).

For each line, text is written at the **first baseline point**, horizontal only, in
Type1 **Times-Roman**. The font size is `xHeight` (in points), or it is fitted to the
baseline length when there is no `xHeight`.

**Oddities:**
- It uses `TextEquiv/PlainText`, the ASCII transliteration, not `Unicode`; the intended
  fallback is a no-op (`:97-100`).
- The image is drawn *after* the text, so the text sits underneath (invisible, but
  searchable).
- A character that Times-Roman/WinAnsi cannot encode raises `IllegalArgumentException`, which
  is not caught in 2.3.0 (only `IOException`), so the whole run fails.

No tests.

### 3.15 `MinionDetectLanguageOfPageXml` — used by the pipeline (`DETECTLANGUAGE=1` by default)

M/MinionDetectLanguageOfPageXml.java. Rewrites the XML(s) in place.

| flag | arg | default | req | notes |
|---|---|---|---|---|
| `-page` | file or dir | — | yes | |
| `-lang_train_data` | dir or file | built-in | no | one text file per language; the file name is the label (should be a PAGE `LanguageSimpleType`) |
| `-threads` | int | 8 | no | |
| `-use_2013_namespace` | — | false | no | |
| `-help` | — | | | |

The model is naive Bayes (`rvankoert/langident`), trained at every start, from scratch. The
built-in data has six languages: Dutch, English, French, German, Italian, Latin
(`minions/src/main/resources/lang-ident-training-data/`). It sets
`TextRegion@primaryLanguage` from the region's Unicode text, `TextLine@primaryLanguage` from
the **`PlainText`** (ASCII transliteration) of each line, and `Page@primaryLanguage` from all
text.

**Oddities:**
- The `detect-language` `MetadataItem` is added **only if a `pom.xml` exists in the current
  working directory** (`:239-263`). It then records that POM's version. In the image the
  working directory is `/src/prima-core-libs/java` (image config), where the build ran
  `mvn clean package`, so it presumably records prima-core-libs' version; not executed to
  confirm.
- Empty text still gets a prediction.

No unit tests; web-service integration fixtures in `detect_language/`.

### 3.16 `MinionGarbageCharacterCalculator` — not used by the pipeline

M/MinionGarbageCharacterCalculator.java. `-page_file <file>` (req), `-characters_file <file>`
(req), `-help`. It joins the `Unicode` text of all lines with `\n` and counts characters not
in the set of characters of the allowed file. Newlines count too, unless the file contains a
newline. It prints total, garbage and an **integer-truncated** percentage to stdout and writes
nothing. The help header names `MinionGeneratePageImages` (copy-paste, `:38`, `:43`). No tests.

### 3.17 `MinionGeneratePageImages` — not used by the pipeline (used by loghi `scripts/generate-synthetic-images.sh`)

M/MinionGeneratePageImages.java. **Reads** `*.txt` from `-text_path` and `*.ttf` from
`-font_path`. **Writes** `<output_path>/synthetic<NNNNNNNNNN>-<font>.png`,
`<output_path>/page/<same>.xml`, optional `_font.txt`, and `<output_path>synthetic.txt`.

| flag | arg | default | req | notes |
|---|---|---|---|---|
| `-text_path` | dir | — | yes | 40-line random windows per file |
| `-output_path` | dir | — | yes | **needs a trailing `/`** for `synthetic.txt` (`:258`) |
| `-font_path` | dir | — | yes | |
| `-make_old` | — | false | no | long s etc. |
| `-add_salt_and_pepper` | — | false | no | noise + Gaussian blur |
| `-random_augment` | — | false | no | **parsed, unused** (`:221`) |
| `-underline` | — | false | no | |
| `-save_font` | — | false | no | |
| `-chance_italic` / `-chance_bold` / `-chance_underline` / `-chance_uppercase` / `-chance_line` | double | 0.2 each | no | |
| `-min_font_size` | int | 36 | no | **also sets the maximum**: `fontMaxSize` is read from `MIN_FONT_SIZE` (`:241`) |
| `-max_font_size` | int | 96 | no | **ignored** (same line) |
| `-max_text_length` | int | 150 | no | |
| `-multiply` | int | 1 | no | |
| `-max_files` | int | 500000 | no | |
| `-blur_window` | int | 11 | no | |
| `-blur_sigmax` | int | 25 | no | |
| `-characters` | str | built-in set | no | `""` = allow everything |
| `-use_2013_namespace` | — | false | no | |
| `-help` | — | | | |

**Oddities:**
- Not reproducible: `new Random()` with no seed option (`:606-611`).
- `synthetic.txt` looks like it is always empty: `merged` is printed but never filled
  (`:253`, `:405`).
- Lines are skipped when the font cannot render them.

No tests.

---

## 4. `loghiwebservice` (Dropwizard)

WS/LoghiWebserviceApplication.java, WS/LoghiWebserviceConfiguration.java. Config template:
`https://github.com/knaw-huc/loghi/blob/2.3.0/webservice/loghi-tooling/configuration.yml`.
Launched as `LoghiWebserviceApplication server <config.yml>` (loghi `webservice/docker-compose.yml`).

**Ports:** application 8080, admin 8081 (env `APPLICATION_PORT`, `ADMIN_PORT`). The image's
`EXPOSE 9006` is a leftover.

**Upload/output root:** `uploadLocation` = `$STORAGE_LOCATION` (default `/tmp/upload`).

Each resource has its own bounded thread pool (`maxThreads` default 1, `queueLength`
1000 or 50) and its own status ledger.

**Results are not returned over HTTP.** Every POST queues a job and returns at once. The job
writes its output under `<uploadLocation>/<identifier>/…`. The only read-back is a status
endpoint. A client must share that file system.

| endpoint (2.3.0) | wraps | form fields (req = required) | hard-wired differences from the CLI |
|---|---|---|---|
| `POST /extract-baselines` | `MinionExtractBaselines` | **`mask`**, **`xml`**, **`image`**, **`identifier`** (req); `margin` (50), `namespace` (2019 URI), `invertImage`, `p2pala_config` (file, read only when `invertImage=true`), `addLaypaMetadata` + `laypa_config` (file), `config_white_list` (repeatable), `splitBaselines` | `asSingleRegion=true`, recalculate contours = true, threshold 32, no region order; output `<id>/<xml filename>` (WS/resources/ExtractBaselinesResource.java:89-196) |
| `POST /cut-from-image-based-on-page-xml-new` | `MinionCutFromImageBasedOnPageXMLNew` | **`image`**, **`page`**, **`identifier`**, **`output_type`**, **`channels`** (req); `min_width` 5, `min_height` 5, `min_width_to_height_ratio` 0, `write_text_contents`, `rescale_height`, `output_conf_file`, `output_box_file` (true), `output_txt_file` (true), `recalculate_text_line_contours_from_baselines` (**true** in 2.3.0, **false** from 2.3.2), `fixed_x_height`, `min_x_height`, `include_text_styles`, `use_tags`, `minimum_baseline_thickness` | does **not** update the PageXML (`overwriteExistingPage=false`); snippets to `<id>/<image base>/` |
| `POST /loghi-htr-merge-page-xml` | `MinionLoghiHTRMergePageXML` | **`page`**, **`identifier`**, **`results`** (file) or **`resultsString`** (req); `namespace`, `comment` | accepts only the **3-tab** line format; other lines are skipped with a warning; ≥4 tabs gives a 500. HTR configs come from the per-line metadata column. The README's `htr-config` and `config_white_list` fields are **never read**. Output `<id>/<page base>.xml` |
| `POST /recalculate-reading-order-new` | `MinionRecalculateReadingOrderNew` | **`page`**, **`border_margin`**, **`identifier`** (req); `namespace`, `interline_clustering_multiplier` 1.5, `dubious_size_width_multiplier` 0.05, `dubious_size_width` | `clean_borders=false`, `as_single_region=false`, default order list |
| `POST /split-page-xml-text-line-into-words` | `MinionSplitPageXMLTextLineIntoWords` | **`xml`**, **`identifier`** (req); `namespace` | |
| `POST /detect-language-of-page-xml` | `MinionDetectLanguageOfPageXml` | **`identifier`**, **`page`** (req); `training_data` (repeatable files); `namespace` | |
| `GET /<each of the above>/status/{identifier}` | — | — | `queued` / `finished` / `not found` (404) |
| `POST /apikey/login` | — | header `Authorization: <uuid>` → 204 + `X_AUTH_TOKEN` | registered only when `securityConfig.enabled` (default false) |
| admin `:8081/prometheus`, Dropwizard admin pages | — | — | |
| `POST /extract-and-cut` | extract + cut in one job | union of the two | **2.3.2 only** (`88b740e`) |

**Web-service bugs and oddities (code reading):**
- **The JSON responses are not valid JSON:**
  - `"queueStatus": 3/1000` is not quoted (WS/configuration/ExecutorServiceConfig.java
    `createQueueUsageStatusSupplier`);
  - `extract-baselines` emits a bare `warnings:` key (ExtractBaselinesResource.java:218-220).
- **`status` says `finished` for a failed job too:** `Future.isDone()` is also true after an
  exception, and the `isCancelled` branch can never be reached (WS/resources/LoghiWebserviceResource.java:27-43).
- **Ledger eviction.** The ledger is a plain `TreeMap`: not thread-safe, and it evicts
  `firstKey()`, the alphabetically first identifier rather than the oldest.
- **Sticky failure.** Once a job has appended to the resource's `minionErrorLog` (e.g. one
  failed write), every later request to that endpoint returns 500 "Minion is failing" until
  the service restarts.
- **Silently ignored metadata.** In `loghi-htr-merge-page-xml`, per-line metadata in
  Python-dict style (`{'model_name': …}`, as in the test fixtures) is only rewritten to
  double quotes when it does *not* already start with `{` and end with `}`
  (LoghiHTRMergePageXMLResource.java:228-235). So Jackson fails to parse it, and the config
  is logged and skipped.
- **The README's curl examples don't work as written:** `extract-baselines` examples omit the
  required `image` field (400), and one example uses `p2palaa_config` where the code reads
  `p2pala_config`.

**Tests:** shell integration scripts that drive a running service with curl
(`loghiwebservice/src/test/resources/integration_tests/test_*.sh`). Only the extract test
checks a value: **105 `<Baseline` elements** for the fixture page.

---

## 5. Unit tests and fixtures (at 2.3.0)

| module | minion-related tests | fixtures |
|---|---|---|
| `minions/src/test` | `MinionExtractBaselinesTest` (3), `BaselinesMapperTest` (6), `MinionLoghiHTRMergePageXMLTest` (22), `MinionRecalculateReadingOrderNewTest` (3), `MinionFixPageXMLTest` (4), `SpecialTagReplacementTest` (a `main()`, no assertions), `MinionCutFromImageBasedOnPageXMLNewTest` (all commented out) | **all inline**: in-memory `PcGts` objects, coordinate strings, result-line strings. There is no `src/test/resources` in minions. |
| `loghiwebservice/src/test` | `LoghiHTRMergePageXMLResourceTest` (result-line parsing) + 7 curl scripts | one page, `NL-0400410000_26_005006_000381`: `.jpg` (5.1 MB, 5382×3756), Laypa baseline mask `.png`, an empty Laypa PageXML, `laypa_config.yaml`, `htr_config.json`, a 110 kB Loghi-HTR `results.txt` (3-tab format), and PageXML at several stages. `detect_language/` adds `NL-HaNA_2.05.31_1_0031.xml` and 6 training files. |
| `layoutanalyzer/src/test` | `LayoutProcTest`, `ConnectedComponentTest`, … (library level, not per minion) | `in/`, `out/`, `out-sep/` (33 PNGs each), `NL-HaNA_1.01.02_3780_0067-in/out.jpg`, `NL-HaNA_1.01.02_3855_0059-in/out.jpg`, `0284_0011.tif` (95 MB), etc. |
| `pagexmlutils/src/test` | `PageUtilsTest` (reading-order rebuild), `PageSerializationTest`, `PageValidatorTest`, `StyledString*`, `GroundTruthTextLineFormatter*` | `NL-HlmNHA_1972_8_0022.xml` |

Minions **without any test**: StartEndNew, StartEndNew3, PyLaiaMerge, ShrinkRegions,
ShrinkTextLines, SplitPageXMLTextLineIntoWords (only the curl script), ConvertOCRResult,
ConvertPageToTxt, ConvertToPdf, DetectLanguageOfPageXml (only the curl script),
GarbageCharacterCalculator, GeneratePageImages. Cut's only test is commented out.

2.3.2 adds `BaseMinionTest`, merge/reading-order/split-image benchmarks and
`SplitImageRecalculateCorrectnessTest`.

No licence statement was found for the fixture scans (the repo is MIT; where the scans come
from is not stated).

---

## 6. Could not be established

- Exit-code behaviour, the `StartEndNew` NPE and the other "bugs" above come from reading the
  code; **nothing was executed**.
- Whether `ShrinkTextLines` recognises `.png` (MIME table of the activation library).
- Whether `ConvertOCRResult`'s ALTO→PageXML output carries the 2019 namespace.
- Which `langident` commit is in the image (cloned unpinned at build time).
- Whether `DetectLanguage` in the image really records prima-core-libs' POM version (inferred
  from WORKDIR plus the build step).
- Licence of the fixture images (`NL-0400410000_…`, `NL-HaNA_…`).

---

## 7. Implications for htr-tooling

1. **Pin = loghi-tooling `7121776` (tag 2.3.0) in image digest `sha256:828d011d…`, run on
   OpenCV 4.12.0 / JDK 11.** Characterisation fixtures should be recorded against that digest,
   not against `latest`, even though `latest` currently points to it. 2.3.2 changes defaults
   (web-service Cut recalculation), image readers and OpenCV (4.14).
2. **Baseline extraction is not "known only as bytecode".** The full Java source of
   `MinionExtractBaselines` (854 lines) and `LayoutProc` (3843 lines at 2.3.0) is public
   under MIT. The
   real algorithm is short:
   - threshold 32, then 8-connected components;
   - per column, the mean row of the label, with ≥3 px per column;
   - one point every 50 columns plus the last, simplified with tolerance 1;
   - drop lines under 15 px or with fewer than 2 points.

   The hard part is `LayoutProc.recalculateTextLineContoursFromBaselines` (the line polygons)
   and `getBinaryLineStrip` (the seam-carved snippets in Cut). This bears directly on the
   "baseline extraction source" decision: translating the Java is an option alongside reusing
   Laypa's Python extractor.
3. **Cut writes PageXML too.** Without `-no_page_update`, it rewrites the input PageXML with
   new line polygons and `xHeight`. In the pipeline the polygons are therefore computed twice
   (ExtractBaselines with `-recalculate_textline_contours_from_baselines`, then Cut by
   default). The compat layer has to reproduce that side effect; the pipeline command can
   compute them once.
4. **RecalculateReadingOrderNew replaces regions.** It discards the incoming regions (Laypa's
   or ExtractBaselines' single region) and clusters again. Region equivalence (§5 of the
   brief) has to be measured *after* this step, against new UUID ids. In the default pipeline,
   region assignment is decided here, not in ExtractBaselines (`-as_single_region` is on by
   default).
5. **Default pipeline output is PageXML 2013** (`USE2013NAMESPACE=1`). The 2013 path drops
   `TextStyle/@xHeight`. The brief's default of 2019 is a deviation from what Loghi users get
   today unless the compat layer honours the pipeline's flag.
6. **Deviation-list candidates found here.** The `FixPageXML -removetext` example in the brief
   is **not** a bug in 2.3.0 (already fixed in `083bd3b`; only the README note was stale).
   Better examples:
   - `StartEndNew` always fails (NPE) and would drop all regions anyway;
   - `StartEndNew3` ignores `-as_single_region`, `-thickness*` and `-minimum_height`, needs
     trailing slashes, and writes debug text into `TextEquiv`;
   - `ExtractBaselines` ignores `-margin` and `-minimum_interlinedistance`;
   - Cut's `-write_done` takes an argument, and done files are never detected;
   - `LoghiHTRMerge` ignores `-htr_code_config_file` and drops every other `custom` entry;
   - `GeneratePageImages` ignores `-max_font_size` and `-random_augment`;
   - `ConvertToPdf` uses ASCII `PlainText`;
   - `DetectLanguage` classifies lines from `PlainText`;
   - `Split` deletes short lines;
   - `ConvertOCRResult` with defaults is a no-op;
   - worker failures still give exit code 0.
7. **Compat-layer surface.**
   - Single-dash commons-cli syntax, with long names also accepted after one dash.
   - Short aliases on `FixPageXML` (`-h` = input path!) and `ConvertOCRResult`.
   - One minion with positional arguments (`ConvertToPdf`).
   - Flags that are accepted but ignored must keep being accepted, so scripts don't break.
   - Version skew: 2.3.2 removed `Split -threads`. Decide which minion version the compat
     layer copies and say so.
8. **Web-service replacement (phase 3).** Today's API is asynchronous and file-system based:
   POST, then poll `/status/{id}`, then read `<STORAGE_LOCATION>/<id>/…` from shared disk.
   Its JSON is invalid and its status reports failed jobs as finished. Copying it exactly is
   probably not worth it; the endpoint list in §4 is the compatibility surface if wanted.
9. **Fixture seed.** loghiwebservice's integration fixtures give a ready end-to-end chain for
   one page:
   - image + Laypa mask + empty PageXML → ExtractBaselines (expected: 105 baselines);
   - → Cut;
   - a recorded Loghi-HTR `results.txt` → Merge.

   Usable for a first characterisation test, if the licence of the scan can be cleared.
10. **Speed baseline caveat.** In 2.3.0 each minion's `main` waits in a busy loop that keeps
    one core at 100 %. Measurements of "no slower than the minion" on a laptop should take
    that into account.
