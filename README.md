# 🚀 Teleport-Walk: A Two-Stage Hybrid Retrieval Architecture for University Knowledge Bases

<p align="center">
  <img src="https://img.shields.io/badge/Status-Published-brightgreen" alt="Status"/>
  <img src="https://img.shields.io/badge/Conference-IEEE%20QPAIN%202026-blue" alt="Conference"/>
  <img src="https://img.shields.io/badge/Institution-East%20West%20University-orange" alt="Institution"/>
  <img src="https://img.shields.io/badge/Python-3.8%2B-yellow" alt="Python"/>
  <img src="https://img.shields.io/badge/License-MIT-lightgrey" alt="License"/>
</p>



---

## 📖 Table of Contents

- [Overview](#overview)
- [The Problem We Solved](#the-problem-we-solved)
- [How It Works](#how-it-works)
- [System Architecture](#system-architecture)
- [Dataset](#dataset)
- [Results](#results)
- [Project Structure](#project-structure)
- [Installation & Usage](#installation--usage)
- [Authors](#authors)
- [Citation](#citation)

---

## Overview

**Teleport-Walk** is a novel two-stage hybrid information retrieval system that intelligently combines **keyword-based search (BM25)** and **vector-based semantic search (FAISS-HNSW)** in a sequential pipeline — not in parallel.

Instead of running both searches independently and merging results (which is costly), Teleport-Walk uses BM25 to find one highly confident "anchor document" first, then uses that anchor as the precise entry point into the semantic graph for deeper exploration.

> Think of it like this: BM25 *teleports* you to the right neighborhood, and the HNSW graph *walk* explores all the semantically related documents from there.

---

## The Problem We Solved

| Search Method | Strength | Weakness |
|---|---|---|
| **Pure BM25 (Keyword)** | Fast, precise exact matches | Fails on synonyms, semantic context |
| **Pure HNSW (Semantic)** | Captures meaning & synonyms | Noisy for short/ambiguous queries, high cost |
| **Parallel Hybrid (RRF)** | Combines both | Double computation, high latency |
| ✅ **Teleport-Walk (Ours)** | Fast + Semantically rich + Precise | — |

Traditional hybrid systems run lexical and semantic search in **parallel** and merge results using algorithms like Reciprocal Rank Fusion (RRF). This doubles the compute cost and increases latency. Our approach is **sequential and structured**, which eliminates redundancy while preserving accuracy.

---

## How It Works

The retrieval pipeline has **4 steps**:

```
Query (q)
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  STEP 1 — Lexical Teleportation (The Jump)              │
│  Run BM25 on the full corpus. Find the top-ranked doc.  │
│  d_top = argmax BM25(q, d)                              │
└─────────────────────────┬───────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  STEP 2 — State Initialization                          │
│  Retrieve the precomputed dense embedding of d_top.     │
│  This becomes the semantic seed vector (v_top).         │
└─────────────────────────┬───────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  STEP 3 — HNSW Graph Entry                              │
│  Skip directly to the node v_top in the HNSW graph.    │
│  No costly top-down traversal from a random entry.      │
└─────────────────────────┬───────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  STEP 4 — Semantic Graph-Walk                           │
│  Greedy nearest-neighbor search from v_top.             │
│  Sim(v_top, v_n) = (v_top · v_n) / (‖v_top‖ ‖v_n‖)   │
│  Returns Top-k semantically similar results.            │
└─────────────────────────┬───────────────────────────────┘
                          │
                          ▼
              Ranked Results (Top-k)
       [Lexical Grounding + Semantic Expansion]
```

### Why This Is Better

- **Noise Attenuation:** Short queries like `"python loop error"` produce poor entry vectors. Using a BM25-verified document as the seed avoids landing in a wrong region of vector space.
- **Latency Optimization:** Entering HNSW at the correct node reduces the number of hops needed to reach the semantic neighborhood.
- **Vocabulary Gap Mitigation:** Documents semantically related but lexically different from the query are discovered through the graph walk from the correct anchor.

---

## System Architecture

### Phase 1: Initialization & Indexing

```
JSON Corpus
    ├── Lexical Pipeline (BM25 Engine)
    │       ├── Tokenization
    │       ├── Inverted Index
    │       └── ──► BM25 Index + Embedding Matrix
    │
    └── Semantic Pipeline (Graph Construction)
            ├── Sentence Transformer (all-MiniLM-L6-v2)
            ├── Dense Embeddings
            └── FAISS HNSW Index ──► HNSW Graph
```

### Phase 2: Retrieval (Teleport-Walk)

```
User Query
    ├── 1. Lexical Teleportation  →  BM25(q, D) → d_top
    ├── 2. State Initialization   →  Retrieve embedding of d_top
    ├── 3. HNSW Graph Entry       →  Jump to v_top node directly
    └── 4. Semantic Walk          →  Greedy 5-NN search
                                  →  Ranked Top-k Results
```

### Core System Components

| Component | Role |
|---|---|
| **BM25 Engine** | Deterministic keyword-based indexing and lexical scoring |
| **Sentence Transformer** | Generates dense embeddings using `all-MiniLM-L6-v2` |
| **FAISS Vector Store** | Manages HNSW structure, vector storage, and memory mapping |
| **HNSW Index** | Multi-layered proximity graph for efficient nearest-neighbor traversal |

---

## Dataset

The system was built and evaluated on a real-world institutional knowledge base scraped from **East West University (EWU)**, Dhaka, Bangladesh.

### Data Collection Pipeline

```
Phase 1: Acquisition
    ├── Institutional PDF Sources  →  PDF-to-Text Extraction
    └── Web Scraper (Faculty Portals)  →  Raw JSON Data Extraction

Phase 2: Transformation & Enrichment
    ├── Atomic Segmentation (Regex Chunking)  →  Course Description Chunks
    ├── Contextual Injection (URL-Path Inference)  →  Faculty Profile Records
    └── Text Normalization (CamelCase & Whitespace Repair)  →  Cleaned Text Data

Phase 3: Unification
    └── Unified Knowledge Repository (Structured JSON + Metadata Enrichment)
             └──► Vector & Keyword Indices (Ready for Teleport-Walk)
```

### Dataset Statistics

| Category | Records | Data Type | Source |
|---|---|---|---|
| Course Catalog | ~1,200 | Textual / Structured | Academic Bulletin |
| Faculty Profiles | ~450 | Semi-Structured | Official Web Portals |
| Tuition & Fees | ~60 | Tabular / Numeric | Academic Bulletin |
| Scholarship Rules | ~35 | Rule-based / Textual | Academic Bulletin |

---

## Results

### Performance Comparison

| Method | Latency (ms) ↓ | Throughput (q/s) ↑ | Lexical Overlap |
|---|---|---|---|
| Lexical — BM25 | 0.4787 | 2,088.79 | 1.0000 |
| Semantic — HNSW | 0.1462 | 6,842.25 | 0.2000 |
| Hybrid Fusion — RRF | 0.8721 | 1,146.61 | 0.6000 |
| **Teleport-Walk (Ours)** | **0.1173** | **8,525.00** | **0.2000** |

### Key Findings

- ✅ **~7× faster** than standard RRF late-fusion (0.1173 ms vs 0.8721 ms)
- ✅ **Fastest overall** — even faster than pure semantic HNSW (0.1462 ms)
- ✅ **Highest throughput** — 8,525 queries/second
- ✅ **Statistically significant** — Student's t-test p-value = 1.63 × 10⁻¹⁶ (p < 0.05)
- ✅ **Maintains lexical alignment** — 20% lexical overlap preserved

### Scalability (FAISS-HNSW Memory Projection)

| Documents | Memory (MB) | Build Time (s) |
|---|---|---|
| 10,000 | 17.24 | 0.35 |
| 50,000 | 86.18 | 1.75 |
| 100,000 | 172.36 | 3.49 |
| 500,000 | 861.78 | 17.47 |
| 1,000,000 | 1,723.56 | 34.95 |

> Memory scales **linearly** — no exponential growth. Suitable for large-scale university deployments.

---

## Project Structure

```
teleport-walk/
│
├── 📓 Teleport-walk.ipynb          # Main implementation notebook
│
├── 📁 data/
│   ├── courses.json                # Course catalog (~1,200 records)
│   ├── faculty.json                # Faculty profiles (~450 records)
│   ├── scholarship.json            # Scholarship rules (~35 records)
│   └── tuition.json                # Tuition & fees (~60 records)
│
├── 📁 index/
│   ├── bm25_index/                 # Prebuilt BM25 index files
│   ├── hnsw_index/                 # FAISS HNSW index files
│   └── embeddings/                 # Precomputed sentence embeddings
│
├── 📁 scripts/
│   ├── scraper.py                  # Web scraping scripts (EWU portals)
│   ├── preprocess.py               # Data cleaning & normalization
│   └── evaluate.py                 # Benchmark & evaluation scripts
│
├── 📄 requirements.txt
└── 📄 README.md
```

---

## Installation & Usage

### Prerequisites

```bash
Python >= 3.8
```

### Install Dependencies

```bash
pip install -r requirements.txt
```

**Core dependencies:**
```
rank_bm25
faiss-cpu
sentence-transformers
numpy
pandas
jupyter
```

### Quick Start

```python
# 1. Load the knowledge base
from teleport_walk import TeleportWalkRetriever

retriever = TeleportWalkRetriever(
    data_path="data/",
    model_name="all-MiniLM-L6-v2"
)

# 2. Build indices (only needed once)
retriever.build_index()

# 3. Run a query
results = retriever.search("CSE488 project requirements", top_k=5)

for r in results:
    print(r['title'], '-', r['score'])
```

### Running the Notebook

```bash
jupyter notebook Teleport-walk.ipynb
```

The notebook walks through the full pipeline:
1. Data loading and preprocessing
2. BM25 index construction
3. HNSW graph construction via FAISS
4. Sentence embedding with `all-MiniLM-L6-v2`
5. The Teleport-Walk retrieval loop
6. Evaluation and benchmarking against BM25, HNSW, and RRF

---

## Limitations & Future Work

The current architecture has one known limitation: **it depends on BM25 finding a valid anchor document in Stage 1.** If there is a severe vocabulary mismatch (zero lexical overlap between query and documents), BM25 may fail to return a relevant seed, causing the semantic walk to start in a wrong region of vector space.

**Planned improvements:**
- Adaptive fallback thresholds — if BM25 confidence is below a threshold, fall back to a centroid-based entry point
- Metaheuristic algorithms for dynamic entry node recalibration
- Multi-anchor support for queries that span multiple knowledge domains

---

## Authors

| Name | Email | Affiliation |
|---|---|---|
| Md. Mahamudur Rahman | rahmanmehraj627@gmail.com | CSE, East West University |
| Md Riyad Hossain | irfanriyad6@gmail.com | CSE, East West University |
| Mehjarin Aklima Jarin | mehjarin010@gmail.com | CSE, East West University |




---

## Acknowledgements

This research was conducted at the **Department of Computer Science and Engineering, East West University, Dhaka-1212, Bangladesh**. The dataset was constructed from the official EWU web portals and academic bulletins for the 2026 academic year.

---

<p align="center">Made with ❤️ at East West University · Published at IEEE QPAIN 2026</p>
