# Testing QMD with a Robotics Research Vault

This guide walks through cloning the [Physical Intelligence Vault](https://github.com/git-kinetix/physical-intelligence-vault) — a research knowledge base of AI/robotics papers — and using QMD to search it.

The examples intentionally avoid `pi-*` names because `pi` can be confused with the Pi coding harness. Locally, this guide uses:

- folder: `~/robotics-vault`
- collection name: `robotics-research`

> Note: the vault does **not** appear to contain a standalone `Attention Is All You Need` paper. Attention is still present through simpler transformer-related entries such as `ACT`, `OpenVLA`, `IRIS`, `GR00T`, and `Attentive Probe Accuracy`.

---

## Setup

### 1. Clone (without PDFs)

```bash
git clone --depth 1 https://github.com/git-kinetix/physical-intelligence-vault.git ~/robotics-vault

# Remove PDFs (~700MB) — we only need markdown
rm -rf ~/robotics-vault/PDFs

# Optional: remove other non-markdown bulk
rm -rf ~/robotics-vault/.git ~/robotics-vault/.playwright-mcp ~/robotics-vault/Explainers
```

### 2. Add to QMD

```bash
qmd collection add ~/robotics-vault --name robotics-research --mask '**/*.md'
qmd update
qmd embed
```

**What gets indexed:**
- `Papers/*.md` — paper summaries with architecture breakdowns
- `Metrics/*.md` — metric definitions
- `Datasets/*.md` — dataset descriptions
- `MOC.md`, `Metrics Index.md`, `Datasets Index.md` — vault indexes

---

## Demo Queries

### Keyword Search (`qmd search`)

Fast BM25 search. Good for names, acronyms, exact terms.

```bash
# Attention / transformer terms available in the vault
qmd search "attention"
qmd search "transformer"
qmd search "cross-attention"
qmd search "Attentive Probe Accuracy"

# Specific simpler transformer/robotics papers
qmd search "ACT"
qmd search "OpenVLA"
qmd search "IRIS"
qmd search "GR00T"

# Metrics
qmd search "FVD"
qmd search "PSNR"
qmd search "mAP"

# Datasets
qmd search "Open X-Embodiment"
qmd search "BridgeData V2"
```

### Vector Search (`qmd vsearch`)

Semantic/meaning-based. Good when you don't know the exact name.

```bash
# Attention / transformer concepts
qmd vsearch "attention mechanism for selecting important tokens"
qmd vsearch "transformers that predict sequences of robot actions"
qmd vsearch "using cross attention to aggregate visual tokens"
qmd vsearch "world models based on autoregressive transformers"

# Robotics concepts
qmd vsearch "robot learning from human demonstrations"
qmd vsearch "vision language action models for robots"
qmd vsearch "evaluating robotic manipulation"
qmd vsearch "transfer from simulation to real world"
```

### Hybrid Search (`qmd query`) — Recommended

Combines keyword + vector + LLM query expansion. Best for complex questions.

```bash
# What/How questions — simple attention/transformer examples
qmd query "what is attention used for in robot policies"
qmd query "what is ACT and how does action chunking work"
qmd query "how do transformers predict future robot actions"
qmd query "what is attentive probe accuracy"

# Comparisons
qmd query "compare ACT OpenVLA and GR00T"
qmd query "how does IRIS use transformers as a world model"
qmd query "how do VLA models differ from world models"

# Cross-cutting (finds connections across papers)
qmd query "which papers use transformer architectures"
qmd query "which papers use attention or cross attention"
qmd query "papers that combine vision language and robot actions"

# Metric-centric
qmd query "what metrics measure robotic success"
qmd query "what is attentive probe accuracy and why use it"
qmd query "what metrics evaluate video quality"

# Dataset-centric
qmd query "datasets for training robot manipulation"
qmd query "what is Open X-Embodiment used for"
qmd query "benchmarks for sim-to-real transfer"

# Architecture deep-dives
qmd query "transformer encoder decoder architectures in robotics"
qmd query "how does cross attention appear in robot policies"
qmd query "how does ACT use a transformer decoder"

# Method categories
qmd query "imitation learning with transformers"
qmd query "vision language action transformer models"
qmd query "world models with discrete tokens and transformers"
```

### Direct Retrieval (`qmd get`)

When you know exactly what you want.

```bash
# Transformer/attention-related papers
qmd get robotics-research/Papers/ACT.md
qmd get robotics-research/Papers/OpenVLA.md
qmd get robotics-research/Papers/IRIS.md
qmd get robotics-research/Papers/GR00T.md

# Attention-related metric definition
qmd get robotics-research/Metrics/Attentive\ Probe\ Accuracy.md

# Dataset description
qmd get robotics-research/Datasets/Open\ X-Embodiment.md

# Master index
qmd get robotics-research/MOC.md

# By docid (shown in search results as #abc123)
qmd get "#abc123"

# Line ranges
qmd get robotics-research/Papers/ACT.md:20:35
```

### Multi-Document Retrieval (`qmd multi-get`)

```bash
# All papers
qmd multi-get "Papers/*.md"

# All metrics
qmd multi-get "Metrics/*.md"

# Specific attention/transformer docs by comma-separated list
qmd multi-get "Papers/ACT.md, Papers/OpenVLA.md, Papers/IRIS.md, Metrics/Attentive Probe Accuracy.md"

# Transformer-related paper names (glob where possible)
qmd multi-get "Papers/*VLA*.md, Papers/ACT.md, Papers/IRIS.md"

# With output format
qmd multi-get "Papers/*.md" --format md
qmd multi-get "Metrics/*.md" --format json
```

---

## Advanced Options

```bash
# Add intent for disambiguation
qmd query "attention" --intent "robotics transformer architectures and action policies"

# More results
qmd query "transformer robot policies" -n 20

# JSON output for scripting
qmd query "attention in robot policies" --format json

# Minimum score threshold
qmd query "character animation" --min-score 0.5

# Skip reranking (faster)
qmd query "VLA models" --no-rerank

# With line numbers (default)
qmd get robotics-research/Papers/ACT.md --line-numbers

# Without line numbers
qmd get robotics-research/Papers/ACT.md --no-line-numbers
```

---

## What to Expect

| Search Type | Speed | Best For |
|-------------|-------|----------|
| `search` | Instant | Exact names, acronyms |
| `vsearch` | Fast | Semantic similarity |
| `query` | Slower | Complex questions, exploration |

**Note:** `qmd query` uses LLM query expansion (generates lex/vec/hyde variants) and optional reranking. First run may trigger model download (~1-2GB GGUF models to `~/.cache/qmd/models`).

---

## Troubleshooting

**"No collections found"**
```bash
qmd collection list
# Ensure robotics-research appears
```

**"sqlite-vec unavailable"**
- Vector search won't work. BM25 still works.
- On macOS with Bun: `brew install sqlite`

**"Need embeddings"**
```bash
qmd embed  # Run after qmd update
```

**Models not downloaded**
```bash
qmd pull  # Downloads default models
```
