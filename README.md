# ACCS — Automated Codebase Comprehension System

**Small footprint. Offline. Fast. Cross-platform.**

A single static C99 binary that analyzes a codebase and produces a structural report — what each file does, how files relate, which files matter most, and where execution starts. The intelligence is baked in at training time via per-language distilled classifiers; nothing calls out to an AI at runtime.

---

## Design goals

Four pillars, in priority order. Every design decision serves these:

1. **Small footprint.** Ship only what you use. Quantized per-language weights stay in the hundreds of KB each.
2. **Offline.** No network, no API calls, no runtime model. Air-gapped, regulated, and security-sensitive environments are first-class citizens.
3. **Fast.** Batch report on a 100k-line repo in seconds, not minutes.
4. **Cross-platform.** One C99 codebase. Cross-compiles to Linux (glibc, musl), macOS, Windows, ARM.

## Why this exists

Most current code-graph tools (Codebase-Memory, Code-Graph-RAG, Graph-sitter, tree-sitter-analyzer) serve graphs to LLM agents over MCP, interactively. ACCS is the other shape:

- **Offline forever.** No network, no runtime model. Useful in air-gapped environments, security workflows, regulated industries, and CI pipelines that can't depend on external services.
- **Batch reports, not query servers.** Point it at a repo, get JSON / DOT / human-readable output.
- **Distilled, not hosted.** Small per-language classifiers learn file-role patterns from AI-labeled training data once, then ship as weights next to the binary.

If you want an interactive code-graph for an LLM agent, use Codebase-Memory or similar. If you want a static binary that produces a report and exits, ACCS is for that.

## Architecture

Five engines, in pipeline order:

1. **Tree-sitter engine** — parses sources, extracts deterministic facts: imports, function/method definitions, call sites, class definitions. One grammar per language, loaded on demand.
2. **Resolver** — maps imports to internal files where possible; treats unresolved imports as external terminal nodes in the graph.
3. **Per-language classifier** — small feedforward neural net per language, loaded from `weights/<lang>.bin`. Takes a fixed-width feature vector and outputs a probability distribution over roles (`controller`, `model`, `test`, `util`, `config`, `middleware`, `entrypoint`, `build`). The argmax is the label; the max probability is the confidence. Quantized to int8.
4. **Entry-point detector** — combines build/config-file declarations (Dockerfile CMD, package.json bin, Cargo.toml [[bin]], pyproject.toml scripts), language-specific structural markers (`if __name__ == "__main__"`, `func main()`, `public static void main`), import-graph position (low in-degree), and filename hints (tiebreaker only). Surfaces entry-point candidates with confidence scores.
5. **Graph + PageRank** — runs on the resolved import graph. Produces importance scores. Deterministic, language-agnostic.

A **template-based narrative formatter** sits at the end and combines all of the above into per-file paragraphs and a whole-codebase summary.

### Why per-language weights

A Rust idiom dilutes a Python classifier and vice versa. Specializing per language means each model can be small *and* accurate. It also means:

- **Ship only what you need.** Python-only project? Drop in `python.bin` and skip the rest.
- **Languages added independently.** A new language pack = grammar + weights file + `lang_table.c` entry. No retraining the world.
- **Bugs in one model don't ship to other languages.** Versioned per-language.
- **Architecture matches tree-sitter's own structure** — one grammar per language, loaded on demand.

Polyglot repos load multiple models. Each file is routed to the classifier for its language. The graph and PageRank stages don't care about language.

### Weights distribution

Weight files live in a `weights/` directory next to the binary. The binary scans the repo, sees which languages are present, and loads only the matching weight files. Missing weights → file gets an `unknown` role label but still appears in the dependency graph and PageRank ranking. **The deterministic parts always work; the ML is additive.**

## Report contents

Output layers, in increasing order of synthesis:

1. **Raw extraction** — per file: defined functions/classes, imports, call sites.
2. **Graph metrics** — PageRank scores, in/out degrees, identified entry-point candidates with confidence.
3. **Role labels** — per-file classification with confidence score.
4. **Per-file narratives** — short template-filled paragraphs describing each file's apparent role and shape. Templates hedge based on classifier confidence.
5. **Codebase summary** — aggregated narrative across all files (role distribution, entry points, key dependencies).

Each layer can be enabled or disabled via CLI flags.

## Build

```bash
make            # build
make clean      # clean
```

## Usage

```bash
./build/accs ./my-project                       # JSON report to stdout
./build/accs ./my-project --format dot          # Graphviz DOT graph
./build/accs ./my-project --format summary      # human-readable summary
./build/accs ./my-project --format narrative    # per-file template narratives
./build/accs ./my-project -o report.json        # write to file
./build/accs ./my-project --weights ./weights   # custom weights directory
./build/accs ./my-project --templates ./tpl     # custom narrative templates
```

## Cross-compile

```bash
CC=aarch64-linux-gnu-gcc make                   # Raspberry Pi / ARM
CC=x86_64-w64-mingw32-gcc make                  # Windows
CC=musl-gcc LDFLAGS=-static make                # fully static Linux
```

## Add a language

1. Download grammar to `grammars/<newlang>/parser.c`.
2. Add an entry to `src/lang_table.c` (~10 lines).
3. Train a classifier (`training/`) and drop the resulting `weights/<newlang>.bin` next to the binary.
4. Rebuild: `make`.

Steps 1–2 alone give dependency-graph, entry-point detection, and PageRank for the language. Step 3 adds role classification and narrative descriptions.

## Output formats

`json` · `dot` (Graphviz) · `summary` · `narrative`

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
├── templates/      # narrative templates (text, swappable)
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
| Output | Interactive query results | Batch report (JSON, DOT, narrative) |

## Honest limits

- Call-graph edges between files are **inferred** by correlating call sites with imports, not resolved via type inference. Marked as `inferred` vs `resolved` in the output. Dynamic dispatch, decorator-registered handlers, and runtime indirection will be missed.
- The role classifier is probabilistic. ACCS reports confidence per label and hedges narrative phrasing when confidence is low. It will not fully eliminate confident-but-wrong labels on out-of-distribution code.
- ACCS describes **structural** properties (what's in the file, what shape it has), not **semantic** ones (what business logic it implements). A human reader provides the semantic interpretation.

## Status

Early-stage scaffold. Milestones:

1. **Deterministic core** — scanner, tree-sitter parser, resolver, graph, PageRank, JSON/DOT reporters. Useful as a polyglot dependency-graph tool on its own.
2. **Entry-point detector** — config-file parsing, structural markers, graph-position heuristics. Deterministic, no ML.
3. **First language pack** — Python training pipeline → quantized `python.bin` → integrated classifier with confidence output.
4. **Narrative templates** — template engine + initial template set per role. Default English; swappable.
5. **Additional language packs** — JavaScript, TypeScript, Java, C, C++, Go, Rust, added one at a time.

## License

MIT — see [LICENSE](LICENSE).
