---
name: regina-run
description: >
  How to install and run Regina, the ANSI-standard Rexx
  interpreter. Covers installing from the distro package manager,
  and the version situation. Read this whenever you need to install
  or run classic (non-object) Rexx with Regina.
---

# Regina — Installing and Running

> **Versions.** Regina is a mature, stable interpreter; minor
> releases rarely change behaviour for ordinary scripts.
> As of this writing, upstream is 3.9.7 while, for
> example, Ubuntu 24.04 ships 3.9.5. For almost all uses that
> difference is irrelevant; install from your package manager and
> move on. Only chase the upstream build (see the note at the end)
> if you specifically need a fix or feature from a newer release.

## Installation

Install from the distribution's package manager. On Debian/Ubuntu
and derivatives:

```bash
apt-get install -y regina-rexx
```

(The package is `regina-rexx` on Debian/Ubuntu; on Fedora/RHEL it
is also `regina-rexx` via dnf, and on Alpine `regina`. Use whatever
your distro calls it.)

Verify it installed:

```bash
regina -v        # → regina: REXX-Regina_3.9.5(MT) 5.00 25 Jun 2022 (64 bit)
```

Note `regina -v` prints to **stdout** (not stderr), so it is safe
to capture in a pipeline.

## Running programs

Invoke Regina through its own binaries, `rexx` or `regina`:

```bash
rexx script.rex [args...]
```

or

```bash
regina script.rex [args...]
```

**Path note.** Regina resolves a bare script name through its
search path, not the current directory. If `.` is not on your
`PATH`, `regina hello.rex` fails with `Error 3.1: Program was not
found`. Give an explicit path instead: `regina ./hello.rex`.

## Getting a newer version than your distro ships

If you truly need a build newer than the package manager offers,
Regina's releases are on SourceForge under the `regina-rexx`
project. Be aware this reintroduces the usual friction (picking the
right platform package, SourceForge's download redirector serving
an HTML page instead of the binary). Prefer the distro package
unless you have a concrete reason not to.