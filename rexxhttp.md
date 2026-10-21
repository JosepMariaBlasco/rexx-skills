# RexxHTTP — Serving Servlets

> **This skill targets RexxHTTP 1.0**, running on **ooRexx 5.3.0 or
> later** and **Apache 2.4 or later** with the `cgid` and `actions`
> modules. It was written and tested against that combination.
>
> **Requires the `apache-cgi-oorexx` skill**, which in turn requires
> `oorexx-run`. This skill assumes you already have ooRexx installed
> (`rexx -v` works) and Apache serving ooRexx CGI scripts. It does *not*
> repeat that setup — do those two first. What this skill adds is the
> RexxHTTP processor on top: deploying its sources, wiring Apache to it,
> writing a servlet, and verifying the whole pipeline answers.
>
> **Environment note.** The paths below (`/var/www/html/...`) and the
> commands (`apt`, `a2enconf`, `/var/log/apache2/`) assume a
> Debian/Ubuntu Apache layout. Substitute the locations that match your
> machine.

## What RexxHTTP is

RexxHTTP is a servlet processor for ooRexx: a servlet is an ordinary
ooRexx program that receives a `request` and a `response` object and
produces its body with plain `Say` or `CharOut`. The processor builds
those objects, runs your servlet, and flushes the response. The central
design idea is *lazy headers* — you can write body content before fixing
the response headers, and they are emitted on the first flush.

The pipeline, end to end: Apache routes a request to a one-line **wrapper**,
which `Call`s `RexxHTTP.rex` (the processor); the processor builds the
request/response objects, runs the servlet named by the request path, and
flushes. You configure that route once; after that, adding a servlet is
one line.

## 1. Get the package

The sources are distributed as a zip. Download it, and extract the
processor sources plus one example servlet to use for verification:

```
curl -sO https://rexx.epbcn.com/rexxhttp/rexxhttp-latest.zip
unzip -qo rexxhttp-latest.zip -d /tmp/rexxhttp
```

You need two things from it: everything under `src/` (the processor
`RexxHTTP.rex` and its four classes — they load each other via
`::Requires`, so they travel together) and one servlet to test with.
Each example is a servlet file named `index` inside its own subfolder, so
the hello-world servlet is `examples/hello-world/index` — a one-liner
(`Say "Hello, world!"`) and ideal for verification. (Don't confuse it with
`examples/index`, which is the examples catalogue, not a servlet.)

## 2. Deploy the processor and the wrapper

The processor sources live in one directory; the servlets you serve live
in another. Deploy the five sources, then create the wrapper:

```
sudo mkdir -p /var/www/html/RexxHTTP
sudo cp /tmp/rexxhttp/src/RexxHTTP.rex \
        /tmp/rexxhttp/src/HTTP.Request.cls \
        /tmp/rexxhttp/src/HTTP.Response.cls \
        /tmp/rexxhttp/src/HTTP.OutputStream.cls \
        /tmp/rexxhttp/src/HTTP.Cookie.cls \
        /var/www/html/RexxHTTP/
```

The **wrapper** is a one-line ooRexx script whose only job is to `Call`
the processor. It must be saved with **LF** line endings and have a clean
`#!` shebang, because the kernel exec's *it* directly:

```
sudo tee /var/www/html/RexxHTTP/wrapper > /dev/null << 'WRAP'
#!/usr/bin/env rexx
Call "/var/www/html/RexxHTTP/RexxHTTP.rex"
WRAP
sudo chmod 755 /var/www/html/RexxHTTP/wrapper
```

Why the wrapper at all: the `.rex`/`.cls` sources are stored and checked
out as **CRLF** (they are edited on Windows; ooRexx reads CRLF fine). A
kernel cannot exec a script whose `#!` line ends in CR — it would look
for `/usr/bin/env rexx\r` and fail with `not found` (exit 127). The
wrapper sidesteps this: its own shebang is clean LF, and `RexxHTTP.rex`
is reached through `Call`, never exec'd, so its CRLF shebang is just an
inert comment. Do **not** "fix" the sources to LF — keep them CRLF and
let the wrapper do its job.

## 3. Deploy a servlet

Servlets live in their own directory, gated by a whitelist `.htaccess`:

```
sudo mkdir -p /var/www/html/test
sudo cp /tmp/rexxhttp/examples/hello-world/index /var/www/html/test/hello-world
```

The `.htaccess` lists *exactly* which files are handled as servlets;
everything else in the directory is served as a static file. This is a
whitelist on purpose — see the warning under "Adding more servlets".

```
sudo tee /var/www/html/test/.htaccess > /dev/null << 'HT'
<Files hello-world>
  SetHandler RexxHTTPCGI
</Files>
HT
```

## 4. Configure Apache

This wires Apache to the processor. The `Action` points at the
**wrapper**, never at `RexxHTTP.rex` directly (for the CRLF-shebang
reason above). Write it as a conf snippet and enable it:

