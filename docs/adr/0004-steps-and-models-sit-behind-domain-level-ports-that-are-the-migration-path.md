# Steps and models sit behind domain-level ports, and those ports are the migration path

Every Step of the chain and every Model sits behind a port in domain terms. Baseline segmentation takes a page image and returns Laypa's class map. Recognition takes Line images and returns text and confidence per Line. A Step takes and returns Pages. Each port has a Docker adapter that runs unmodified Loghi: Laypa and Loghi-HTR for the models, the pinned minion for a Step. The Docker adapters are the Reference and the fallback, and only unmodified Loghi keeps them so. A Step that is not yet translated runs on its minion adapter, so the whole chain works from phase 1. Its default switches to the Python adapter, per release, once it passes Step equivalence or the Chain outcome. The minion adapters stay afterwards, opt-in, for comparing against Loghi and for recording the Reference. See [Model runtime ports, adapters and the minion fallback](https://github.com/pslits/htr-tooling/issues/19).

## Considered Options

- **A network-level model port (tensors in, logits out), with pre-processing, post-processing and CTC decoding shared above it.** Rejected. The Docker adapter would then need logits from Loghi, which is only possible with a hook that replaces Loghi-HTR's `main.py` and monkeypatches `utils.threading.decode_batch_predictions`, as genealogie-transcriptie does today. The Reference would then be patched Loghi, tied to internals that change between releases. The shared processing stays a module inside the ONNX adapter instead, behind an internal seam with its own characterisation tests.
- **Step ports that take and return PageXML files.** Rejected. PageXML would sit in the core of the system, and Pages would be rewritten as files at every Step, as in `inference-pipeline.sh`. Only the minion adapter reads and writes PageXML, so the Page ↔ PageXML round trip must lose nothing a later Step needs.
- **A minion adapter only for a Step that fails both rules.** Rejected. Untranslated Steps need it anyway to run the full chain, and users need it to tell their own input from a difference with Loghi.
- **Switching to the minion automatically when the Python adapter fails.** Rejected. The adapter is chosen per Step by the release default or the user's override. A failure stops the run.

## Consequences

- Under Docker, every call starts a container (about 24 s for Loghi-HTR), because no warm Loghi container is kept. The interface therefore asks callers to pass as many Pages or Lines per call as they can.
- Recognition's batching (groups of `batch_size` in the order given, padded with −10) is part of the port's interface, because batch composition changes the text. Both adapters honour it.
- A switch of a Step's default changes the output, so it is a version bump and goes in the release notes.
