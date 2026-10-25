---
name: oorexx-run
description: >
  How to install and run ooRexx. Three choices: the tested 5.3.0
  beta revision (r13289, the default), the latest 5.3.0 beta build,
  or the 5.2.0 release. Covers installation from SourceForge,
  running programs, the ::Requires search path, and common errors.
  Read this whenever you need to install or execute ooRexx code.
---

# ooRexx — Installing and Running

> **Environment note.** This skill is consumed by more than one
> project. The download and verification steps are universal; the
> Linux distribution (`ubuntu2404` below) and the fallback upload
> path are environment-specific — substitute the values that match
> the machine you are installing on.

## Which ooRexx

There are three choices. **Install the tested revision unless the
user asks for another one.**

| Choice | Version | SourceForge folder | When |
|---|---|---|---|
| **Tested** (default) | 5.3.0 beta, build **r13289** | `oorexx/5.3.0beta/` | Always, unless the user says otherwise |
| Latest | 5.3.0 beta, the most recent build | `oorexx/5.3.0beta/` | The user asks for the latest build, or needs a fix newer than the tested revision |
| Release | **5.2.0**, build r13156 | `oorexx/5.2.0/` | The user asks for the released (non-beta) ooRexx |

- **Tested** is the revision the maintainers of this collection have
  tested their own ooRexx projects against (r13289 since 25 October
  2026). 5.3.0 is a beta; its builds come from the development
  trunk, and this one is known to be good.
- **Latest** is the newest build in the `5.3.0beta` folder. Builds
  appear there as the trunk moves, sometimes several a day, and are
  not individually tested: use it only when asked.
- **Release** is the latest official release. Some features of
  5.3.0 are missing (for example `.LogicalWrapper`); use it only
  when asked, or when the code must run on the released version.

The packages are named
`oorexx-<version>-<build>.<distro>.<arch>.deb`, e.g.
`oorexx-5.3.0-13289.ubuntu2404.x86_64.deb` (tested),
`oorexx-5.2.0-13156.ubuntu2404.x86_64.deb` (release). Confirm your
distro with `cat /etc/os-release` if unsure.

## Installation

ooRexx is installed from an official `.deb` published by the Rexx
Language Association on SourceForge.

1. **Choose the package** for your choice above, distribution and
   architecture. For **latest**, find the most recent build for
   your distribution in the folder's listing (builds for the
   different platforms appear over a few hours, so the newest
   number may not exist yet for yours):

   ```bash
   curl -sL https://sourceforge.net/projects/oorexx/files/oorexx/5.3.0beta/ \
     | grep -o 'oorexx-5\.3\.0-1[0-9]*\.ubuntu2404\.x86_64\.deb' | sort -u | tail -1
   ```

2. **Download from `downloads.sourceforge.net`** (it redirects to a
   mirror that serves the binary), with the folder and the file
   name of step 1:

   ```bash
   curl -L -f -o /tmp/oorexx.deb \
     "https://downloads.sourceforge.net/project/oorexx/oorexx/5.3.0beta/oorexx-5.3.0-13289.ubuntu2404.x86_64.deb"
   ```

   For the release, the folder is `5.2.0` and the file
   `oorexx-5.2.0-13156.ubuntu2404.x86_64.deb`. Do not use the
   `/download` links of the SourceForge web pages: they return an
   HTML page, not the binary. The `master.dl.sourceforge.net` mirror
   is no use either: it now redirects to plain `http://`, which
   proxies may refuse.

3. **Verify it's really a `.deb` before installing** (catches the
   case where you got an HTML error page instead):

   ```bash
   file /tmp/oorexx.deb            # → "Debian binary package"
   dpkg-deb -I /tmp/oorexx.deb     # → Package: oorexx, Version: 5.3.0
   ```

   If `file` reports HTML, the download served an error page rather
   than the binary — recheck the URL.

4. **Install, then refresh the library cache:**

   ```bash
   dpkg -i /tmp/oorexx.deb && ldconfig
   ```

   The package runs no `ldconfig` of its own. `rexx` does not need
   it (it finds its libraries through its RUNPATH,
   `$ORIGIN/../lib`), but anything else that loads `librexx.so.4`
   does: a Python bridge, a native extension, a C program linked
   against the API. Without `ldconfig` they fail with
   `librexx.so.4: cannot open shared object file`. Installing
   another package later replaces the one installed (both ways:
   beta to release and back); run `ldconfig` again after it.

**Fallback (network blocked, e.g. proxy 403).** Ask the user to
upload the `.deb` and install from the upload. Where uploads land
depends on the environment; in Claude's containers they have been
seen in `/mnt/user-data/uploads/`, and in
`/mnt/user-data/attachments/<session>/` or
`/root/.claude/uploads/<session>/` with a prefix added to the file
name. Look in all three:

```bash
DEB=$(find -H /mnt/user-data/uploads /mnt/user-data/attachments /root/.claude/uploads \
        -name '*oorexx-*.deb' 2>/dev/null | head -1)
dpkg -i "$DEB" && ldconfig
```

**Verify the interpreter runs, and that it is the build you
chose:** `rexx -v` → `Open Object Rexx Version 5.3.0 r13289` (for
the release, `5.2.0 r13156`).

## Running programs

Scripts: `rexx script.rex [args...]`

To check that a `.cls` loads without errors:
`rexx -e "Call 'path/to/file.cls'"`

## Dependency lookup (::Requires)

ooRexx searches in this order:
1. Directory of the program that issues the `::Requires`
2. Current directory
3. Directories in `REXX_PATH`
4. Directories in `PATH`

This allows referencing dependencies in the same directory without a path.

To add a directory: `export REXX_PATH="/path/to/libs"`

## Common errors

- **"rexx: command not found"** — `.deb` not installed. `dpkg -i`.
- **"Error 43: Routine not found"** — a `::Requires` is missing.
  Check that the `.cls` exists at the expected path.
- **"Error 98: Execution error"** — a `::Requires` failed to load.
  Read the full error, it usually includes the failing path.
- **"librexx.so.4: cannot open shared object file"** (from Python,
  a native extension or a program linked against ooRexx, while
  `rexx` itself works) — the library cache was not refreshed after
  installing: `ldconfig`.
- **Other shared library errors** — `apt-get install -f`.
