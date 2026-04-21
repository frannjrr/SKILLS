---
name: graphify-legaltech
description: >
  Use this skill whenever the user needs to build, query, or extend a Graphify
  knowledge graph from a corpus of documents — especially legal, regulatory, or
  structured Markdown repositories (e.g. legalize-es, Código Civil, LGT, IRPF).
  Trigger on any of: "graphify", "knowledge graph from docs", "graph RAG",
  "legal graph", "graphify install", "build a graph from markdown",
  "consultar grafo legal", "cross-reference between laws", "MCP graph server",
  "Graph-RAG pipeline", or any request to ingest folders of documents into a
  queryable graph structure. Always use this skill before writing any code that
  touches graphify, graph.json, GRAPH_REPORT.md, or the graphify MCP server.
---

# Graphify LegalTech Skill

Graphify (PyPI: `graphifyy`) turns a folder of files into a queryable
knowledge graph. This skill covers installation, corpus design, graph building,
querying, MCP server usage, and LegalTech-specific patterns for Spanish law
corpora (legalize-es, Código Civil, LGT, IRPF, etc.).

---

## 1. Installation

```bash
# Always install with double-y on PyPI
pip install graphifyy

# Install the skill into the AI coding assistant
graphify install                          # Claude Code (Linux/Mac/Windows auto-detect)
graphify install --platform windows       # explicit Windows
graphify install --platform codex         # OpenAI Codex
graphify install --platform cursor        # Cursor
graphify install --platform gemini        # Gemini CLI
graphify install --platform copilot       # GitHub Copilot CLI

# Optional extras
pip install 'graphifyy[office]'           # .docx/.xlsx ingestion
pip install 'graphifyy[video]'            # video/audio transcription
pip install 'graphifyy[mcp]'              # MCP server
```

> **Common mistake:** `pip install graphify` (single-y) installs an unrelated package.
> The CLI command is always `graphify` regardless.

---

## 2. Building a Graph

```bash
/graphify .                               # current directory
/graphify ./legalize-es                   # specific corpus folder
/graphify ./legalize-es --mode deep       # more aggressive INFERRED edge extraction
/graphify ./legalize-es --directed        # preserves edge direction (A→B)
/graphify ./legalize-es --update          # re-extract only changed files
/graphify ./legalize-es --no-viz          # skip HTML, produce JSON + report only
/graphify ./legalize-es --cluster-only    # rerun clustering without re-extraction
```

### Output structure

```
graphify-out/
├── graph.html         # interactive visual (click nodes, search, filter)
├── GRAPH_REPORT.md    # god nodes, surprising connections, suggested questions
├── graph.json         # persistent queryable graph
└── cache/             # SHA256 cache — only changed files re-processed on next run
```

### .graphifyignore

Create at corpus root to exclude noise:

```
# .graphifyignore
node_modules/
.git/
*.generated.md
exposicion_de_motivos/      # exclude preambles from node extraction
```

---

## 3. Extraction Schema

Every file produces nodes and edges following this contract:

```json
{
  "nodes": [
    {
      "id": "lgt_art_3_1",
      "label": "LGT Art.3.1 — Principios de ordenación",
      "source_file": "lgt/titulo_preliminar/articulo_3.md",
      "source_location": "L12"
    }
  ],
  "edges": [
    {
      "source": "irpf_art_14",
      "target": "lgt_art_3_1",
      "relation": "remite_a",
      "confidence": "EXTRACTED"
    }
  ]
}
```

### Confidence labels

| Label | Meaning |
|-------|---------|
| `EXTRACTED` | Relation explicitly stated in source (e.g. "conforme al artículo 3 LGT") |
| `INFERRED` | Reasonable deduction (co-occurrence, semantic similarity) |
| `AMBIGUOUS` | Uncertain — flagged in GRAPH_REPORT.md for human review |

---

## 4. Legal Corpus Node Mapping (LegalTech Pattern)

Map Spanish legal hierarchy to Graphify node IDs deterministically:

```
{ley}_{division}_{id}
```

| Level | Example ID | Example Label |
|-------|-----------|---------------|
| Ley | `lgt` | Ley General Tributaria |
| Título | `lgt_tit_1` | LGT Título I |
| Capítulo | `lgt_tit_1_cap_2` | LGT Título I Capítulo II |
| Sección | `lgt_tit_1_cap_2_sec_1` | LGT T.I C.II Sección 1ª |
| Artículo | `lgt_art_58` | LGT Artículo 58 |
| Apartado | `lgt_art_58_1` | LGT Art.58.1 |
| Letra | `lgt_art_58_1_a` | LGT Art.58.1.a) |

**Critical rule:** Never split an *apartado* or *letra* across multiple nodes.
A single semantic unit (one numbered/lettered paragraph) = one leaf node.

### Metadata to embed in node attributes

