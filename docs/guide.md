# Writing a lexer with Lexie

> Works with pooka version >= 0.3.2

A lexer gives names to the pieces of a language. Lexie needs a small TOML
specification; it writes the token classes, scanner, source locations and an
optional highlighter as ordinary Pooka modules.

## Start with a private specification

```sh
pooka add lexie --dev
pooka run -m lexie init
```

`init` creates `.lexie/lexie.toml` and adds `/.lexie/` to `.gitignore`.
It leaves `pooka.toml` unchanged. The specification stays local; the generated
Pooka files are the part to commit. Existing specifications are never replaced.

Open `.lexie/lexie.toml`, choose the tokens, then generate:

```sh
pooka run -m lexie
```

Use `--config .lexie/another-lexer.toml` to select a different specification.
`init --config .lexie/another-lexer.toml` creates that file inside the ignored directory too.

## Choose the generated directory

```toml
location-info = true

[output]
dir = "src"

[module]
name = "my_app/scanner"
```

This writes:

```text
src/my_app/scanner/location.pooka
src/my_app/scanner/token.pooka
src/my_app/scanner/lexer.pooka
```

Import the scanner with `import my_app/scanner/lexer`. Change `module.name` to
choose another directory, or omit it to generate directly in `output.dir`.
A nested source root such as `dir = "src/generated"` is included in imports:
`src/generated/my_app/scanner/lexer.pooka` imports its siblings through
`generated/my_app/scanner`. Filesystem output roots outside `src` are also
available to tools that collect the generated files elsewhere.

## Describe the words

Here is a small complete specification:

```toml
location-info = true

[output]
dir = "src"

[module]
name = "my_app/scanner"

[keywords]
items = ["let", "if", "else"]

[operators]
items = ["==", "=", "+", "-"]

[punctuation]
items = ["(", ")", ","]

[comments.line]
start = "//"
emit = false
```

Fixed spellings become classes: `let` becomes `Let`, `==` becomes `EqualEqual`.
Longer spellings are tried first. A keyword is recognised only at an identifier
boundary, so `if` and `iffy` remain different things.

Use `[literals] items = ["true", "false", "nil"]` for fixed literal values.
Names that clash with Pooka's built-in types receive their section as a suffix,
such as `TrueLiteral`. Set a name explicitly when a symbol deserves one:

```toml
[tokens.names]
"|>" = "Pipe"
```

The symbol must also appear in its token section. Word underscores join words:
`else_if` becomes `ElseIf`. Leading, trailing and repeated underscores are
spelled out, as are punctuation marks. An unsupported spelling needs a name
in `[tokens.names]`.

## Identifiers and whitespace

```toml
[identifier]
enabled = true
start = ["a-z", "A-Z", "_"]
continue = ["a-z", "A-Z", "0-9", "_"]
unicode = true
token = "Ident"

[whitespace]
items = [" ", "\t", "\n", "\r"]
emit = false
token = "Whitespace"
```

Character classes accept ASCII ranges or explicit sets. Unicode identifiers
are optional. Skipped whitespace still advances the location. Set `emit = true`
when a formatter or other tool needs whitespace tokens.

## Numbers

```toml
[numbers.int]
enabled = true
token = "IntLiteral"
separator = "_"

[numbers.float]
enabled = true
token = "FloatLiteral"
exponent = ["e", "E"]
exponent_after_int = true

[numbers.prefixes]
hex = "0x"
binary = "0b"
octal = "0o"

[numbers.suffixes]
items = ["u8", "i64"]
```

Numbers retain their spelling. The scanner does not turn a literal into a
machine number: that belongs to the next stage of the language. Integer and
float scanning can be disabled independently, but floats require integers.

## Strings and characters

```toml
[strings.double]
quote = '"'
escapes = true
multiline = false
token = "StringLiteral"

[strings.raw]
quote = '"'
prefix = "r"
escapes = false
token = "RawString"
```

