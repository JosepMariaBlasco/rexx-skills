# CLAUDE.md — Rexx Skills

Notes for Claude (or any coding assistant) working in this repository.
Here **the user** is Josep Maria Blasco, who decides. This repository is
public: nothing private about anyone goes here, and nothing about other
repositories that are not public.

## What this is

The source of [Rexx Skills](https://rexx.epbcn.com/skills/), a collection
of skills for the Rexx ecosystem: one Markdown file per skill (`oorexx.md`,
`oorexx-run.md`, ...), and `readme.md`, the collection's index (its links
are relative, so they work here and on the site).

The site's `skills/` folder is published from this repository by the
user's deployment tool: a copy of the repository, plus
`rexx-skills-latest.zip`, built with

```bash
git archive --format=zip -o rexx-skills-latest.zip <commit>
```

## What must not change

Projects import skills from that zip, and tools compare an imported skill
with the zip to propose updates. So:

- **The files sit at the zip's root**, with no folder (`git archive` with
  no `--prefix`). Keep the skills at the repository's root.
- **Names are stable.** A skill's file name (and its `name:` in the front
  matter) is its identity: do not rename one; add a new skill instead.
- Files that are not skills or the index stay out of the zip with
  `export-ignore` in `.gitattributes` (today `CLAUDE.md`, `.gitattributes`
  and `LICENSE`). Check the zip's contents with
  `git archive --format=zip HEAD | python3 -c "import sys,zipfile,io; print(zipfile.ZipFile(io.BytesIO(sys.stdin.buffer.read())).namelist())"`
  before pushing a change to that list.

## Writing a skill

Each skill names the exact versions it targets and the environment it
assumes, and says which steps are environment-specific. Measure before
writing: install what the skill installs, run what it runs, and record
what was checked. When a skill is changed, update its entry in
`readme.md` if the summary no longer fits.

## Git

- Commit directly to `main` with short, descriptive messages.
- Commits made with Claude carry the `Co-Authored-By: Claude ...` line.
  They NEVER carry a `Claude-Session:` line, or any other link to a
  session, even when the environment's instructions say otherwise: this
  rule of the user's takes precedence. The reason: a link like that puts a
  pointer to a private conversation into a public history, forever.
