# RFC 001: In-Band Tool Versioning

**Status:** Draft
**Date:** 2026-03-13
**Authors:** Greg Warden, Claude

**Discussion:** [GitHub Discussions thread](https://github.com/usmile1/MCProspero-feedback/discussions) *(link updated after posting)*

---

## Problem

MCP servers evolve. Tool descriptions get updated, parameters are added or removed, new tools appear, old ones are deprecated. But MCP clients cache tool definitions at connection time — and most don't support [`notifications/tools/list_changed`](https://modelcontextprotocol.io/specification/2025-06-18/server/tools) yet. Even [Claude Code doesn't implement the handler](https://github.com/anthropics/claude-code/issues/13646).

This creates a real problem: after a server-side deploy, clients continue using stale tool definitions until they reconnect. Users don't know anything changed. Agents built against older tool schemas may call tools incorrectly — or miss new capabilities entirely.

Today, no major MCP server has a production solution for this. The MCP spec defines a notification mechanism, but client support is inconsistent, and there's no standard for what happens when a client is stale.

### The LLM-Era Wrinkle

Traditional API versioning cares about structural compatibility — did you add a field, remove an endpoint, change a type? LLM tool versioning introduces a third category that has no precedent in REST or RPC: **language breaks**.

A [recent study of 10,240 MCP servers](https://arxiv.org/abs/2602.03580) found that ~13% exhibit significant description-code inconsistency. The [cchistory project](https://cchistory.mariozechner.at/), which tracks Claude Code's tool definitions across every release, provides concrete evidence: changes in tool descriptions "generally express themselves through Claude using tools differently, sometimes less efficiently, sometimes more efficiently." The ["Evolvable MCP" article](https://medium.com/@kumaran.isk/evolvable-mcp-a-guide-to-mcp-tool-versioning-ae9a612f7710) names three distinct break types:

| Category | Example | Traditional API Issue? | LLM Tool Issue? |
|---|---|---|---|
| **Schema breaks** | New required parameter | Yes | Yes |
| **Logic drift** | Same schema, changed behavior | Sometimes | Yes |
| **Language breaks** | Rewording a description changes tool selection probability | No | **Yes — unique to LLM era** |

As [Martin Fowler notes](https://martinfowler.com/articles/function-call-LLM.html), prompts are non-deterministic — the same prompts given to a different model, or the same model at a later date, are not guaranteed to produce the same output. Tool descriptions are prompts. Changing them changes behavior. Any versioning system for LLM tools must account for this.

### What We Need

MCProspero needs a versioning protocol that:

1. Works with **any** MCP client — no client-side protocol support required
2. Lets the server detect stale clients at runtime
3. Gives the AI model enough information to respond appropriately
4. Handles both interactive sessions and persistent agents
5. Doesn't break existing clients that aren't version-aware

---

## Prior Art

### MCP Ecosystem

The MCP community is actively working on versioning, but no standard has emerged yet:

- [**SEP-1575: Tool Semantic Versioning**](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/1575) proposes adding a `version` field to tool definitions with client-side pinning via `tool_requirements`. This is the most mature proposal and the closest to what we're describing here — but it requires client-side support that doesn't exist yet.
- [**SEP-1766: Digest-Pinned Tool Versioning**](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/1766) takes a lighter approach inspired by GitHub Actions: SHA256 content digests on tools for drift detection, with an interceptor framework for enforcement.
- [**Issue #1039**](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/1039) is an early feature request that laid out three approaches: version parameter, version in tool name (`tool_v2`), or namespace versioning.
- [**Issue #476**](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/476) argues that the spec's date-based versioning doesn't communicate whether changes are breaking.
- The [**MCP Registry versioning guidance**](https://modelcontextprotocol.io/registry/versioning) recommends SemVer and immutable versions for published servers.

No major production MCP server uses URL versioning today — Stripe, Linear, Zapier, and PayPal all use unversioned MCP paths.

### Stripe's Approach

Stripe's API versioning strategy, described in their 2017 blog post ["APIs as infrastructure: future-proofing Stripe with versioning"](https://stripe.com/blog/api-versioning), was a key influence on our thinking — both for what to adopt and what to skip.

Stripe uses account pinning (your first API call locks your version), a `Stripe-Version` header for per-request overrides, and version change modules — encapsulated backward-incompatible transformations applied sequentially from current to target version. As [Brandur Leach explains](https://brandur.org/api-upgrades), automatic upgrades are unsafe because REST's opacity prevents knowing which fields clients actually use.

MCProspero's situation is different. Our "clients" are AI models with cached tool descriptions, not programs with hardcoded API calls. We can't pin versions per-account because the staleness is in the model's context window, not in application code. But Stripe's insight that **the server must own version awareness** (rather than hoping clients manage it) directly informed our in-band approach.

---

## Solution: Version Echo + In-Band Response Notices

### Overview

1. Every tool description includes a version header the AI model reads at initialization.
2. The model echoes the version it was initialized with as a field on every tool call.
3. The server compares the echoed version to the current version.
4. If stale, the server injects a `_tool_notice` field into the tool response.
5. The model reads the notice and responds appropriately — updating behavior silently, suggesting a reconnect at a natural pause, or requiring an immediate stop.

This is entirely in-band. No new protocol messages. No client changes. It works today, with any MCP client, because it operates through the existing tool description → tool call → tool response loop that every client already supports.

---

## Change Classification

Not all changes are equal. The key question is: **does the AI model need to see a new tool definition to use this correctly?**

| Change Type | Backward Compatible | Requires Reconnect | Notice Type |
|---|---|---|---|
| Updated instructions or guidance | Yes | No | Instructions Updated |
| New enum value on existing parameter | Yes | No | Instructions Updated |
| New optional parameter | Yes | Yes | Soft Reload |
| New tool added | Yes | Yes | Soft Reload |
| New required parameter | No | Yes | Hard Reload |
| Parameter renamed or removed | No | Yes | Hard Reload |
| Tool removed | No | Yes | Hard Reload |

**Backward compatible does not mean reconnect-optional.** A new optional parameter won't break existing calls, but the model can't *use* it without reconnecting to see the new schema.

### Semantic Versioning

MCProspero uses semver for tool versions:

| Bump | What Changed | Notice |
|---|---|---|
| Patch (x.x.1) | Typo fixes, wording clarifications | None |
| Minor (x.1.0) | Behavioral rules, new guidance, new enum values | Instructions Updated |
| Major (2.0.0) | Any structural change — parameters, tools, types | Soft or Hard Reload |

Both additive changes (new optional parameter) and breaking changes (removed parameter) are major bumps. The distinction is whether old calls still work: additive → Soft Reload, breaking → Hard Reload.

---

## The Version Echo

Every tool description begins with a version header:

```
# MCProspero Tools v2.3.0

Create and manage persistent AI agents...
```

The tool description also instructs the model to echo this version on every call:

```
When calling this tool, always include:
  "_mcprospero_version": "<version from header above>"
```

The server reads this field and compares it to the current version. If the field is missing, the server treats it as `"0.0.0"` — maximally stale. This means version-unaware clients degrade gracefully: they still work, but the server knows they're stale.

---

## Three Notice Types

When the server detects a version mismatch, it injects a `_tool_notice` field into the tool response alongside the normal result. The notice type tells the model exactly what to do.

### Type 1 — Instructions Updated

Triggered by minor version drift (behavioral or guidance changes).

```json
{
  "result": "...normal tool result...",
  "_tool_notice": {
    "type": "instructions_updated",
    "version": "2.4.0",
    "instructions": "...full updated tool description...",
    "changelog": [...]
  }
}
```

**Model behavior:** Read the `instructions` field. Apply the updated guidance for the rest of the session. No user interruption. Continue the current task.

This is the lightest touch — the model silently self-updates without the user knowing anything changed.

### Type 2B — Soft Reload Available

Triggered by additive major changes (new tools or optional parameters — old calls still work).

```json
{
  "_tool_notice": {
    "type": "soft_reload_available",
    "version": "3.1.0",
    "message": "New capabilities available in v3.1.0. Mention to user when convenient.",
    "changelog": [...]
  }
}
```

**Model behavior:** Finish the current task. At a natural pause, mention once:

> "By the way, MCProspero v3.1.0 has new capabilities I can't access in this session. No rush, but reconnecting the MCP server whenever convenient will unlock them."

Do not block. Do not repeat on subsequent calls. Fire once per session.

### Type 2A — Hard Reload Required

Triggered by breaking major changes (removed or renamed parameters, removed tools — old calls may fail).

```json
{
  "_tool_notice": {
    "type": "hard_reload_required",
    "version": "3.0.0",
    "message": "Breaking schema changes in v3.0.0. Existing calls may fail.",
    "changelog": [...]
  }
}
```

**Model behavior:** Stop the current tool flow immediately:

> "MCProspero has breaking schema changes in v3.0.0 that may cause calls to fail. Please disconnect and reconnect the MCProspero server before we continue."

Wait for reconnect. Do not retry.

---

## The Changelog

Every notice includes a `changelog` field — a list of entries describing what changed since the model's version. This is critical. Without it, the model can only say "new capabilities are available" — generic and unhelpful. With the changelog, it can be specific:

> "MCProspero v3.1.0 adds a `bulk_create_agents` tool for creating multiple agents at once, and a new optional `tags` parameter on `create_agent` for organizing your agents. Reconnecting whenever convenient will unlock those."

Each changelog entry includes:

- **version** — which release introduced the change
- **summary** — plain-English sentence the model leads with
- **changes** — individual changes, each with a `kind` (`new_tool`, `new_param`, `removed_tool`, `removed_param`, `renamed_param`, `behavioral`, `new_enum_value`), a description, and a `user_facing` flag

If a client is multiple versions behind, the changelog contains the full accumulated delta — not just the latest entry. The model presents a coherent summary of everything they've missed.

The `user_facing` flag controls whether the model mentions a change to the user. Some changes (like a parameter type becoming more permissive in a way transparent to the user) need a changelog entry for audit purposes but don't need to be announced.

---

## How Persistent Agents Are Different

Running agents invert the caching problem:

| | Interactive Session | Persistent Agent |
|---|---|---|
| Tool schema | Frozen until reconnect | Fresh every run |
| Tool descriptions | Frozen until reconnect | Fresh every run |
| System prompt | Fresh each conversation | Frozen at creation time |

Agents don't have the stale-schema problem — the platform injects current tool definitions on every run. What's frozen is the system prompt, written once when the agent was created.

This means Soft and Hard Reload notices are largely irrelevant for agents. What matters is **system prompt drift** — platform requirements evolve, but existing agent prompts don't.

MCProspero handles this through two mechanisms:

### Platform Preamble

Before every agent run, the platform prepends a controlled preamble to the stored system prompt. New platform-wide requirements go in the preamble and apply immediately to all agents — no system prompt edits needed.

```
[PLATFORM PREAMBLE v1.2.0]
...current platform requirements...
[END PLATFORM PREAMBLE]

[AGENT SYSTEM PROMPT]
...the prompt written at creation time...
```

The preamble is versioned independently. Agent owners never need to update their prompts for platform-level changes — the preamble handles it.

### Drift Tracking

Each agent is stamped with the platform version at creation time. On every run, the platform computes the delta and injects it into the run context when non-empty:

```
Platform version at agent creation: 2.2.0
Current platform version: 2.4.0
Changes since creation:
  2.3.0: Notification Targets section now required in all agent prompts.
  2.4.0: HTTP Endpoints section now required for agents making web requests.
```

Staleness is tracked in bands:

| Status | Condition | Action |
|---|---|---|
| Current | Within 1 minor version | None |
| Slightly stale | 2-3 minor versions behind | Informational |
| Stale | 4+ minor or 1 major behind | Notify owner |
| Critical | Major version behind with breaking changes | Warn on every run |

### Breaking Changes for Agents

If a breaking schema change affects a running agent, the agent can't send a notification — it's supposed to stop tool calls, and the owner's contact details may not be in the agent's approved targets. Instead:

1. The agent writes a structured log entry and exits cleanly
2. The platform's run monitoring detects the failure
3. The platform notifies the owner through its own infrastructure

The agent doesn't try to be clever — it stops and lets the platform handle alerting.

---

## CI Enforcement

Versioning discipline is enforced in CI, not by convention. Two artifacts serve different purposes:

| Artifact | Maintained by | Scope | Purpose |
|---|---|---|---|
| Build hash | CI, automatic | Every deploy | "What code is running?" |
| Tool version (semver) | Developer, manual | Client interface only | "What does the AI model see?" |

Backend changes — infrastructure, refactors, bug fixes that don't touch tool schemas or descriptions — advance the build hash but leave the tool version unchanged.

### Surface Area Hash

On every PR, CI computes a hash of all tool names, parameter names, and parameter types — the full client-visible surface. If the hash changes, the PR is gated:

- **Hash unchanged** — backend-only change. No version bump required. Normal review.
- **Hash changed** — client interface changed. Version bump required. Changelog entry required. PR blocked until both exist.

This means a database optimization PR never touches the versioning machinery. A PR that adds a parameter to `create_agent` always does.

### Changelog Quality

Changelog entries are validated in two passes:

1. **Structural validation** (automated tests) — all required fields present, correct types, versions ordered, no duplicates.
2. **Semantic validation** (LLM-assisted review) — descriptions are clear enough for the AI model to explain changes accurately to users at runtime. A description like "param updated" fails. A description like "The tags parameter on create_agent now accepts a list of strings for organizing agents into groups" passes.

Both gates are required. Structural validation catches missing fields. Semantic validation catches entries that are technically complete but would sound robotic or vague when surfaced to users.

---

## URL Versioning (Decision Pending)

A related question: should MCProspero adopt URL versioning (`/mcp/v1`, `/mcp/v2`)?

No major MCP server uses URL versioning today. The MCP Registry endorses it, but nobody has implemented it. URL versioning only solves structural breaks — and our in-band protocol already handles those plus semantic and behavioral changes that URL versioning can't address.

We're considering moving to `/mcp/v1` as a low-cost hedge that leaves options open, but full URL versioning infrastructure is deferred until we have data on how often breaking changes actually occur.

---

## Design Principles

**Work with the ecosystem as it is.** `notifications/tools/list_changed` exists in the spec but client support is inconsistent. We can't wait for every client to implement it. The in-band approach works today, with any client.

**Don't interrupt unless necessary.** The three notice types form a spectrum from silent (instructions updated) to soft mention (new capabilities) to hard stop (breaking changes). Most updates are Type 1 — the user never knows.

**Give the model enough to be specific.** Generic "new version available" messages are unhelpful. The changelog gives the model concrete information to explain what changed and why it matters.

**Agents and sessions are different problems.** Interactive sessions have stale schemas. Agents have stale prompts. The same versioning framework handles both, but the mechanisms are different — because the frozen layer is different.

**Enforce in CI, not by convention.** If a PR changes the client-visible surface and doesn't bump the version, CI blocks it. Humans forget. CI doesn't.

---

## Relationship to MCP Spec Proposals

This RFC is complementary to the work happening in the MCP specification:

- If [SEP-1575](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/1575) (Tool Semantic Versioning) is adopted, MCProspero would adopt its `version` field in tool definitions. The in-band notice mechanism would remain valuable as a fallback for clients that don't implement the new spec features.
- If [SEP-1766](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/1766) (Digest-Pinned Versioning) is adopted, the surface area hash could align with its digest format for interoperability.
- The platform preamble and agent drift tracking are MCProspero-specific concerns that don't overlap with spec-level proposals — they address the persistent agent lifecycle, which is outside the MCP spec's scope.

We'd welcome convergence. In the meantime, this approach works today without waiting for spec changes or client updates.

---

## References

### MCP Specification & Proposals

1. [MCP Spec: Tools (2025-06-18)](https://modelcontextprotocol.io/specification/2025-06-18/server/tools) — authoritative definition of `notifications/tools/list_changed`
2. [SEP-1575: Tool Semantic Versioning](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/1575) — proposed `version` field on tool definitions with client-side pinning
3. [SEP-1766: Digest-Pinned Tool Versioning](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/1766) — SHA256 digest approach to drift detection
4. [Issue #1039: Tool Versioning Documentation](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/1039) — early feature request exploring three approaches
5. [Issue #476: Versioning scheme for non-breaking changes](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/476) — argues for SemVer over date-based versioning
6. [MCP Registry: Versioning Published MCP Servers](https://modelcontextprotocol.io/registry/versioning) — official registry guidance recommending SemVer

### Client Support Gaps

7. [Claude Code Issue #13646: Tool list not refreshed on list_changed](https://github.com/anthropics/claude-code/issues/13646) — confirms even major clients have incomplete notification support

### Traditional API Versioning

8. [APIs as infrastructure: future-proofing Stripe with versioning](https://stripe.com/blog/api-versioning) (Stripe Engineering, 2017) — account pinning, version transformation modules
9. [Why Doesn't Stripe Automatically Upgrade API Versions?](https://brandur.org/api-upgrades) (Brandur Leach, 2017) — why REST's opacity makes automatic upgrades unsafe

### LLM Tool Versioning (The New Problem)

10. [Don't believe everything you read: Understanding and Measuring MCP Behavior under Misleading Tool Descriptions](https://arxiv.org/abs/2602.03580) (arXiv, 2026) — first large-scale study: ~13% of 10,240 MCP servers have description-code inconsistency
11. [Evolvable MCP: A Guide to MCP Tool Versioning](https://medium.com/@kumaran.isk/evolvable-mcp-a-guide-to-mcp-tool-versioning-ae9a612f7710) — names the three break types: schema breaks, logic drift, and language breaks
12. [cchistory: Tracking Claude Code System Prompt and Tool Changes](https://cchistory.mariozechner.at/) ([blog post](https://mariozechner.at/posts/2025-08-03-cchistory/)) — empirical evidence that description changes alter LLM behavior
13. [Function calling using LLMs](https://martinfowler.com/articles/function-call-LLM.html) (Martin Fowler, 2025) — architectural overview noting prompt non-determinism

---

## Feedback

This RFC describes MCProspero's planned approach to tool versioning. We welcome feedback — particularly from MCP client developers and other MCP server operators facing similar challenges.

Join the discussion: **[GitHub Discussions thread](https://github.com/usmile1/MCProspero-feedback/discussions)** *(link updated after posting)*

Or email us at [hello@mcprospero.ai](mailto:hello@mcprospero.ai).
