# ACCS — Automated Codebase Comprehension System

A single static C99 binary that analyzes any codebase and produces a structured report:
**what each file does, how files relate, and which files matter most.**

No runtime dependencies. No API calls. No internet required. Cross-compiles anywhere.

---

## Architecture

Three engines working together:

1. **Tree-sitter engine** — parses source files, extracts actual import/export statements deterministically. Builds the real dependency graph. No guessing.
2. **ML classifier** — small feedforward neural network loaded from `weights.bin`. Takes file-level features (filename patterns, directory position, token frequencies, AST statistics) and predicts file role: `controller`, `model`, `test`, `util`, `config`, `middleware`, etc. Trained offline using AI-labeled data.
3. **PageRank** — runs on the real dependency graph from engine 1. Produces importance scores. Deterministic, no ML needed.

A template formatter combines all three outputs into human-readable summaries.

## Two pieces

### Training pipeline (`/training`, throwaway tooling)
- Feed codebases to AI API
- the AI labels each file with role, purpose, features
- Train feedforward net on those labels
- Export `weights.bin`

### C99 inference binary (the product)
- Built entirely on **Canon-C** (arenas, Results, slices, ownership)
- Scanner → tree-sitter parser → feature extractor → ML classifier → graph builder → PageRank → template reporter
- Ships as one static binary with `weights.bin`

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

1. Download grammar to `grammars/<newlang>/parser.c`
2. Add an entry to `src/lang_table.c` (~10 lines)
3. Rebuild: `make`

## Supported languages

Extensible via `lang_table.c`. Selectable by design.

## Output formats

`json` · `dot` (Graphviz) · `summary` (human-readable)

## Project layout

```
accs/
├── Makefile
├── include/        # all core types (.h)
├── src/            # implementation (.c)
├── vendor/
│   ├── Canon-C/    # arenas, Results, slices
│   └── tree-sitter/
├── grammars/       # one parser.c per language
├── training/       # Training pipeline (separate)
└── build/          # make output
```

## How ACCS differs from CodeMap

| | CodeMap | ACCS |
|---|---|---|
| Runtime LLM | Required every session | None — distilled into `weights.bin` |
| Form factor | Research prototype | Single static binary |
| Mode | Interactive exploration | Batch analysis |
| Stack | Python / JS / web | C99 + Canon-C + weights.bin |
| ML approach | Uses LLM directly each call | Uses an AI to teach a small model, then ships without it |

Designed for CI pipelines, onboarding docs, automated audits — and applicable to defensive security workflows (e.g. architectural overview of a Ghidra export of an unknown binary).

## License

MIT — see [LICENSE](LICENSE).
