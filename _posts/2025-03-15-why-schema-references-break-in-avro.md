---
layout: post
title: "Why Schema References Break in Avro (And How to Design Around It)"
date: 2025-03-15
description: "Avro schema references promise reusable types across schemas. In practice, they break in subtle ways — circular dependencies, version pinning drift, namespace collisions. Here's what I learned contributing to Apicurio Registry."
tags: [Avro, Schema Registry, Apicurio, Distributed Systems]
---

Avro schema references promise reusable types across schemas. In practice, they break in subtle ways that won't show up until you're trying to deploy on a Friday afternoon. This article is based on what I learned contributing to [Apicurio Registry](https://www.apicur.io/registry/) and building schema governance for a real-time ML pipeline.

> Full article with runnable examples: [distributed-deep-dives/schema-references-avro](https://github.com/carlesarnal/distributed-deep-dives/tree/main/schema-references-avro)

## The Promise

Avro supports named types — records, enums, and fixed types that can be referenced by name across schemas. This seems like a great idea for DRY (Don't Repeat Yourself) schema design:

```json
{
  "type": "record",
  "name": "Order",
  "namespace": "com.example",
  "fields": [
    {"name": "id", "type": "string"},
    {"name": "customer", "type": "com.example.Customer"}
  ]
}
```

The `Customer` type is defined elsewhere and referenced by its fully qualified name. Clean. Reusable. What could go wrong?

## What Goes Wrong

### 1. Circular References

Two schemas that reference each other create a chicken-and-egg problem during registration:

```json
// Order references Customer
{"type": "record", "name": "Order", "fields": [
  {"name": "customer", "type": "com.example.Customer"}
]}

// Customer references Order
{"type": "record", "name": "Customer", "fields": [
  {"name": "lastOrder", "type": ["null", "com.example.Order"]}
]}
```

You can't register `Order` without `Customer` already existing, and vice versa. Confluent Schema Registry and Apicurio handle this differently, but neither makes it seamless.

### 2. Version Pinning Drift

When Schema A references Schema B version 1, and Schema B evolves to version 2, Schema A still points to v1. This creates a silent contract mismatch — consumers using the latest Schema B see different types than what Schema A promises.

### 3. Registry Resolution Differences

Confluent uses a `references` array in the registration API with global subject resolution. Apicurio uses artifact references with group-based resolution. If you're migrating between registries or using a multi-registry setup, references that work in one may not resolve in the other.

### 4. Namespace Collisions

When two independent schemas define the same fully qualified name (`com.example.Address`), Avro's name-based resolution can't distinguish between them. The registry may accept both, but consumers get the wrong type at deserialization time.

## What I Learned from Apicurio Registry

Contributing to Apicurio's schema handling taught me several things:

**Content canonicalization matters.** When comparing schemas for compatibility, the registry normalizes schemas to a canonical form. For Avro, this means sorting fields, resolving aliases, and expanding named type references. A schema that looks different in JSON can be semantically identical after canonicalization.

**Reference resolution during compatibility checks is expensive.** Checking whether Schema A v2 is backward-compatible with Schema A v1 requires resolving all references in both versions, then comparing the fully expanded schemas. If references form a deep tree, this becomes a graph traversal problem.

**JSON Schema references are simpler.** JSON Schema uses `$ref` with URI resolution — a well-defined standard. Avro uses name-based resolution with namespace scoping, which is more implicit and harder to debug. This is one reason I chose JSON Schema for the [reddit-realtime-classification](https://github.com/carlesarnal/reddit-realtime-classification) pipeline.

## Practical Recommendations

1. **Design schemas as self-contained where possible.** Inline shared types rather than referencing them across schemas. The duplication is worth the independence.

2. **Use explicit version pinning for references.** Never reference "latest" — always pin to a specific version and update deliberately.

3. **Prefer flat schemas for Kafka topics.** Deeply nested references add complexity with minimal benefit for event schemas that are typically read by one consumer group.

4. **Register leaf schemas first.** If you must use references, register in bottom-up order: shared types first, then schemas that depend on them.

5. **Test schema evolution with references in CI.** Use the registry's compatibility API to validate changes before deployment. A breaking change in a leaf schema can cascade to every schema that references it.

6. **Consider JSON Schema instead.** If you don't need Avro's binary encoding efficiency, JSON Schema's `$ref` with URI resolution is simpler and better tooled.

---

*This is the first article in the [distributed-deep-dives](https://github.com/carlesarnal/distributed-deep-dives) series. The full article includes runnable examples demonstrating each failure mode and the recommended patterns to avoid them.*
