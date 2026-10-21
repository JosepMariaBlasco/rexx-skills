---
name: oorexx-run
description: >
  How to install and run ooRexx. Pinned to the ooRexx 5.3.0beta
  release branch (latest build of that branch). Covers
  installation from SourceForge, running programs, the ::Requires
  search path, and common errors. Read this whenever you need to
  install or execute ooRexx code.
---

# ooRexx — Installing and Running

> **This skill targets ooRexx 5.3.0beta.** It was written and
> tested against that release branch. The *version* is pinned on
> purpose: the SourceForge layout and the official download page
> are not dependable enough to automate a "latest version" lookup,
> and a newer major/minor line may behave differently. The *build*
> within that branch is not pinned — use the most recent build of
> `5.3.0beta` unless the user asks for a specific one. If you need
> a different version entirely, adjust the steps accordingly.

> **Environment note.** This skill is consumed by more than one
> project. The download and verification steps are universal; the
> Linux distribution (`ubuntu2404` below) and the fallback upload
> path are environment-specific — substitute the values that match
> the machine you are installing on.

## Installation

ooRexx is installed from an official `.deb` published by the Rexx
Language Association on SourceForge.

1. **Choose the build and the matching package.** In the
   `5.3.0beta` folder on SourceForge —
   `https://sourceforge.net/projects/oorexx/files/oorexx/5.3.0beta/` —
   pick the `.deb` for your distribution and architecture with the
   **most recent build number** (`rNNNNN`), *unless the user
   requested a specific build*, in which case use that one. Within
   this single pinned folder the files are all the same version, so
   "most recent build" is unambiguous (sort by date) — this is
   safe in a way that picking the folder itself is not. The
   filename has the form
   `oorexx-5.3.0-<build>.<distro>.<arch>.deb`, e.g.
   `oorexx-5.3.0-13169.ubuntu2404.x86_64.deb`. Confirm your distro
   with `cat /etc/os-release` if unsure.

2. **Download from the direct mirror, not the `/download`
   redirector.** SourceForge's `/download` URL returns an HTML
   interstitial page (~130 KB of HTML, not the binary). Use the
   `master.dl.sourceforge.net` mirror with a browser user-agent
   (substitute the filename you chose in step 1):

   ```bash
   curl -L -f -A "Mozilla/5.0" -o /tmp/oorexx.deb \
     "https://master.dl.sourceforge.net/project/oorexx/oorexx/5.3.0beta/oorexx-5.3.0-13169.ubuntu2404.x86_64.deb?viasf=1"
   ```

3. **Verify it's really a `.deb` before installing** (catches the
   case where you got an HTML error page instead):

   ```bash
   file /tmp/oorexx.deb            # → "Debian binary package"
   dpkg-deb -I /tmp/oorexx.deb     # → Package: oorexx, Version: 5.3.0
   ```

   If `file` reports HTML, the download served an error page rather
   than the binary — recheck the URL.

4. **Install:** `dpkg -i /tmp/oorexx.deb`

**Fallback (network blocked, e.g. proxy 403).** Ask the user to
upload the `.deb` and install from the upload — adjusting the path
to wherever uploads land in your environment (in Claude's
container that is `/mnt/user-data/uploads/`):
`dpkg -i /path/to/uploads/oorexx-*.deb`.

Verify the interpreter runs: `rexx -v` → `Open Object Rexx Version
5.3.0`.

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
- **Shared library errors** — `apt-get install -f`.
