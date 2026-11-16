---
name: md2slides
description: >
  Write, convert and check slide decks in md2slides Markdown, the
  slide format of RexxPub (the Rexx Parser's publishing tools): one
  Markdown file in, one self-contained HTML deck out, with Rexx code
  highlighted by the Rexx Parser. Covers the shape of a deck, the
  `.` lines (steps, layout, code, marks, arrows), the patterns a
  lecture uses, and the rules that save a rebuild. Read this whenever
  a deck, slides or a lecture is to be written for md2slides; the
  deliverable is the .md file, not a .pptx or an HTML file.
---

# md2slides — Writing Decks

> **Version note.** This skill describes md2slides v404 (15 November
> 2026). The manual online is the authority: if the two disagree,
> the manual wins.
>
> - Manual: `https://rexx.epbcn.com/rexx-parser/doc/utilities/md2slides/`
> - The `.` lines, complete: the manual's section *The `.` Lines — a
>   Simpler Way to Write* (`#the-.-lines-a-simpler-way-to-write`)
> - Every option, in braces form: `.../md2slides/reference/`
> - Which marks are Markdown's, Pandoc's and md2slides':
>   `.../md2slides/primer/`

## What md2slides is, and what you deliver

md2slides reads **one Markdown file** and writes **one self-contained HTML
file**: the deck, with its pictures, its style sheets and a runtime that
does the steps, the timer, the presenter's window and so on. Rexx code in
the deck is highlighted by the Rexx Parser. Everything else is Pandoc
Markdown with a few words of md2slides' own.

- **Your deliverable is the `.md` file**, UTF-8, ready to build. A `.pptx`,
  a slide deck in some other tool, or the HTML itself is not what is asked
  for, even when the request just says "slides". The author builds the deck.
- **Building it:** `[rexx] md2slides deck.md` writes `deck.html` beside it.
  It needs ooRexx 5.3 (beta, r13514 or later: the first with CATCH,
  which the Rexx Parser uses), Pandoc and the Rexx Parser
  (`https://rexx.epbcn.com/rexx-parser/Rexx-Parser-latest.zip`; run
  `bin/md2slides.rex` with `bin` on the PATH). If your environment cannot
  reach those (a sandbox whose network only reaches package registries),
  don't spend effort trying: write carefully, and ask the author to build
  it and paste you the messages.
- **The look is not in the deck.** It comes from a *skin* (`-s` /
  `--skin`, a CSS file: fonts, colours, institutional strings) and,
  optionally, *masters* (`-mp`, repeatable: layout and chrome). Both are
  named by path. With no `-s` md2slides looks for `skin-default.css`
  beside the deck, then in the current directory, and a missing skin is an
  error. If the author has skins, use theirs. The package has two to start
  from, in `samples/md2slides/skins/`: `default`, neutral, and `rexxla`,
  in the colours of RexxLA, each a skin and its master;
  `samples/md2slides/decks/` has five sample decks and `makedeck`, which
  builds one with either skin (`makedeck rexxla trees-demo`). A minimal
  skin is one line, `:root { --skin-body-size: 34px; }`.
- **Installing ImageMagick pays off.** The pictures (`img/...`) are
  embedded in the deck, and any PNG or JPEG longer than 1920 px
  (`--picture-size`, `picture-size:`) is reduced with ImageMagick, if it
  is installed; without it, big pictures go in as they are. On Windows
  only ImageMagick 7's `magick` is used, never `convert` (there, `convert`
  is the system's FAT-to-NTFS converter).
- **For a PDF of the deck, print from Chrome or Edge.** Firefox's "Save as
  PDF" makes files several times larger.
- **The build reports its problems** with line numbers: unknown words that
  are almost md2slides', marks that find nothing, steps that point nowhere.
  Ask for those messages, and fix what they say. In the built deck, the
  `d` key diagnoses what only shows on screen (nodes that overlap, a slide
  that does not fit, a logo with no picture), and `v` or Shift+V shows a slide's Markdown when the
  deck has `view-source: yes`.

## Before writing: study the author's decks

If the author has decks already, read two or three of them before you
write anything, and copy what you find:

- the **front matter**: often just a `defaults:` file (which holds the
  presenter, the affiliation, the footer and the timer), plus `logo:` and a
  few fields. Copy the block from a deck of the same course or event.
  Don't invent fields: a field that nothing sets prints empty and is
  reported.
- the **slide options**: `kicker=`, `reference=`, `logo=`, written on a
  `. slide` line above the title;
- the **steps**: how much comes on a press, what comes by itself
  (`after`), how programs and their output are revealed, and how marks and
  arrows are used;
- the **wording**: how long the bullets are, what is bold or italic, how
  code is written in prose.

When the author's habits and this skill differ, the author's decks win.

## The shape of a deck

```
---
rexxpub:
  presenter: Jane Doe
  affiliation: Example University
  logo: oorexx
  code: rexx
  footer:
    left: "{presenter}"
    right: "{page} / {pages}"
---

. slide cover
# Procedural and Object-oriented Programming

. subtitle
Introduction to ooRexx

. slide kicker="Basics"
# What Is Rexx?

. steps
- A "human-centric" language
    - Designed by Mike F. Cowlishaw at IBM, 1979
- Few rules, few keywords
```

- **The front matter** is the `---` block on the very first line.
  md2slides' settings go under `rexxpub:`. It is YAML: a text with a colon
  and a blank in it, or one that reads `yes`, `no`, `on` or `off`, goes in
  quotes.
- **A slide opens with a level-1 heading**, `# Title`. Two `#` lines in a
  row make a two-line title. `## ...` and below are not slides.
- **The `. slide` line** goes just above the `#` line and gives the slide's
  options. `. slide` with no `#` after it is a slide with no title.
- **Roles:** `. slide cover` (the title slide, with `. subtitle` and
  `. cover-art`), `. slide section` (a part opener) and `. slide card` (the
  closing business card). A card slide with a title and nothing under it
  (`. slide card` and then `# Thank you`) takes its card from the front
  matter, `card:`, or else shows the presenter and the affiliation. Up to
  v357 the bare `. slide card`, as the last line, did nothing (with a
  warning); from v358 on it is that card too, with no title.
- **Logos** come from the master: the sample masters offer `rexx`,
  `oorexx` and `bsf4oorexx` (and `rexxla`, the RexxLA one). `logo:` in the front matter puts one on every
  slide; `logo=` on a `. slide` line changes it from that slide on. A logo
  the master has no picture for is not shown (the deck's `d` key says so).

## The `.` lines

A line that starts with `.`, on its own, **just before the thing it is
about**. No blank line goes between the line and its block. Several `.`
lines in a row all apply to the next block. Within a line, the order does
not matter, and the line holds:

- a **step word**: `click` (on a press), `after` (by itself, when the step
  before has finished), `with` (together with the step before);
- **options**: `plain`, `steps`, `blanks`, or `name=value` (`scale=0.8`,
  `level=2`). A value with blanks goes in double quotes;
- a **name**: `id=#prog`. It is used afterwards as `#prog`;
- a **keyword with operands**: `columns 7 5`, `arrow #a -> #b`,
  `mark 2 "NOVALUE"`, `picture img/a.png`.

In a list, the line goes just above the item, **indented as the item is**.
`>` is never a md2slides line: it is a Markdown quotation. A paragraph
that must start with a dot is written `\.`. The braces form
(`{.fragment}`, `::: {.incremental}`) says the same things and mixes with
the dots; if the author's decks use one form, write that one. Dots are
shorter and easier to get right.

## Cheat sheet

The full list is in the manual (see the version note). These are the ones
a lecture uses most.

**Steps**

| Write | Meaning |
|---|---|
| `. click` / `. after` / `. with` | The next block is a step |
| `. steps` | Before a list: each item a step (sublists too). Before a table: each row. Before a listing or output: each line. Above a later item: each item a step from there on (v363) |
| `. steps each=after` / `. after steps` | Each one comes by itself / the first one comes by itself too |
| `. static` above an item | In a `steps` list, it comes with its parent, not on its own step |
| `. after` / `. with` above an item | In a `steps` list, this item comes by itself, or with the one before |
| `. steps="1-2,3-5"` | A listing or output, a group of lines per step. In a listing numbered from N (`. numbered=13`), the numbers are the ones shown (from v386) |
| `. later id=#x` … `. click show #x` | A block written in its place that comes in when the `show` says |
| `. click anim=fade-up`, `. after wait=2` | How and when a step comes in |

**Layout**

| Write | Meaning |
|---|---|
| `. columns 7 5` … `. next column` … `. end columns` | Columns with their shares; `ruled` draws a line between them |
| `. group` … `. end group` | Several blocks that are one step |
| `. level=2` | Lined up with the text of a level-2 item (code, output, a table) |
| `. label` | A small bold caption, typically `Output:` |
| `. plain`, `. tight`, `. flow=3` | A list without bullets; rows close together; a long list in 3 columns |
| `. scale=0.8` | The block at 80% |
| `. pin=top-right` | The next picture or block laid over the slide |
| `. picture img/a.png width=40%` | A picture (also `![](img/a.png)`) |
| `. notes` | The next block is a presenter's note (key `n`), never on the slide |

**Code**

| Write | Meaning |
|---|---|
| ```` ```rexx ```` | A Rexx listing, highlighted by the Rexx Parser |
| ```` ```output ```` | What a program printed; not highlighted |
| `. file=data.txt` before ```` ```output ```` | A file box: the data in a file, named on top |
| ```` ```output {.keys} ```` | `<Enter>` drawn as a key cap |
| `. numbered`, `. blanks`, `. program=p1` before a listing | Line numbers; blanks shown; pieces of one program over several slides |
| `` `say "hi"` `` with `code: rexx` | Rexx in a sentence, highlighted |
| `` `3 4`{.output} ``, `[Enter]{.key}` | Output in a sentence; a key |
| ```` ```java ````, ```` ```python ````… | Other languages, highlighted by Pandoc |

**Marks, arrows, callouts**

| Write | Meaning |
|---|---|
| `. mark id=#nv 2 "NOVALUE"` before a listing | A mark on those words of line 2. Also: `5-7` (lines), `"text"` (anywhere), `20,30,60,70` (a region of a picture) |
| `box`, `ring`, `ink`, `hatch`, `bold`, `pen=2` on a mark | How it looks (`wash` is the default) |
| `caseless`, `part` on a mark | Its texts in any case; inside words too (`. mark "%20" part` marks the `%20` of `Rony%20G.`, v360). Without `part`, a text is found only as a whole word, and from v360 the build says when a mark's text is not in its listing |
| `. after mark 7-10` | A mark that comes in as its own step, just after its block |
| `` [`https`{.output}]{spot=u1} `` in a sentence | A mark in the prose: it lights with beat `u1` and looks like that mark in the listing, code inside included (from v361; no `<style>` is needed for that) |
| `. explain "https" pen=1` after a listing, then the line | The line is a step that marks `https` in the listing above and its own `` `https` `` with the same look (v364). `. explain 2 "https"`, `. after explain`, the words of `. mark`. Use it for a mark and the line that explains it |
| `. click show #nv`, `. click hide #nv`, `clear` | A named mark comes in, or goes; `clear` puts out the ones before. `. click show #a #b` is one step, and each mark keeps its own look (from v380); what the `show` line says (`box`, `pen=`) wins for all |
| `. after arrow #a -> #b` | An arrow (`<-`, `<->`, `--`); `label="..."`, `bend=30`, `line=dashed` |
| `. click callout --> #code:2 at=right` | The next paragraph is a balloon pointing at line 2 of `#code` |
| `#code:4`, `#code:4:"word"`, `#tree:"Node"`, `#pic:60,40` | An end: a line, a word in a line, a node, a point on a picture |

**Trees:** `. tree` before a list draws it as a tree (`. steps` grows
it), with one of three layouts:

- `layout=down`, the default: each node over its children;
- `layout=columns`, a class tree in boxes: the root on top, its children in
  `flow=4` columns, or a new column from each item with `. break` above it;
  `. under` above a grandchild puts it straight under its parent;
- `layout=folders`, as a file manager shows them: one name per line, thin
  lines, no boxes; `icons` puts a folder or a page before each name. From
  v403 it takes `flow=` and `. break` too, columns side by side, which
  makes a class tree about a third as tall as in boxes.

`[the document root]{.note}` in an item is a note beside its node. `. id=#t`
just before `. tree` names the tree (`#t:"Alarm"` is a node). A tree ends
at an item of its top level that comes after a blank line: that item starts
another list. `. mark id=#f "Object" "Class" hull label="Fundamental"`
before the tree, and `. click show #f`, draws one region round the nodes;
`hatch` fills it with strokes.

**Drawings:** `. drawing ratio=2.4 width=80% size=2` …
`. end drawing` holds blocks placed by `. at=x,y,w,h` (per cent) and
framed by `. box` (`round`, `dashed`, `bare`, `arrow=right`), joined with
`. arrow`.

**Steps in a list:** an item with no step after one that has is on the slide
from the start, before the item above it; from v363 the build says so. Give
it `. click`, or `. steps` above the first item that steps. In a list with
`steps` before its first item nothing is said: every item steps already
(v365; v363 and v364 said it there too, wrongly). In `. layers`,
blank lines between the pictures are not needed from v363. The
`{md2slides-version}` field (v363) shows which md2slides built a deck.

**Front matter keys you will meet:** `defaults:` (a file, or several, that
hold what decks share; the deck wins over them), `logo:`, `code: rexx`,
`style:` (the Rexx colours), `fit-min:` (how far a slide that does not
fit may shrink: `50%`, or from v387 `8pt`, the size its smallest text may
reach; 45% by default), `number-titles: yes` (slides with the same
title get ", 1", ", 2"), `timer:` with `total:`, `view-source: yes`
(Shift+V shows the Markdown), `print-notes:`, `footnotes:` (from v376:
`slide`, the default, puts a slide's notes at its foot; `end`, on a last
slide, "Notes"; `presenter`, in the presenter's notes, with no mark on the
slide), `blocks:` (the ground and
the frame of every block, or of `code:` or `output:`: `background`,
`border`, `border-width`, `border-color`; the same words on a fence, from
v359 on; from v366, also of one language, `bash:`, or of `other:`, the
code that is not Rexx), `footer:` with `left:`,
`center:`, `right:`, and fields of your own like `course:`, used as
`{course}`. A `#` in a kicker numbers it: `kicker="Commands, #"`.

## Patterns that lectures use

**A program, then its output, line by line**, under a level-2 bullet. The
listing ends the list, and `level=2` lines it up with the text of the
level-2 item (see rule 8 for the other way).

    - Loops
        - `do` with a control variable

    . click level=2
    ```rexx
    do i = 1 to 3
      say "Line" i
    end
    ```
    . click label level=2
    Output:
    . with level=2 steps
    ```output
    Line 1
    Line 2
    Line 3
    ```

**Walking through a listing**: name the marks before it, then light them
one press at a time.

    . id=#code
    . mark id=#cls 1 "::class"
    . mark id=#meth 3-5 box
    ```rexx
    ::class Person
    ::attribute name
    ::method init
      expose name
      use arg name
    ```
    . click show #cls
    . click show #meth clear

**Code beside its explanation**:

    . columns 7 5
    . id=#prog
    ```rexx
    ...
    ```
    . next column
    . steps
    - First point
    - Second point
    . end columns

**A picture that explains, then the text**: `. click show #pic` at the
top, and `. later id=#pic` above the picture where it stands.

**Long topics**: split them. Give the slides the same title and set
`number-titles: yes`, so they are numbered for you.

## Rules that save a rebuild

1. **Only md2slides' words.** An unknown class or option does nothing,
   silently, unless it is near a word of ours (then the build warns). A
   misspelled word in a `.` line is reported when the line has other
   words of ours (`. clik level=2`); a misspelled option *name* is not
   (`. click lvel=2` keeps `lvel` as an attribute, and nothing happens).
   A `.` line of one misspelled word (`. clik`) is text on the slide:
   reported from v358 on, silent before. Don't invent options: if it is
   not in the manual, it does not exist. Ask the author, or say what you
   would need.
2. **The Rexx must be real ooRexx.** Use straight quotes, `--` for
   comments (not an en dash) and `/* */`. Code copied from a word
   processor, a slide program or a PDF often carries curly quotes and
   dashes: put them back. Rexx that does not parse is not highlighted.
3. **With `code: rexx`, every plain code span must be Rexx.** A lone
   operator is ambiguous: write `` `=`{.rexx comparison} `` (or
   `assignment`). Something that is not Rexx in prose (a file name, a
   command, a Java call) is written `` `x`{.output} `` or as plain text.
   A span that does not parse shows up **red on the slide**; md2slides
   v357 and later warn about it, with the line.
4. **Never put a class on a ```` ```rexx ```` fence** (`{.fragment}`). It
   is dropped, with a warning. Put a `. click` (or `. after`...) line
   before the listing.
5. **One slide is one screen** (16:9). Aim for at most 7 or 8 bullets, or
   about 18 lines of code with a few bullets. md2slides shrinks a slide
   that does not fit, down to 45% (or the author's `fit-min:`), but small
   type is not readable from the back of a lecture hall. Split instead, or
   use `. columns`.
6. **Pictures you have not seen**: use the paths the author gives you
   (`img/...`). Don't invent file names. When a picture is still to be
   made, put a clear placeholder paragraph
   (`[PICTURE: class diagram of Person]`), and list them for the author at
   the end.
7. **Ids are unique on a slide**, and an arrow, a callout or a `show` must
   name something on the same slide.
8. **Code and output under a bullet.** There are two ways. The first ends
   the list and puts `level=2` (or `1`) on the `.` line before the
   listing, at the margin (the first pattern above). The second indents
   the `.` line and the listing to where the item's **text** starts, two
   columns past its `-`. Don't indent them to the item's dash: up to
   md2slides v356, a `.` line and a listing at the dash give stray `:::`
   text on the slide; from v357 on, they work, but the listing goes into
   the *parent* item, after the sublist.
9. **Characters**: UTF-8 only. Write `—` and `…` as the characters
   themselves.

## Before you hand a deck over

- The front matter opens on line 1, and every slide opens with `#`.
- Each `.` line stands just above its block, with no blank line between,
  and in lists it is indented as the item is.
- Every `columns`, `group`, `drawing` and `layers` has its `end`.
- Every `#name` used in `show`, `arrow`, `callout` and `over=` exists on
  its slide.
- No `. word` shows up as text in a slide: every `.` line is spelled
  right, and so is every option name in it.
- The Rexx listings would run, and the outputs are what they print.
- No slide is crowded. Give the timing with `time=` on section slides
  when the author gives you the minutes.

Then tell the author what you could not decide: missing pictures, a step
order you guessed, a slide you would split.
