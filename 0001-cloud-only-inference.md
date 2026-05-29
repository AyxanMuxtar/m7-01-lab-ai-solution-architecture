# ADR 0001: Cloud-only inference — no edge pre-filter

## Context

The platform moderates UGC images with legal exposure for prohibited content. The system must decide per-upload within 600 ms. A cloud-GPU inference call is the primary path, but a lightweight edge pre-filter (running on CDN nodes or client devices) could deflect obviously benign traffic before it ever reaches the GPU, reducing cloud cost and latency on the happy path.

## Decision

We run inference exclusively in the cloud on a managed GPU endpoint. No edge model is deployed. Every upload is scored by the full production model before a pass/block/queue decision is made.

The core reason: the legal consequences of a missed detection are asymmetric. A false negative (prohibited content auto-passed because the edge model marked it benign) creates regulatory liability and trust damage that exceeds any compute savings. A production-grade vision model capable of catching subtle violations (e.g. near-nude imagery, CSAM, graphic violence in stylized formats) requires 500M–1B+ parameters. That does not fit in a CDN worker or mobile device within the 600 ms budget. A smaller edge model capable of running at the edge would have materially higher false-negative rates on the adversarial content categories that matter most.

## Alternatives rejected

- **Edge pre-filter (client or CDN) + cloud full-model for flagged traffic:** Rejected because the edge model's false-negative rate on adversarial content is unknown and hard to bound. The edge model becomes a bypassable gate: a motivated bad actor can probe it to find content that passes edge but would be caught by the full model. Maintaining two models in sync also doubles the MLOps surface.
- **Edge-only inference (full model on-device):** Rejected on model size and latency grounds. A model powerful enough for legal-weight decisions at this task is too large to run at p95 < 600 ms on a CDN node without specialized hardware. Client-side inference is additionally unacceptable because the client cannot be trusted to run the model faithfully.
- **Streaming pre-filter (frame-level or metadata-only pass):** Rejected because image metadata (filename, EXIF) is an unreliable signal for content classification and would produce an unacceptable false-negative rate with no latency benefit over just running the model.

## Consequences

- **Committed to:** Every upload incurs a GPU inference call. At 20 req/s peak this is ~1.7M calls/day — GPU cost scales linearly with volume.
- **Committed to:** The cloud serving fleet must auto-scale to handle peak without queuing beyond the latency budget. Requires pre-warmed instances or a fast cold-start path.
- **Benefit:** Single model version to maintain, audit, and explain in legal proceedings. No edge/cloud model drift to track.
- **Benefit:** Threshold logic lives in one place (the decision router), making regulatory compliance changes (e.g. adjusting block thresholds by jurisdiction) a config update, not a model deployment.

## Revisit if

Traffic grows to a scale (e.g. 500+ req/s sustained) where GPU cost becomes a primary business constraint AND a distillation study demonstrates that a small edge model achieves false-negative parity with the production model on the adversarial content test set.
