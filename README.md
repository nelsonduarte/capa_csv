# capa_csv

Pure-Capa CSV parsing and writing to the RFC 4180 grammar. Zero
capabilities: the parser is a `(String) -> Result<Table, CsvError>`
function. The caller reads the file (or fetches the bytes) with
whatever authority it already holds and hands this library a
plain string; nothing here can touch the filesystem, the
network, the clock, or anything else. `capa --manifest` proves
it (see [Audit claim](#audit-claim)).

Real Capa programs that read tabular data (audit-trail
reporters, account ledgers, risk feeds) today split on `","` by
hand, which silently corrupts any field that contains a comma,
a quote, or a newline. This library replaces that hand-rolled
split with a correct state machine, behind a typed model, with
typed errors for malformed input.

## Status

v0.1 (seed library). Covers the RFC 4180 core (quoted fields,
doubled-quote escaping, embedded delimiters and newlines, CRLF
and LF terminators, empty fields, a final record with or without
a trailing newline), a configurable delimiter, a header view
that addresses fields by column name, a writer with correct
quoting, and typed errors for every malformed shape. Validated
against the Python `csv` module as an oracle.

Out of scope for v0.1, by design:

- **Encoding detection.** Input is a Capa `String` (already
  decoded text). Byte-level encoding sniffing (BOM, latin-1,
  UTF-16) is the caller's job before it hands us the string.
- **Column type inference.** Fields are `String`; parsing
  `"42"` to an `Int` or `"3.14"` to a `Float` is the consumer's
  decision, with the consumer's error handling.
- **Exotic dialects.** Configurable quote character, comment
  lines, quoting modes (`QUOTE_ALL`, `QUOTE_NONNUMERIC`), and
  per-field type hints are not modelled. The one knob is the
  delimiter.
- **Streaming large files.** The whole document is parsed into a
  `Table` in memory; there is no incremental row iterator over a
  file handle. CSV files in the hundreds-of-MB range want a
  streaming API this library does not provide.

## Quick start

```capa
import capa_csv.model
import capa_csv.parse
import capa_csv.header

fun main(stdio: Stdio, fs: Fs)
    let text = match fs.read("data.csv")
        Ok(s)  -> s
        Err(e) -> return stdio.eprintln("read failed: ${e}")

    // Raw parse: a Table is a list of Rows, each a list of fields.
    match parse(text)
        Err(e) -> stdio.eprintln(error_message(e))
        Ok(table) ->
            for row in table.rows
                stdio.println("${row.fields.length()} field(s)")

    // Header view: address fields by column name.
    match parse_headed(text)
        Err(e) -> stdio.eprintln(error_message(e))
        Ok(view) ->
            match view.get(0, "amount")
                Ok(v)  -> stdio.println("row 0 amount: ${v}")
                Err(e) -> stdio.eprintln(error_message(e))
```

The full runnable example is [`example.capa`](./example.capa);
it parses [`data/employees.csv`](./data/employees.csv) (which
exercises quoted fields, a doubled-quote escape, an embedded
comma, and an empty field), prints a summary, walks the rows by
header name, and shows the typed error for a missing column.

```bash
capa --run example.capa
capa --wasm --run example.capa   # byte-identical output
```

## Install via capa.toml

```toml
[dependencies.capa_csv]
git = "https://github.com/nelsonduarte/capa_csv"
tag = "v0.1.1"
verify_key = "6C1D222D491FB88031E041A536CFB426101AA24B"
```

`capa install` runs `git verify-tag` against your GPG keyring;
import the publisher's key first (see [`SECURITY.md`](SECURITY.md)
for the fingerprint provenance and `gpg --import` instructions).

## API surface

### Model (from `capa_csv.model`)

```capa
pub type Dialect { delimiter: String }
pub fun comma() -> Dialect                     // ","
pub fun tab() -> Dialect                       // "\t"
pub fun with_delimiter(delim: String) -> Dialect

pub type Row   { fields: List<String> }
pub type Table { rows: List<Row> }

pub type CsvError =
    UnclosedQuote(Int)    // record number of an unterminated quoted field
    UnexpectedChar(Int)   // text after a closing quote, e.g. "ab"c
    BadDialect(String)    // delimiter not a single non-quote character
    NoSuchColumn(String)  // header view: unknown column name
    NoSuchRow(Int)        // header view: no data record at that index
    ShortRow(String)      // header view: row exists but too short for that column

pub fun error_message(e: CsvError) -> String
```

### Parser (from `capa_csv.parse`)

```capa
pub fun parse(text: String) -> Result<Table, CsvError>
pub fun parse_with(text: String, dialect: Dialect) -> Result<Table, CsvError>
```

### Header view (from `capa_csv.header`)

```capa
pub type HeaderTable { columns: List<String>, rows: List<Row> }

pub fun parse_headed(text: String) -> Result<HeaderTable, CsvError>
pub fun parse_headed_with(text: String, dialect: Dialect) -> Result<HeaderTable, CsvError>
pub fun from_table(table: Table) -> HeaderTable

impl HeaderTable
    pub fun length(self) -> Int                          // data rows, header excluded
    pub fun column_index(self, name: String) -> Option<Int>
    pub fun has_column(self, name: String) -> Bool
    pub fun get(self, row_index: Int, column: String) -> Result<String, CsvError>
```

### Writer (from `capa_csv.write`)

```capa
pub fun write(table: Table) -> String
pub fun write_with(table: Table, dialect: Dialect) -> String
pub fun write_row(fields: List<String>) -> String
pub fun write_row_with(fields: List<String>, dialect: Dialect) -> String
```

## Why a writer is in v0.1

The writer earns its place by making the round trip
`parse -> write -> parse` a tested invariant: any `Table` the
parser produced serialises back to text that parses to the same
field values. That is the property a consumer relies on when it
reads a CSV, edits a cell, and writes it back. Quoting follows
RFC 4180 exactly: a field is quoted when it contains the
delimiter, a double quote, CR, or LF, and an interior double
quote is doubled. Records are LF-terminated (CRLF parses back
identically, so LF is the simpler stable choice).

## Parsing rules

RFC 4180, with two deliberate, documented decisions:

- **A wholly blank line between records is skipped** (it yields
  no `Row`). Python's `csv` module instead yields a zero-field
  row for a blank line; this library does not, because every
  real tabular consumer treats a blank line as nothing, and a
  zero-field row is a footgun a caller would have to filter out
  by hand. A blank field inside a record (`a,,c`) is of course
  preserved as the empty string; this rule is only about lines
  with no content at all.
- **Whitespace is significant.** Fields are not trimmed;
  `a, b` parses to `["a", " b"]`. Trim at the call site if you
  want it, so the parser never guesses.

Everything else is the standard grammar: a quoted field may
contain the delimiter, CR, and LF literally; `""` inside a
quoted field decodes to one `"`; a closing quote must be
followed by the delimiter, a record terminator, or end of
input (anything else is `UnexpectedChar`); an unterminated
quoted field is `UnclosedQuote`. Both LF and CRLF terminate a
record, and the final record needs no terminator. Field access
is Unicode-correct (codepoint indexing), identically on both
backends.

Malformed input never crashes and never panics: it returns a
typed `Err` whose record number points an operator at the bad
line.

## Tests

Four deterministic suites under [`tests/`](./tests/), run with
the real `capa test` subcommand and asserted with the
[`capa_test`](https://github.com/nelsonduarte/capa_test)
assertion library (this repository's `tests/` are the first
real dogfood of the `capa test` + dev-dependency flow on a fresh
library):

- `test_parse.capa` - the RFC 4180 core against the Python
  `csv` oracle: simple rows, quoted commas, escaped quotes,
  newline-in-field, CRLF vs LF, empty fields, the blank-line
  decision, trailing/leading delimiters, unicode, tab and
  semicolon dialects.
- `test_errors.capa` - every malformed shape returns the right
  typed `Err` (unclosed quote, text after quote, bad delimiter)
  with the record number carried through.
- `test_header.capa` - column access by name, `column_index`,
  `has_column`, and the `NoSuchColumn` / `ShortRow` errors.
- `test_write.capa` - quoting correctness and the
  `parse -> write -> parse` round trip.

```bash
capa test          # Python backend
capa test --both   # Python + Wasm, byte-identical stdout required
```

Current output of `capa test --both`:

```
capa test: 4 file(s) under .../capa_csv/tests [backend: python+wasm]
test_errors.capa ... ok
test_header.capa ... ok
test_parse.capa ... ok
test_write.capa ... ok
4 test(s): 4 passed, 0 failed
```

`capa_test` is declared under `[dev-dependencies]` with the same
git + tag + verify_key shape as any published dependency, pinned
to its `v0.1.0` tag and verified against the publisher key, so
`capa install` runs the full three-layer check (lockfile SHA +
GPG tag signature + SLSA L2 provenance) on it. Dev-dependencies
are resolved only when this repository is the install root, so a
consumer of `capa_csv` never fetches the test library.

## Audit claim

A parser is exactly the kind of dependency a supply-chain
attacker wants to own, so this one proves the empty claim about
itself. `capa --manifest` over every library module reports, for
every one of the 24 functions defined across the four modules
(`model` 4, `parse` 6, `header` 7, `write` 7):

```
declared_capabilities:                []
transitively_reachable_capabilities:  []
has_unsafe:                           false
```

0 functions with capabilities, 0 crossing `unsafe`, in every
module (`model`, `parse`, `header`, `write`). The only
capabilities anywhere in this repository are in the example and
are the example's own (`Fs` to read the fixture, `Stdio` to
print), and the example attenuates its `Fs` with
`restrict_to("data/")` before the first read. A program using
`capa_csv` declares only the authority its own code needs to
obtain the CSV text.

## Honest limitations (v0.1)

- **In-memory only.** The whole document is parsed into a
  `Table`; no streaming row iterator over a file handle.
- **One delimiter knob.** The quote character is fixed to `"`;
  comment lines and quoting modes are not modelled.
- **Strings in, Strings out.** No column type inference; a field
  is always a `String`.
- **No encoding detection.** Input is already-decoded text.
- **Blank lines are dropped.** A deliberate divergence from
  Python's `csv` (see Parsing rules).

## License

MIT. See [`LICENSE`](./LICENSE). Release tags are GPG-signed;
see [`SECURITY.md`](./SECURITY.md) for the fingerprint and
verification instructions.
