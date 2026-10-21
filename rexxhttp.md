---
name: rexxhttp
description: >
  How to install RexxHTTP (ooRexx rexxlets served by Apache over CGI),
  with its automated installer or by hand, verify it, and write
  rexxlets: the request/response API, lazy headers, and the traps.
  Read this whenever you need to serve ooRexx programs as web pages.
---

# RexxHTTP — Serving Rexxlets

> **This skill targets RexxHTTP 1.0** (the package
> `rexxhttp-latest.zip` of 30 July 2026; its manual, `rexxhttp.pdf`,
> edition 2026.07.10), which needs **ooRexx 5.2.0 or later** and
> **Apache 2.4** with the `cgid` (or `cgi`) and `actions` modules.
> Every step below was run on 21 October 2026 on Ubuntu 24.04,
> Apache 2.4.58, ooRexx 5.3.0 r13282.
>
> **Requires `oorexx-run`** (ooRexx installed, `rexx -v` works).
> Apache is installed here, by the installer or by hand.
>
> **Environment note.** The paths and commands (`apt-get`, `a2enmod`,
> `/usr/lib/cgi-bin`, `/var/www/html`, `/var/log/apache2/`) are those
> of Debian and Ubuntu. The manual covers macOS (Homebrew) and Windows
> (Apache Lounge) too. Run as root, or prefix the commands with `sudo`.

## What RexxHTTP is

