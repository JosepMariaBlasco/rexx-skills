---
name: oorexx-docbook-build
description: >
  How to reproduce the official ooRexx documentation build (DocBook +
  Publican layout + Apache FOP) in a Linux environment, and add Rexx
  syntax highlighting to the program listings. Covers getting the
  sources over svn-https, the official build scripts plus the Rexx
  Parser's drop-in highlighting tools (`hldocprep`/`hldoc2pdf`), and the
  Common_Content. Read this whenever you need to build an ooRexx book to
  PDF, with or without highlighting.
---

# Building official ooRexx literature (with highlighting)

This skill builds an ooRexx documentation book to a PDF that looks like
the official ones (cover, brand colours, admonitions) and, optionally,
colours its Rexx listings using the Rexx Highligher distributed as
part of the Rexx Parser.

The recipe assumes a Linux environment with command-line tools, the Rexx
Parser project checked out in a directory referred to below as `$PARSER`,
the official build scripts from svn (`$BLDOC`), and the ooRexx docs laid
out as docpath expects (`$DOCS`). Adjust all three to your own machine.

## Setup (do these in order)

1. **Install ooRexx** — the whole toolchain runs on it. Follow the
   install steps in the companion skill `oorexx-run.md`, which you should
   have received alongside this one. Use it — do not install ooRexx by any
   other route (SourceForge guesswork, building from source, apt); those
   steps are kept current in that skill, and reconstructing them by hand
   wastes time and drifts from the tested method. **If you did not receive
   `oorexx-run.md`, stop and tell the user** rather than improvising an
   install.

2. **Check out the Rexx Parser project** — it provides the highlighting
   engine in `bin/` and the DocBook tools in `bin/docbook/`. Download it
   from <https://rexx.epbcn.com/rexx-parser/Rexx-Parser-latest.zip> (the
   canonical source, not GitHub — it usually carries the latest patches
   before GitHub does).

3. **Install the apt packages:**
   `apt-get install -y subversion fop xsltproc docbook-xsl docbook-xml libxml2-utils poppler-utils`
   (poppler-utils only to rasterise/verify the PDF).

If the session needs to build railroad diagrams from EBNF, that's a
separate task — use the companion skill `ebnf2svg.md` (received alongside
this one). If you need it and did not receive it, stop and tell the user.

## Key facts (the non-obvious ones)

- **Prefer `https://svn.code.sf.net/...`** for checkout/export/diff. The
  native `svn://` protocol is often blocked by firewalls and proxies in
  sandboxed environments; the `https://` form behaves identically and
  works through them.
- **The book layout is Publican's, but the build does NOT run Publican.**
  The sources follow the Publican convention — `publican.cfg`, `en-US/`,
  `Common_Content/` — so the obvious move is to install the `publican`
  tool and run it. Don't: the toolchain reproduces that convention
  without invoking the executable. `docprep` copies the Common_Content by
  hand; `doc2fo` is just `xsltproc pdf.xsl`; `fo2pdf` is just `fop`.
- **The "official look" lives entirely in `pdf.xsl` + `defaults.xsl` +
  `Common_Content`** — there is no magic Publican brand to replicate. The
  cover comes from `Book_Info.xml` + `Common_Content/images/oorexx.jpg`,
  rendered by docbook-xsl.
- **Highlighting in production does NOT use the `highlight.rex` CLI.**
  The Rexx Parser ships a standalone command-line highlighter,
  `highlight.rex`, and it is the obvious tool to reach for — but it is
  the wrong one here. The production path is different: `hldocprep` loads
  `Parser.DocBook.cls` and calls its `::Routine ProcessProgramListings`
  to colour each listing, and the matching XSL is generated separately by
  `css2xsl --style STYLE`. The colouring carries fine-grained token
  classes (operators, specials, constants, and so on — see the attribute
  table below), and those classes only line up with the style definitions
  when both come from this pairing. Highlight with the standalone CLI
  instead, and the listings still render, but their token granularity no
  longer matches the generated XSL, so colours land on the wrong tokens —
  a failure that produces output and so is easy to miss.

## Where the pieces live in svn

Documentation trunk: `https://svn.code.sf.net/p/oorexx/code-0/docs/trunk`

| What | Path under the trunk |
|---|---|
| A book's sources | `<bookname>/` (e.g. `rexxref/`, `rexxextensions/`) |
| Build tools | `tools/bldoc_orx/` (`pdf.xsl`, `defaults.xsl`, `fop.cfg`, `*.rex`) |
| **Official Common_Content** | `oorexx/en-US/` (logo, Notices, CPL, Conventions, admonition images) |
| Liberation fonts | `tools/LiberationFonts/` |
| DRAFT watermark | `oorexx/en-US/images/watermark-draft.png` |

