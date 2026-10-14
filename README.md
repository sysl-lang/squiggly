# squiggly

A template engine for sysl: text with `{{ … }}` tags in it, rendered against a value. It is the
language juicer's themes are written in — Go templates and Liquid in spirit — ported from the Scala
[squiggly](https://github.com/edadma/squiggly) and improved where the port showed how.

```
dependencies {
  squiggly { git = "github.com/sysl-lang/squiggly", version = "0.1.0" }
}
```

The module is `sh.sysl.squiggly`.

```sysl
import sh.sysl.squiggly.*

main()
    val t = parse("hello.html", "<h1>{{ .title }}</h1>{{ for t <- .tags }} #{{ t }}{{ end }}").unwrap()
    val data = new_obj()

    data.put("title", Str("Notes"))
    data.put("tags", strings_of(["sysl", "templates"]))

    engine().render(t, data) match
        Ok(html) -> print(html)
        Err(e) -> print(e.render())
```

## The language

**Text is copied; a tag is evaluated.** `{{ expr }}` prints a value, `{{- ` trims the whitespace
before the tag and ` -}}` the whitespace after it, and `{{// … }}` is a comment (as is `/* … */`
inside a tag, and `//` to the end of a line).

| tag | does |
|---|---|
| `{{ if c }}…{{ elsif c }}…{{ else }}…{{ end }}` | the first true arm (`else if` is `elsif` too) |
| `{{ for .items }}` / `{{ for x <- e }}` / `{{ for x, i <- e }}` | a list's elements (and indexes), a map's values (or `for k, v <- m`); `else` runs when there are none |
| `{{ with e }}…{{ else }}…{{ end }}` | the body with `.` bound to `e`, if `e` is true |
| `{{ match e }}{{ case 1, 2 }}…{{ case 3 }}…{{ else }}…{{ end }}` | the first case equal to `e` |
| `{{ name := e }}` | sets a variable — one table per render, so a loop can count |
| `{{ define name }}…{{ end }}` / `{{ block name e }}…{{ end }}` | layouts: a block renders its definition, or its own body |
| `{{ return e }}` | stops the render and hands `e` back |

**Expressions**, loosest first:

| | |
|---|---|
| `if c then a else b` | a conditional value |
| `or`, `and`, `not` | `or`/`and` answer the deciding operand: `.title or 'Untitled'` |
| `= != < <= > >= div` | chained: `1 < x <= 9`; `a div b` is "a divides b" |
| `x \| f a` | the pipe puts `x` last: `f a x`; pipes chain |
| `+ - ++`, `* / \ mod`, unary `-`, `^` | `++` joins strings or lists; `\` is integer division; `^` is right-associative |
| `f a b` | a function applied to operands |
| `x.name`, `x[i]` | a field, or a one-argument function; an index (negative counts from the end) |
| `.`, `.a.b`, `$`, `$.a`, `$name`, `name` | the current value, a path from it, the root, a path from the root, a variable, a variable or nullary function |
| `` `e` `` | a lambda, evaluated per element by `filter`, `map`, `sortBy` |
| `1`, `2.5`, `'s'`, `"s"`, `true`, `null`, `[1, 2]`, `{a: 1, 'b c': 2}` | literals |

**A space decides two things, and only those.** `a.b` is a field and `a .b` passes `.b` to `a`;
`xs[0]` is an index and `f [0]` passes a list. And `-1` written with a space before it and none after
is a negative argument — `take -1 .xs` — while `n - 1` and `n-1` subtract.

## The value

```sysl
enum Value
    Null
    Bool(b: bool)
    Int(n: long)
    Real(x: real)
    Str(s: string)
    List(items: Buf[Value])
    Obj(entries: Buf[Entry])      // a map that keeps the order it was built in
    Lambda(body: &Expr)
```

- **One `Null`.** A missing key, any field of something missing, and `null` are the same: false,
  printed as nothing. `.page.author.name` is safe to write where there is no author.
- **Two numbers.** Integers stay integers while they can — `6 / 2` is `3`, `7 / 2` is `3.5` — and an
  integer that would overflow continues as a real. `3 = 3.0`. A real prints in its shortest form that
  reads back the same.
- **False** are `null`, `false`, `0`, `""`, `[]` and `{}`.
- **A map keeps its order**, so `for k, v <- .params` follows the front matter.
- `from_toml` and `from_json` convert those packages' values; TOML's dates become ISO 8601 text, which
  `format` and `unix` read. `new_obj().put(k, v)`, `new_list().push(v)`, `list_of`, `strings_of` build
  one by hand.

## The engine

```sysl
val e = engine()                                   // every builtin
e.define("relURL", 1, 1, (c, a) -> ...)            // name, min and max arguments, body
e.partials(name -> ...)                            // Option[Result[Template, TemplateError]]
e.render(t, data)                                  // Result[string, TemplateError]
e.render_layout(baseof, page, data)                // the layout, with the page's `define`s
e.run(t, data, t)                                  // the text and the `return` value
```

A function is `&Fn(*Ctx, Buf[Value]) -> Result[Value, Fail]`; it answers `fail("why")` to refuse,
and the engine places the refusal at the call. `c.apply(lambda, x)` evaluates a lambda and
`c.partial(name, data)` renders a partial, as a builtin of the template's own would.

**A partial answers its text**, so `{{ partial 'card' . }}` prints it and `{{ c := partial 'card' . }}`
keeps it — unless it ran `{{ return x }}`, when it answers `x`.

**Every error is a `TemplateError`**: `message()`, `place()` (`page.html:3:14`), and `render()`,
which quotes the line with a caret under the spot. A parse error says what was expected; an error in
a partial, a block, or a lambda names the line in the template it is written in.

## Builtins

The argument a function works on comes last, so a pipe can supply it.

- **Numbers**: `abs ceil floor round max min sum number int float bool range`
- **Text**: `capitalize lower upper trim ltrim rtrim htmlEscape newline_to_br remove removeFirst
  replace startsWith endsWith split substring truncate urlize slugify urlEncode urlDecode jsonStr
  jsonify toString printf findRE replaceRE`
- **Lists and maps**: `length isEmpty nonEmpty head last tail append prepend reverse distinct compact
  flatten take takeRight drop dropRight slice join contains toSeq keys values sort sortBy intersect
  union symdiff complement filter filterNot map querify default`
- **Dates and the rest**: `format unix partial`

`format` takes `java.time` pattern letters (`yyyy-MM-dd`, `EEEE, MMMM d`, `h:mm a`, `XXX`) and the
names `:date_full`, `:date_long`, `:date_medium`, `:date_short`. `printf` takes C's conversions,
flags, width and precision, and Java's `,` on `%d`. `findRE` and `replaceRE` take POSIX extended
regular expressions.

**Deliberately absent**: `now`, `random` and `shuffle` — a site that renders differently each build
cannot be checked by comparing builds, so a host that wants them defines them. `markdownify`,
`emojify` and `fileExists` belong to the host's Markdown renderer, emoji table and filesystem; this
package opens nothing and reads no clock, and `requires` only `heap`.

## Improvements over Scala squiggly

1. **Application binds tighter than operators, and an argument is one operand.** Scala read `x - 1`
   as the call `x(-1)` and `xs[0]` as the call `xs([0])`; here both mean what they say, and `f x + 1`
   is `(f x) + 1`.
2. **Pipes chain**: `x | f | g a`. Scala's grammar took one.
3. **A tag ends at the delimiter outside any string or map literal**: `{{ '}}' }}` and
   `{{ {a: {b: 1}} }}` work. Scala cut the tag at the first `}}`.
4. **Errors are values with places.** Every parse and render error names the template, line and
   column; Scala threw, and a tag's parse error named a column inside the tag.
5. **`define` is collected when parsing**, and `render_layout(layout, page, data)` uses the page's
   blocks. Scala needed a throwaway render of the page first, and a block a page did not define kept
   the *previous* page's definition.
6. **One `Null`** for null and undefined, and **null-safe paths**: `x.a.b` with `x.a` missing is
   null, where Scala raised "attempt to lookup property of undefined" (`.a.b` was safe, `x.a.b` was
   not).
7. **`or` and `and` answer an operand**, so `.title or 'Untitled'` is a default.
8. **Integers are integers.** BigDecimal printed `1.5 * 2` as `3.0` and `6.02214076e23` as
   `6.02214076E+23`; here `3` and `602214076000000000000000`. (*Changed: LiteralsTests "decimal".*)
9. **A negative index counts from the end**: `xs[-1]`. (*Changed: IndexTests 2 and 8, which expected
   an error.*)
10. **`round` takes a half away from zero**: `round 2.5` is `3`. (*Changed: BuiltinTests "round".*)
11. **A string escape `\\` works**; Scala's lexer refused it. **A string or map key may be quoted**:
    `{'two words': 1}`. **Any word may be a field**: `.end`, `.for`.
12. **`case` may list values**: `{{ case 1, 2 }}`. **`else if`** is accepted for `elsif`. **`block`'s
    expression is optional** and defaults to `.`. Text before a `match`'s first `case` must be
    whitespace, which Scala only noted as a todo.
13. **New builtins**: `range`, `keys`, `values`, `sort`, `sortBy`, `jsonify`, `replaceRE`; `max` and
    `min` take one list too; `complement` takes any number of lists; `querify` encodes.
14. **`default` replaces `null` too**, as well as empty and zero, and keeps `false`. Scala's let a
    `null` through.
15. **A variable shadows a function of the same name**, and a variable given arguments says so.
16. **Builtins dropped**, for the reasons above: `now`, `random`, `shuffle`, `fileExists`, `context`,
    `print`, `println`, `markdownify`, `emojify`, and `time` (a date is ISO 8601 text, which `format`
    and `unix` read directly). (*Changed: BuiltinTests' `time`/`now`/`markdownify`/`emojify`/
    `fileExists` tests, rewritten or dropped.*)
17. **Regular expressions are POSIX ERE**, sysl's engine, which has no lazy `+?`. (*Changed:
    BuiltinTests "findRE (with groups)" uses `[^<]+`.*)

## Migration: Scala squiggly → sysl squiggly

**The templates.** Every layout of juicer's themes and demos, and of sysl's and texish's docs themes
(103 files, in `testdata/corpus/`), parses unchanged, and none relies on a behaviour that changed:
they use no `x -1`, no `f a + b`, no output of an `and`/`or`, no `round`, and no `time`. What to watch for
in a template that is not in the corpus:

- `name -1` is now a call; write `name - 1`. `xs[0]` now indexes.
- `f a + b` is now `(f a) + b`; write `f (a + b)`.
- `{{ a or b }}` prints the operand, not `true`.
- A date is ISO 8601 text: `format 'yyyy' .date` instead of `format 'yyyy' (time .date)`.

**The host (juicer).**

| Scala | sysl |
|---|---|
| `TemplateParser.default.parse(text)` | `parse(name, text)` → `Result[Template, TemplateError]` |
| `new TemplateParser(start, end)` | `parse_with(name, text, start, end)` |
| `new TemplateRenderer(partials, functions = builtins ++ mine)` | `var e = engine()`, then `e.define(...)` per function and `e.partials(loader)` |
| `TemplateFunction(name, arity, { case (con, Seq(...)) => ... })` | `e.define(name, min, max, (c, args) -> ...)`, answering `Ok(v)` or `fail("why")` |
| `TemplateLoader = String => Option[TemplateAST]` | `Loader = &Fn(string) -> Option[Result[Template, TemplateError]]` |
| render the page (to fill blocks), then render baseof | `e.render_layout(baseof, page, data)` |
| `renderer.render(data, ast, out)` | `e.render(t, data)` → `Result[string, TemplateError]` |
| data as `Map[String, Any]`, case classes | `Value`: `new_obj()` + `put`, `from_toml`, `from_json` |
| `NonStrictExpr` argument | `Lambda`, evaluated with `c.apply(f, x)` |
| `problem(...)` / exceptions | `fail("message")`; an error from `c.apply` or `c.partial` passes through with `?` |

juicer defines `relURL`, `absURL`, `relLangURL`, `absLangURL`, `markdownify`, `imageDims`,
`imageVariants`, `srcset`, `asset`, `ogTags` and `i18n` itself, as it did; add `now` if a theme
wants it.

## Tests

`sysl test .` — 153 tests: every Scala squiggly test ported (with the changed expectations above
named where they are), the parse and render errors with their places, every builtin's answer and
refusal, the conversions, and the corpus. The suite passes under
`SYSL_EXTRA_CFLAGS="-fsanitize=address -g"`; the package is pure sysl, so all of it is instrumented.
