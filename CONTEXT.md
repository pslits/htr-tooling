# htr-tooling

The tooling around Laypa and Loghi-HTR: everything between finding baselines on a page image and reading the text on its lines. PAGE is a format at the edge; this glossary uses its own terms.

## Language

### The page and what is found on it

**Page**:
One page image together with everything found on it: its regions, lines and reading order.
_Avoid_: PcGts, PageXML (that is the file format), scan (genealogie-transcriptie's term)

**Region**:
An area of a page that groups lines, such as a paragraph or a margin note.
_Avoid_: TextRegion (PAGE element), regio, block

**Line**:
One line of handwriting on a page.
_Avoid_: TextLine (PAGE element), regel, text line

**Baseline**:
The polyline that the letters of a line stand on.
_Avoid_: basislijn

**Line polygon**:
The outline that encloses a line's handwriting.
_Avoid_: Coords (PAGE element), polygoon, bounding polygon

**Character**:
One recognised character on a line, with its position, confidence and alternatives.
_Avoid_: Glyph (PAGE element), TextEquiv

**Reading order**:
The order in which the regions and lines of a page are meant to be read.
_Avoid_: ReadingOrder (PAGE element), leesvolgorde

### Checking against Loghi

**Reference**:
The recorded output of the real Loghi chain on the fixtures, kept as a Page snapshot after each step, that htr-tooling is checked against.
_Avoid_: golden output, expected output, baseline (that is a Line's polyline)

**Step equivalence**:
Whether one step, given the Reference's input for that step, produces the Reference's output for that step within the agreed tolerances.
_Avoid_: step test, per-step match

**Chain outcome**:
The text quality of a whole chain (in reading order, against ground truth) compared with that of the Reference chain on the same fixtures; it decides whether a step that fails Step equivalence is still accepted.
_Avoid_: end-to-end score, fallback test

**Deviation**:
A deliberate, recorded difference from the Reference or from what a replaced minion promises (its flags, defaults and exit codes): either a fixed Loghi bug or a change that the Chain outcome shows is no worse. A bug in code htr-tooling doesn't replace is not a Deviation, only something to report upstream.
_Avoid_: divergence, difference, known issue

### Models

**Model**:
A published Laypa or Loghi-HTR model as Loghi itself reads it: a Loghi-HTR model folder, or a Laypa config with its weights. Named by its path; htr-tooling never writes into it.
_Avoid_: model folder (when the identity is meant), Loghi model, eigen model / generiek model (genealogie-transcriptie's terms for who made it)

**Converted model**:
An ONNX version of a Model, kept apart from it, together with the fingerprint of the Model's files it was made from and the evidence that it was verified.
_Avoid_: ONNX model, exported model

**Model store**:
The place where htr-tooling keeps Converted models, found by the fingerprint of their Model.
_Avoid_: model cache, registry
