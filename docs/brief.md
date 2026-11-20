# htr-tooling — brief

*Working name. Status: brief, the first of three documents (brief → specification → design). Written 2026-11-20 after a system design interview with the owner. Decisions in this brief are the owner's; everything marked "for the specification" is still open.*

## 1. Problem

[Loghi](https://github.com/knaw-huc/loghi) reads handwriting in three stages: Laypa finds baselines, Loghi-HTR reads the lines, and **loghi-tooling** does everything in between and around them. That tooling is 17 Java command-line programs ("minions") plus a web service. They pass work to each other as PageXML files on disk.

What this costs, as seen in `loghi/docker.loghi-tooling:2.3.0` (loghi-tooling 2.1.0) and measured on the owner's laptop:

- **The rules of the domain are scattered.** Each minion is a `main` + `run` script. The PageXML classes copy the XML schema and store points as text. One class (`LayoutProc`, 103 kB of bytecode) mixes image processing, geometry and layout rules.
- **New behaviour comes by copying.** `MinionExtractBaselinesStartEndNew`, `…New2` and `…New3` are three copies. The PageXML version is a flag on every command (`-use_2013_namespace`).
- **The steps can't be tested or extended on their own.** No step owns a model that keeps itself consistent. Each one reads a file, changes it and writes it back.
- **Most of the time goes to starting up.** Each step starts its own container for each scan. Loghi-HTR alone takes 24 s to start against 0.14 s per line; with about 23 lines per scan, roughly 90 % of the reading time goes to startup.
- **CPU only on Intel.** Loghi-HTR runs on TensorFlow 2.20, which cannot use an Intel Arc GPU.

**Goal.** A testable foundation: the whole chain in one process around Laypa and Loghi-HTR, with an explicit domain model, that later improvements can build on. Speed follows from that. It is not the goal.

## 2. Users

1. **The Loghi community**: archives and researchers who run Loghi's `inference-pipeline.sh` or call the minions from their own scripts. They come first.
2. **genealogie-transcriptie**: one of the users, with no priority over the first group. It will call the library behind its own segmentation port.

## 3. Scope

| | what | how |
|---|---|---|
| **Replace** | all of loghi-tooling: the 17 minions and `loghiwebservice` | new implementation in Python |
| **Drive** | Laypa and Loghi-HTR, for inference and for training | through adapters; not rewritten |
| **Leave to Loghi** | the APIs of Laypa and Loghi-HTR, the Loghi-HTR GUI | out of scope |
| **Out** | anything Loghi cannot do today (new models, new architectures, decoding with a language model) | see §9 |

The 17 minions: `ExtractBaselines`, `ExtractBaselinesStartEndNew`, `ExtractBaselinesStartEndNew3`, `CutFromImageBasedOnPageXMLNew`, `LoghiHTRMergePageXML`, `PyLaiaMergePageXML`, `RecalculateReadingOrderNew`, `ShrinkRegions`, `ShrinkTextLines`, `SplitPageXMLTextLineIntoWords`, `FixPageXML`, `ConvertOCRResult`, `ConvertPageToTxt`, `ConvertToPdf`, `DetectLanguageOfPageXml`, `GarbageCharacterCalculator` and `GeneratePageImages` (all prefixed `Minion`).

## 4. What it must do

1. **Library.** A Python library that offers the use cases on an in-memory domain model. Programs such as genealogie-transcriptie use this.
2. **Modern CLI.** The main entry point. Subcommands share one style, good `--help` and sensible defaults. One **pipeline command** runs the whole `inference-pipeline.sh` chain in one process, loading each model once.
3. **Drop-in compatibility.** All 17 minion commands, with their names, every flag and Loghi's folder layout (images in a folder, PageXML in `<folder>/page/<image>.xml`). Existing scripts must keep working unchanged. This layer and the CLI are both thin layers on the same library; neither calls the other.
4. **Known bugs are fixed, not copied.** A known minion bug is fixed, listed as a deviation, and reported to Loghi later. Example: in `FixPageXML`, `-removetext` sets words instead of removing text (Loghi's own README says so).
5. **PageXML at the edge.** Read and write PageXML 2019 (the default) and 2013. Per character: text, position, confidence and alternatives, written as `Word`/`Glyph` with `Coords` and `TextEquiv@conf`/`@index`. There is no storage format of its own.
6. **Two model runtimes per model.** Every Laypa or Loghi-HTR model can run in two ways:
   - **Loghi in Docker:** the reference and the fallback.
   - **ONNX Runtime:** with a selectable execution provider (CPU, OpenVINO, CUDA, DirectML).
7. **Training** (phase 2). Train Laypa and Loghi-HTR from the library and the CLI, using their own training code in Docker.
8. **Web service** (phase 3). A replacement for `loghiwebservice`.

## 5. Quality requirements

- **Equivalence.** Each replaced step is checked by characterisation tests against the recorded output of the real minion on fixed fixtures. That means:
  - the same lines, with baselines within a few pixels;
  - the same line polygons, with IoU ≥ 0.9 as a starting point;
  - the same region assignment;
  - the same Loghi-HTR text on the line crops.

  A step may differ only where a measurement shows it is better, and that difference is recorded. Exact tolerances: for the specification.
- **Fallback.** If a step does not match within tolerance, the end result decides: a chain that gives the same or better Loghi-HTR text on the fixtures is accepted, and the deviation is recorded. If it does not, that one step keeps running the real minion (Docker) behind the same port, so the system stays usable.
- **Speed.** Relative, on the owner's laptop (Intel Core Ultra 7 255H, Arc 140T), after a baseline measurement of `inference-pipeline.sh` on the fixtures:
  - the pipeline command on the CPU is at least **3× faster per scan**;
  - with OpenVINO on the Arc it is faster than on the CPU;
  - each compatibility command is no slower than the minion it replaces.
- **Platforms.** Python, running natively on Windows and Linux.

  | | tested | supported, not tested |
  |---|---|---|
  | Windows | CPU, Arc/NPU via OpenVINO | DirectML |
  | Linux | CPU, in CI | CUDA |

  Docker Desktop on Windows cannot see the Arc, so the Arc is reached only through native ONNX Runtime.
- **Language.** Code, documentation and the domain glossary are in English. A translation table links the terms to those of genealogie-transcriptie (§10).

## 6. Constraints

- **Public, MIT.** A NOTICE keeps the copyright of the translated minions (loghi-tooling, MIT, © rvankoert) and of any reused Laypa code (MIT, © Stefan Klut). Loghi-HTR is MIT too. detectron2 and TensorFlow (Apache-2.0) stay inside the Docker images.
- **No model weights in the repository.** No licence file was found for the published Loghi models (they come from a SURFdrive share). Users fetch the weights themselves.
- **Test data.** A handful of pages from an openly licensed set are kept in the repository, ideally 17th-century Dutch. The owner's own scans are used only locally, through an environment variable, and never committed. Which set: for the specification.
- **Built independently.** The owner builds this without the Loghi maintainers for now, and may contact them later.

## 7. Phases

There is no fixed appetite. The fallback rule in §5 stops a step from blocking a phase.

1. **Phase 1:** `ExtractBaselines`, `CutFromImageBasedOnPageXMLNew`, `LoghiHTRMergePageXML` and `RecalculateReadingOrderNew`, the pipeline command, and an **ONNX exploration**: convert both models and compare them with the Docker reference.
2. **Phase 2:** the other PageXML minions (`ExtractBaselinesStartEnd…`, `PyLaiaMergePageXML`, `ShrinkRegions`, `ShrinkTextLines`, `SplitPageXMLTextLineIntoWords`, `FixPageXML`, `ConvertPageToTxt`, `GarbageCharacterCalculator`, `DetectLanguageOfPageXml`), and **driving training**.
3. **Phase 3:** `ConvertToPdf`, `ConvertOCRResult`, `GeneratePageImages`, and the **web service**.

Each phase is done when its steps pass the equivalence or fallback rule and the speed requirements that apply to them.

## 8. Rabbit holes

- **Baseline extraction.** The heuristics of `ExtractBaselines` are known only as bytecode. This risk is reduced by Laypa's own Python extractor (`page_xml/baseline_extractor.py`, 443 lines, MIT, versions 1–3 including start/end orientation), which Laypa already uses to write baselines into PageXML. The Loghi pipeline uses the Java minion instead, which also recomputes the line polygons.
- **Laypa to ONNX.** detectron2 `SemanticSegmentor` with ResNet-50-FPN has not been tested. Laypa's pre- and post-processing (scaling, turning the output into a mask) moves into this system.
- **Loghi-HTR to ONNX.** Keras 3 / TF 2.20, with five BiLSTM layers, through `tf2onnx`: not tested. A known trap from converting models: Loghi loads weights with `skip_mismatch=True`, so a broken conversion can give an untrained network without any error. Every converted model needs a check of its output.
- **Java and Python image processing differ.** Interpolation, rounding and OpenCV versions are not the same, which is why §5 uses tolerances and not identical bytes.
- **Minions nobody here uses.** Language detection (with its own training), PDF output and synthetic page generation still need fixtures and expected output.

## 9. No-gos

- No new models, architectures or decoding with a language model. That is a later step on top of this foundation.
- No rewrite of Laypa or Loghi-HTR, of their APIs, or of the Loghi-HTR GUI.
- No model weights in the repository.
- No promise to test on NVIDIA hardware.

## 10. Glossary seed

| htr-tooling (PAGE terms) | genealogie-transcriptie |
|---|---|
| page image | scan |
| TextLine | regel |
| line id | regel-id |
| Baseline | basislijn |
| line polygon (`Coords`) | polygoon |
| TextRegion | regio (voorgestelde akte) |
| ReadingOrder | leesvolgorde |
| Glyph confidence | zekerheid |

## 11. For the specification

- Exact equivalence tolerances, and how to measure them.
- Which open dataset to use for the fixtures, and the baseline speed measurement.
- Python version, dependencies, packaging and releases.
- How training is driven: inputs, outputs and model provenance.
- The shape of the web service API.
- Model loading and verification, both after an ONNX conversion and at run time.
- The deviation list: where it lives and how it is reported upstream.

The domain model (aggregates, modules) is for the design, after the specification.
