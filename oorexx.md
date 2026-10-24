---
name: oorexx
description: >
  Pitfalls, best practices and traps for ooRexx code. Read this
  whenever writing, reading, editing, or debugging ooRexx code.
  Does NOT cover coding style, only language semantics and traps.
---

# ooRexx — Pitfalls & Best Practices

This is not a language reference. It covers what trips up code
generation — special variables, non-obvious semantics, and common
mistakes — rather than teaching the language from scratch.

---

## Special variables — NEVER use as variable names

`RESULT`, `RC`, and `SIGL` are silently overwritten by the interpreter.
**Never use any of them as a variable name.** Use `response`, `res`,
`outcome`, `ret`, etc. instead.

- `RESULT` — set by unassigned message sends and `CALL`: to the value
  returned, or **dropped** when nothing is returned. Assigned sends
  (`x = obj~method`) do NOT touch it, but unassigned sends
  (`obj~method`) DO.
- `RC` — set by `ADDRESS` commands (command instructions). Equally
  dangerous if the program uses command instructions.
- `SIGL` — set by `CALL` and `SIGNAL` to the source line number.
- `SELF` — the receiver inside a method (safe to read, not to assign).
- `SUPER` — the superclass scope inside a method.

```rexx
-- WRONG: result gets clobbered by the unassigned ~setEntry send
result = .Directory~new
result~setEntry("status", "200")   -- setEntry returns nothing: RESULT is
                                   -- dropped, and result is now "RESULT"!
                                   -- (a send that returns a value, like
                                   -- ~hasIndex, would leave that value)

-- RIGHT: use a different name
response = .Directory~new
response~setEntry("status", "200")
-- response is still the Directory
```

---

## Common pitfalls

### 1. Continuation characters: `,` and `-`

Rexx has TWO continuation characters, `-` and `,`. Both also have
other jobs in the language (the comma separates arguments and acts as
short-circuit AND in `If`, `When` and other logical contests;
the minus is used for subtraction and to start line comments with `--`).
A character is treated as a continuation ONLY when it is the last token
on the line, modulo trailing whitespace and comments. In that case it
is functionally replaced by a blank — which may or may not be
significant depending on context — and, crucially, the automatic
end-of-clause `;` that a line ending normally generates is NOT
emitted: the clause continues onto the next physical line.

```rexx
msg = "hello" -      -- the "-" is a continuation -> replaced by a blank
      "world"        -- msg is "hello world"
```

Only `-` and `,` do this. A line left hanging on some other token
is NOT continued — for example, a trailing `+` is an incomplete
expression and raises Error 35.

The trap is that a continuation SUPPRESSES the `;`. Most of the time
that is what you want. But when the character you put at end-of-line
was meant to do its OTHER job — a comma meant as short-circuit AND, say
— consuming it as a continuation silently changes the meaning of the
clause. That is the subject of the next pitfall.

### 2. Blank between two terms is CONCATENATION-WITH-A-SPACE

Rexx has three concatenation forms, and the dangerous one is invisible.
`a||b` joins with nothing; `a b` (a blank between two terms) joins **with
one blank inserted between them**; abuttal (`a''b`) joins with nothing.
The blank form is *concatenation by juxtaposition* — the space is not
decorative whitespace the parser discards, it is an operator that puts a
literal blank into the result.

```rexx
x = "a" "b"     -- x is "a b"  (3 chars) — the blank is DATA
x = "a"||"b"    -- x is "ab"   (2 chars)
```

This is trivial to get wrong when **building a string out of literals and
values**, because visually the blank reads like harmless code spacing.
The classic bite is assembling SQL or HTML by concatenation:

```rexx
-- WRONG: blank before the closing literal -> space injected INTO the value
sql = "slug = '"slug~changeStr("'","''") "',"
--                                       ^ this blank becomes part of the SQL:
--   slug = 'bankinter ',     <- trailing space written into the column

-- RIGHT: no blank -> the pieces abut exactly as intended
sql = "slug = '"slug~changeStr("'","''")"',"
--   slug = 'bankinter',
```

Here the `~strip` on the *input* is irrelevant: the blank is added *later*,
at build time, by the juxtaposition operator. The value goes into the
database (or the page, or the URL) with a trailing space that is invisible
in most displays and only bites when it lands somewhere space-sensitive —
a slug in a URL path, a key in a lookup, a UNIQUE column. It also survives
re-editing (the form re-injects it on every save), so it does not
self-heal.

Defensive rules when concatenating to build a literal:

