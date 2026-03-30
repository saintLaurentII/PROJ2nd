# README — Automotive Financial Crisis Knowledge Graph

This project builds and queries a **knowledge graph about the automotive financial crisis** over 4 lab sessions in a single Jupyter notebook.

---

## Prerequisites

| Requirement | Details |
|---|---|
| **Python** | 3.9+ |
| **Conda** | Anaconda or Miniconda |
| **Ollama** | [https://ollama.com](https://ollama.com) — local LLM server (Session 4 only) |
| **Java** | JDK ≥ 11 (required by OWLReady2 / Pellet reasoner in Session 3) |

---

## Hardware Requirements

| Session | Minimum RAM | Recommended | GPU |
|---|---|---|---|
| Session 1 (Crawling + NER) | 8 GB | 16 GB | Not required |
| Session 2 (RDF + SPARQL) | 4 GB | 8 GB | Not required |
| Session 3 (KGE — TransE/DistMult) | 8 GB | 16 GB | Optional (CPU training ~20 min) |
| Session 4 (RAG — Gemma 2B) | 8 GB | 16 GB | Optional (CPU inference ~10 s/query) |

> **Note:** The spaCy `en_core_web_trf` transformer model (Session 1) peaks at ~4 GB RAM. Gemma 2B (Session 4) requires ~2 GB RAM when served by Ollama.

## Installation

### 1. Create the Conda environment

```bash
conda create -n WDS python=3.10 -y
conda activate WDS
```

### 2. Install Python packages

```bash
pip install -r requirements.txt
```

### 3. Download the Gemma model (Session 4)

```bash
ollama pull gemma:2b
```

Make sure the Ollama application is running before starting the notebook (`http://localhost:11434` must be reachable).

---

## Project Structure

```
├── notebook.ipynb                          # Main notebook (115 cells, Sessions 1–4)
├── automotive_kb.nt                        # Knowledge base (~125,873 triples, N-Triples)
├── alignment.ttl                           # owl:sameAs alignment triples (68 triples, Turtle)
├── ontology.ttl                            # Domain ontology (Turtle)
├── family.owl                              # Test ontology used in Session 3
├── crawler_output.jsonl                    # Raw crawled articles (Session 1)
├── extracted_knowledge.csv                 # Extracted entities/relations (Session 1)
├── extracted_relations.csv                 # Extracted relations (Session 1)
├── entity_alignment.csv                    # Entity alignment results (Session 2)
├── train.txt / valid.txt / test.txt        # KGE splits (Session 3)
├── train_*.txt / valid_*.txt / test_*.txt  # Sized KGE splits (20k, 50k, full)
├── models/                                 # Trained KGE models
│   ├── transe/                             # TransE model + results
│   └── distmult/                           # DistMult model + results
├── notebook_session1_documentation.md      # Session 1 documentation (French)
├── notebook_session2_documentation.md      # Session 2 documentation (French)
├── notebook_session3_documentation.md      # Session 3 documentation (French)
├── notebook_session4_documentation.md      # Session 4 documentation (French)
└── README.md                               # This file
```

---

## How to Run the Notebook

### 1. Activate the environment

```bash
conda activate WDS
```

### 2. Start Ollama (Session 4 only)

Launch the Ollama application, or run:

```bash
ollama serve
```

Verify it is running:

```bash
curl http://localhost:11434
```

### 3. Launch Jupyter

```bash
jupyter notebook notebook.ipynb
```

Or use **VS Code** with the Jupyter extension: open `notebook.ipynb` and select the `WDS` kernel.

### 4. Run cells in order

The notebook is designed to be executed **sequentially from top to bottom**.

| Cells | Session | Topic |
|---|---|---|
| 1–24 | Session 1 | Web crawling, NLP extraction, KG construction |
| 25–56 | Session 2 | Wikidata enrichment, ontology alignment |
| 57–96 | Session 3 | Knowledge Graph Embeddings (TransE, DistMult) |
| 97–115 | **Session 4** | RAG with RDF/SPARQL + local LLM (Gemma 2B) |

> **Important:** Session 4 (cells 97–115) requires Ollama to be running with `gemma:2b` downloaded. Other sessions do not require Ollama.

---

## Session 4 Quick Start

If you only want to run Session 4:

1. Make sure `automotive_kb.nt` and `alignment.ttl` are present in the project root.
2. Start Ollama and pull the model: `ollama pull gemma:2b`.
3. Run cells **97 → 115** in order.
4. Cell 110 produces the evaluation table (5 SPARQL queries + 1 baseline comparison).
5. Cell 112 produces the CLI demo (2 queries + baseline).
6. Cell 114 launches an interactive chatbot widget (optional).

---

## Troubleshooting

| Problem | Solution |
|---|---|
| `ModuleNotFoundError: rdflib` | Run `conda activate WDS` before launching Jupyter |
| `ConnectionError` on Ollama calls | Start the Ollama app or run `ollama serve` |
| `404: model 'gemma:2b' not found` | Run `ollama pull gemma:2b` |
| BCE date warnings flood the output | Already suppressed in cell 100 — restart kernel if they reappear |
| `Unknown namespace prefix: wdt` | The `add_prefixes()` function in cell 106 handles this automatically |
| PC crashes / high memory usage | Gemma 2B needs ~2 GB RAM. Close other applications. The evaluation uses hand-crafted SPARQL to minimize LLM calls |

---

## Lab Session Overview

- **Session 1** — Web crawling of automotive crisis articles, NLP-based entity and relation extraction, initial KG construction in RDF.
- **Session 2** — Enrichment via Wikidata SPARQL queries, ontology alignment (`owl:sameAs`), export to N-Triples.
- **Session 3** — Knowledge Graph Embedding with PyKEEN (TransE, DistMult), link prediction evaluation.
- **Session 4** — Retrieval-Augmented Generation: local LLM (Gemma 2B) translates natural language questions to SPARQL, queries the RDF graph, and returns structured answers.

---

## Demo Screenshot

The CLI demo (cell 112) produces the following output comparing RAG vs. baseline:

```
============================================================
 Q: Which organization owned Jaguar Land Rover?
============================================================
--- RAG (SPARQL on graph) ---
SPARQL: SELECT ?owner WHERE {
          auto:Jaguar_Land_Rover_Automotive_PLC auto:ownBy ?owner }
[1 result(s)]
   owner
   ─────
   http://automotive-crisis.org/kb/Tata_Motors

--- Baseline (No RAG) ---
I am unable to access real-time or specific knowledge sources,
therefore I cannot provide the owner of Jaguar Land Rover.

============================================================
 Q: Mention frequency of Volkswagen?
============================================================
--- RAG (SPARQL on graph) ---
[1 result(s)]
   freq
   ────
   19
```

> RAG returns a grounded, structured fact from the knowledge graph. The baseline LLM refuses to answer because it has no access to the private KB.

---

## Data Notes

- `automotive_kb.nt` (~70 MB, ~125,873 triples) and `train_full.txt` / `valid_full.txt` / `test_full.txt` are large files.  
  If they exceed GitHub's 100 MB file limit, use [Git LFS](https://git-lfs.com/) or upload them to an external host (Google Drive, Zenodo) and link them here.
- Smaller splits `train_20k.txt`, `train_50k.txt` (and their valid/test counterparts) are included for reproducibility without Git LFS.

