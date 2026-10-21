---
name: oorexx-docbook-build
description: >
  How to build an ooRexx documentation book (DocBook, the official
  bldoc_orx scripts, Apache FOP) to PDF and to chunked HTML, with
  Rexx syntax highlighting, on Linux: the hl* scripts (hldocprep,
  hldoc2pdf, hldoc2HTML) plus the Rexx Parser. Covers getting the
  sources over svn, the build, rebuilding after an edit, marking
  listings for highlighting, and the traps. Read this whenever you
  need to build an ooRexx book, official or your own.
---

# Building ooRexx books (with highlighting)

The official ooRexx books are DocBook 4.5 sources, built by the scripts
in the documentation tree's `tools/bldoc_orx/`: to PDF through
`xsltproc` and Apache FOP, and to chunked HTML through the DocBook
XHTML stylesheets. The **hl\* series** in the same folder (`hldocprep`,
`hldoc2fo`, `hldoc2pdf`, `hldoc2HTML`) does the same and also colours
the Rexx listings, using the Rexx Highlighter of the Rexx Parser. The
result looks like the official books (cover, colours, admonitions).

> **Tested** on 21 October 2026 on Ubuntu 24.04 with ooRexx 5.3.0
> r13282, the documentation trunk's `tools/bldoc_orx` (last changed in
> SVN r13235), the Rexx Parser's `Rexx-Parser-latest.zip` of that day,
> FOP 2.8 and docbook-xsl 1.79: the RexxHTTP manual (83 pages of PDF,
> 182 HTML pages) built with no errors.
>
> **Requires `oorexx-run`** (the whole toolchain runs on ooRexx). If a
> book builds railroad diagrams from EBNF, that is a separate step: see
> the `ebnf2svg` skill.
>
> **Environment note.** The paths below (`/work/...`) are placeholders.
> The commands assume Debian/Ubuntu; run them as root or with `sudo`.
> The documentation tree's `setup.rex` is Windows-only: on Linux the
> toolchain is set up by hand, as here.

## Setup (once)

1. **ooRexx**, with the `oorexx-run` skill.

2. **The packages:**

   ```bash
   apt-get install -y subversion fop xsltproc docbook-xsl docbook-xml \
                      libxml2-utils fonts-liberation poppler-utils
   ```

   `fonts-liberation` gives the fonts the stylesheets ask for (FOP finds
   them by itself: `fop.cfg` has `<auto-detect/>`); `poppler-utils` is
   only for checking the PDF. With Debian's `docbook-xsl`, the remote
   stylesheet URLs that `pdf.xsl` and `html.xsl` import resolve to the
   local copy through the system XML catalog: no network is needed for
   the build.

3. **The Rexx Parser**, from its canonical site (it carries the latest
   fixes before GitHub does):

   ```bash
   curl -sLO https://rexx.epbcn.com/rexx-parser/Rexx-Parser-latest.zip
   unzip -qo Rexx-Parser-latest.zip -d /work/rexx-parser
   export REXX_PATH=/work/rexx-parser/bin
   ```

   Only its `bin/` has to be on `REXX_PATH` (or `PATH`); the HTML build
   finds the Parser's `css/` and `js/` from there. Nothing is copied
   into the build scripts.

4. **The documentation tree**, laid out as the scripts expect: each
   book in `$DOCS/<book>/` (with `publican.cfg` and `en-US/`), and the
   shared Common_Content in `$DOCS/oorexx/`. The scripts themselves can
   live anywhere. Use the `https://` form of the SVN URL (the `svn://`
   protocol is often blocked by proxies):

   ```bash
   TRUNK=https://svn.code.sf.net/p/oorexx/code-0/docs/trunk
   DOCS=/work/docs
   svn checkout $TRUNK/tools/bldoc_orx /work/bldoc_orx   # the scripts, hl* included
   svn checkout $TRUNK/oorexx          $DOCS/oorexx      # the Common_Content
   svn checkout $TRUNK/rexxref         $DOCS/rexxref     # a book (skip for your own)
   ```

   A book of your own goes in the same layout, `$DOCS/<book>/`. `svn
   export` instead of `checkout` works too; the only difference is that
   `docprep` asks `svnversion` for the book's revision and prints a
   harmless `svn: E155007: ... is not a working copy`.

5. **Point the scripts at the tree**, from the scripts' folder (it
   saves the path in the scripts' properties; redo it only if `$DOCS`
   changes):

   ```bash
   cd /work/bldoc_orx
   rexx docpath $DOCS        # → The docpath is set to /work/docs
   ```

## Build

From `/work/bldoc_orx`, with `REXX_PATH` set as above:

```bash
rexx hldoc2pdf <book>        # → pdf_files/<book>.pdf
rexx hldoc2HTML <book>       # → html_folders/<book>/ and html_zipfiles/<book>.zip
```

