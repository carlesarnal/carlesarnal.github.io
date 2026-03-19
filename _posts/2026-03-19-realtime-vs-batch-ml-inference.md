---
layout: post
title: "Real-Time vs. Batch ML Inference: Engineering Tradeoffs You Won't Find in Tutorials"
date: 2026-03-19
description: "The inference spectrum isn't binary. Micro-batch, dual-model architectures, PySpark UDF tracing challenges, and what changes at 100x scale — lessons from building a real-time classification pipeline."
tags: [ML Inference, Spark, Kafka, Distributed Systems, Observability]
---

Most articles frame ML inference as a binary choice: real-time or batch. In practice, it's a spectrum with five or six distinct points, each with different infrastructure requirements, failure modes, and cost profiles. This article is based on building a [real-time Reddit classification pipeline](https://github.com/carlesarnal/reddit-realtime-classification) that sits in the middle of that spectrum.

> Full article with runnable examples: [distributed-deep-dives/realtime-vs-batch-inference](https://github.com/carlesarnal/distributed-deep-dives/tree/main/realtime-vs-batch-inference)

## The Inference Spectrum

| Mode | Latency | Examples | Infrastructure |
|------|---------|----------|----------------|
| True real-time | < 100ms | TF Serving, Triton, KServe | Dedicated model servers, GPU |
| Near-real-time | 100ms - 5s | Kafka Streams, Flink | Stream processors, stateful |
| Micro-batch | 10s - minutes | Spark Structured Streaming | Batch-oriented, simpler state |
| Batch | minutes - hours | Spark batch, Airflow | Cheapest, highest throughput |

The reddit pipeline uses **micro-batch** via Spark Structured Streaming with a 1-minute trigger interval. This was deliberate — Reddit data is inherently non-real-time (posts, not clicks), there's no user-facing latency requirement, and micro-batch gives us Spark's DataFrame APIs for feature engineering with simpler failure handling than true streaming.

## Dual-Model Inference

The pipeline runs two fundamentally different models on every post:

- **DistilBERT (Transformer)** — fine-tuned for semantic understanding, higher accuracy, slower
- **scikit-learn (TF-IDF + LSA + classifier)** — bag-of-words features, fast, less capable with nuance

Why two models? **Agreement rate as a reliability proxy.** In a streaming pipeline, you don't have ground truth labels. When both models agree on a classification, confidence is higher. When they disagree, it signals ambiguity — either in the post or in one model's weakness.

The Quarkus consumer tracks the agreement rate as a Prometheus gauge, per-flair confidence gaps, and stores predictions from both models. This gives us concrete data for deciding whether the Transformer's accuracy gain justifies its 10x resource cost.

## The Hard Parts Nobody Talks About

### PySpark UDF Serialization

You can't pass a Transformer model directly to a PySpark UDF — `AutoModelForSequenceClassification` objects aren't picklable. The solution: load models as module-level globals on the driver, and let PySpark serialize the UDF function (which captures the model references via closure). This works because Spark's local mode and single-executor setups share the same process, but it breaks in multi-node clusters without proper model distribution.

### OTel Tracing in UDFs

OpenTelemetry spans created inside a PySpark UDF don't inherit the parent trace context. UDFs execute on executors, not the driver, so the OTel context doesn't propagate across the driver/executor boundary. The solution: create independent spans with a `reddit.post.id` attribute for correlation, rather than trying to maintain a parent-child span hierarchy.

```python
with tracer.start_as_current_span("dual-model-inference", attributes={
    "reddit.post.id": post_id,
}) as span:
    with tracer.start_as_current_span("transformer-inference"):
        # ... transformer prediction
    with tracer.start_as_current_span("sklearn-inference"):
        # ... sklearn prediction
    span.set_attribute("prediction.models_agree", t_flair == sk_flair)
```

### Backpressure via maxOffsetsPerTrigger

What happens when inference takes longer than the trigger interval? Without limits, Spark pulls all available messages from Kafka, loads them into memory, and potentially OOMs. The fix is `maxOffsetsPerTrigger=100` — each micro-batch processes at most 100 messages. If the queue grows, Spark processes it incrementally. Consumer lag increases but the application stays alive.

### The Checkpoint Trap

Spark Structured Streaming uses checkpoints for exactly-once semantics. But if you change the UDF output schema (e.g., adding a third model's predictions), Spark can't deserialize the old checkpoint state. You have to wipe checkpoints and reprocess from the earliest offset — which means temporary duplicate processing.

## DLQ Design for ML Inference

Failed inference shouldn't silently drop messages. The pipeline wraps the UDF in try/except and returns a `__dlq: true` flag on failure. The output DataFrame is split into success and DLQ streams with separate Kafka sinks:

```python
is_dlq = get_json_object(col("value"), "$.__dlq") == "true"
success_df = predicted_df.filter(~is_dlq)
dlq_df = predicted_df.filter(is_dlq)
```

This is pragmatic but not ideal — a proper implementation would use separate error schemas for the DLQ topic. It works because the DLQ consumer (if any) can filter on the flag and still access the original content + error message for debugging.

## What I'd Change at 100x Scale

1. **Separate model serving.** Move the Transformer to a dedicated Triton or TF Serving endpoint. Call it via HTTP from Spark. Keep sklearn inline — it's fast enough.
2. **GPU nodes.** The Transformer currently runs on CPU. At scale, GPU inference is 10-50x faster per prediction.
3. **Replace Spark with Flink.** Flink has true per-record streaming, better state management, and lower latency. Spark Structured Streaming's micro-batch model adds unnecessary delay for high-volume scenarios.
4. **Feature store.** Pre-compute TF-IDF vectors and cache them. Avoid recomputing features for posts that appear in multiple queries.
5. **A/B testing infrastructure.** Currently both models run on every post. At scale, you'd want traffic splitting and statistically rigorous comparison.

## The Cost-Accuracy Decision Framework

The real question isn't "which model is better" — it's "when is the simpler model good enough?"

- If agreement rate is consistently > 95%, the sklearn model is sufficient for most use cases. Drop the Transformer and save 10x on compute.
- If agreement rate drops below 80% for specific flairs, investigate those categories. The disagreement is the signal.
- The infrastructure cost of running both models is fixed. The real cost is engineering complexity — maintaining two model training pipelines, two sets of hyperparameters, two deployment processes.

Track the agreement rate over time. If it's stable, you've learned something about your problem space. If it's dropping, you're seeing model drift — and that's worth investigating regardless of which inference mode you use.

---

*This is the third article in the [distributed-deep-dives](https://github.com/carlesarnal/distributed-deep-dives) series. The full article includes the complete UDF implementation, Spark configuration examples, and a hypothetical refactor showing the migration from inline inference to model serving endpoints.*
