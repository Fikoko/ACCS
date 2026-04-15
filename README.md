# ACCS — Automated Codebase Comprehension System

**Small footprint. Offline. Fast. Cross-platform.**

A single static C99 binary that analyzes a codebase and produces a report — what each file does, how files relate, and which files matter most. The intelligence is baked in at training time via per-language distilled classifiers; nothing calls out to an AI at runtime.

---

## Design goals

Four pillars, in priority order. Every design decision serves these:

1. **Small footprint.** Ship only what you use. A Python-only project pulls a Python-only weights file. Quantized models keep individual weight files in the hundreds of KB.
2. **Offline.** No network, no API calls, no runtime model. Air-gapped, regulated, security-sensitive environments are first-class citizens.
3. **Fast.** Batch report on a 100k-line repo in seconds, not minutes. Tree-sitter parses, fixed-width feature vectors classify, deterministic graph algorithms rank.
4. **Cross-platform.** One C99 codebase. Cross-compiles to Linux (glibc, musl), macOS, Windows, ARM (Raspberry Pi, Apple Silicon, server ARM).

## Why this exists

Most current code-graph tools (Codebase-Memory, Code-Graph-RAG, Graph-sitter, tree-sitter-analyzer) serve graphs to LLM agents over MCP, interactively. ACCS is the other shape:

- **Offline forever.** No network, no runtime model. Useful in air-gapped environments, security workflows, regulated industries, and CI pipelines that can't depend on external services.
- **Batch reports, not query servers.** Point it at a repo, get JSON / DOT / human-readable output. Fits onboarding docs, automated audits, and CI checks rather than chat-style exploration.
- **Distilled, not hosted.** Small per-language classifiers learn file-role patterns from AI-labeled training data once, then ship as weights next to the binary.

If you want an interactive code-graph for an LLM agent, use Codebase-Memory or similar. If you want a static binary that produces a report and exits, ACCS is for that.

## Architecture

Three engines:

1. **Tree-sitter engine** — parses sources, extracts real import/export edges deterministically. One grammar per language, loaded on demand.
2. **Per-language distilled classifier** — small feedforward neural net per language, loaded from `weights/<lang>.bin`. Each takes a fixed-width feature vector (filename patterns, directory position, token frequencies, AST statistics) and predicts a role: `controller`, `model`, `test`, `util`, `config`, `middleware`, `entrypoint`, `build`. Trained offline using AI-labeled data; no runtime AI. Quantized (int8) to keep each weight file in the hundreds of KB.
3. **PageRank** — runs on the dependency graph from engine 1. Produces importance scores. Deterministic, language-agnostic.

A template formatter combines all three into the chosen output format.

### Why per-language weights

A Rust idiom dilutes a Python classifier and vice versa. Specializing per language means each model can be small *and* accurate. It also means:

- **You only ship what you need.** Analyzing a Python project? Drop in `python.bin`. Don't ship the JavaScript model.
- **Languages can be added independently.** A new language pack = one tree-sitter grammar + one weights file + a `lang_table.c` entry. No retraining the world.
- **Bug in one model doesn't ship to other languages.** Versioned per-language.
- **The architecture matches tree-sitter's own structure** — one grammar per language, loaded on demand. Consistent end-to-end.

Polyglot repos load multiple models. Each file is routed to the classifier for its language. The graph and PageRank stages don't care about language — they operate on the unified import graph.

### Weights distribution

Weight files live in a `weights/` directory next to the binary. Users copy in only the language packs they need:

```
accs
weights/
├── python.bin
├── javascript.bin
└── rust.bin
```

The binary scans the repo, sees which languages are present, and loads only the matching weight files. Missing weights = file gets an `unknown` role label but still appears in the dependency graph and PageRank ranking. The deterministic parts always work; the ML is additive.

## Two pieces

### C99 inference binary (the product)
- Built on **Canon-C** (arenas, Results, slices, ownership).
- Pipeline: scanner → tree-sitter parser → feature extractor → per-language classifier → graph builder → PageRank → reporter.
- Ships as one static binary plus the language weight files you choose.

### Training pipeline (`/training`, throwaway tooling)
- Crawl representative codebases per language.
- Featurize each file into a fixed-width vector.
- Label via AI API.
- Train one feedforward net per language.
- Quantize and export to `weights/<lang>.bin`.

The training pipeline runs once per language. The binary runs anywhere, forever, with no external dependencies.

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
./build/accs ./my-project --weights ./weights  # custom weights directory
```

## Cross-compile

```bash
CC=aarch64-linux-gnu-gcc make                  # Raspberry Pi / ARM
CC=x86_64-w64-mingw32-gcc make                 # Windows
CC=musl-gcc LDFLAGS=-static make               # fully static Linux
```

## Add a language

1. Download grammar to `grammars/<newlang>/parser.c`.
2. Add an entry to `src/lang_table.c` (~10 lines).
3. Train a classifier (`training/`) and drop the resulting `weights/<newlang>.bin` next to the binary.
4. Rebuild: `make`.

Steps 1–2 alone give you dependency-graph and PageRank coverage for the language. Step 3 adds role classification.

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
├── weights/        # per-language weight files (not committed)
└── build/          # make output
```

## How ACCS sits next to similar tools

| | Codebase-Memory / Code-Graph-RAG / Graph-sitter | ACCS |
|---|---|---|
| Primary consumer | LLM agents over MCP | Humans, CI pipelines |
| Mode | Interactive query server | Batch report |
| Runtime AI | Yes (LLM in the loop) | No |
| ML role labels | None or LLM-on-demand | Per-language distilled classifier |
| Form factor | Daemon + DB (often SQLite) | Static binary + per-language weight files |
| Footprint | Tens of MB + dependencies | Binary + only the language packs you need |
| Network required | Usually yes | No |

The shared core — Tree-sitter parsing, dependency graph, polyglot — is well-trodden ground at this point. What ACCS does differently is the deployment shape, the per-language distilled classifiers, and the offline-first commitment.

## Status

Early-stage scaffold. Milestones:

1. **Deterministic core** — scanner, tree-sitter parser, resolver, graph, PageRank, reporters. Useful on its own.
2. **First language pack** — Python training pipeline → quantized `python.bin` → integrated classifier in the binary.
3. **Additional language packs** — JavaScript, TypeScript, Java, C, C++, Go, Rust, added one at a time.

## License

MIT — see [LICENSE](LICENSE).
