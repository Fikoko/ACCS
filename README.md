# ACCS — Automated Codebase Comprehension System

**The offline, batch-report alternative to LLM-served code graphs.**

A single static C99 binary that analyzes a codebase and produces a report — what each file does, how files relate, and which files matter most. The intelligence is baked in at training time via a distilled classifier; nothing calls out to an AI at runtime.

---

## Why this exists

Most current code-graph tools (Codebase-Memory, Code-Graph-RAG, Graph-sitter, tree-sitter-analyzer) are built to serve graphs to LLM agents over MCP, interactively. ACCS is the other shape:

- **Offline forever.** No API calls, no network, no runtime model. Useful in air-gapped environments, security workflows, regulated industries, and CI pipelines that can't depend on external services.
- **Batch reports, not query servers.** Point it at a repo, get JSON / DOT / human-readable output. Fits onboarding docs, automated audits, and CI checks rather than chat-style exploration.
- **Distilled, not hosted.** A small classifier learns file-role patterns from AI-labeled training data once, then ships as weights inside the binary.

If you want an interactive code-graph for an LLM agent, use Codebase-Memory or similar. If you want a static binary that produces a report and exits, ACCS is for that.

## Architecture

Three engines:

1. **Tree-sitter engine** — parses sources, extracts real import/export edges deterministically across 8 languages.
2. **Distilled classifier** — small feedforward neural net loaded from `weights.bin`. Takes a fixed-width feature vector per file (filename patterns, directory position, token frequencies, AST statistics) and predicts a role: `controller`, `model`, `test`, `util`, `config`, `middleware`, `entrypoint`, `build`. Trained offline using AI-labeled data; no runtime AI.
3. **PageRank** — runs on the dependency graph from engine 1. Produces importance scores. Deterministic.

A template formatter combines all three into the chosen output format.

## Two pieces

### C99 inference binary (the product)
- Built on **Canon-C** (arenas, Results, slices, ownership).
- Pipeline: scanner → tree-sitter parser → feature extractor → classifier → graph builder → PageRank → reporter.
- Ships as one static binary plus `weights.bin`.

### Training pipeline (`/training`, throwaway tooling)
- Crawl representative codebases.
- Featurize each file into a fixed-width vector.
- Label via AI API.
- Train feedforward net.
- Export `weights.bin`.

The training pipeline runs once. The binary runs anywhere, forever, with no external dependencies.

## Build

```bash
make            # build
make clean      # clean
```

## Usage

```bash
./build/accs ./my-project                      # JSON report to stdout
./build/accs ./my-project --format dot         # Graphviz DOT graph
./build/accs ./my-project --format summary     # human-readable summary
./build/accs ./my-project -o report.json       # write to file
```

## Cross-compile

```bash
CC=aarch64-linux-gnu-gcc make                  # Raspberry Pi
CC=x86_64-w64-mingw32-gcc make                 # Windows
CC=musl-gcc LDFLAGS=-static make               # fully static Linux
```

## Add a language

1. Download grammar to `grammars/<newlang>/parser.c`.
2. Add an entry to `src/lang_table.c` (~10 lines).
3. Rebuild: `make`.

## Supported languages

Extensible via `lang_table.c`. Selectable by design.

## Output formats

`json` · `dot` (Graphviz) · `summary` (human-readable)

## Project layout

```
accs/
├── Makefile
├── include/        # core types (.h)
├── src/            # implementation (.c)
├── vendor/
│   ├── Canon-C/    # arenas, Results, slices
│   └── tree-sitter/
├── grammars/       # one parser.c per language
├── training/       # Python training pipeline (separate)
└── build/          # make output
```

## How ACCS sits next to similar tools

| | Codebase-Memory / Code-Graph-RAG / Graph-sitter | ACCS |
|---|---|---|
| Primary consumer | LLM agents over MCP | Humans, CI pipelines |
| Mode | Interactive query server | Batch report |
| Runtime AI | Yes (LLM in the loop) | No |
| ML role labels | None or LLM-on-demand | Small distilled classifier in the binary |
| Form factor | Daemon + DB (often SQLite) | Single static binary + `weights.bin` |
| Network required | Usually yes | No |

The shared core — Tree-sitter parsing, dependency graph, polyglot — is well-trodden ground at this point. What ACCS does differently is the deployment shape and the distilled-classifier approach.

## Status

Early-stage scaffold. The deterministic parts (scanner, parser, graph, PageRank, reporters) are the first milestone. The classifier and training pipeline are the second.

## License

MIT — see [LICENSE](LICENSE).