The new highlighting tools (`hldocprep`/`hldoc2fo`/`hldoc2pdf` + a
highlighting-aware `pdf.xsl`) live in the **Rexx Parser** project, in
`bin/docbook/`. That `pdf.xsl` is currently a verbatim copy of the official
one — byte-for-byte identical. Neither version embeds the `rexx_*`
highlighting templates; those are generated separately by css2xsl and pulled
in via the driver XSL.

## Book layout (Publican / DocBook 4.5)

```
publican.cfg                 brand: oorexx, mainfile: <id>
en-US/
  <id>.xml                   mainfile: xi:include of every component
  <id>.ent                   entities (VERSION, EDITION, ...)
  Book_Info.xml              title/subtitle/cover (xi:includes Legal_Notice + Author_Group)
  Author_Group.xml
  Preface.xml
  <chapter>.xml              one per chapter
  Revision_History.xml
  Common_Content/            from svn oorexx/en-US/ (logo, Notices, CPL, images)
```
Every file starts with the DOCTYPE for its root element + the BOOK_ENTITIES
declaration + the standard CPL licence comment block. Clone an existing
book (e.g. `rexxextensions`) for exact boilerplate.

## Build recipe (official tools, official look + highlighting)

The Rexx Parser ships three drop-in companions to the official build
scripts — `hldocprep` (highlighted docprep), `hldoc2fo`, `hldoc2pdf`.
Use them directly: they reproduce the whole production path, including
the parts that are easy to get wrong by hand (discovering every style
the book actually uses, generating one XSL per style with `css2xsl`,
gluing them together, and injecting the result into a `pdf-hl.xsl`
derived from the official `pdf.xsl`). Don't hand-roll the highlighting
loop — these tools are the production path.

The recipe splits in two: a **one-time setup** (get the scripts, drop in
the highlighting tools) and a **rebuild** you run after every source edit.
Keeping them separate is the point — most build failures come from
half-remembering the rebuild steps, so the rebuild is packaged as an
idempotent script you can rerun blindly.

### Setup (once)

Set `PARSER` to the Rexx Parser checkout, `DOCS` to a directory laid out
as docpath expects (`$DOCS/<book>/en-US` for each book, plus
`$DOCS/oorexx/en-US` for the Common_Content), and `BLDOC` to where the
official build scripts will live; adjust all to your machine.

```bash
export PARSER=/path/to/rexx-parser
export DOCS=/path/to/oorexx-docs        # holds <book>/ and oorexx/ trees
export BLDOC=/path/to/tools/bldoc_orx   # the official build scripts (svn)
TRUNK=https://svn.code.sf.net/p/oorexx/code-0/docs/trunk

# 0. Get the official build scripts, the book, and the Common_Content.
#    docpath expects $DOCS/<book>/en-US for the book and $DOCS/oorexx/en-US
#    for the shared Common_Content. (A book of your own goes in the same
#    layout: put its sources under $DOCS/<book>/en-US.)
svn export --force $TRUNK/tools/bldoc_orx $BLDOC
svn export --force $TRUNK/<book>          $DOCS/<book>     # skip if the book is your own
svn export --force $TRUNK/oorexx          $DOCS/oorexx     # Common_Content

# 1. Put the Parser's highlighting tools alongside the official scripts,
#    and use the Parser's highlighting-aware pdf.xsl. (The readme calls
#    this a drop-in: no official file needs editing.)
cp $PARSER/bin/docbook/hldocprep.rex $PARSER/bin/docbook/hldoc2fo.rex \
   $PARSER/bin/docbook/hldoc2pdf.rex $BLDOC/
cp $PARSER/bin/docbook/pdf.xsl $BLDOC/pdf.xsl

# 2. Parser bin on REXX_PATH so the highlighting engine is found.
export REXX_PATH="$PARSER/bin:$PARSER/bin/docbook"
```

That is the **one-time setup**: get the scripts, drop in the Parser's
tools, set `REXX_PATH`. You do NOT repeat it after a source edit. The
build itself is the next section.

### Rebuild (run after EVERY source edit)

**Use the `oorexx-docbook-rebuild.sh` script shipped alongside this
skill.** It is idempotent — safe to run repeatedly — and it does, in the
right order, the four things that are easy to forget and that produce
misleading errors when skipped (see the troubleshooting table). Do not
hand-run `hldocprep`/`hldoc2pdf` from memory between edits; that is
exactly where the "Unable to find fo_files" and stale-`whichdoc` traps
bite. Run the script:

```bash
export PARSER=/path/to/rexx-parser
export DOCS=/path/to/oorexx-docs
export BLDOC=/path/to/tools/bldoc_orx
bash oorexx-docbook-rebuild.sh <book> <path-to-live-source>/en-US
# e.g. bash oorexx-docbook-rebuild.sh rexxhttp \
#        /home/claude/state/docbook-tree/rexxhttp/en-US
# -> pdf_files/<book>.pdf ; prints the page count
```

