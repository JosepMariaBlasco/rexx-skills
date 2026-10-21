# Rexx Skills

A small collection of skills for working in the Rexx ecosystem.
Each skill is a self-contained Markdown document written
for an AI agent (or a person) to follow: how to install something,
how to run it, what traps to avoid.

Each skill names the exact versions it targets and the environment
it assumes. Where a step is environment-specific,
the skill says so — adjust those to your own machine.

## Skills

### ooRexx-specific skills

[**oorexx**](oorexx.md) — Pitfalls and best practices for ooRexx code.
Covers the special variables that get silently overwritten
(`RESULT`, `RC`, `SIGL`), non-short-circuit logical operators,
method scoping with `Expose`, where labels are illegal,
and idioms for strings, stems, and line endings.

Works with: Claude.

[**oorexx-run**](oorexx-run.md) — Install and run ooRexx. Three choices:
the tested 5.3.0 beta revision (the default), the latest beta build,
or the 5.2.0 release; covers downloading the right `.deb` from SourceForge,
verifying it, the `::Requires` lookup order, and the common errors.

Works with: Claude.

[**apache-cgi-oorexx**](apache-cgi-oorexx.md) — Serve ooRexx scripts
over CGI with Apache. Covers installation, the two configuration patterns
(running `.rex` scripts directly and processing `.md` files through a Rexx handler),
a shell wrapper for the CRLF-shebang trap when scripts are edited on Windows,
and Linux case-sensitivity gotchas.

Works with: Claude.

[**rexxhttp**](rexxhttp.md) — Serve ooRexx rexxlets over Apache with
[the RexxHTTP processor](https://rexx.epbcn.com/rexxhttp/).
Requires [`oorexx-run`](oorexx-run.md);
covers the automated installer and the installation by hand,
verification with curl and the package's integration suite,
the request/response API (arguments, cookies, lazy headers, errors and redirects),
and the traps that bite when writing rexxlets
(non-short-circuit &, the JSON ::Requires, case-sensitivity,
what is served as source).

Works with: Claude.

[**ebnf2svg**](ebnf2svg.md) — Generate one SVG railroad diagram per production
from a W3C-style EBNF grammar. Covers the toolchain (the Java-based Railroad Diagram
Generator rr.war plus two ooRexx scripts), where to download each piece,
prerequisites, usage and the no-extension argument convention, the EBNF notation,
and how the orchestrator chains generation and SVG extraction.

Works with: Claude.

[**oorexx-docbook-build**](oorexx-docbook-build.md) — Build the official
ooRexx documentation books (DocBook + Publican layout + Apache FOP) to PDF,
with optional Rexx syntax highlighting on the program listings.
Requires [`oorexx-run`](oorexx-run.md), and [`ebnf2svg`](ebnf2svg.md)
only if a book builds railroad diagrams from EBNF.
Covers checking out the sources over svn-https, the official build scripts plus
the Rexx Parser's drop-in highlighting tools (`hldocprep`/`hldoc2pdf`),
the `docpath` layout for a book of your own, and the traps that bite
(the wrong `highlight.rex` shortcut, re-highlighting a clean checkout).

Works with: Claude.

### Regina-specific skills

[**regina-run**](regina-run.md) — Install and run Regina, the
ANSI-standard classic Rexx interpreter. Covers installing from the
distro package manager, invoking it, and getting a newer build from
SourceForge if you need one.

Works with: Claude.

[**apache-cgi-regina**](apache-cgi-regina.md) — Serve Regina Rexx scripts
over CGI with Apache. Covers installation, the two configuration patterns
(running .rex scripts directly and processing .md files through a Rexx handler),
loading external routines with CALL and the REGINA_MACROS search path,
a shell wrapper for the CRLF-shebang trap when scripts are edited on Windows,
and Linux case-sensitivity gotchas.

Works with: Claude.

## Download

[rexx-skills-latest.zip](rexx-skills-latest.zip).

## Using a skill

Open the one you need and follow it top to bottom. They assume basic
command-line comfort and, for most, a working Rexx interpreter — so
if you don't have one yet, start with **oorexx-run** or
**regina-run**.