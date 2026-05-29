# JUSTIFICATION — Scenario C: Image Moderation (UGC Platform)

## Serving pattern: Online inference with a hybrid human-in-the-loop fallback

**Pattern chosen: Online (synchronous) with async human escalation.**

The scenario specifies 2 uploads/second average, 20/second peak, and a p95 latency budget of 600 ms. That is a throughput level — not a high-volume streaming problem — where a stateless, per-request inference call is both sufficient and operationally simpler than a streaming pipeline. A Kafka-style streaming system (like Flink or Spark Streaming) adds broker latency, consumer lag, and operational overhead that buys nothing here: there are no multi-event joins needed and no aggregation window required to produce a score. Batch inference is ruled out immediately — the scenario requires moderation before the upload reaches other users, which means the decision must happen inside the request lifecycle, not overnight.

The "ambiguous → human review" path is *asynchronous*: the upload is held in a pending state, the user sees a "processing" status, and the human verdict resolves it within a defined SLA (e.g. 4 hours). This is not a fallback to a slower serving mode; it is a deliberate three-way routing decision by the classifier itself.

## Where inference runs: Cloud, single-region

All inference runs in the cloud (e.g. AWS GPU instances or a managed endpoint like SageMaker). Edge inference is rejected — see ADR 0001 for full reasoning. The short version: the model needs to be large enough to catch subtle prohibited content (legal weight), and edge devices cannot run a production-grade vision model within this latency budget. A hybrid approach where a lightweight edge filter pre-passes obviously benign images is plausible as a future optimization but is not in scope for v1; the cloud path already fits within 600 ms.

## Latency, throughput, and cost targets

| Dimension | Target | Role |
|---|---|---|
| Latency | p95 < 600 ms end-to-end | **Hard budget** — user is waiting for upload confirmation |
| Throughput | 20 req/s peak, sustained | **Primary optimization** — must not drop requests at peak |
| Cost | Minimize GPU-hours on human-review path | **Secondary optimization** — only pay for GPU on clear-cut cases |

The latency budget of 600 ms is generous compared to fraud scoring (80 ms) because the user is uploading, not staring at a payment screen. This allows a larger, more accurate model. The two dimensions we optimize for are throughput (auto-scaling the serving fleet to handle 20 req/s peaks without queuing) and cost (routing unambiguous cases through the model quickly and routing ambiguous cases to cheaper human reviewers rather than a second, heavier model pass). We accept higher per-image cost on ambiguous cases because the legal risk of a wrong auto-decision outweighs the compute cost.

## Fallback behavior

**Model unavailable (serving outage):** Route all uploads to the human review queue. Do not auto-pass. The system defaults to conservative behavior because the cost of letting prohibited content through is higher than the cost of review queue backlog.

**Model wrong (post-hoc):** Human review verdicts flow into the label store and are included in the next training run. If a systematic error class is detected (monitoring flags a spike in human overrides on a specific content type), the drift alert triggers an emergency fine-tune cycle before the next scheduled retraining.

**Human review backlog (SLA breach):** Alert the on-call team. If backlog exceeds the SLA window, temporarily lower the high-confidence block threshold to auto-block more aggressively rather than auto-passing. The upload stays in "pending" state; the user is notified of the delay. We never auto-pass under capacity pressure.
