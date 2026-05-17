# Wikipedia Network Backboning

Pipeline for building a multilingual Wikipedia knowledge network and producing a corrected, weighted edge list for downstream analysis.

---

## Pipeline Overview

```
┌─────────────────────────────────────────────────────────────┐
│  Stage 1: Entity Filtering                                  │
│  notebooks/filter_by_languages.ipynb                        │
│                                                             │
│  Input:  data/cross-verified-database.csv.gz                │
│  Output: data/entities_filtered_by_languages.csv            │
│  Purpose: Keep entities with articles in all target langs   │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  Stage 2: Graph Building                                    │
│  python src/main.py build                                   │
│                                                             │
│  Input:  data/entities_filtered_by_languages.csv            │
│  Output: data/out/graph.db (DuckDB database)                │
│  Purpose: Fetch Wikipedia articles and links for all nodes  │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  Stage 3: Weight Assignment                                 │
│  python src/main.py weight                                  │
│                                                             │
│  Input:  data/out/graph.db                                  │
│  Output: data/out/SpotlightWeightSource_MMDD_HHMM.csv       │
│  Purpose: Count entity mentions via DBpedia Spotlight       │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  Stage 4: Full-Match Counting                               │
│  notebooks/edge_fullmatch_count.ipynb                       │
│                                                             │
│  Input:  SpotlightWeightSource_MMDD_HHMM.csv + graph.db     │
│  Output: SpotlightWeightSource_MMDD_HHMM_fullmatch.csv      │
│  Purpose: Add exact substring match count per edge          │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  Stage 5: Spurious Edge Correction                          │
│  notebooks/spurious_edge_correction.ipynb                   │
│                                                             │
│  Input:  SpotlightWeightSource_MMDD_HHMM_fullmatch.csv      │
│  Output: SpotlightWeightSource_MMDD_HHMM_fullmatch_corrected.csv │
│  Purpose: Replace weights on pseudo-self-loops with         │
│           exact match counts                                │
└─────────────────────────────────────────────────────────────┘
```

**Final outputs**:
- `data/entities_filtered_by_languages.csv` — filtered entity metadata
- `data/out/SpotlightWeightSource_MMDD_HHMM_fullmatch_corrected.csv` — corrected weighted edge list

---

## Data Schema Conventions

Wikidata identifier column naming:

- **`wikidata_code`**: Input column from cross-verified database
- **`wikidata_id`**: Output column after filtering (metadata files)
- **`source_wikidata_id`, `target_wikidata_id`**: Edge data columns
- **Format**: Always includes Q prefix (e.g., `Q42`, `Q5`)

**Pipeline column transformations**:
1. Cross-verified database: `wikidata_code` column
2. `filter_by_languages.ipynb`: Renames `wikidata_code` → `wikidata_id`
3. `python src/main.py build`: Reads `wikidata_code` from input CSV
4. `python src/main.py weight`: Outputs `source_wikidata_id`, `target_wikidata_id` in edges CSV

---

## Prerequisites

**System Requirements:**
- Python 3.10+
- 16GB RAM (32GB recommended for large datasets)
- ~50GB disk space
- Docker

**External Services:**
- DBpedia Spotlight instances (must be configured and running before weight assignment)

**Data:**
- Cross-verified notable entities database (see Data Sources section below)

---

## Installation

```bash
uv sync
source .venv/bin/activate   # On Windows: .venv\Scripts\activate
```

---

## Configuration

Copy `.env.example` to `.env` and edit as needed:

```bash
cp .env.example .env
```

| Variable | Description | Default |
|----------|-------------|---------|
| `SPOTLIGHT_PORTS` | Language:port mapping for Spotlight instances | `it:2223,en:2221,es:2225,fr:2224,de:2222` |
| `SPOTLIGHT_MIN_CONFIDENCE` | Minimum confidence for entity recognition | `0.8` |
| `DEFAULT_LANGUAGES` | Default languages for processing | `it,en,es,fr,de` |
| `WIKIPEDIA_USER_AGENT` | User agent for Wikipedia API requests | `WikipediaBiasProject/1.0 (...)` |
| `WIKIPEDIA_ACCESS_TOKEN` | Optional API token for higher rate limits | (empty) |

### DBpedia Spotlight Setup

Weight assignment requires local DBpedia Spotlight instances:

1. Configure ports in `.env`:
   ```
   SPOTLIGHT_PORTS=it:2223,en:2221,es:2225,fr:2224,de:2222
   ```

2. Generate docker-compose file (optional):
   ```bash
   python scripts/generate_spotlight_compose.py > container/spotlight-compose.yml
   ```

3. Start Spotlight services:
   ```bash
   cd container
   docker-compose -f spotlight-compose.yml up -d
   ```