1. Between a value and the next quoted fragment, use `||` or abut the
   quotes directly (`"..."value"..."`). Do **not** leave a blank unless
   you actually want a blank in the output.
2. Never add blanks "for alignment" inside a concatenation chain. If you
   want the source readable, break the line with `-` (continuation, which
   *does* insert a blank — so put it where a blank is harmless or use `||`
   explicitly), or assemble via `.mutableBuffer` (see best practices).
3. When a bug looks like "the data has a mysterious trailing space",
   suspect a stray blank in the string-building code before suspecting the
   input or the database.

### 3. Logical operators are NOT short-circuit

`&`, `|`, and `&&` evaluate **both** operands unconditionally. Use
commas (short-circuit AND) in `If` and `When` when a condition guards
a subsequent one:

```rexx
-- WRONG: crashes if i == 1
If ch == "#" & i > 1 & text[i - 1] == " " Then ...

-- RIGHT: commas as short-circuit AND
If ch == "#", i > 1, text[i - 1] == " " Then ...
```

Two conditions make the comma behave as short-circuit AND, and both
must hold:

1. **Top level only** — not inside parentheses. `If a, b Then` is AND;
   `If (a, b) Then` is not (inside parens the comma has other roles:
   argument syntax, array terms, ... —  and the guard breaks).
2. **The comma must not fall at end-of-line**, or it is consumed as a
   continuation (pitfall 1) instead of AND. If the guard spans physical
   lines and a condition ends in a trailing comma, that comma is
   swallowed, the `;` is suppressed, and the conditions concatenate
   rather than AND. Concretely:

   ```rexx
   -- BROKEN: the trailing comma is a continuation, not AND.
   -- Parses as `If a b ; Then ...` -> `a b` concatenates to "1 0",
   -- which is not a logical value -> Error 34.
   If a,
      b -
   Then Say "..."
   ```

   Keep the whole comma-guard on ONE logical line. If it is too long,
   do not break it with `-`; restructure (compute a boolean into a
   variable first, or nest `If a Then If b Then ...`).

### 4. There is no `main` routine

The prolog (code before the first `::directive`) is the entry point.

### 5. Loop Until evaluates at the end

Variables assigned in the loop body are available when `Until` is
first evaluated. This is by design, not a bug. `While` evaluates
at the start.

### 6. Uninitialised variables evaluate to their own uppercase name

```rexx
Say foo        -- prints "FOO" (not an error)
```

Use `Signal On Novalue` to trap this during development.

### 7. EXPOSE and variable scoping in methods

Methods do not see the caller's variables. Instance variables must
be explicitly exposed:

```rexx
::Method increment
  Expose count
  count = count + 1
```

Without `Expose count`, the body starts a fresh local `count` (which
evaluates to `"COUNT"`, see pitfall 6) and the assignment never
touches the instance variable.

Don't write a plain getter or setter this way: `::Method name; Expose
name; Return name` is exactly what `::Attribute name Get` generates
for you. Reach for `Expose` when the method does more than relay an
attribute — like the increment above.

### 8. Assignment operators are pure syntactic expansion

`x[f(n)] += 1` becomes `x[f(n)] = x[f(n)] + 1` — evaluates `f(n)`
twice. Be aware when the left-hand side has side effects.

### 9. `=` is non-strict, `==` is strict

`=` is non-strict: it ignores leading and trailing blanks, and
compares numerically when both sides are numbers (`"1.0" = "1"` and
`" a" = "a"` are both true). `==` is strict: character by character
for strings, and for other objects whatever their class defines,
which is object identity unless the class says otherwise.

Some classes do say otherwise. `.LogicalWrapper` (ooRexx 5.3.0, the
base of `.JsonBoolean` and `.YamlBoolean`) compares by its logical
value with both operators:

```rexx
.LogicalWrapper~true =  1    -- 1
.LogicalWrapper~true == 1    -- 1 as well: == is not identity here
```

**Rule of thumb**: default to `=` for ordinary comparisons; use `==`
when the exact characters matter, and for `.nil` tests. **In tests,
use `assertSame`** (it compares with `==`), not `assertEquals` (it
compares with `=`, so `"1.0"` would pass for `"1"`); use
`assertEquals` only when non-strict equality is really what is meant.

### 10. `stem. = value` resets the entire stem

It drops all existing tails and installs a new default. After
`x.1 = 12; x. = 3`, `x.1` evaluates to `3`, not `12`.

### 11. `stem.tail` is NOT a message send

It is variable name resolution by the interpreter. The dot is a
tail separator, not the `~` operator.

### 12. A label inside a block causes error 47.2

A label may not appear inside a `DO`/`LOOP` block (including the
`do...end` that hangs off a `Select` `When` or an `If`). The
interpreter rejects it: *"Error 47.2: Labels are not allowed within
a DO/LOOP block."* The culprit is the **label**, not `Signal`.

`Signal` itself jumps *out* of a loop perfectly well — it terminates
the active control structures and transfers control to the target
label, which lives at prolog level (outside any block):

```rexx
do i = 1 to 3
  if i = 2 then signal done   -- fine: jumps out of the loop
