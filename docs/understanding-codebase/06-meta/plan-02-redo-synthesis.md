---
title: "redo synthesis Q&A"
experiment: 001-understanding-codebase
created: "2026-06-08 06:55 UTC"
---

# Redo Synthesis in Q&A Format

## Problem
Current L0/L1/L2 docs are too dense, technical, and not consumable for giving someone an overview of the QMD repo. They read like API reference, not a guide.

## Goal
Produce a set of docs in **question-answer format** that someone can read (or present) to understand:
1. What QMD is and why it exists
2. How to use it (CLI, SDK, MCP)
3. How it's structured (architecture)
4. What nuances to watch for (gotchas, design decisions)

## Output Format

Each doc is a series of Q&A pairs. Every answer should be:
- 1-3 sentences for simple things
- Up to a short paragraph for complex things
- Include a concrete example where it helps
- Reference file paths for the curious

## Docs to Produce

1. `00-what-is-qmd.md` — Elevator pitch, problem, capabilities, tech stack
2. `01-how-to-use.md` — CLI commands with examples, SDK code samples, MCP setup
3. `02-architecture.md` — High-level architecture, module map, data flow
4. `03-gotchas.md` — Design decisions, common pitfalls, nuanced behaviors
5. `04-deep-dives.md` — Per-subsystem Q&A: chunking, search, embeddings, config

## Input

Read all explorer outputs:
- `workers/cli-explorer/output/findings.md`
- `workers/store-explorer/output/findings.md`
- `workers/db-coll-explorer/output/findings.md`
- `workers/llm-ast-explorer/output/findings.md`
- `workers/sdk-mcp-explorer/output/findings.md`

And the current synthesizer outputs for reference (but do not copy their density):
- `workers/synthesizer/output/L0-overview.md`
- `workers/synthesizer/output/L1-modules.md`
- `workers/synthesizer/output/L2-implementation.md`

## Agent

Launch a single codex gpt-5.5 medium agent to produce all 5 docs. No fleet needed — one focused synthesis task.
