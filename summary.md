# GOAT-TS-SUPERLITE: Repository Summary

## What Is This?

**GOAT-TS-SUPERLITE** is a lightweight **Contextual Information Generator (CIG)** — a system that takes text as input, extracts concepts from it, runs graph-based influence propagation across those concepts, and surfaces exploratory hypotheses about relationships between them.

It is designed to run **fully offline** with **no external APIs or LLMs**, on **low-end hardware** (tested down to machines with < 4 GB RAM and 2 CPU cores). The optional Rust extension accelerates the core computation, but the system falls back gracefully to pure Python when Rust is unavailable.

---

## What Does It Do?

The system follows a four-step pipeline:

1. **Ingest** — Raw text is split into sentences and concept nodes are extracted using rule-based NLP (regex + stopword filtering, no NLTK or spaCy). Words that co-occur in the same sentence are linked with `relates_to` edges.

2. **Wave propagation** — Influence spreads through the graph over multiple "ticks". Each node's influence is updated as a blend of its own current value and the weighted sum of its neighbours, decayed by a configurable factor. Propagation stops early once influence values converge (delta < epsilon). This is the compute-intensive step; the Rust extension parallelises it with Rayon.

3. **Idea map** — Nodes are ranked by final influence score. The top nodes and their strongest outgoing edges form the "idea map" — a high-level view of what concepts matter most in the input.

4. **Hypothesis generation** — The engine pairs high-influence nodes with low-influence nodes to identify "tensions" and generates exploratory hypotheses, e.g. *"Explore connection between ethics and policy under tension 0.23."* These can be output as structured JSON or plain text.

The system can also generate **instruction-tuning JSONL datasets** from raw text files, suitable for fine-tuning small LLMs (e.g. TinyLlama) using PEFT/LoRA.

---

## Tech Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **Python 3.10+** | Core language | Config, ingestion, graph management, CLI, data generation |
| **Rust (optional)** | Extension via PyO3 | Fast, parallel wave propagation (`goat_ts_core`) |
| **SQLite** | Built-in | Optional persistence of the knowledge graph |
| **PyYAML** | `pyyaml` | Config file parsing |
| **psutil** | `psutil` | Hardware autodetection (RAM, CPU cores) |
| **Rayon** | Rust crate | Work-stealing parallelism for wave propagation |
| **petgraph** | Rust crate | Graph data structures |
| **sysinfo** | Rust crate | Hardware detection from Rust |
| **Maturin** | Build tool | Compiles the Rust extension into a Python-importable module |

---

## Project Structure

```
GOAT-TS-DEVELOPMENT/
├── run.py                    # Main entry point: --seed | --text | --file
├── run_tests.py              # Test runner (no pytest required)
├── benchmark.py              # Performance timing + hardware limit reporting
├── config.yaml               # Default config (graph backend, wave params)
├── requirements.txt          # Python dependencies
├── README.md                 # User-facing documentation
├── OPTIMIZATION.md           # Tuning and profiling guide
│
├── python/
│   ├── chat_ingest.py        # Text → concept nodes + co-occurrence edges
│   ├── generate_dataset.py   # Raw texts → JSONL for LLM fine-tuning
│   ├── bindings/
│   │   └── rust_bindings.py  # Imports goat_ts_core (Rust extension)
│   └── goat_ts_cig/
│       ├── interface.py      # Full CIG workflow + CLI argument parsing
│       ├── config.py         # YAML/JSON config loader + hardware autodetection
│       ├── knowledge_graph.py # In-memory graph with optional SQLite persistence
│       ├── ts_engine.py      # Wave propagation (Rust or Python fallback)
│       ├── cig_generator.py  # Builds the idea map (nodes ranked by influence)
│       ├── hypothesis_engine.py # Tension-based hypothesis generation
│       └── utils.py          # Graph format conversion for Rust interop
│
├── rust/
│   ├── Cargo.toml            # Rust package: goat-ts-core v1.0.0
│   └── src/
│       ├── lib.rs            # PyO3 module registration
│       ├── wave_engine.rs    # Double-buffered wave propagation with Rayon
│       └── graph_engine.rs   # Graph data structures
│
├── tests/
│   ├── test_cig.py           # Unit tests (propagation, graph CRUD, smoke test)
│   └── test_data_gen.py      # Data generation tests
│
├── examples/
│   ├── raw_texts/sample.txt  # Example input text
│   └── sample_usage.py       # Example programmatic usage
│
└── data/                     # Generated DB and JSONL files (gitignored)
```

---

## How to Run It

### Setup (Python only)

```bash
python -m venv venv
source venv/bin/activate       # Linux/macOS
# venv\Scripts\activate        # Windows
pip install -r requirements.txt
```

### Run the CIG

```bash
# From a seed concept (uses persisted graph)
python run.py --seed "AI ethics"

# From raw text (ingest + propagate + hypotheses)
python run.py --text "Machine learning and AI ethics are important."

# From a file, with JSON output
python run.py --file examples/raw_texts/sample.txt --json
```

### Build the Rust Extension (optional, for better performance)

```bash
pip install maturin
cd rust
# On Python 3.13+: export PYO3_USE_ABI3_FORWARD_COMPATIBILITY=1
python -m maturin develop
cd ..
```

### Generate Instruction-Tuning Data

```bash
# From a folder of .txt/.md files
python python/generate_dataset.py --input examples/raw_texts --output data/train.jsonl

# Alpaca format, capped at 20 examples
python python/generate_dataset.py --input examples/raw_texts -n 20 --format alpaca
```

### Run Tests

```bash
python run_tests.py
# → TS propagation, wave convergence, graph CRUD, full CIG smoke test
```

---

## Configuration

The system reads `config.yaml` (or `config.json`) from the repo root. All settings have sensible defaults:

```yaml
graph:
  backend: sqlite
  path: data/cig_graph.db
  in_memory_max_nodes: 1000

wave:
  ticks: 20
  decay: 0.15
  epsilon: 0.01
  max_ticks: 50

system:
  max_threads: null
  lazy_load: true
```

Environment variable overrides are also supported:

| Variable | Effect |
|----------|--------|
| `CIG_MAX_TICKS` | Override maximum wave propagation ticks |
| `CIG_BATCH_SIZE` | Override batch size for data generation |
| `CIG_PARALLELISM` | `0`/`false` to disable Rayon parallelism |
| `CIG_GRAPH_PATH` | Override the SQLite graph database path |

### Hardware Autodetection

At startup, the system detects available RAM and CPU cores (via `psutil` in Python, `sysinfo` in Rust) and adjusts its behaviour automatically:

| RAM | Max ticks | Batch size | Parallelism |
|-----|-----------|-----------|-------------|
| < 4 GB | 5–25 | 50–256 | Disabled |
| 4–8 GB | 15–50 | 200–512 | Enabled |
| ≥ 8 GB | 15–100 | 200–1024 | Enabled |

Parallelism is always disabled on machines with fewer than 2 physical CPU cores.

---

## Programmatic API

```python
from goat_ts_cig.interface import run_cig, run_cig_from_text

# From a seed concept
result = run_cig("quantum computing", output_json=True)
# → {"seed": "...", "idea_map": [...], "hypotheses": [...], ...}

# From raw text
result = run_cig_from_text("AI and ethics intersect in policy.", output_json=True)
# → {"source": "...", "idea_map": [...], "hypotheses": [...], ...}
```
