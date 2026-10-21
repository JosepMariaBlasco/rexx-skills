---
name: apache-cgi-oorexx
description: >
  How to set up Apache with CGI to run ooRexx scripts. Covers
  installation, an example Apache config, and Linux case-sensitivity traps.
  Read this whenever you need to serve ooRexx via CGI.
---

# Apache + CGI with ooRexx

This skill serves **ooRexx** scripts over CGI, using the `rexx`
interpreter and `REXX_PATH` so that scripts can load their external
routines through `Call` or `::Requires`.

Prerequisite: ooRexx installed and runnable (`rexx -v`). See
the `oorexx-run` skill if you need to install it.

> **Environment note.** The paths below (`/path/to/cgi`, `/path/to/bin`,
> etc.) are placeholders — substitute the actual locations on your
> machine. The commands assume a Debian/Ubuntu Apache layout
> (`apt`, `a2enmod`, `/var/log/apache2/`); adapt for other distros.

## Installation

```bash
apt-get update -qq && apt-get install -y -qq apache2
a2enmod cgid actions
```

## Verification

Check that Apache starts and answers before configuring anything
further:

```bash
service apache2 start
curl -s http://localhost/
```

Once your CGI is configured (below), request one of
those URLs instead, e.g. `curl -s http://localhost/cgi-bin/foo.rex`.
Errors go to `/var/log/apache2/error.log`.

## Apache configuration

Two example setups, illustrating different patterns. They can coexist
(the second overrides the handler for `.md` files only).

### Serving CGI scripts directly

The standard CGI pattern: a request for `/cgi-bin/foo.rex` runs that
script and returns its output.

```apache
ScriptAlias /cgi-bin/ /path/to/cgi/
<Directory /path/to/cgi>
    Options +ExecCGI
    AllowOverride None
    Require all granted
    SetHandler cgi-script
</Directory>
```

### Serving `.md` files through a Rexx handler

A different pattern: instead of executing a requested `.rex`, every
`.md` file is fed *through* a Rexx CGI that processes it and returns
the result. This is how a Markdown file gets rendered on the fly
rather than served raw.

```apache
Action Markdown /cgi-bin/markdown_processor.rex
<FilesMatch "\.md$">
    SetHandler Markdown
</FilesMatch>
```

## Shell wrapper (for CRLF shebangs)

A shell wrapper is useful when the `.rex` CGI scripts are edited on
Windows: a `#!/usr/local/bin/rexx` shebang saved with `CRLF` line
endings leaves a trailing `\r` glued to the interpreter path, so the
kernel looks for `/usr/local/bin/rexx\r`, fails to find it, and the
script dies with `not found` (exit 127). Because Apache invokes the
wrapper through `Action`/`ScriptAlias` rather than through the
script's own shebang line, the wrapper's clean shebang is what runs,
and its `exec rexx ...` launches the real script regardless of the
broken shebang inside it.

```bash
cat > /path/to/cgi-wrapper.sh << 'WRAPPER'
#!/bin/bash
export REXX_PATH="/path/to/cgi:/path/to/bin"
exec rexx /path/to/CGI-script.rex "$@"
WRAPPER
chmod +x /path/to/cgi-wrapper.sh
chmod +x /path/to/CGI-script.rex
```

If your scripts have clean `LF` shebangs (or you run them with
`SetHandler cgi-script` directly), you don't need the wrapper — set
`REXX_PATH` with `SetEnv` in the `<Directory>` block instead.

## Case sensitivity on Linux

`::Requires` and `CALL` must reference filenames that match what is
on disk exactly. On Windows it doesn't matter; on Linux,
`HTTP.Request.cls` ≠ `http.request.cls`.
