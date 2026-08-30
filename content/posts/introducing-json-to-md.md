+++
title = "json-to-md: The Same Markdown From JSON, in TypeScript and Go"
date = 2026-08-30
description = "A converter that turns JSON into deterministic GitHub Flavored Markdown with byte-identical output from its TypeScript and Go implementations. Install steps, usage, and token and speed benchmarks measured on real documents."
[taxonomies]
tags = ["typescript", "go", "llm", "markdown", "open-source", "performance"]
[extra]
toc = true
+++

I built [json-to-md](https://github.com/rajnandan1/json-to-md) to turn a JSON document into Markdown that a person can read and a model can follow. It runs in Node, in the browser, in Go, and on the command line. The TypeScript and Go implementations produce byte-identical output, and a shared corpus plus a cross-implementation fuzz gate in CI keep them that way.

An order object going through it:

```json
{
  "order": {
    "id": "ord_1042",
    "status": "paid",
    "customer": { "name": "Asha", "email": "asha@example.com" },
    "items": [
      { "sku": "A1", "qty": 2, "price": 499.00 },
      { "sku": "B7", "qty": 1, "price": 1299.50 }
    ]
  }
}
```

```markdown
# Results

## order

### id

ord\_1042

### status

paid

### customer

#### name

Asha

#### email

asha@example.com

### items

| sku | qty | price |
| --- | --- | --- |
| A1 | 2 | 499.00 |
| B7 | 1 | 1299.50 |
```

Object keys become headings. Arrays of objects become tables. Strings come out escaped, so `ord_1042` renders as `ord\_1042`, and a value containing `<script>` or `**bold**` shows up as literal text. Data never injects formatting into the document. The prices kept their spelling too: `499.00` stayed `499.00` because the text entry point preserves each number's original lexeme instead of round-tripping it through a float.

## Why I needed this

JSON keeps landing in places where a person or a model reads it:

1. Prompts. Tool results and retrieved records go into the context window, and the model reads `{`, `"`, and `,` before it reads any data.
2. Agent logs and memory files. You will not open a 400-line escaped JSON blob to check what an agent did.
3. Docs and PR descriptions. Pasting an API response means someone reformats it by hand.

Markdown fixes the reading problem. Before I would trust a converter inside a pipeline, I wanted four guarantees from it, so I wrote them down first and built to them.

- **Deterministic.** Same input, same bytes. LF line endings, one blank line between blocks, no trailing spaces, one final newline.
- **One contract, two languages.** The TypeScript and Go code share a corpus of expected outputs, and CI fuzzes both implementations against each other on every change.
- **Safe by construction.** Every string is escaped. A value cannot open a heading, a link, or an HTML tag.
- **One-way.** The output is a projection for reading. When a downstream step must parse, validate, or store the data, keep the JSON.

## How values render

| Input | Rendering |
| --- | --- |
| Object keys | Headings H2 through H6, then nested lists once nesting passes H6 |
| Array of objects | One GFM table |
| Any other array | Unordered list, in source order |
| Non-empty object or array inside a table cell | Link to a detail section headed by the value's JSON Pointer |
| String | Escaped literal text |
| Whole-string absolute `http(s)` URL | Markdown link |
| `null`, `""`, `[]`, `{}` | Inline code, each kept distinct from a missing cell |

Tables are where the format pays for itself. Three rows of the classic `cars.json` dataset:

```markdown
# Cars

| Name | Miles\_per\_Gallon | Cylinders | Displacement | Horsepower | Weight\_in\_lbs | Acceleration | Year | Origin |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| chevrolet chevelle malibu | 18 | 8 | 307 | 130 | 3504 | 12 | 1970-01-01 | USA |
| buick skylark 320 | 15 | 8 | 350 | 165 | 3693 | 11.5 | 1970-01-01 | USA |
| plymouth satellite | 18 | 8 | 318 | 150 | 3436 | 11 | 1970-01-01 | USA |
```

Pretty-printed, those three records cost 265 tokens on the `o200k_base` tokenizer. Minified, 173. The table, 168. Each key name appears once in the header row instead of once per record, and the gap widens with every row you add.

There is no input-size, nesting-depth, or table-size limit in the converter. Parsing, validation, and rendering use explicit stacks instead of recursion, so a deeply nested document cannot overflow the call stack.

## Install

### Node

```sh
npm install @rajnandan1/json-to-md
```

Ships ESM and CommonJS builds with TypeScript declarations and no runtime dependencies. Node 18 or newer.

### Browser

The same package. Bundle it, or load the release build from a CDN:

```html
<script type="module">
    import { convertJsonText } from "https://cdn.jsdelivr.net/npm/@rajnandan1/json-to-md@3/dist/index.js";
    document.body.textContent = convertJsonText('{"hello":"world"}');
</script>
```

An IIFE build at `dist/index.global.js` exposes a `jsonToMd` global for classic script tags. Pin the version and add a Subresource Integrity hash if you go that route.

### Go

```sh
go get github.com/rajnandan1/json-to-md/go/v3
```

### CLI

```sh
brew tap rajnandan1/homebrew-rajnandan
brew trust rajnandan1/rajnandan     # once per machine
brew install json-to-md
```

Or through the Go toolchain:

```sh
go install github.com/rajnandan1/json-to-md/go/v3/cmd/json-to-md@latest
```

## Use

### TypeScript and JavaScript

```ts
import { convertJsonText, convertJsonValue } from "@rajnandan1/json-to-md";

convertJsonText('{ "hello": "world" }'); // untrusted serialized JSON text
convertJsonValue({ hello: "world" });    // already-parsed data you trust
// # Results
//
// ## hello
//
// world
```

Pick the entry point by where the data came from. `convertJsonText` takes serialized JSON, parses it without running caller code, rejects duplicate member names, and keeps each number's spelling. `convertJsonValue` takes a parsed value and validates it at runtime, rejecting cycles, sparse arrays, `undefined`, `BigInt`, `NaN`, functions, `Date`, `Map`, `Set`, and class instances.

The difference shows on large integers:

```ts
convertJsonText("9007199254740993"); // 9007199254740993  (preserved)
convertJsonValue(9007199254740993);  // 9007199254740992  (JS rounded it before you called)
```

Two options cover the output shape:

```ts
convertJsonValue(data, { heading: "Orders" }); // custom H1
convertJsonValue(data, { heading: null });     // no H1
convertJsonValue(data, { showTypes: true });   // annotate values with their JSON type
```

With `showTypes`, scalars get a trailing ` *(string)*`, ` *(integer)*`, ` *(number)*`, or ` *(boolean)*`, and a table column gets the annotation in its header when every cell shares one type. The order document from the top, with types on and the H1 off:

```markdown
## order

### id

ord\_1042 *(string)*

### status

paid *(string)*

### items

| sku *(string)* | qty *(integer)* | price |
| --- | --- | --- |
| A1 | 2 | 499.00 |
| B7 | 1 | 1299.50 |
```

`price` gets no header annotation because one cell is an integer and the other is not. A literal `*` in your data is always escaped, so the `*(…)*` token cannot be forged.

Conversion fails at the first error and never returns partial Markdown:

```ts
try {
    convertJsonText('{"name":"a","name":"b"}');
} catch (e) {
    if (e instanceof JsonToMarkdownError) {
        e.code;          // "DUPLICATE_MEMBER_NAME"
        e.location;      // { offset, line, column }
        e.firstLocation; // where the first "name" was
        e.pointer;       // JSON Pointer, when the bad value is locatable
    }
}
```

Codes: `INVALID_JSON_SYNTAX`, `DUPLICATE_MEMBER_NAME`, `INVALID_PARSED_VALUE`, `CYCLIC_REFERENCE`, `SPARSE_ARRAY`, `INVALID_OPTION`.

### Go

```go
import jsontomd "github.com/rajnandan1/json-to-md/go/v3"

md, err := jsontomd.ConvertText(src)                              // byte-identical to convertJsonText
md, err  = jsontomd.ConvertValue(v)                                // json.Marshal(v), then the same core
md, err  = jsontomd.ConvertText(src, jsontomd.WithHeading("Orders"))
md, err  = jsontomd.ConvertText(src, jsontomd.WithoutHeading())
md, err  = jsontomd.ConvertText(src, jsontomd.WithTypes())

var convErr *jsontomd.Error
if errors.As(err, &convErr) {
    // convErr.Code, .Pointer, .Location: same codes and UTF-16 locations as the TS errors
}
```

`ConvertValue` renders struct fields in declaration order and map keys sorted, because that is what `json.Marshal` does. The library package has no dependencies.

### CLI

```sh
json-to-md data.json > out.md                       # file in, Markdown out
curl -s https://api.example.com/items | json-to-md  # stdin
json-to-md --types data.json                        # showTypes
json-to-md --json broken.json 2> error.json         # structured error on stderr
```

Exit codes: `0` success, `1` conversion failed, `2` usage or I/O error. A failure prints one greppable line:

```
json-to-md: DUPLICATE_MEMBER_NAME at 1:13 (first at 1:2): duplicate object member name "name"
```

With `--json` you get the full error object instead:

```json
{"code":"DUPLICATE_MEMBER_NAME","location":{"offset":12,"line":1,"column":13},"firstLocation":{"offset":1,"line":1,"column":2},"message":"duplicate object member name \"name\""}
```

## Benchmarks

I ran everything below on 2026-08-30 on an Apple M3 Pro with 18 GB of RAM, Node 24.14.1, Go 1.23.1, and `@rajnandan1/json-to-md` 3.0.0 (Go module `go/v3.0.0`). Every input file was downloaded the same day. The numbers in the project README come from an earlier run on a smaller copy of the Stripe spec; the ones here are fresh.

### Byte-identical output

The input is Stripe's OpenAPI spec, [spec3.json](https://github.com/stripe/openapi/blob/master/openapi/spec3.json): 8,028,700 bytes of JSON. Both implementations produced 6,568,122 bytes of Markdown, and the SHA-256 matched:

```
dc322742c7f7c00de38da45e8f871fe60cdad67dc67fe8e5eb5d3478cecb7a89  go-out.md
dc322742c7f7c00de38da45e8f871fe60cdad67dc67fe8e5eb5d3478cecb7a89  ts-out.md
```

The corpus promise holds on an 8 MB real-world document as well as on the fixtures.

### Speed

Same file, same machine:

| Surface | Result |
| --- | --- |
| Go `ConvertText` | 125 ms per conversion, 64 MB/s (`go test -bench`, 10 iterations) |
| TypeScript `convertJsonText`, Node 24 | 207 ms median over 11 runs (min 185, max 240) |
| CLI, whole process, `spec3.json` | 0.13 s wall clock, consistent across 10 runs |
| CLI, whole process, a 6 KB GitHub API response | under 10 ms |

Go runs about 1.7x faster than Node on this document. Both are fast enough that conversion will never be the slow step in a pipeline that also makes a network call.

Reproduce with any large JSON file:

```sh
git clone https://github.com/rajnandan1/json-to-md && cd json-to-md
pnpm install && pnpm build && node scripts/bench-file.mjs path/to/big.json 11
cd go && BENCH_FILE=path/to/big.json go test -bench ConvertTextFile -benchtime 10x -run '^$'
```

### Tokens: where Markdown wins and where it loses

The pitch for Markdown in prompts is that models pay for JSON punctuation. I wanted a number from my own files instead of a quoted one, so I counted tokens with tiktoken's `o200k_base` encoding (the GPT-4o and GPT-5 tokenizer) for each document in three forms: pretty-printed JSON with two-space indentation, minified JSON, and json-to-md output. `cl100k_base` gave the same direction on every file, within a few points.

| Document | Shape | Pretty JSON | Minified JSON | Markdown | vs pretty | vs minified |
| --- | --- | --- | --- | --- | --- | --- |
| [cars.json](https://github.com/vega/vega-datasets/blob/main/data/cars.json) | 406 flat records, 9 columns | 36,106 | 23,575 | 15,414 | -57% | -35% |
| [movies.json](https://github.com/vega/vega-datasets/blob/main/data/movies.json) | 3,201 flat records, 16 columns | 500,615 | 343,404 | 217,508 | -57% | -37% |
| JSONPlaceholder `/comments` | 500 flat records | 45,260 | 35,761 | 32,499 | -28% | -9% |
| JSONPlaceholder `/posts` | 100 flat records | 7,690 | 6,091 | 5,659 | -26% | -7% |
| JSONPlaceholder `/users` | 10 records, two nested objects each | 1,829 | 1,223 | 1,503 | -18% | +23% |
| PyPI `requests` metadata | deep object, release history | 97,138 | 82,258 | 119,833 | +23% | +46% |
| Stripe `spec3.json` | deep object, 8 MB | 1,371,581 | 902,685 | 1,729,974 | +26% | +92% |
| npm registry doc for this package | deep object | 7,971 | 7,105 | 10,858 | +36% | +53% |
| GitHub API, 30 VS Code issues | nested records, URL heavy | 64,952 | 55,638 | 88,931 | +37% | +60% |
| GitHub API, 30 commits | nested records, URL heavy | 49,892 | 43,335 | 70,638 | +42% | +63% |
| GitHub API, one repo | one object, 49 URL fields | 1,848 | 1,556 | 2,846 | +54% | +83% |

Two shapes, two answers.

**Flat arrays of records win, and win big.** The two Vega datasets drop 57% against pretty JSON and 35% to 37% against minified. The header row spells each key once; JSON spells it once per record. If you are feeding query results, CSV-shaped exports, or log rows to a model, this is the case the tool was built for.

**Nested API responses lose.** Every GitHub, npm, PyPI, and Stripe document cost more as Markdown than as JSON, by 23% to 54% against pretty-printed and 46% to 92% against minified. The README cites a 16% saving on one document. On my files the answer depends on shape, and I would rather publish the spread than the best case.

I traced where the extra tokens go on the GitHub commit list, which is the worst realistic case:

| Step | Tokens |
| --- | --- |
| Minified JSON | 43,335 |
| Pretty JSON | 49,892 |
| Markdown as generated | 70,638 |
| Markdown with URL links collapsed to bare URLs | 52,814 |
| ...and with backslash escapes removed | 50,136 |

The URL rule is the biggest cost. Any string that is a whole `http(s)` URL renders as `[url](url)`, which spells the URL twice. That commit list has 770 URL-valued fields, and the doubling alone accounts for 17,824 tokens, a quarter of the output. Escaping is second: GitHub's keys are snake_case, and each `_` becomes `\_`, which splits a key like `node_id` into more tokens than the bare word. 1,369 escape sequences, 1,324 of them underscores, cost another 2,678 tokens. Take both away and the Markdown lands within 1% of pretty-printed JSON and 16% above minified; that remainder is the heading structure, where every leaf under a nested object gets a `###` line and two blank lines.

The Stripe spec has almost no URLs and still loses, because it is objects nested in objects nested in objects. Its Markdown carries 127,117 escape sequences, 80,733 of them `\_`, and even with all of them removed it sits at 1,540,277 tokens against 902,685 minified. Deep object trees are the shape this projection handles worst for token count, even though they are the shape it handles best for reading.

So my rule for prompts:

- Tabular data, log rows, anything a person will also read: convert it.
- Deeply nested, URL-dense API payloads that only the model will read: keep them as minified JSON.

An option to render URLs as bare text would recover most of the gap on API responses. It is not in 3.0.0.

## Try it

The [playground](https://json-to-md.rajnandan.com/demo/) loads the released library from the CDN. Paste JSON, switch between the rendered preview and the raw Markdown, and flip between the text and value entry points to watch numeric spelling survive or not.

Source, corpus, and CI are at [github.com/rajnandan1/json-to-md](https://github.com/rajnandan1/json-to-md). MIT licensed.
