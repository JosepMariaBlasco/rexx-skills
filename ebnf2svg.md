---
name: ebnf2svg
description: >
  How to generate individual SVG railroad diagrams from an EBNF grammar
  using the Railroad Diagram Generator (rr.war, a Java tool) plus two
  ooRexx scripts. Covers downloading the toolchain, prerequisites, usage,
  EBNF format, and output. Read this whenever you need to produce railroad
  diagrams from an EBNF source.
---

# EBNF → SVG — Railroad Diagram Generation

The toolchain turns an EBNF grammar into one SVG railroad diagram per
production. It chains a Java tool and two ooRexx scripts:

```
grammar.ebnf → rr.war (Java) → XHTML with embedded SVGs
             → extract_svg_from_rr_xhtml.rex → individual .svg files
```

`ebnf2svg.rex` is the orchestrator: it runs `rr.war` and then calls the
extractor. You invoke only `ebnf2svg.rex`.

## Prerequisites

- ooRexx — the scripts use ooRexx-specific syntax and will not run on
  other Rexx interpreters.
- Java 11+ on the `PATH`. On Debian/Ubuntu:
  `apt-get install -y default-jre-headless`.

## Getting the toolchain

Put all three files in one directory (referred to as `bin/` below).

**`rr.war`** — the Railroad Diagram Generator, by Gunther Rademacher. The
download is a zip that contains `rr.war`; extract it:

```bash
curl -L -o rr.zip https://www.bottlecaps.de/rr/download/rr-2.6-java11.zip
unzip rr.zip rr.war -d bin/
```

The exact version may differ; check `https://www.bottlecaps.de/rr` for the
current download. `rr.war` is an executable war (`java -jar rr.war`), so
Java is all it needs.

**The two ooRexx scripts** — from the Rexx Parser distribution:

```bash
curl -L -o bin/ebnf2svg.rex \
  https://rexx.epbcn.com/rexx-parser/bin/ebnf2svg.rex
curl -L -o bin/extract_svg_from_rr_xhtml.rex \
  https://rexx.epbcn.com/rexx-parser/bin/extract_svg_from_rr_xhtml.rex
```

`ebnf2svg.rex` locates `rr.war` next to itself by default (override with
`--rr`), and finds the extractor through the standard Rexx search order —
so the simplest setup is all three in the same directory.

## Usage

```bash
rexx bin/ebnf2svg.rex grammar
```

The argument is the grammar's path **without** the `.ebnf` extension —
the script appends `.ebnf` itself. So `grammar` reads `grammar.ebnf`.

One SVG file is written per production, named after the production, in the
same directory as the EBNF file.

Options:

- `--rr <path>` — path to `rr.war`, if it is not next to `ebnf2svg.rex`.
- `--help`, `-h` — usage.

If the extractor is not in the same directory as the orchestrator, point
`REXX_PATH` at its directory so the search order can find it:

```bash
REXX_PATH=/path/to/bin rexx /path/to/bin/ebnf2svg.rex grammar
```

## EBNF format

W3C-style EBNF, one production per line:

```
production_name ::= definition
Statement ::= 'IF' Expression 'THEN' Statement ( 'ELSE' Statement )?
```

Elements: `'KEYWORD'` (terminal), `name` (non-terminal),
`( ... | ... )` (alternatives), `?` (optional), `*` (zero or more),
`+` (one or more), `[0-9]` (character class), `/* ... */` (comments).

Each production yields one SVG named after it (e.g. `Statement.svg`).

## Notes

The orchestrator runs `rr.war` with `-suppressebnf` (no EBNF text beside
the diagram) and `-noinline` (do not fold single-literal productions into
their references), then extracts each embedded SVG into its own file with
CRLF line endings. On success it exits zero; on error it prints a
diagnostic and exits with a code identifying the failure: 2 bad usage
(`--rr` without a path), 3 EBNF not found, 4 `rr.war` not found, 5 Java
not found, 6 cannot create temp file, 7 `rr.war` error, 8 no output.
