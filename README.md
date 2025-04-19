# MCP Server × PostgreSQL on MIMIC‑IV

> **Project goal:** Evaluate whether a Model Context Protocol (MCP) Server layered on Azure Database for PostgreSQL can match—or outperform—direct SQL while reducing developer effort for large‑scale clinical analytics.

---

## 1. Problem Statement

Clinical datasets such as MIMIC‑IV contain tens of millions of rows and hundreds of attributes. Writing and maintaining raw SQL against such breadth is error‑prone and slow. We ask:

_Can an MCP‑based abstraction simultaneously improve developer productivity **and** sustain (or enhance) runtime efficiency compared with traditional SQL‑only workflows?_

To answer this we compare direct SQL against an **MCP Server → Postgres** path on identical workloads—ranging from a simple `COUNT(*)` on `mimiciv_hosp.emar_detail` (≈ 87 M rows) to multi‑table note aggregations.

---

## 2. Repository Layout

```
.
├── mimic-postgres/          # Scripts to create schemas & ingest MIMIC‑IV CSVs
├── src/            # Fork of azure_postgresql_mcp with env configs
└── README.md              # This file
```

---

## 3. Prerequisites

- Python 3.10+
- Node 18+ (optional, for additional MCP tooling)
- **Azure Database for PostgreSQL – Flexible Server** (Standard_D4s v5 or larger)
- MIMIC‑IV credential & data download permission

```bash
# core libs
pip install psycopg[binary] rich click pandas matplotlib
# benchmarking
pip install httpx pytest pytest‑benchmark
# MCP server
pip install mcp[cli] azure-identity azure-mgmt-postgresqlflexibleservers
```

---

## 4. Dataset Loading

```bash
cd data-loading
psql "$PGURI" -f 00_create_mimiciv_note_schema.sql
psql "$PGURI" -f 01_create_emar_tables.sql
python 02_bulk_copy.py --csv-root /path/to/mimic-iv
```

Estimated ingest time on Standard_D4s: **~45 min** for 72 GB (11.7 M note rows + 87 M EMAR rows).

---

## 5. Starting the MCP Server

```bash
cd mcp-server
python azure_postgresql_mcp.py
```

Environment variables (`PGHOST`, `PGUSER`, `PGPASSWORD`, `PGDATABASE`) must be set or supplied in your Claude Desktop / VS Code configuration.

To use **Microsoft Entra** authentication instead of a password, set:

```bash
export AZURE_USE_AAD=True
export AZURE_SUBSCRIPTION_ID=...
export AZURE_RESOURCE_GROUP=...
```

---

## 6. Running Baselines

### 6.1 Direct SQL

```bash
python benchmarks/run_sql.py --query count_emar_detail
```

Outputs a single line like:

```
count=87,371,064 elapsed=101550.2 ms
```

### 6.2 MCP Server

```bash
python benchmarks/run_mcp.py --query count_emar_detail
```

Sample output:

```
count=87,371,064 elapsed=67560.1 ms
```

Multiple runs (default = 30) are aggregated into CSV under `benchmarks/results/`.

---

## 7. Reproducing Paper Figures

```bash
jupyter lab notebooks/03_make_plots.ipynb
```

Generates latency histograms and developer‑effort bar charts used in the final report.

---

## 8. Key Results (Preview)

| Query                        | Direct SQL |    MCP Server |  Δ (ms) |   Δ % |
| ---------------------------- | ---------: | ------------: | ------: | ----: |
| `COUNT(*)` on `emar_detail`  | 101,550 ms | **67,560 ms** | −33,990 | −33 % |
| LOC to implement workload #1 |         68 |        **23** |     −45 | −66 % |

Full tables in `benchmarks/results/final_summary.csv`.

---

## 9. Future Work

- Enable Entra ID + RBAC and re‑benchmark connection latency.
- Add pgvector semantic‑search benchmark via a custom MCP tool.
- Run clinician UX trials comparing MCP+Claude vs. SQL IDE.

---

## 10. License & Citation

MIT License — see `LICENSE`.

If you use this code or results, please cite:

```
Menon S.K., N. (2025). MCP Server × PostgreSQL on MIMIC‑IV: productivity and performance comparison. Georgia Tech CS 8803 Final Project Report.
```

---

## 11. Acknowledgements

- PhysioNet for access to the MIMIC‑IV dataset
- Azure Database for PostgreSQL product team
- Anthropic for the MCP specification and Claude Desktop tooling
