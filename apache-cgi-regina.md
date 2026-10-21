---
name: apache-cgi-regina
description: >
  How to set up Apache with CGI to run Regina Rexx scripts. Covers
  installation, an example Apache config, loading external routines via
  REGINA_MACROS, and Linux case-sensitivity traps. Read this whenever you
  need to serve Regina Rexx via CGI.
---

# Apache + CGI with Regina Rexx

## Installation

```bash
apt-get update -qq && apt-get install -y -qq apache2 regina-rexx
a2enmod cgid actions
a2disconf serve-cgi-bin
```

The last line matters if you use `/cgi-bin/` as below. Debian and
Ubuntu's Apache enables `serve-cgi-bin.conf` by default, which has
its own `ScriptAlias /cgi-bin/ /usr/lib/cgi-bin/`, and that one wins:
a request for `/cgi-bin/foo.rex` then gives a 404, with `script not
found or unable to stat: /usr/lib/cgi-bin/foo.rex` in the error log.
Disable it (or use another URL prefix for your scripts).

Run `apt-get update` first: a stale package index can make the Apache
`.deb` fetches 404.

## Verification

Check that Apache starts and answers before configuring anything
further:

```bash
service apache2 start
curl -s http://localhost/
```

Once your CGI is configured (below), request one of those URLs instead,
e.g. `curl -s http://localhost/cgi-bin/foo.rex`. Errors go to
`/var/log/apache2/error.log`.

## Apache configuration

Two example setups, illustrating different patterns. They can coexist
(the second overrides the handler for `.md` files only).

### Serving CGI scripts directly

The standard CGI pattern: a request for `/cgi-bin/foo.rex` runs that
script and returns its output. Give the script a Regina shebang, e.g.
`#!/usr/bin/regina` (or `#!/usr/bin/rexx`, depending on where the
interpreter lives on your system).

```apache
ScriptAlias /cgi-bin/ /path/to/cgi/
<Directory /path/to/cgi>
    Options +ExecCGI
    AllowOverride None
    Require all granted
    SetHandler cgi-script
    SetEnv REGINA_MACROS "/path/to/cgi/lib"
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

## Loading external routines

A CGI script pulls in external code with `CALL`:

```rexx
Call "render.rex"
```

To resolve the name, Regina searches `REGINA_MACROS`, then the current
directory, then `PATH`, in that order. In a CGI you don't control the
working directory or `PATH` of the Apache process, so the reliable route
is `REGINA_MACROS`, set in the `<Directory>` block with `SetEnv` (shown
above) — verified with mod_cgid. For command-line runs, set it in the
environment:

```bash
REGINA_MACROS=/path/to/cgi/lib regina ./main.rex
```

## Shell wrapper (for CRLF shebangs)

A shell wrapper is useful when the `.rex` CGI scripts are edited on
Windows: a `#!/usr/bin/regina` shebang saved with `CRLF` line endings
leaves a trailing `\r` glued to the interpreter path, so the kernel looks
for `/usr/bin/regina\r`, fails to find it, and the script dies with
`not found` (exit 127). Because Apache invokes the wrapper through
`Action`/`ScriptAlias` rather than through the script's own shebang line,
the wrapper's clean shebang is what runs, and its `exec regina ...`
launches the real script regardless of the broken shebang inside it.

```bash
cat > /path/to/cgi-wrapper.sh << 'WRAPPER'
#!/bin/bash
export REGINA_MACROS="/path/to/cgi/lib"
exec regina /path/to/CGI-script.rex "$@"
WRAPPER
chmod +x /path/to/cgi-wrapper.sh
chmod +x /path/to/CGI-script.rex
```

If your scripts have clean `LF` shebangs (or you run them with
`SetHandler cgi-script` directly), you don't need the wrapper — set
`REGINA_MACROS` with `SetEnv` in the `<Directory>` block instead.

## Case sensitivity on Linux

`CALL` must reference filenames that match what is on disk exactly. On
Windows it doesn't matter; on Linux, `Render.rex` ≠ `render.rex`.
