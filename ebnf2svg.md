---
name: ebnf2svg
description: >
  How to generate SVG railroad (syntax) diagrams from an EBNF grammar with
  ebnf2svg.rex, the pure-ooRexx generator of the ooRexx documentation
  (docs/trunk/tools/RailRoadDiagrams in the ooRexx SVN): no Java, no other
  tools. Covers getting it, prerequisites, usage and options, the EBNF
  notation (including '$' for defaults, drawn above the main path as in
  the IBM syntax diagrams), and what the output looks like. Read this
  whenever you need to produce railroad diagrams from an EBNF source.
---

# EBNF → SVG — Railroad Diagram Generation

`ebnf2svg.rex` reads a grammar in W3C-style EBNF and writes one SVG
railroad diagram per production. It is a single ooRexx program: it parses
the grammar, lays the diagrams out and writes the SVG files itself.

It is the generator of the syntax diagrams of the ooRexx documentation.
Until November 2026 it was a wrapper around the Railroad Diagram Generator
(`rr.war`, a Java tool by Gunther Rademacher); since docs r13668 it does
everything in ooRexx, with the look of rr's diagrams (the same boxes,
arcs, text widths and CSS).

## Prerequisites

- ooRexx 5.3.0, **r13514 or later**: the program uses CATCH. It does not
  run on other Rexx interpreters.
- Nothing else.

Tested (26/11/2026) on Ubuntu 24.04 with ooRexx 5.3.0 r13665, on the 24
`.ebnf` files of `docs/trunk` (1442 diagrams; all the SVG files are
well-formed XML; the largest grammar, 345 productions, takes about 0.1 s).

## Getting it

It lives in the ooRexx documentation's SVN, in
`docs/trunk/tools/RailRoadDiagrams/ebnf2svg.rex`. Either of these works:

```bash
# Without Subversion: SourceForge's raw view of the file
curl -sSL -o ebnf2svg.rex \
  "https://sourceforge.net/p/oorexx/code-0/HEAD/tree/docs/trunk/tools/RailRoadDiagrams/ebnf2svg.rex?format=raw"

# With Subversion (apt-get install -y subversion)
svn export https://svn.code.sf.net/p/oorexx/code-0/docs/trunk/tools/RailRoadDiagrams/ebnf2svg.rex
```

(Both checked on 26/11/2026: they give the same file.) The same folder
still holds `rr-1.67-java8/rr.war` and `extract_svg_from_rr_xhtml.rex`,
the old toolchain, and `RailDiagramGenerator-readme.txt`, which describes
both; you do not need them.

## Usage

```bash
rexx ebnf2svg.rex [options] grammar [production...]
```

- `grammar` is the EBNF file; the `.ebnf` extension may be omitted
  (`grammar` reads `grammar.ebnf` if `grammar` does not exist).
- One SVG file is written per production, named after it
  (`Statement.svg`), by default in the directory of the grammar.
- If production names follow the grammar, only those diagrams are
  written.

Options:

- `--width n` — wrap a diagram whose main sequence is wider than `n`
  pixels into several rows joined by dotted lines. The default, 805,
  reproduces the wrapping of the ooRexx books.
- `--out dir` — write the SVG files to `dir`.
- `--help`, `-h` — usage.

Exit code: 0 on success; 1 for an error in the grammar (the message gives
the file, the line and the production); 2 for a bad option; 3 when the
grammar or the output directory does not exist.

## EBNF format

W3C-style EBNF:

```
Statement ::= 'IF' Expression 'THEN' Statement ( 'ELSE' Statement )?
List      ::= 'LIST(' item ( ',' item )* ')'
```

- `name ::= expression` — a production. It ends where the next one
  starts, so it may span several lines. Several productions with the same
  name are drawn as one diagram, as alternatives.
- `'text'` or `"text"` — terminal (rounded box). `name` — nonterminal
  (square box, linked to `#name`). Names may contain letters, digits,
  `_`, `.` and `-`.
- `A B` sequence, `A | B` choice (an empty alternative, `( A | )`, makes it
  optional), `A?`, `A*`, `A+`, `( ... )` grouping, `/* ... */` comments.
- `A - B` — drawn as one hexagon labelled `A - B`; the ooRexx books use it
  for references to a diagram fragment, as in `( WITH - fragment )`.
- Not supported: character classes (`[0-9]`) and `#x` characters.
- Text before the first production is ignored.

**Defaults.** A `$` before an alternative of a choice marks it as the
default. A choice with a default is optional: the main path is empty, the
default is drawn **above** it and the other alternatives below it, as in
the IBM syntax diagrams:

```
Scope ::= '::METHOD' name ( $'PUBLIC' | 'PACKAGE' | 'PRIVATE' )
```

When an optional group ends with a choice with a default, what precedes
the choice goes above the main path with the default:
`( ',' ( $'"Normal"' | '"Count"' ) )?` is drawn like
`( $( ',' '"Normal"' ) | ',' '"Count"' )`. The `$` notation comes from
Franz Hohenegger's BNF4OOo (WU Vienna, 2011).

## What the diagrams look like

Like rr's: a double arrow at each end of the main path, required items on
it, optional items below it, repetitions with a return path above (a
separator, if any, on the return path). Before drawing, the generator
factors a few patterns as rr did: `X (S X)*` and `(X S)* X` become one `X`
with `S` on the return path; `X X*` becomes `X+`; alternatives that start
or end with the same items share them; nested choices are flattened.

Each SVG carries the CSS of the ooRexx diagrams (`.line`, `rect.terminal`,
`text.nonterminal`, ...), with Verdana 12px. Text widths are a table
measured on rr's output, so the boxes fit Verdana and similar fonts.