```
sudo tee /etc/apache2/conf-available/rexxhttp.conf > /dev/null << 'CONF'
ServerName localhost

<Directory "/var/www/html/RexxHTTP">
    Options +ExecCGI
    AddHandler cgi-script .rex
    Require all granted
</Directory>

ScriptAlias /RexxHTTP/ /var/www/html/RexxHTTP/

# Action points at the wrapper (one-line LF Rexx that Calls RexxHTTP.rex),
# not at RexxHTTP.rex directly: the wrapper's LF shebang keeps the kernel
# happy, and RexxHTTP.rex is reached via Call so its CRLF shebang is inert.
Action RexxHTTPCGI /RexxHTTP/wrapper

<Directory "/var/www/html/test">
    AllowOverride All
    Options +ExecCGI
    Require all granted
</Directory>
CONF
sudo a2enconf rexxhttp
sudo service apache2 restart
```

`AllowOverride All` on the servlet directory is what lets the `.htaccess`
take effect. Without it, the whitelist is ignored and your servlets are
served as static text.

## 5. Verify

Request the servlet. You should get **HTTP 200** and the body:

```
curl -s -i http://127.0.0.1/test/hello-world
```

Expected:

```
HTTP/1.1 200 OK
...
Content-Type: text/plain; charset=utf-8

Hello, world!
```

Note the response convention: **headers are CRLF, the body is LF** (per
RFC for the headers; the body carries whatever the servlet wrote). A
plain `Say` produces `text/plain; charset=utf-8` by default — a servlet
sets `response~content_type = "text/html"` when it wants HTML.

If you get the *source code* of the servlet back instead of `Hello,
world!`, the `.htaccess` is not taking effect — check `AllowOverride All`
and that `actions` is enabled. If you get a 500, look in
`/var/log/apache2/error.log`.

## Adding more servlets

This is where the two-directory montage pays off. To add a servlet, drop
the file in `/var/www/html/test/` and add one stanza to the `.htaccess`:

```
<Files my-servlet>
  SetHandler RexxHTTPCGI
</Files>
```

`.htaccess` is read per request, so a new servlet is live immediately —
**no Apache reload**. The existing servlets keep working untouched.

**The whitelist is a safety boundary, not a formality.** A file that is
*not* listed is served as a static file — its source code goes out
verbatim. Never rely on "it's in the servlets directory" to mean "it
runs": if it is not in the `.htaccess`, its source is public. List every
servlet, and keep nothing secret in that directory that is not listed.

## Writing servlets — traps that bite

A servlet is ordinary ooRexx, but a few non-obvious rules will catch you:

- **The invocation contract is frozen.** A servlet receives
  `Use Arg request, response`; `.request` and `.response` also exist in
  the environment; `Say` and `CharOut` write the body. Production
  servlets rely on this exactly — don't expect a different signature.

- **Line endings: `.rex`/`.cls` are CRLF, the wrapper is LF.** Keep the
  processor sources CRLF (ooRexx reads them fine; the wrapper protects the
  exec path). Only the wrapper must be LF.

- **`&` does not short-circuit in ooRexx.** For an "exists *and* matches"
  guard, use a comma in the `If`, not `&`:
  `If request~exists("q"), request~arg("q") \== "" Then`. With `&` both
  sides evaluate and a missing value blows up at runtime — and a test
  outside Apache may not reproduce it.

- **JSON needs its own `::Requires`.** `.json` is not one of the four
  classes the processor loads. A servlet that parses or builds JSON must
  end with `::Requires "json.cls"`, or it dies with *Error 97, method not
  found*. (A `Signal On Syntax` handler can mask this as a parse failure —
  if JSON "won't parse", check the requires first.)

- **Linux is case-sensitive.** `::Requires "HTTP.Request.cls"` must match
  the filename on disk exactly. `http.request.cls` is a different file.

- **Output encoding.** The response object offers encoders for output
  (`encodeHTML` for the five HTML entities `& < > " '`,
  `encodeURIComponent`, `encodeURI`, `encodeForm`) and the request offers
  the matching decoders (`decodeURIComponent`, `decodeForm`). Escape any
  external value you echo into a page with `encodeHTML`. See the manual's
  Class Reference for the full set.

## Common errors

- **Servlet source served instead of run** — `.htaccess` not taking
  effect. Check `AllowOverride All` on the servlet `<Directory>`, that the
  file is listed in the whitelist, and that `a2enmod actions` was run.
- **500, `error.log` shows exit 127 / "not found"** — the CRLF-shebang
  trap. The `Action` is pointing at `RexxHTTP.rex` directly, or the
  wrapper was saved CRLF. The wrapper must be LF and the `Action` must
  point at it.
- **500, `Error 43: Routine not found`** — a source did not deploy. All
  five files (`RexxHTTP.rex` + the four `HTTP.*.cls`) must sit together in
  the processor directory.
- **500, `Error 97: ... json`** — a JSON servlet without
  `::Requires "json.cls"`. Add it.
- **404 from the processor itself** — `RexxHTTP.rex` guards against being
  run as a servlet directly and returns 404. Requests must go through the
  wrapper to a servlet, not to the processor.

## A full deployment, for reference

The montage above is the real two-directory pattern, kept minimal. For a
complete, multi-servlet deployment with environment overrides and an
automated montage — the same shape, exercised end to end — see
`tests/http/` in the package: `RunHTTPTests.rex` deploys the five sources
plus the wrapper, writes this exact Apache configuration, and runs a suite
of fixture servlets against the live stack. Its `readme.md` documents the
montage and the fixtures.