Each named string style chooses its delimiter, optional prefix, escaping and
whether it crosses line breaks. Token payloads exclude the delimiters and
preserve escape spellings. An unfinished literal becomes an unexpected token;
scanning continues where possible.

Characters may use matching quotes or a prefix:

```toml
[chars]
quote = "'"
escapes = true
token = "Character"
```

For a prefix form such as `$a`, use `prefix = "$"` instead of `quote`.

## Comments

```toml
[comments.line]
start = "//"
emit = false

[comments.doc]
start = "///"
emit = true
token = "DocComment"

[comments.module]
start = "////"
emit = true
token = "ModuleComment"

[comments.block]
start = "/*"
end = "*/"
nested = true
emit = false
```

More specific line-comment prefixes win. Block comments may nest when asked.
Skipped comments keep their source positions, just as skipped whitespace does.

## Tokens and locations

With `location-info = true` (the default), `lexer.lex(source)` returns
`List(Lexeme)`. Each lexeme has `token` and `span` fields:

```pooka
import my_app/scanner/lexer
import my_app/scanner/location.{Location}

let lexemes = lexer.lex("let answer = 42")
loop lexeme in lexemes {
  echo #(lexeme.token, Location("input.tiny", lexeme.span).format())
}
```

`Position(line, column, byte_offset)` uses one-based lines and character columns,
and zero-based UTF-8 byte offsets. `Span(start, end)` is half-open: `end` points
just beyond the token. CRLF counts as one line break; standalone CR and LF are
also recognised. `position.format()` returns `line:column`,
`span.byte_length()` returns its byte length, and `Location(path, span).format()`
adds a file name, such as `input.tiny:1:1`.

Set `location-info = false` for `List(Token)` instead. The generated
`lexer.lex_with_locations(source)` always returns located lexemes, independently
of that switch. The final token is always `End`; unexpected input is represented
by `Unexpected(text)`, which may be renamed with `[errors] token = "Invalid"`.

## Highlighting

```toml
[highlighter]
enabled = true
```

The extra `highlighter.pooka` module returns `Highlight(category, text, span)`
values. Categories describe meaning—keyword, identifier, number, comment and so
on—without choosing terminal colours or HTML. It keeps spans even when the
ordinary `lex` entry point returns plain tokens. End-of-file is not a highlight.

## Commands

| Command | What it does |
| --- | --- |
| `pooka run -m lexie init` | Create and ignore `.lexie/lexie.toml` |
| `pooka run -m lexie` | Validate, generate and format the Pooka modules |
| `pooka run -m lexie check` | Show all specification problems together |
| `pooka run -m lexie check --verbose` | Also list the generated token classes |
| `pooka run -m lexie clean` | Remove files carrying Lexie's generated marker |
| `pooka run -m lexie --config .lexie/other.toml` | Generate from another private specification |
| `pooka run -m lexie --help` | Show command help |

Generation checks all destinations before writing. A handwritten file or
symlink is not overwritten. `clean` applies the same ownership rule.

## Generate without writing

```pooka
import lexie

let text = "[keywords]\nitems = [\"let\"]\n"
let assert Ok(#(specification, problems)) = lexie.read(text)
let result = lexie.generate(specification)
```

`read` reports malformed TOML as an error. Otherwise it returns the specification
and its reading problems. `generate` validates the configuration and returns
`GeneratedFile(path, content)` values or all configuration errors. It does not
touch the filesystem. Check `problems` before using a specification in another
tool. Unknown settings, wrong types, duplicate spellings, ambiguous names and
invalid delimiters are reported rather than guessed.

## Tests and snapshots

Lexie's test suite uses Poke as a development dependency. `pooka test` runs unit
cases, named regressions, seeded properties and generated-source snapshots. It
also generates fixture languages in a temporary project and executes their Poke
tests against the resulting code.

Review deliberate generator changes with:

```sh
pooka run -m poke review
```

Accepted snapshots live in `test/snapshots`. Poke is not a runtime dependency of
Lexie or of its generated lexer. Applications can add it to their own development
dependencies when writing lexer tests.