Given the book name, each one first runs `hldocprep <book>`, which does
everything the official `docprep` does (it copies the book and the
Common_Content into `work_folder/`) and then highlights every
`<programlisting language="rexx">` and generates the XSL the PDF and the
HTML need (`hl-styles/`, `rexx-highlights.xsl`, `pdf-hl.xsl`,
`rexx-highlights-html.xsl`, `html-hl.xsl`, all in the scripts'
folder). Its log says what it did:

```
44 block(s) highlighted in 6 file(s).
1 style(s) found: print.
```

The steps can also be run one by one: `rexx hldocprep <book>`, then
`rexx hldoc2fo`, then `rexx fo2pdf` (or `rexx hldoc2pdf` / `rexx
hldoc2HTML` with no book name, to reuse the prepared work folder).
`hldocprep` takes two options before the book name: `--style S` (the
default style for listings that do not name one; default `print`) and
`--regen` (regenerate all the XSL; needed after updating the Parser).

**Rebuilding after an edit:** edit the sources in `$DOCS/<book>/en-US/`
(that is what the scripts read) and run `rexx hldoc2pdf <book>` again.
The work folder is rebuilt from the sources each time, so repeated runs
are safe.

**Checking the result:**

```bash
pdfinfo pdf_files/<book>.pdf | grep Pages
pdftoppm -png -r 80 -f 10 -l 10 pdf_files/<book>.pdf page   # look at a page with code
```

**Without highlighting**, the official pair still works from the same
sources: `rexx docprep <book>` then `rexx doc2pdf`. For HTML on Linux,
use `hldoc2HTML` even then: the stock `doc2HTML` looks for the Common
Content in `ooRexx` while the folder is `oorexx`, copies nothing
("0 files copied"), and the book comes out with no stylesheet at all.

## Marking listings for highlighting

Only listings with `language="rexx"` are highlighted; the others are
left alone, so a book can be migrated a few listings at a time:

```xml
<programlisting language="rexx">Say "Hello, world!"</programlisting>
```

Per-listing attributes:

| Attribute | Values | Default |
|---|---|---|
| `hl-style` | any of the Parser's styles (`print`, `dark`, ...) | `print` |
| `operator` / `special` / `constant` / `assignment` | `group` / `full` / `detail` | `group` |
| `doccomments` | `detailed` / `block` | `detailed` |
| `dialect` | `executor` / `cms` / `rexxvm` / `tutor` / `unicode` / `experimental` | none (ooRexx) |

`hl-style` (not `style`: the DTD would drop an unknown attribute) also
pins the listing in the HTML, where the reader can otherwise switch
styles with the style chooser. A book may use several styles; the
scripts find the ones in use.

## Book layout (Publican / DocBook 4.5)

```
publican.cfg                 brand: oorexx, mainfile: <book>
en-US/
  <book>.xml                 the main file: xi:include of every component
  <book>.ent                 entities (VERSION, EDITION, ...)
  Book_Info.xml              title, subtitle, cover
  Preface.xml, <chapter>.xml ...
  images/
```

Every file starts with the DOCTYPE of its root element and the book's
entity declaration. Clone an existing book (`rexxextensions`, say) for
the exact boilerplate. The layout is Publican's, but **the build does
not run Publican**: do not install it. `docprep` copies the Common
Content itself, `doc2fo` is `xsltproc` with `pdf.xsl`, `fo2pdf` is
`fop`.

## Traps

- **Do not highlight with the `highlight.rex` command line** and paste
  the result. The book path is `hldocprep`, which highlights through
  `Parser.DocBook.cls` and generates the XSL that matches it. The
  output is standard DocBook (`<phrase role="rx-kw">...`, with the
  style on the listing's `role`, `highlight-rexx-print`), so a
  highlighted book still validates.
- **Old highlighted XML** (before July 2026) used invented elements
  (`<rexx_print_kw>`...). Re-highlight from clean sources and run
  `hldocprep --regen`.
- **Code with no colour, and no error**: the listing's style and the
  generated XSL do not match. Run `hldocprep --regen <book>`.
- **`Warning: highlighting failed for block at line N`**: that listing
  does not parse (a fragment out of context, say an `Expose` outside a
  method, or a dialect's syntax: try `dialect=`). The block is left
  uncoloured and the build goes on.
- **`hldocprep` cannot find the Rexx Parser** ("Please ensure that the
  Rexx Parser's bin/ directory is on your PATH or REXX_PATH"): set
  `REXX_PATH` as in the setup, in the same shell.
- **FOP cannot overwrite a PDF that is open** in a reader: close it
  before rebuilding.
- In the browser console of the HTML, `ReferenceError: $ is not
  defined` comes from the official XHTML stylesheets, in a plain build
  too. Harmless.

The Parser documents all of this in more depth in
`doc/highlighter/docbook/readme.md` (in its zip).
