---
layout: default
title: Home
---

<div class="hero">
  <h1>Carles Arnal</h1>
  <p class="subtitle">
    Principal Software Engineer at IBM, based in Barcelona. I work on distributed systems, schema governance, and ML pipelines. Core contributor to <a href="https://www.apicur.io/registry/">Apicurio Registry</a> — an open-source schema and API registry for event-driven architectures.
  </p>
  <div class="hero-meta">
    <span>IBM</span>
    <span>Barcelona</span>
    <span><a href="https://github.com/carlesarnal">github.com/carlesarnal</a></span>
  </div>
</div>

<h2 class="section-title">What I Work On</h2>

<div class="expertise-grid">
  <div class="expertise-item">
    <h3>Schema Governance</h3>
    <p>Core contributor to Apicurio Registry. Schema validation, compatibility rules, content canonicalization, and API design for multi-format registries (Avro, JSON Schema, Protobuf, OpenAPI).</p>
  </div>
  <div class="expertise-item">
    <h3>Distributed Systems</h3>
    <p>Kafka-based event pipelines, Spark Structured Streaming, distributed tracing with OpenTelemetry, failure handling patterns (DLQ, backpressure, circuit breakers).</p>
  </div>
  <div class="expertise-item">
    <h3>ML Pipelines</h3>
    <p>Real-time inference with dual-model architectures, model agreement tracking, Prometheus/Grafana observability, and schema-governed model metadata.</p>
  </div>
  <div class="expertise-item">
    <h3>Cloud-Native Java</h3>
    <p>Quarkus, SmallRye Reactive Messaging, Kubernetes operators (Strimzi, Spark Operator), health probes, and production deployment patterns.</p>
  </div>
</div>

<h2 class="section-title">Deep Dives</h2>

<p style="color: var(--text-secondary); margin-bottom: 1.5rem;">Technical articles rooted in real engineering experience — not tutorials. Each one references actual code, architecture decisions, and production lessons.</p>

<div class="card-grid">
{% assign sorted_posts = site.posts | sort: 'date' | reverse %}
{% for post in sorted_posts %}
  <div class="card">
    <h3><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h3>
    <p>{{ post.description }}</p>
    <div class="card-tags">
      {% for tag in post.tags %}
      <span class="tag">{{ tag }}</span>
      {% endfor %}
    </div>
  </div>
{% endfor %}
</div>

<h2 class="section-title">Featured Projects</h2>

<div class="card-grid">
  <div class="card">
    <h3><a href="https://github.com/carlesarnal/reddit-realtime-classification">reddit-realtime-classification</a></h3>
    <p>End-to-end ML pipeline: Reddit API &rarr; Kafka (Strimzi) &rarr; Spark Structured Streaming (dual-model inference) &rarr; Quarkus consumer. Full observability with OpenTelemetry, Prometheus, and Grafana. Dead letter queues, backpressure, and 5 architecture decision records.</p>
    <div class="card-tags">
      <span class="tag">Kafka</span>
      <span class="tag">Spark</span>
      <span class="tag">Quarkus</span>
      <span class="tag">ML</span>
      <span class="tag">Kubernetes</span>
    </div>
  </div>
  <div class="card">
    <h3><a href="https://github.com/carlesarnal/distributed-deep-dives">distributed-deep-dives</a></h3>
    <p>Technical deep dives into distributed systems, schema evolution, and ML inference tradeoffs. Each article includes runnable examples and references real engineering work.</p>
    <div class="card-tags">
      <span class="tag">Writing</span>
      <span class="tag">Distributed Systems</span>
      <span class="tag">Schema Evolution</span>
    </div>
  </div>
  <div class="card">
    <h3><a href="https://github.com/carlesarnal/apicurio-registry-support-chat">apicurio-registry-support-chat</a></h3>
    <p>RAG-powered support chatbot for Apicurio Registry using LangChain4j, Ollama, and prompt templates stored as versioned registry artifacts.</p>
    <div class="card-tags">
      <span class="tag">RAG</span>
      <span class="tag">LangChain4j</span>
      <span class="tag">Ollama</span>
    </div>
  </div>
</div>
