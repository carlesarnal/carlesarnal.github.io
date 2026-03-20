---
layout: default
title: Projects
permalink: /projects/
---

<div class="hero">
  <h1>Projects</h1>
  <p class="subtitle">Open-source projects spanning distributed systems, ML pipelines, schema governance, and cloud-native Java.</p>
</div>

<h2 class="section-title">Flagship</h2>

<div class="project-grid" style="margin-bottom: 3rem;">
  <div class="project-card">
    <h3><a href="https://github.com/carlesarnal/reddit-realtime-classification">reddit-realtime-classification</a></h3>
    <p>Real-time Reddit post classification pipeline: Reddit API &rarr; Kafka (Strimzi) &rarr; Spark Structured Streaming with dual-model inference (DistilBERT + scikit-learn) &rarr; Quarkus consumer. Includes OpenTelemetry distributed tracing, Prometheus + Grafana dashboards, dead letter queues, backpressure patterns, and architecture decision records.</p>
    <span class="project-lang jupyter">Jupyter Notebook / Python / Java</span>
  </div>
  <div class="project-card">
    <h3><a href="https://github.com/carlesarnal/distributed-deep-dives">distributed-deep-dives</a></h3>
    <p>Technical deep dives into distributed systems engineering. Articles on schema references in Avro, schema evolution for event-driven systems, and real-time vs. batch ML inference tradeoffs. Each includes runnable examples.</p>
  </div>
</div>

<h2 class="section-title">Schema Governance & Apicurio</h2>

<div class="project-grid" style="margin-bottom: 3rem;">
  <div class="project-card">
    <h3><a href="https://github.com/Apicurio/apicurio-registry/tree/main/support-chat">apicurio-registry/support-chat</a></h3>
    <p>RAG-powered support chatbot for Apicurio Registry using LangChain4j, Ollama, and prompt templates stored as versioned registry artifacts.</p>
    <span class="project-lang java">Java</span>
  </div>
  <div class="project-card">
    <h3><a href="https://github.com/carlesarnal/apicurio-registry-canary-application">apicurio-registry-canary-application</a></h3>
    <p>Canary application for Apicurio Registry — continuous health monitoring and smoke testing.</p>
    <span class="project-lang java">Java</span>
  </div>
  <div class="project-card">
    <h3><a href="https://github.com/carlesarnal/apicurio-registry-client-demo">apicurio-registry-client-demo</a></h3>
    <p>Demo application for the Apicurio Registry client SDK.</p>
    <span class="project-lang java">Java</span>
  </div>
  <div class="project-card">
    <h3><a href="https://github.com/carlesarnal/kroxylicious-schema-validation">kroxylicious-schema-validation</a></h3>
    <p>Schema validation filter for Kroxylicious Kafka proxy — enforce schema compliance at the broker level.</p>
  </div>
</div>

<h2 class="section-title">Quarkus & Cloud-Native</h2>

<div class="project-grid" style="margin-bottom: 3rem;">
  <div class="project-card">
    <h3><a href="https://github.com/carlesarnal/dynamic-config-properties">dynamic-config-properties</a></h3>
    <p>Dynamic configuration properties for Quarkus applications.</p>
    <span class="project-lang java">Java</span>
  </div>
  <div class="project-card">
    <h3><a href="https://github.com/carlesarnal/keycloak-admin-app">keycloak-admin-app</a></h3>
    <p>Keycloak administration web application.</p>
    <span class="project-lang html">HTML</span>
  </div>
</div>

<h2 class="section-title">Data & Visualization</h2>

<div class="project-grid">
  <div class="project-card">
    <h3><a href="https://github.com/carlesarnal/data-visualization">data-visualization</a></h3>
    <p>Data visualization notebooks analyzing Barcelona bike-sharing (Bicing) data.</p>
    <span class="project-lang html">HTML</span>
  </div>
  <div class="project-card">
    <h3><a href="https://github.com/carlesarnal/currency-exchange-graph">currency-exchange-graph</a></h3>
    <p>Currency exchange rate graph visualization in Java.</p>
    <span class="project-lang java">Java</span>
  </div>
</div>