---

## CLI Commands

All commands support `--log-level` (choices: `DEBUG`, `INFO`, `WARNING`, `ERROR`, `CRITICAL`; default: `INFO`).

### 1. Build a Graph

```bash
python src/main.py build \
    --input data/entities_filtered_by_languages.csv \
    --output data/out/graph.db \
    --languages en de fr it es
```

**Arguments:**
- `--input` (required): Path to CSV with a `wikidata_code` or `wikidata_id` column
- `--output` (required): Output database filename
- `--languages` (optional): Language codes (default: `en`)
- `--limit` (optional): Limit entities for testing

### 2. Assign Weights

```bash
python src/main.py weight \
    --graph-db data/out/graph.db \
    --output-dir data/out \
    --languages en de fr it es \
    --refresh-notable-links
```

**Arguments:**
- `--graph-db` (required): Path to existing graph database
- `--output-dir` (optional): Output directory for CSV file (default: `data`)
- `--languages` (optional): Language codes to process (default: `en de it fr es`)
- `--checkpoint-file` (optional): Path to checkpoint file for resumable processing
- `--refresh-notable-links` (optional): Force refresh of notable_page_link table

**Output:**
- `data/out/SpotlightWeightSource_MMDD_HHMM.csv`
- Columns: `source_wikidata_id`, `target_wikidata_id`, `weight`, `language_code`

---

## Complete Pipeline Example

```bash
source .venv/bin/activate

# Stage 1 — run filter_by_languages.ipynb in Jupyter
# Output: data/entities_filtered_by_languages.csv

# Stage 2
python src/main.py build \
    --input data/entities_filtered_by_languages.csv \
    --output data/out/graph.db \
    --languages en de fr it es

# Stage 3
python src/main.py weight \
    --graph-db data/out/graph.db \
    --output-dir data/out \
    --languages en de fr it es \
    --checkpoint-file data/out/weight_checkpoint.json \
    --refresh-notable-links

# Stages 4-5 — run notebooks/edge_fullmatch_count.ipynb,
#              then notebooks/spurious_edge_correction.ipynb in Jupyter
# Final output: data/out/SpotlightWeightSource_MMDD_HHMM_fullmatch_corrected.csv
```

## Resuming Interrupted Weight Assignment

```bash
python src/main.py weight \
    --graph-db data/out/graph.db \
    --output-dir data/out \
    --checkpoint-file data/out/weight_checkpoint.json
```

---

## Logs

Logs are saved to the `logs/` directory:
- Build: `logs/builder_logfile_YYYYMMDD_HHMMSS.log`
- Weight: `logs/weighter_logfile_YYYYMMDD_HHMMSS.log`

---

## Statistical Methodology

### Weight Calculation

Edge weights represent **the number of entity mentions** detected by DBpedia Spotlight:

- **Source**: The Wikipedia article being analyzed
- **Target**: An entity mentioned within that article's text
- **Weight**: Count of mentions above confidence threshold (≥ 0.8)

### Spurious Edge Correction

Pseudo-self-loops arise when an entity linker confuses an entity with a family member sharing the same surname (e.g., "Darwin" → George Darwin instead of Charles Darwin). They are detected by requiring **both**:
1. Shared name components between source and target titles
2. Statistical outlier weight (z-score > 1.5) relative to the source node's distribution

Detected edges have their weight replaced with the exact full-match count (from stage 4), which is a more reliable signal.

---

## Data Sources

### Cross-Verified Database

- **File**: `data/cross-verified-database.csv.gz`
- **Source**: [BHHT Datascape](https://medialab.github.io/bhht-datascape/)
- **Citation**: Laouenan, M., Bhargava, P., Eyméoud, J.-B., Gergaud, O., Plique, G., & Wasmer, E. (2022). A cross-verified database of notable people, 3500BC-2018AD. *Scientific Data*, 9(1), 290. https://doi.org/10.1038/s41597-022-01369-4

  ```bibtex
  @article{bhht3,
    author = {Laouenan, Morgane and Bhargava, Palaash and Eyméoud, Jean-Benoît and Gergaud, Olivier and Plique, Guillaume and Wasmer, Etienne},
    title = {A cross-verified database of notable people, 3500BC-2018AD},
    journal = {Scientific Data},
    publisher = {Nature Publishing Group},
    year = {2022},
    volume = {9},
    number = {1},
    pages = {290},
    doi = {10.1038/s41597-022-01369-4},
    url = {https://doi.org/10.1038/s41597-022-01369-4}
  }
  ```

- **License**: CC-BY-SA. Derived from Wikipedia (CC-BY-SA) and Wikidata (CC0).
