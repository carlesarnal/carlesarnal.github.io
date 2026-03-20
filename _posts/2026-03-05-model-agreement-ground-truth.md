---
layout: post
title: "Model Agreement as a Proxy for Ground Truth in Streaming ML"
date: 2026-03-05
description: "When you deploy ML models in a streaming pipeline, you don't have labels. Dual-model agreement rate, disagreement taxonomy, confidence calibration, and drift detection without ground truth."
tags: [ML, Model Monitoring, Streaming, Prometheus, Distributed Systems]
---

In batch ML, you measure accuracy against labeled test sets. In streaming, you don't have labels. Posts arrive, get classified, and are consumed — there's no feedback loop telling you if the prediction was correct. So how do you know your model is working?

This article explores what I learned running two fundamentally different models in parallel on every message in a [real-time classification pipeline](https://github.com/carlesarnal/reddit-realtime-classification), and how agreement rate serves as a proxy for the ground truth you don't have.

> Full article with examples: [distributed-deep-dives/model-agreement-ground-truth](https://github.com/carlesarnal/distributed-deep-dives/tree/main/model-agreement-ground-truth)

## The Dual-Model Architecture

The pipeline runs a **DistilBERT Transformer** (semantic understanding, higher accuracy, slower) and a **scikit-learn TF-IDF + LSA classifier** (bag-of-words, fast, simpler) on every Reddit post. Both predict one of 13 flair categories.

The key insight from ADR-001: *"A single model gives us predictions but no way to gauge trustworthiness. If the model is wrong, we have no signal to detect it without ground truth labels."*

When two architecturally different models agree, it's a stronger signal than either alone. A bag-of-words model and a semantic model reaching the same conclusion means the signal is in both surface-level features and deeper meaning.

## Disagreement Taxonomy

Not all disagreements are equal. Four quadrants:

| | sklearn confident | sklearn uncertain |
|---|---|---|
| **Transformer confident** | If they agree: highest trust. If they disagree: genuine ambiguity or systematic bias. | Transformer is likely right. Investigate why sklearn struggles with this category. |
| **Transformer uncertain** | sklearn is likely right. The Transformer may need more training data for this pattern. | Hard example. Consider routing to DLQ for human review. |

The pipeline tracks these zones via the `ModelUncertaintyZoneResource`, and the Grafana dashboard visualizes the distribution over time.

## Detecting Drift Without Labels

If agreement rate drops over time, something changed. Either:
- **Data distribution shifted** — new topics, different writing styles, seasonal patterns
- **One model is degrading** — perhaps the TF-IDF vocabulary doesn't cover new terms

Agreement rate trend is an early warning for drift. You can set a Prometheus alert:

```yaml
- alert: ModelAgreementDrop
  expr: model_agreement_rate < 0.75
  for: 15m
  annotations:
    summary: "Model agreement rate below 75% for 15 minutes"
```

This fires before traditional metrics would catch it — because traditional metrics require labels you don't have.

## When to Stop Running Both Models

The dual-model architecture is a diagnostic tool, not a permanent requirement:

- **Agreement > 95% consistently**: The simpler model suffices. Drop the Transformer, save 10x compute.
- **Agreement < 80%**: You're getting valuable disagreement signal. Keep both.
- **Agreement varies by category**: Consider per-category model routing — use the Transformer only for categories where sklearn struggles.

The meta-lesson: running both models isn't about getting better predictions. It's about understanding your problem space well enough to make informed architecture decisions.

## Beyond Classification

This pattern generalizes to any prediction task where you can run two architecturally different models:
- **NER**: Rule-based extractor vs. neural NER
- **Anomaly detection**: Statistical (z-score) vs. ML (isolation forest)
- **Recommendations**: Collaborative filtering vs. content-based

The requirement: models must be *architecturally* different. Two neural networks with different hyperparameters will agree on the same failure modes. A statistical model and a neural model fail differently — that's what makes disagreement informative.

---

*Full article with agreement tracking code, Prometheus alert rules, and disagreement analysis scripts: [distributed-deep-dives/model-agreement-ground-truth](https://github.com/carlesarnal/distributed-deep-dives/tree/main/model-agreement-ground-truth)*
