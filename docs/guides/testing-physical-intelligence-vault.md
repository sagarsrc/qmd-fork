# Testing QMD with the Physical Intelligence Vault

This guide walks through cloning the [Physical Intelligence Vault](https://github.com/git-kinetix/physical-intelligence-vault) — a research knowledge base of 60 AI/robotics papers — and using QMD to search it.

---

## Setup

### 1. Clone (without PDFs)

```bash
git clone --depth 1 https://github.com/git-kinetix/physical-intelligence-vault.git ~/pi-vault

# Remove PDFs (~700MB) — we only need markdown
rm -rf ~/pi-vault/PDFs

# Optional: remove other non-markdown bulk
rm -rf ~/pi-vault/.git ~/pi-vault/.playwright-mcp ~/pi-vault/Explainers
```

### 2. Add to QMD

```bash
qmd collection add ~/pi-vault --name pi-research --mask '**/*.md'
qmd update
qmd embed
```

**What gets indexed:**
- `Papers/*.md` — 60 paper summaries with architecture breakdowns
- `Metrics/*.md` — 64 metric definitions
- `Datasets/*.md` — 78 dataset descriptions
- `MOC.md`, `Metrics Index.md`, `Datasets Index.md` — vault indexes

---

## Demo Queries

### Keyword Search (`qmd search`)

Fast BM25 search. Good for names, acronyms, exact terms.

```bash
# Acronyms
qmd search "JEPA"
qmd search "VLA"
qmd search "TD-MPC2"

# Specific methods/models
qmd search "DreamerV3"
qmd search "Gemini Robotics"
qmd search "OpenVLA"

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
# Concepts
qmd vsearch "predicting future states from video"
qmd vsearch "robot learning from human demonstrations"
qmd vsearch "embedding spaces for video understanding"
qmd vsearch "physics-based character animation"
qmd vsearch "transfer from simulation to real world"
qmd vsearch "generating video for robot planning"
qmd vsearch "evaluating robotic manipulation"
```

### Hybrid Search (`qmd query`) — Recommended

Combines keyword + vector + LLM query expansion. Best for complex questions.

```bash
# What/How questions
qmd query "what is JEPA and how does it work"
qmd query "what is a world model in robotics"
qmd query "how does video generation help robot planning"

# Comparisons
qmd query "compare world models to VLA models"
qmd query "how does DreamerV3 differ from DreamerV2"
qmd query "JEPA vs V-JEPA vs I-JEPA"

# Cross-cutting (finds connections across papers)
qmd query "which papers use joint embedding architectures"
qmd query "how have world models evolved from Dreamer to JEPA"
qmd query "papers that combine video generation with control"

# Metric-centric
qmd query "what metrics measure robotic success"
qmd query "what is PSNR and which papers report it"
qmd query "metrics for video quality in world models"

# Dataset-centric
qmd query "datasets for training robot manipulation"
qmd query "papers that use EPIC-KITCHENS dataset"
qmd query "benchmarks for sim-to-real transfer"

# Architecture deep-dives
qmd query "encoder decoder architectures in robotics"
qmd query "transformer architectures for robot control"
qmd query "how does ACT-JEPA architecture work"

# Method categories
qmd query "methods for sim to real transfer"
qmd query "temporal difference learning in world models"
qmd query "reinforcement learning for character animation"

# Negative space (finds what's NOT there)
qmd query "reinforcement learning without simulators"
qmd query "robot learning without human demonstrations"
```

### Direct Retrieval (`qmd get`)

When you know exactly what you want.

```bash
# Specific paper
qmd get pi-research/Papers/V-JEPA\ 2.md

# Metric definition
qmd get pi-research/Metrics/FVD.md

# Dataset description
qmd get pi-research/Datasets/BridgeData\ V2.md

# Master index
qmd get pi-research/MOC.md

# By docid (shown in search results as #abc123)
qmd get "#abc123"

# Line ranges
qmd get pi-research/Papers/DreamerV3.md:10:30
```

### Multi-Document Retrieval (`qmd multi-get`)

```bash
# All papers in a category
qmd multi-get "Papers/*.md"

# All metrics
qmd multi-get "Metrics/*.md"

# Specific papers by comma-separated list
qmd multi-get "Papers/JEPA*, Papers/V-JEPA*"

# With output format
qmd multi-get "Papers/*.md" --format md
qmd multi-get "Metrics/*.md" --format json
```

---

## Advanced Options

```bash
# Add intent for disambiguation
qmd query "transformer" --intent "robotics vision language action"

# More results
qmd query "world models" -n 20

# JSON output for scripting
qmd query "JEPA" --format json

# Minimum score threshold
qmd query "character animation" --min-score 0.5

# Skip reranking (faster)
qmd query "VLA models" --no-rerank

# With line numbers (default)
qmd get pi-research/Papers/V-JEPA\ 2.md --line-numbers

# Without line numbers
qmd get pi-research/Papers/V-JEPA\ 2.md --no-line-numbers
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
# Ensure pi-research appears
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
