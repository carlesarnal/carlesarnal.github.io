---
layout: post
title: "Building a RAG Chatbot with Schema Registry as the Knowledge Backend"
date: 2025-06-10
description: "Using Apicurio Registry to store versioned prompt templates as registry artifacts, then feeding them to a RAG chatbot. A unique intersection of schema governance and LLM tooling that nobody else has explored."
tags: [RAG, LangChain4j, Ollama, Apicurio, LLM]
---

Prompt templates are code. They have versions, they break when changed, they need rollback capability, and different versions produce different outputs. Yet most teams store them as hardcoded strings or, at best, in a config file. Schema registries already solve versioning, governance, and compatibility for schemas — why not extend the pattern to prompts?

This article is based on [support-chat](https://github.com/Apicurio/apicurio-registry/tree/main/support-chat), a RAG-powered chatbot I built as part of Apicurio Registry that uses the registry itself as its prompt management backend.

> Full article with examples: [distributed-deep-dives/rag-with-schema-registry](https://github.com/carlesarnal/distributed-deep-dives/tree/main/rag-with-schema-registry)

## The Architecture

The system has three components running in Docker Compose:

- **Apicurio Registry** — Stores two artifact types: `PROMPT_TEMPLATE` (versioned prompts with variable substitution) and `MODEL_SCHEMA` (model metadata searchable by capability)
- **Ollama** — Local LLM provider running llama3.2 for chat and nomic-embed-text for embeddings
- **Quarkus app** — LangChain4j integration that orchestrates RAG retrieval, prompt rendering, and multi-turn conversation management

The key insight: the registry isn't just storing schemas for Kafka topics — it's a general-purpose versioned artifact store. Prompt templates are artifacts that benefit from the same governance patterns: versioning, rollback, compatibility rules, and audit trails.

## Prompt Templates as Registry Artifacts

Instead of hardcoding the system prompt:

```java
String systemPrompt = "You are a helpful assistant for Apicurio Registry...";
```

The chatbot fetches it from the registry:

```java
ApicurioPromptTemplate template = promptRegistry.getTemplate("apicurio-support-system-prompt", version);
String rendered = template.apply(Map.of("artifactTypes", supportedTypes));
```

This means I can update the system prompt — changing tone, adding context, adjusting behavior — without redeploying the application. I can A/B test prompt versions by passing different version parameters. And if a prompt change makes the chatbot worse, I roll back to the previous version in the registry.

## RAG Pipeline

At startup, the `DocumentIngestionService` asynchronously fetches 12 pages of Apicurio Registry documentation, parses the HTML with JSoup, chunks the text (500 tokens, 50 token overlap), and embeds it with nomic-embed-text. The embedding store provides context for each question:

```java
ContentRetriever retriever = EmbeddingStoreContentRetriever.builder()
    .embeddingStore(embeddingStore)
    .embeddingModel(embeddingModel)
    .maxResults(5)
    .minScore(0.6)
    .build();
```

The `minScore(0.6)` threshold is critical — without it, the retriever returns marginally relevant chunks that confuse the LLM into hallucinating. Better to return fewer, higher-quality results.

## What I Learned

**Embedding model choice matters more than LLM choice for RAG quality.** A better embedding model retrieves more relevant chunks, which gives the LLM better context. Switching from a generic embedding model to nomic-embed-text improved answer relevance more than switching between LLM models.

**Chunk size is the most impactful hyperparameter.** Too small (100 tokens) and you lose context. Too large (1000 tokens) and you dilute the relevant information. 500 tokens with 50 token overlap was the sweet spot for technical documentation.

**Prompt versioning prevents "it worked yesterday" debugging.** When the chatbot's behavior changes, I can diff prompt versions in the registry. This is the same benefit schema versioning gives you for data contracts.

**This pattern generalizes beyond chatbots.** Any system that uses prompts — summarizers, classifiers, code generators — benefits from versioned prompt management. The registry already has the infrastructure for versioning, compatibility rules, and access control.

---

*Full article with Docker Compose setup and prompt template examples: [distributed-deep-dives/rag-with-schema-registry](https://github.com/carlesarnal/distributed-deep-dives/tree/main/rag-with-schema-registry)*