```json
{
  "id": "lgt_art_58_1",
  "label": "LGT Art.58.1 — La deuda tributaria",
  "source_file": "lgt/titulo_ii/capitulo_iv/articulo_58.md",
  "source_location": "L45",
  "ley": "LGT",
  "numero_articulo": 58,
  "apartado": "1",
  "fecha_vigencia_desde": "2004-12-18",
  "fecha_vigencia_hasta": null,
  "ambito_territorial": "estatal",
  "es_exposicion_motivos": false,
  "modificado_por": ["Ley 34/2015"],
  "tipo_nodo": "articulo_apartado"
}
```

---

## 5. Cross-Referencing Between Laws

When a Markdown article contains a legal reference (e.g. "conforme al artículo 3 de la LGT"), Graphify extracts it as an `EXTRACTED` edge if the node IDs are consistent across corpora.

**Strategy:**

1. Use a shared node-ID registry (`node_registry.json`) across all law folders.
2. Teach the extractor to recognize citation patterns:
   - `"artículo \d+ (de la |del )?(LGT|IRPF|CC)"` → edge type `remite_a`
   - `"según lo dispuesto en..."` → edge type `segun_lo_dispuesto`
   - `"sin perjuicio de..."` → edge type `sin_perjuicio_de`
3. Mark cross-law edges with `cross_law: true` attribute for priority traversal.

```json
{
  "source": "irpf_art_14_1",
  "target": "lgt_art_3_1",
  "relation": "remite_a",
  "confidence": "EXTRACTED",
  "cross_law": true,
  "citation_text": "conforme al artículo 3 de la LGT"
}
```

---

## 6. Querying the Graph

### CLI queries (no assistant needed)

```bash
graphify query "¿Qué artículos de la LGT regula la prescripción?"
graphify query "camino entre IRPF Art.14 y LGT Art.58" --dfs
graphify query "¿Qué es la base imponible?" --budget 1500
graphify path "irpf_art_14" "lgt_art_58"         # shortest path
graphify explain "lgt_art_58"                      # plain-language explanation
```

### MCP server (for agent integration)

```bash
# Start the MCP stdio server
python -m graphify.serve graphify-out/graph.json

# .mcp.json (project root)
{
  "mcpServers": {
    "graphify-legal": {
      "type": "stdio",
      "command": ".venv/bin/python3",
      "args": ["-m", "graphify.serve", "graphify-out/graph.json"]
    }
  }
}
```

Available MCP tools: `query_graph`, `get_node`, `get_neighbors`, `shortest_path`.

### Graph-RAG query pattern for the agent

```python
# 1. Retrieve — focused subgraph
subgraph = mcp.query_graph(
    "base imponible IRPF reducciones",
    budget=1500
)

# 2. Augment — inject into LLM prompt
prompt = f"""
Responde SOLO con base en este subgrafo legal. Cita el nodo fuente exacto.
Si no está en el grafo, di "No encontrado en corpus".

SUBGRAFO:
{subgraph}

PREGUNTA: {user_question}
"""

# 3. Generate — LLM answers citing node IDs
response = llm.complete(prompt)
```

---

## 7. Always-On Integration (Claude Code)

```bash
graphify claude install
```

This writes a `CLAUDE.md` section and installs a **PreToolUse hook** that fires
before every Glob/Grep call. Claude navigates via `GRAPH_REPORT.md` (structure)
before searching raw files.

---

## 8. Incremental Updates

```bash
# Add a new BOE amendment
graphify add https://boe.es/... --author "BOE" --contributor "tu-nombre"

# Re-extract only changed files
/graphify ./legalize-es --update

# Watch mode (auto-rebuild on file save)
/graphify ./legalize-es --watch

# Git hook (rebuild on commit/branch switch)
graphify hook install
```

---

## 9. LegalTech Anti-Patterns to Avoid

| Anti-pattern | Problem | Fix |
|---|---|---|
| Split an *apartado* at mid-sentence | Breaks semantic unit | One node = one numbered/lettered paragraph |
| Mix *Exposición de Motivos* with articulado | Preamble is not normative | Use `.graphifyignore` or `es_exposicion_motivos: true` flag |
| No `fecha_vigencia` metadata | Agente puede citar norma derogada | Always inject vigency dates as node attributes |
| Node IDs non-deterministic (UUIDs) | Cross-referencing breaks | Use `{ley}_{nivel}_{numero}` scheme strictly |
| Dump full `graph.json` into prompt | Token explosion | Use `graphify query --budget N` to extract subgraph |

---

## 10. Quick Reference

```bash
pip install graphifyy                   # install
graphify install                        # register skill
/graphify ./legalize-es                 # build graph
/graphify ./legalize-es --directed      # preserve citation direction
graphify query "pregunta legal"         # query without assistant
graphify path "nodo_a" "nodo_b"        # find path between articles
python -m graphify.serve graph.json     # start MCP server
graphify claude install                 # always-on hook
graphify hook install                   # git hook
```

See `references/legal-schema.md` for the full Spanish legal hierarchy schema.
See `references/mcp-integration.md` for the full Graph-RAG agent integration pattern.
