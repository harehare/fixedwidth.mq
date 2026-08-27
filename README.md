<h1 align="center">fixedwidth.mq</h1>

A fixed-width (fixed-length) text parser implemented as an [mq](https://github.com/harehare/mq) module.

Fixed-width text has no delimiter; each field occupies a fixed number of
characters in column order. It's still common in legacy COBOL/mainframe
exports (the same format csvq calls its "Fixed-Length Format").

## Features

- Parses fixed-width records (one per line) given an array of column widths
- Trims surrounding whitespace from every field, handling both left-padded
  (numeric) and right-padded (text) columns
- Optional header row, producing dicts keyed by column name
- Skips blank lines (empty or whitespace-only)

## Installation

Copy `fixedwidth.mq` to your mq module directory, or place it anywhere and reference it with `-L`.

```sh
cp fixedwidth.mq ~/.local/mq/config/
```

Alternatively, skip the local copy entirely and import it straight from GitHub over HTTP:

```sh
mq --allow-http-import -I raw \
  'import "github.com/harehare/fixedwidth.mq" | fixedwidth::fixedwidth_parse(., [10, 3], true)' input.txt
```

See [HTTP Import](#http-import) below for details.

## Usage

```sh
mq -L /path/to/modules -I raw \
  'import "fixedwidth" | fixedwidth::fixedwidth_parse(., [10, 3], true)' input.txt
```

If you copied it to the mq built-in module directory:

```sh
mq -I raw 'import "fixedwidth" | fixedwidth::fixedwidth_parse(., [10, 3], true)' input.txt
```

## API

### `fixedwidth_parse(input, widths, has_header)`

Parses fixed-width content and returns an array.

| Input type | `has_header` | Output |
|---|---|---|
| String | `false` | Array of arrays, one per line, all values are trimmed strings |
| String | `true` | Array of dicts, keyed by the first non-blank line's field values |

`widths` is an array of **character** counts (not bytes), one per column, in order. Blank lines
(empty or whitespace-only) are skipped. If a line is longer than the sum of `widths`, the
trailing characters are dropped; if it's shorter, the missing trailing fields are empty strings.

> **Note:** Widths count Unicode characters, not bytes. Byte-oriented specs (e.g. Shift-JIS,
> EBCDIC) count full-width/multi-byte characters as 2+ bytes, so a byte-based spec's widths can't
> be passed to `widths` directly. mq v0.8.4+ can measure a field's real byte length for you via
> its built-in [`encode`](https://github.com/harehare/mq)/`decode` functions, which take a
> [WHATWG encoding label](https://encoding.spec.whatwg.org/#names-and-labels):
>
> ```sh
> mq -I raw 'encode("氏名", "shift_jis") | len'
> # => 4  (2 characters, 2 bytes each)
> ```
>
> Use that to convert a byte-based column spec to character counts before calling
> `fixedwidth_parse`, or to re-derive byte offsets from the parsed character-based fields.

## Example

Given `users.txt`:

```
name      age
Alice     30
Bob       25
```

```sh
mq -L . -I raw 'import "fixedwidth" | fixedwidth::fixedwidth_parse(., [10, 3], true)' users.txt
# => [{"name": "Alice", "age": "30"}, {"name": "Bob", "age": "25"}]

mq -L . -I raw 'import "fixedwidth" | fixedwidth::fixedwidth_parse(., [10, 3], true)[0]["name"]' users.txt
# => "Alice"
```

## HTTP Import

`import`/`include` also accept `https://` URLs, or the shorter `github.com/{owner}/{repo}` form,
so you can pull the module straight from this repository without cloning it. HTTP imports are
disabled by default, so pass `--allow-http-import`:

```sh
mq --allow-http-import -I raw \
  'import "github.com/harehare/fixedwidth.mq" | fixedwidth::fixedwidth_parse(., [10, 3], true)' input.txt
```

Pin a released version instead of tracking the default branch with `@{tag}`:

```sh
mq --allow-http-import -I raw \
  'import "github.com/harehare/fixedwidth.mq@v1.0.0" | fixedwidth::fixedwidth_parse(., [10, 3], true)' input.txt
```

By default only `raw.githubusercontent.com/harehare` is reachable this way; pass `--allowed-domain`
to allow other hosts. The first fetch is cached and recorded in `mq.lock` (commit it alongside
scripts that rely on this import so re-runs use the exact content that was locked); pass
`--refresh-modules` to pick up new commits on an unpinned import, or `--frozen` in CI to fail
instead of writing a new `mq.lock` entry.

## Compatibility

Requires [mq](https://github.com/harehare/mq) v0.5 or later.

## License

MIT
