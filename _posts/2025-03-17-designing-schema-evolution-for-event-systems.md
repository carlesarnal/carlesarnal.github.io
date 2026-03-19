---
layout: post
title: "Designing Schema Evolution for Event-Driven Systems"
date: 2025-03-17
description: "Schema evolution is the hardest coordination problem in event-driven architectures. Compatibility modes, common mistakes, and real-world patterns from building a Kafka-based ML pipeline with Apicurio Registry."
tags: [Schema Evolution, Kafka, Apicurio, Event-Driven Architecture]
---

Schema evolution is the hardest coordination problem in event-driven architectures. Producers want to add features. Consumers want stability. You can't deploy all services simultaneously. And unlike REST APIs where you can version the URL, Kafka topics are shared channels where old and new messages coexist.

This article covers what I've learned governing schemas for a [real-time ML pipeline](https://github.com/carlesarnal/reddit-realtime-classification) using Apicurio Registry with JSON Schema.

> Full article with runnable examples: [distributed-deep-dives/schema-evolution-events](https://github.com/carlesarnal/distributed-deep-dives/tree/main/schema-evolution-events)

## The Core Tension

In a monolith, changing a data structure means updating one codebase. In a distributed system with Kafka, changing a schema means coordinating:

- The **producer** that writes the new format
- The **broker** that stores both old and new messages
- Every **consumer** that reads the topic — some of which you may not even control

Schema compatibility modes exist to make this coordination tractable.

## Compatibility Modes in Practice

| Mode | What it means | Safe changes | Unsafe changes |
|------|--------------|--------------|----------------|
| **BACKWARD** | New schema reads old data | Add optional fields, remove fields | Add required fields |
| **FORWARD** | Old schema reads new data | Remove optional fields, add fields with defaults | Remove required fields |
| **FULL** | Both directions | Add/remove optional fields with defaults | Almost everything else |
| **NONE** | No enforcement | Anything | Nothing is checked |

Most teams default to BACKWARD because it's the most intuitive: "my new consumer can read old messages." But FORWARD compatibility matters when you can't guarantee all consumers upgrade before the producer starts writing new-format messages.

## Common Mistakes

### Adding a Required Field

```json
// v1
{"required": ["id", "content"]}

// v2 — BREAKS BACKWARD compatibility
{"required": ["id", "content", "timestamp"]}
```

Old messages don't have `timestamp`. New consumers using v2 will fail to validate them. The fix: make `timestamp` optional with a default, or don't add it to `required`.

### The "Optional Everything" Trap

After getting burned by required fields, teams sometimes make everything optional:

```json
{"required": []}
```

Now your schema is technically compatible with everything, but it's also useless as a contract. A consumer has no guarantee that *any* field will be present. You've traded type safety for deployment convenience.

### Renaming a Field

Renaming `content` to `body` is equivalent to removing `content` and adding `body` — it breaks both directions. If you need to rename, use the expand-and-contract pattern: add the new field, migrate consumers, then remove the old field in a later version.

## Schema Evolution in the Reddit Pipeline

The pipeline has two Kafka topics:

**`reddit-stream`** — simple schema:
```json
{
  "properties": {
    "id": {"type": "string"},
    "content": {"type": "string"}
  },
  "required": ["id", "content"]
}
```

If I want to add `source_subreddit` and `timestamp`, the BACKWARD-compatible evolution is:

```json
{
  "properties": {
    "id": {"type": "string"},
    "content": {"type": "string"},
    "source_subreddit": {"type": "string", "default": "AskEurope"},
    "timestamp": {"type": "string", "format": "date-time"}
  },
  "required": ["id", "content"]
}
```

New fields are optional. Old messages without them are still valid. The Spark consumer handles missing fields gracefully.

**`kafka-predictions`** — more complex, with 7 fields for dual-model output. Adding a third model means adding `model3_flair` and `model3_confidence` as optional fields. The existing consumer ignores fields it doesn't know about.

## When to Break Compatibility

Sometimes you must. Strategies:

1. **Version the topic name** — `reddit-stream-v2`. Clean break, but requires migrating all consumers.
2. **Dual-write** — Produce to both old and new topics during migration. Stop writing to the old topic after all consumers switch.
3. **Expand and contract** — Add new fields (expand), migrate consumers, remove old fields (contract). Works within FULL compatibility.
4. **Schema ID in headers** — Consumers read the schema ID from the message header and branch on version. Complex but allows gradual migration.

## Multi-Schema Governance

Patterns that scale beyond a single team:

- **Producer owns the schema.** The team that writes to a topic owns its schema. Consumers adapt.
- **Compatibility checks in CI.** Validate schema changes against the registry's compatibility rules before merging. This is your first line of defense.
- **Groups for organization.** Apicurio Registry supports groups — use them to organize schemas by domain or team (`payments`, `ml-pipeline`, `user-events`).
- **Schema documentation as contract.** The schema *is* the API contract for event-driven systems. Treat it with the same rigor as an OpenAPI spec.

---

*This is the second article in the [distributed-deep-dives](https://github.com/carlesarnal/distributed-deep-dives) series. The full article includes runnable examples showing valid and invalid schema evolutions, plus scripts for validating compatibility via the Apicurio Registry API.*
