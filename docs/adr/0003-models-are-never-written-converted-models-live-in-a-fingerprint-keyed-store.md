# Models are never written; Converted models live in a store keyed by the Model's fingerprint

htr-tooling never writes into a Model (a Loghi-HTR model folder, or a Laypa config with its weights). A Converted model (the `.onnx` file plus its conversion record) lives in htr-tooling's own Model store in the user data dir, overridable, and is found by a per-file SHA-256 fingerprint of the Model's files. At every load, htr-tooling re-hashes the Model and looks that fingerprint up, so an ONNX file can never run against a Model it was not verified for. The Docker runtime hands Loghi a disposable per-process copy of the Model, because Loghi-HTR writes into its model folder: it writes `tokenizer.json` next to a lone `charlist.txt`, and its `skip_mismatch` fallback replaces `model.keras` and keeps the original as `model.keras.old`, both without error handling and the second without any message. htr-tooling logs those writes and discards them. Callers such as genealogie-transcriptie pin a checksum over the Model's files, so a Model that changes under them breaks their provenance, and the owner's own folders already show such a silent re-save. See [Model loading and verification](https://github.com/pslits/htr-tooling/issues/14).

## Considered Options

- **The `.onnx` as a sidecar inside the Model folder.** Rejected. It writes into folders meant to be immutable, fails on read-only folders, and is lost anyway when genealogie-transcriptie's `gt model plaats` copies only the four model files.
- **The user names the `.onnx` path explicitly.** Rejected. Every caller and the Docker fallback would have to keep two paths per model in step, and nothing would tie the ONNX file to the Model it came from.
- **Mount the Model read-only for Docker.** Rejected. `generic_new_17_2023_05_25_4channel` fresh from SURFdrive needs both writes under Loghi 2.3.x, so the Reference could not run on the recommended model.
- **Mount the Model writable, as `inference-pipeline.sh` does.** Rejected. A silent re-save would change the Model, and the checksums callers rely on, behind their backs.

## Consequences

- Copying a Model to another machine does not bring its Converted model. It has to be converted again there. Identical copies on one machine share one conversion.
- Each Docker process copies about 400 MB per Model, roughly a second against Loghi-HTR's ~24 s container start.
- Each load re-hashes the Model (roughly a second per model), once per process.