A processor for **rexxlets**: ordinary ooRexx programs served as web
pages by Apache over CGI. A rexxlet receives a `request` and a
`response` object (also in the environment as `.request` and
`.response`) and writes its body with plain `Say` or `CharOut`. The
central idea is *lazy headers*: the body is buffered, so a rexxlet can
write body before it sets headers; everything is sent at the first
flush (normally the processor's, when the rexxlet ends).

The pipeline: Apache's `Action` routes a request for a rexxlet to the
processor, `RexxHTTP.rex`, which builds the two objects, runs the
rexxlet, and flushes. The manual (`rexxhttp.pdf`, in the package) is
the reference; this skill follows its *Installation* chapter.

## 1. Get the package

```bash
curl -sO https://rexx.epbcn.com/rexxhttp/rexxhttp-latest.zip
unzip -qo rexxhttp-latest.zip -d /tmp/rexxhttp
```

`src/` holds the processor and its classes; `install/` the automated
installer; `examples/` the examples; `tests/http/` the integration
suite.

## 2a. Install with the automated installer (a clean machine)

For a machine **without Apache**: it installs Apache, deploys
RexxHTTP, plants a welcome page and the examples, and checks that the
result serves a request. If Apache is already installed, it stops
without changing anything: install by hand (2b).

```bash
cd /tmp/rexxhttp/install
rexx install --yes          # --yes: no prompts; --help lists the options
```

It ends with `SUCCESS: a minimal RexxHTTP installation is up and
running`. Check it:

```bash
curl -s http://localhost/                            # the welcome page (itself a rexxlet)
curl -s http://localhost/examples/hello-world-plain/ # → Hello, world!
```

What it leaves, on Debian/Ubuntu: the processor in `/usr/lib/cgi-bin/`
(its shebang pinned to the `rexx` it found), the site in
`/var/www/html/`, and `/etc/apache2/conf-available/rexxhttp.conf`, where
**any `*.rxl` file is a rexxlet** (`AddHandler RexxHTTP .rxl`),
`DirectoryIndex` is `wrapper.rxl index.rxl readme.md`, `.md` files go
through a Markdown renderer and `.rex`/`.cls` files through a source
viewer (their source is shown, on purpose). `REXXHTTP_BASE` and
`REXXHTTP_URLBASE` are set for that layout (section 2b explains them).
A new rexxlet is then just a file `something.rxl` in the site.

`--port N` changes the port when 80 is taken. To undo the
installation (it removes Apache too, unless `--keep-apache`):

```bash
cd /usr/local/share/rexxhttp && rexx uninstall --yes
```

The manual also documents the Rexx Highlighter option (the installer
records the Rexx Parser's location if it finds it at install time).

## 2b. Install by hand

When Apache is already there, or to place each piece yourself. These
are the manual's steps for Debian/Ubuntu.

**Apache**, with the two modules:

```bash
apt-get install -y apache2
a2enmod cgid actions
service apache2 restart
```

This recipe uses Debian's own `/cgi-bin/` (the `serve-cgi-bin`
configuration, enabled by default, maps it to `/usr/lib/cgi-bin/`), so
**leave `serve-cgi-bin` enabled** here (the `apache-cgi-oorexx` skill
disables it for its own `/cgi-bin/`; the two set-ups do not mix).

**The sources:** everything in `src/` goes to `/usr/lib/cgi-bin/`,
together (the processor finds its classes there), and the processor
must be executable:

```bash
cp /tmp/rexxhttp/src/*.rex /tmp/rexxhttp/src/*.cls /usr/lib/cgi-bin/
chmod +x /usr/lib/cgi-bin/RexxHTTP.rex
```

That is six files today: `RexxHTTP.rex`, `HTTP.Request.cls`,
`HTTP.Response.cls`, `HTTP.OutputStream.cls`, `HTTP.Cookie.cls`, and
`RexxHTTP.CommonRoutines.cls` (an empty home for helper routines; the
manual still says five files). `RexxHTTP.rex` starts with
`#!/usr/bin/env rexx`, so `rexx` must be on the `PATH` Apache gives
its CGI programs (it is, with SourceForge's `.deb`, which installs
`/usr/local/bin/rexx`); if not, pin the shebang to the interpreter's
absolute path.

**Which files are rexxlets.** The handler is declared once, and each
rexxlet is whitelisted by name; everything else in the directory is
served as a static file:

```bash
cat > /etc/apache2/conf-available/rexxhttp.conf << 'CONF'
Action RexxHTTP /cgi-bin/RexxHTTP.rex
<Directory "/var/www/html">
    Options +ExecCGI
    DirectoryIndex index
    <Files "index">
        SetHandler RexxHTTP
    </Files>
</Directory>
CONF
a2enconf rexxhttp
service apache2 reload
```

Each further rexxlet is one more `<Files>` block. The same blocks work
in an `.htaccess` file inside the directory, **only if** the
`<Directory>` allows it (`AllowOverride All`); otherwise the
`.htaccess` is ignored and the rexxlet's source is served as text.

**Smoke test.** Put a rexxlet named `index` in `/var/www/html/`:

```bash
printf '.response~content_type = "text/plain"\nSay "Hola, mundo"\n' > /var/www/html/index
curl -s http://localhost/        # → Hola, mundo
```

**`REXXHTTP_BASE` and `REXXHTTP_URLBASE`** (`SetEnv` in the
`<Directory>`) matter only when a rexxlet is reached as an *action* for
a resource that lives elsewhere (as the installer's Markdown renderer
is). `BASE` is the zone's root on disk; `URLBASE` is the URL prefix in
front of it that `BASE` does not already include: `*` when there is
none (the document root), `/prefix` for a subdirectory or an `Alias`.
Never leave `URLBASE` empty: with `BASE` set, an empty `URLBASE` is a
500. The manual's table *Three installation geometries* has the cases.

## 3. Verify with the integration suite

The package's own suite stands up a throwaway Apache montage, runs
182 tests through Apache and curl, and removes what it created.
**Stop Apache first**: the runner starts Apache with `apachectl start`,
which does nothing to a server that is already running, so its
configuration is never loaded and most tests fail (151 of 182).

```bash
service apache2 stop
cd /tmp/rexxhttp && rexx tests/http/RunHTTPTests.rex   # → Tests ran: 182, Failures: 0
```

## Writing rexxlets

A rexxlet is an ordinary ooRexx program: no class to subclass, no
entry point. `Use Arg request, response` names the two objects (or use
`.request` and `.response`).

```rexx
.response~content_type = "text/plain"   -- the default: text/plain; charset=utf-8
Say "Hello, world"
```

- **Arguments** (the query string of a GET, the form of a POST) come
  through `arg`, modelled on the built-in ARG: `.request~arg()` is the
  count; `.request~arg("q")` the value of `q` (by name, caseless) and
  `.request~arg(2)` the second one; a second argument chooses `Value`
  (the default), `Name`, `Exists` or `Omitted`. Names and values arrive
  decoded. A request with a malformed or duplicated parameter (`?q=1&Q=2`)
  gets **400** and never reaches the rexxlet.
- **Cookies:** `c = .HTTP.Cookie~new("session"); c~value = "abc";
  c~path = "/"; .response~addcookie(c)`. Read them back with
  `.request~cookie("session", "Value")` (case-sensitive, unlike `arg`).
- **Headers** are message names: `.response~x_late_header = "yes"`
  sends `X-Late-Header: yes` (underscores become hyphens), even after
  body was written, as long as nothing was flushed. After a flush a
  header assignment raises an error.
- **Errors and redirects**, as the last statement:
  `Exit .response~error(404, "no such record")`,
  `Exit .response~redirect("/elsewhere/")` (302 by default).
- **Escaping:** `.response~encodeHTML(text)` for anything external you
  echo into HTML; also `encodeURIComponent`, `encodeURI`, `encodeForm`,
  and on the request `decodeURIComponent`, `decodeForm`.

The traps:

- **`&` does not short-circuit in ooRexx.** For "exists and is not
  empty", use a comma in the `If`:
  `If .request~arg("q", "Exists"), .request~arg("q") \== "" Then`.
- **JSON needs its own `::Requires "json.cls"`**: the processor does not
  load it. Without it: `Error 97.1: Object ".JSON" does not understand
  message "NEW"`.
- **Linux is case-sensitive**: `::Requires` and `Call` must name files
  exactly as on disk.
- **What is not a rexxlet is served as it is.** With the whitelist
  (2b), a file left out of the `<Files>` blocks is sent as static text,
  source included; with the installer's layout (2a), `.rex` and `.cls`
  files are shown as highlighted source by design. Keep nothing secret
  in the served directories.

## Common errors

- **The rexxlet's source comes back instead of its output**: it is not
  marked as a rexxlet (missing from the `<Files>` whitelist, an
  `.htaccess` without `AllowOverride`, or a file without `.rxl` in the
  installer's layout).
- **500, `End of script output before headers`** in
  `/var/log/apache2/error.log`: Apache cannot run `RexxHTTP.rex` (no
  execute bit, or `rexx` not on Apache's `PATH`).
- **A page headed `Internal server error`** with an ooRexx traceback:
  the rexxlet itself failed; the traceback says where.
- **404 when requesting `/cgi-bin/RexxHTTP.rex`**: by design; the
  processor answers 404 when called directly, not through a rexxlet.
- **400**: a malformed or duplicated request argument.
