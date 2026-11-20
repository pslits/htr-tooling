# Translate loghi-tooling's Java for the phase-1 layout code, not Laypa's extractor

The brief (§8) expected Laypa's Python `page_xml/baseline_extractor.py` to take the risk out of baseline extraction. It doesn't. That file ports only the first step of `MinionExtractBaselines` (mask → Baselines). It differs from the Java (the per-column minimum height is capped at 1 instead of 3, it runs on the network's resized output, and its Line polygon is the Baseline's bounding box). It has no Line id mapping, no Region assignment and no Line polygon recalculation. Those exist only in the Java, and its source is public under MIT. So for the four phase-1 minions we translate the Java: `MinionExtractBaselines` and every `LayoutProc`/`PageUtils` routine those minions use (Line polygon recalculation, `getBinaryLineStrip`, `interlineMedian`, `reorderRegions`, `attachTextLines`, `splitBaselines`). The translation is faithful in behaviour, not in structure. It is written on the domain model, but it keeps everything that affects results exactly: constants, truncation and rounding, the processing order and the OpenCV operations. Each translated function cites the Java file and line at the reference tag.

## Considered Options

- **Laypa's extractor for mask → Baselines, and the Java for the rest.** Rejected. It saves only the easy part, and its differences would have to be patched or listed as deviations.
- **An implementation of our own, tuned to the tolerances.** Rejected. It gives up the head start of the reference and risks a long tuning loop on the Line polygons.

## Consequences

- `ExtractBaselines`' no-op flags are fixed and listed as deviations. `-minimum_interlinedistance` takes effect. `-margin` takes effect with an effective default of 0, so output with default flags matches the reference.
- Every path of `ExtractBaselines`, `-split_baselines` included, is translated in phase 1. Paths outside the default need their own fixtures.
- The NOTICE carries loghi-tooling's copyright (© rvankoert) for the translated code. Laypa's copyright is needed only for the Laypa pre- and post-processing that is reused.
