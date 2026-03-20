---
layout: post
title: "MCP Servers for Domain-Specific AI Tooling: Lessons from Apicurio Registry"
date: 2025-12-05
description: "Building an MCP server that wraps a schema registry for AI agent use. Designing the tool surface, security considerations, and patterns for wrapping any REST API as an MCP server."
tags: [MCP, AI Agents, Apicurio, Schema Registry, LLM Tooling]
---

AI agents are powerful reasoners but they lack domain-specific knowledge. An LLM knows what a schema registry is in theory, but it can't query *your* registry, check *your* compatibility rules, or validate *your* schema changes — unless you give it tools. MCP (Model Context Protocol) is a standard protocol for exposing domain-specific operations to AI agents.

This article is based on building an MCP server that wraps the Apicurio Registry API, exposing 25+ schema governance operations as tools that AI agents like Claude Code can call directly.

> Full article with examples: [distributed-deep-dives/mcp-servers-domain-ai](https://github.com/carlesarnal/distributed-deep-dives/tree/main/mcp-servers-domain-ai)

## Why MCP Over Alternatives

Before MCP, you had three options for giving AI agents domain access:

- **Function calling** — Model-specific (OpenAI, Claude each have their own format). Not portable.
- **LangChain tools** — Framework-specific. Ties you to a Python ecosystem.
- **Custom prompting** — "Here's the curl command, run it." Fragile, no type safety.

MCP solves this with a standard protocol: servers expose typed tools with JSON Schema parameters, clients (AI agents) discover and call them. One MCP server works with any MCP-compatible client.

## Designing the Tool Surface

The hardest decision isn't implementation — it's what to expose. Not every REST endpoint should be an MCP tool. My design principles:

**Expose operations that benefit from AI reasoning:** Search, analysis, comparison, validation. "Find all schemas related to predictions" is a natural language query that maps well to `search_artifacts`.

**Limit operations that could cause damage:** The MCP server exposes `create_artifact` and `update_version_content`, but these are guarded by the registry's own compatibility rules. Destructive operations (delete, force-update) require explicit confirmation.

**Name tools for discoverability:** The AI reads tool names and descriptions to decide which to use. `get_version_content` is clear. `fetch_v` is not. Rich descriptions with examples help the AI choose correctly.

## Real-World Usage

With the MCP server configured, I can ask my AI assistant:

- *"What schemas are registered in the reddit-realtime group?"* — calls `list_artifacts`
- *"Is this schema change backward compatible?"* — calls `create_version` with dry-run header
- *"Show me the prompt templates available"* — calls `list_mcp_prompts`
- *"Compare version 1 and version 2 of the predictions schema"* — chains `get_version_content` twice

The AI handles the orchestration — chaining multiple tool calls, interpreting results, and presenting them in context. This is more productive than manually running curl commands or navigating the registry UI.

## Security Considerations

An MCP server is an attack surface. The AI agent operates with whatever permissions the server grants. Principles:

- **Least privilege**: Read-only by default. Write operations require explicit context.
- **Dry-run for validation**: Use the registry's dry-run mode (`X-Registry-DryRun: true`) to validate changes without persisting them.
- **Audit logging**: Log every AI-initiated operation with the session context.
- **No delete operations**: In my server, there's no tool for deleting artifacts. If I need to delete, I do it manually.

## Generalizable Patterns

Building this MCP server taught me patterns that apply to wrapping any REST API:

1. **Map REST resources to tool groups** — artifacts, versions, groups become natural tool categories
2. **Use the API's own error messages** — don't hide them from the AI. A 409 Conflict with "schema not backward compatible" is more useful than a generic "operation failed"
3. **Handle pagination transparently** — the AI shouldn't need to manage page tokens
4. **Provide rich descriptions** — the AI reads them to decide which tool to use
5. **Consider caching** — schema content changes rarely; cache reads aggressively

---

*Full article with MCP tool definitions and multi-tool workflow examples: [distributed-deep-dives/mcp-servers-domain-ai](https://github.com/carlesarnal/distributed-deep-dives/tree/main/mcp-servers-domain-ai)*