The **live source is the tree you edit** (for this project,
`state/docbook-tree/<book>/en-US`), which is NOT the same directory the
build reads (`$DOCS/<book>/en-US`). The script's first job is to copy
the former into the latter, so an edit that isn't copied over is an edit
the build never sees — the single most common "why didn't my change
show up" cause. Passing the live source path to the script handles this
for you.

What the script does, and why each step matters (the same four things,
if you ever build by hand):

1. **Refresh `$DOCS/<book>/en-US` from the live source.** docpath reads
   only from `$DOCS`; the tree you edit lives elsewhere. Skip this and
   you build the previous version.
2. **Clean `work_folder` and the XSL artifacts.** Re-highlighting an
   already-highlighted file fails with `Unmatched comment delimiter /*`.
   Always start highlighting from clean sources.
3. **`mkdir -p fo_files pdf_files log_files`.** `hldoc2fo` reads the
   `fo_files` property (relative path, default `fo_files/`) and bails
   with `Unable to find fo_files.` if the directory is absent — it does
   NOT create it. The message reads like missing *input*; it is a
   missing *output dir*. This is the trap that looks like a broken build
   but is a missing `mkdir`.
4. **Re-run `docpath.rex $DOCS`.** It writes `bldoc.env`, the properties
   file that `hldocprep`/`hldoc2fo` read for `whichdoc`, `fo_files`, etc.
   A fresh shell (or a wiped tree) has no `bldoc.env`; without this step
   `whichdoc` is empty and the FO step can't find its inputs.

Then it runs `hldocprep <book>` (prep + highlight + generate all XSL)
and `hldoc2pdf`, and prints the page count. Eyeball a page to confirm
highlighting:

```bash
pdftoppm -png -r 120 -f 1 -l 1 pdf_files/<book>.pdf cover
```

### Troubleshooting (the misleading messages, decoded)

| Message | Real cause | Fix |
|---|---|---|
| `Unable to find fo_files.` | Output dir missing (not input). | `mkdir -p fo_files pdf_files log_files` before the FO step. |
| `Unmatched comment delimiter /*` | Re-highlighting an already-highlighted file. | Clean `work_folder` and rebuild from fresh source. |
| `whichdoc`/`fo_files` empty; FO step finds nothing | `bldoc.env` absent or stale (fresh shell, wiped tree). | Re-run `rexx docpath.rex $DOCS` before `hldocprep`. |
| Edit didn't show up in the PDF | Edited the live tree but built `$DOCS`, which wasn't refreshed. | Re-copy live source into `$DOCS/<book>/en-US` (the script's step 1). |
| `hldoc2pdf` skips XSL: `... already exists` after a partial clean | Half-cleaned tree: some XSL survived, some didn't. | Clean ALL of `work_folder fo_files pdf_files hl-styles rexx-highlights.xsl pdf-hl.xsl`, then rebuild. |

Notes on this path:

- **The unhighlighted build is the same minus highlighting:** `docprep
  <book>` then `doc2pdf`. Both builds read the same sources and only
  touch the work folder; the originals are never modified.
- **`hldocprep` runs `docprep` for you** — it copies the Common_Content,
  builds the work folder, highlights every `<programlisting
  language="rexx">`, then generates `hl-styles/`, `rexx-highlights.xsl`
  and `pdf-hl.xsl`. The DRAFT watermark and FOP base-dir are handled by
  the official scripts; no manual driver XSL or `fop.cfg` surgery needed.
- **Offline?** Only then do you need an XML catalog to resolve the remote
  docbook-xsl import in `pdf.xsl` to a local copy. Point
  `XML_CATALOG_FILES` at a catalog that rewrites
  `http://cdn.docbook.org/release/xsl-nons/current/` and
  `http://docbook.sourceforge.net/release/xsl/current/` to your local
  `docbook-xsl` directory. With network access this is unnecessary.

## Enabling highlighting in the source

Mark a listing with `language="rexx"`; unmarked listings are left alone.
Per-listing attributes on `<programlisting>` (handled by
`Parser.DocBook.cls~BuildOptions`):

| Attribute | Values | Default |
|---|---|---|
| `hl-style` | any style | `print` |
| `operator` / `special` / `constant` / `assignment` | `group` / `full` / `detail` | `group` |
| `doccomments` | `detailed` / `block` | `detailed` |
| `dialect` | `executor` / `cms` / `rexxvm` / `tutor` / `unicode` / `experimental` | (none = standard ooRexx) |

The `dialect` attribute is rarely needed; use it only when a listing
contains dialect-only constructs (e.g. Executor's `::EXTENSION`). Without
it, those constructs fail to parse and the block is left un-highlighted.

## Known defects / open points

- Cover logo etc. all resolve once Common_Content is the official one;
  `hldoc2pdf` (via the official `fo2pdf`) handles the FOP base-dir.
- 80-column lines: the official `shade.verbatim` style sets
  `wrap-option: wrap` (hyphen char `\`), which wraps long listings
  cleanly — no need to shrink the font.