end
done:
```

What fails is putting the `label:` *inside* the block. If you need a
jump target near loop logic, hoist it out of the block — or extract
the logic into a separate routine.

### 13. `Call LineOut` with no arguments closes standard output

With no stream name, `LineOut` acts on the default output stream,
and with no line it closes that stream. So a bare `Call LineOut`
closes standard output for the **whole program**, even when it runs
inside a `::Routine`: from then on nothing more reaches stdout —
not `Say`, not `LineOut 'STDOUT'`, not `.output~lineOut` — and
there is no error; the program ends with return code 0. Standard
error still works.

```rexx
say 'a'
call lineout          -- closes stdout
say 'b'               -- never appears
```

If the intent is an empty line, write `Say ''` (or
`Call LineOut , ''`). If the intent is to flush standard output,
flush it without closing it: `.output~flush` (or
`.stream~new('STDOUT')~flush`).

### 14. A `RANDOM` seed does not reach a `::Routine`

Seeding with `Random(min, max, seed)` makes the following calls
repeatable in the code that seeded, but not inside a `::Routine` or
a `::Method` it calls: there, `Random` gives different values on
every run. Each call of a routine or method starts a generator of
its own, so a seed given in one call does not carry to the next
call either; and the calls consume nothing from the caller's
sequence.

```rexx
x = random(1, 1000, 42)
say random(1, 1000)   -- the same on every run
say draw()            -- different on every run
exit
::routine draw
  return random(1, 1000)
```

For a repeatable sequence inside a routine, seed inside the
routine, on every call (or pass the random values to it as
arguments).

---

## Best practices

### Prefer `text[i]` over `text~substr(i, 1)`

More legible and more performant for single characters. Use `substr`
only for multiple characters.

### Use `select case` for same variable or expression

```rexx
select case value
  when "null", "Null", "NULL" then ...
  when "true", "True", "TRUE" then ...
  otherwise nop
end
```

### Use MutableBuffer for string accumulation

```rexx
mb = .mutableBuffer~new
do item over collection
  mb~append(item, "0A"x)
end
out = mb~string
```

`append` accepts multiple arguments:
`mb~append("prefix", value, "0A"x)`.

### StringTable iteration order is undefined

Do not rely on key order. For sorted keys: `~allIndexes~sort`.

### Use `makeString('L', separator)` for array-to-string

```rexx
-- WRONG: the first argument is the type ("L" or "C"):
-- Error 93.915: Method option must be one of "CL"
arr~makeString("0A"x)

-- RIGHT
arr~makeString('L', "0A"x)
```

### Line endings: native vs. forced LF

For ordinary text output, use `Say` or `LineOut`. They emit the
platform's **native** line terminator — `LF` on Unix/Linux, `CRLF`
on Windows — which is what the rest of that platform's tools expect.
That is the portable, do-the-right-thing default.

```rexx
do line over arr
  s~lineOut(line)        -- native terminator on each platform
end
```

Only reach for `charOut` with an explicit `"0A"x` when you
deliberately want `LF` everywhere regardless of platform — e.g.
emitting a file destined for a Unix system, or a format that
mandates `LF`:

```rexx
s~charOut(arr~makeString('L', "0A"x))   -- forces LF on every platform
```

Note this is the *opposite* of platform independence: it pins
Unix-style endings even on Windows. Don't use it as a generic way to
"avoid CRLF" — use `LineOut` for native endings, and `charOut`/`"0A"x`
only when a fixed `LF` is the actual requirement. (Also mind that
`makeString('L', sep)` places the separator *between* elements, so
there is no trailing terminator after the last line.)

---

## Resources

- **ooRexx on SourceForge**: <https://sourceforge.net/projects/oorexx/>
- **RexxLA (Rexx Language Association)**: <https://www.rexxla.org>
- **ooRexx Reference (rexxref)**: distributed with every installation
