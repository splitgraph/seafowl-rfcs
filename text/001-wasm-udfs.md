- Feature Name: WASM UDFs
- Start Date: 2022-10-31
- RFC PR: [001](https://github.com/splitgraph/seafowl-rfcs/issues/001)
- ClickUp task: [CU-3jv2jw7](https://app.clickup.com/t/3jv2jw7)
- GH issue: [106](https://github.com/splitgraph/seafowl/issues/106)

# Summary

This RFC is the response to
[#106](https://github.com/splitgraph/seafowl/issues/106). It describes a way to
extend the existing support for WASM UDFs allowing for non-primitive data types
such as strings in arguments and return values.

# Motivation

One of PostgreSQL's most important advantages lies in the rich embedded language
support, allowing stored procedures to be authored in javascript, lua, python,
etc in addition to the natively support pl/pgsql.

Since Seafowl is meant to be run close to users (perhaps eventually as close as
within their browsers), any extension mechanism supported by Seafowl should be
portable across environments. With the
[widespread adoption](https://caniuse.com/wasm) of WebAssembly, it seems like a
[natural](https://sqlite.org/wasm/doc/tip/about.md)
[choice](https://duckdb.org/2021/10/29/duckdb-wasm.html) for
[user-defined functions](https://www.scylladb.com/2022/04/14/wasmtime/).

Seafowl already has the ability to introduce
[custom UDFs](https://seafowl.io/docs/guides/custom-udf-wasm) using WebAssembly.
These functions are capable of receiving a tuple of WebAssembly primitives and
returning a primitive. The currently available data types are thus currently
limited to integer and floating point numbers: `i32`, `i64`, `f32` and `f64`.

The WASM virtual machine is re-created for each invocation, isolating UDF
invocations between rows.

Assuming Seafowl's users will build upon the UDF interface, it will be difficult
to change in a backward-incompatible way without breaking users' applications.
As a result, any significant investment in the UDF logic should address
near-term future requirements in addition to currently experienced pain points,
specifically:

1. Seafowl's [data types](https://seafowl.io/docs/reference/types) are much
   richer than what can be trivially represented with numbers (consider `TEXT`
   and `DATE`). Most data types cannot currently be used as UDF arguments or
   return values.
1. In the long term, UDFs should be able to emit and receive Arrow data types
   like lists or
   [Structs](https://docs.rs/arrow/latest/arrow/datatypes/enum.DataType.html#variant.Struct).
1. Planned support for UDF window and aggregate functions require UDFs to be
   capable of retaining persistent state while processing the tuples of a single
   batch.
1. Planned support for user-defined aggregate functions (UDAFs) require support
   for emitting a scalar result from input consisting of multiple tuples (e.g.:
   a table).
1. Aggregate functions require Seafowl to implement
   [an interface](https://docs.rs/datafusion/latest/datafusion/physical_plan/udaf/struct.AggregateUDF.html)
   with several methods (currently registering a UDF allows only a single
   entrypoint into the WASM module).
1. Planned support for UDF table functions require support for returning one or
   more tuples.
1. Planned support for UDFs with variable number of arguments, e.g.:
   `CONCAT('prefix', table1.col1, 'suffix', ...)`.
1. Planned support for UDFs with multiple support argument types, e.g.:
   `MAX(2, 1)`, `MAX(0.5, 0.7)` and `MAX(0.2, 8)`.

# Terminology

- UDF: A _User Defined Function_ which can be used in SQL queries just like
  built-in functions.

- Aggregate function: a function which returns a scalar value based on a set of
  tuples, e.g.:
  [`avg()`](https://github.com/apache/arrow-datafusion/blob/8dcef91806443f9b9b512bf6d819dc20961b29c8/datafusion/physical-expr/src/aggregate/average.rs).

- [Window function](https://www.postgresql.org/docs/current/functions-window.html):
  a function with some persistent state which is shared between invocations. An
  example is `lag()`.

- [Table function](https://docs.snowflake.com/en/sql-reference/functions-table.html#what-are-table-functions):
  A function returning potentially multiple tuples (a table) for each
  invocation.

- WASI - WebAssembly System Interface - a standard set of functions which
  essentially act as a `libc` for `wasm` programs.

# Calling conventions

A calling convention refers to the rules of invoking a function. In this case,
it describes the following:

1. A method of exchanging arguments and return values between Seafowl and the
   UDF.
1. A data serialization format.
1. A mechanism for signaling errors.

Ideally the calling convention for UDFs should be:

- _efficient_, requiring no more work copying or (de)serializing objects in
  memory than necessary.
- _simple to implement_, since UDFs may be written in a number of languages
  which compile to WASM and each will need its own implementation of the calling
  convention logic.
- _flexible_ enough to allow addressing the issues listed in the "Motivation"
  section.

## Serialization format

**tl;dr** I recommend MessagePack.

`JSON` is so commonly used as a serialization format, that it's become the
default, especially on the web. The problem with JSON in this case is that it
has a single numeric type: `f64`. Seafowl's set of supported numeric types is
much richer, including `SMALLINT` (aka `i16`), `CHAR`, `VARCHAR`, `TEXT`,
`DECIMAL`, `BOOLEAN` and `TIMESTAMP`. Representing integers with floats may lead
to unexpected rounding / representational issues. For this reason projects like
[`node-pg`](https://node-postgres.com/) default to encoding 64-bit integers as
strings.

Serialization formats with an explicit schema such as
[Google's Protocol Buffers](https://developers.google.com/protocol-buffers)
often prioritize development workflows which use the schema for code generation.
In this case, the UDFs' signature is not known at the time Seafowl is compiled,
so we need a solution which does not require compiling the schema.

Seafowl uses Apache Arrow, which could open the door to using
[Apache Arrow Flight](https://github.com/apache/arrow-rs/tree/master/arrow-flight),
a solution for minimizing data (de)serialization during processing with multiple
network nodes (see
[1](https://voltrondata.com/blog/apache-arrow-flight-primer/),
[2](https://arrow.apache.org/blog/2019/10/13/introducing-arrow-flight/)).
Theoretically, Seafowl and the WASM UDF
[could communicate over gRPC](https://www.verygoodsecurity.com/blog/posts/using-grpc-and-wasm-to-transform-data-inside-envoy-proxy)
as required by Flight. While tempting, there are some significant drawbacks to
this approach:

- Only languages for which an Apache Arrow Flight binding exists can be used to
  write UDFs.
- The WASM module must include Apache Arrow Flight with all its dependencies.
- The data Seafowl receives from DataFusion is columnar. For each column,
  Seafowl gets an array of column values, one for each row. UDFs expect row-like
  tuples. This transformation prevents us from using the original serialized
  Arrow data as originally received. While it could be possible to transpose the
  columnar input to row-tuples but keep the serialization of individual scalar
  values, this seems a lot more complex than what its worth.

[MessagePack](https://msgpack.org/index.html) is an efficient binary format
similar to JSON with support for many programming languages. It has first class
support for the numeric types offered by Seafowl as well as strings, maps and
arrays. Any tuple which can be currently `SELECT`-ed in Seafowl has a trivial
mapping to a MessagePack-encodable object. WASM MessagePack implementations
include:

- [msgpack-rust](https://github.com/3Hren/msgpack-rust) for Rust.
- [MPack](https://github.com/dsyer/mpack-wasm) for C and C++.
- [AssemblyScript](https://github.com/wapc/as-msgpack) support by the
  [waPC project](https://wapc.io/).
- [TinyGo](https://github.com/wapc/tinygo-msgpack) support also by waPC.

## Existing WASM function call solutions

### Shared linear memory

Conceptually, the simplest approach is to write UDF input arguments to a section
of WASM linear memory and pass a pointer to it as the UDF argument. In a similar
vein, the result can be written to linear memory, with a pointer to the result
buffer returned by the UDF. Spoiler: this is the recommended approach.
Advantages include low overhead, minimal dependencies and maximal flexibility.

Drawbacks:

- Requires reimplementing the lightweight host-guest protocol for reading
  intput, writing output and memory management in each supported language.
- No type safety (UDF must validate input, host must validate returned values).

### WIT

In the long run, WebAssembly Interface Types (`WIT`) promise to provide an
elegant solution to the problem of passing complex data between webassembly
functions written in various high-level languages and the host. WIT includes an
IDL, also called "wit" which can be used
[for code generation](https://bytecodealliance.github.io/wit-bindgen/).

There exists a very early pre-alpha
[WIT implementation for rust](https://github.com/bytecodealliance/wit-bindgen)
supporting both rust hosts and WASM guests. The developers urge everyone
interested in using this in production to hold their horses and look for other
alternatives while the WIT standard is finalized, I'd guess somewhere between
12 - 18 months from now.

WIT is designed to be used with code generators; Seafowl need to load UDFs
directly, so even when WIT is mature enough to be used, it might not be a good
fit for the same reason Protocol Buffers isn't a convenient serialization
format.

### WaPC

The [waPC](https://github.com/wapc) project attempts to simplify wasm host-guest
RPC. They provide a rust host and a number of supported guest languages. WaPC
has its own GraphQL-inspired IDL language (WIDL). Based on GitHub activity, it
seems to be an active project but lacks significant backing (written and used
mostly by 3 guys at a startup formerly called Vino). WaPC uses MessagePack to
serialize data by default.

### WASM-bindgen

[wasm-bindgen](https://crates.io/crates/wasm-bindgen) is an important project in
the Rust WASM community. Its a mature solution for WASM RPC, but unfortunately
limited to JavaScript host -> Rust WASM module guest calls. There
[was](https://github.com/bytecodealliance/wasmtime/issues/677) experimental
support for WIT, but its not longer supported. In a future where WIT support
returns, `wasm-bindgen` could be an ergonomic route to UDFs with complex inputs
/ outputs. Currently the
[guide on using it with rust hosts](https://github.com/bytecodealliance/wasmtime/blob/main/docs/wasm-rust.md)
does not work as advertised.

### WASI-based communication

The WebAssembly System Interface is an extension to WASM providing an interface
to module functions for interacting with the host filesystem, command line
arguments, environment variables, etc. Like most things WASM-related, WASI
itself is still in it's infancy and subject to change. Currently, there are two
versions, `wasi_unstable` and `wasi_snapshot_preview1`. These version names are
also the names of modules housing WASI functions. Thanks to this namespacing,
both WASI version modules may be present in the runtime simultaneously.

Using WASI's `stdin` stream to pass function arguments and `stdout` to receive
return values seemed like a promising solution. Unfortuantely, I didn't find a
convenient way to reuse these streams for multiple invocations, and recreating
the entire WASM instance resulted in too much overhead.

Being able to print to `stderr` was a huge help in debugging UDFs during
development. For this, I think we should continue to use WASI. Although
initially meant to be used on the backend,
[wasmer-js](https://github.com/wasmerio/wasmer-js) provides a solution for
running WASI-consuming WASM programs in the browser, making a WASI a promising
"run anywhere" standard.

## UDF function definition

DataFusion includes a `CREATE FUNCTION` statement which accepts
[arbitary data](https://github.com/splitgraph/seafowl/blob/27f105eda4a3076d95334a754c571c7fd147d4a7/src/context.rs#L1142)
which Seafowl uses to encode the UDF's properties as JSON, eg:

```sql
CREATE FUNCTION my_udf AS '
{
  "entrypoint": "wasm_function_name",
  "language": "wasmMessagePack",
  "input_types": ["TEXT", "TEXT"],
  "return_type": "TEXT",
  "data": "AGFzbQEAAAABDQJgAX0BfWADfX9/AX0DBQQAAAABBQQBAUREBxgDBnNpb..."
}';
```

Fields:

- `entrypoint` - name of WASM module export to execute.
- `language` - A string determining the calling convention used by the WASM UDF.
  Currently supported values are `Wasm` and `WasiMessagePack`.
- `input_types` - An array of strings denoting the
  [Seafowl data types](https://seafowl.io/docs/reference/types) passed as
  arguments to the function. The WASM native types `i32`, `i64`, `f32`, `f64`
  may also be used).
- `return_type` - A single string determining return type.
- `data` - Base64 encoded bytecode of the WASM module exporting the UDF.

## UDF languages

Proof-of-concept implementations of a `concat(string, string) -> string` UDF
using linear memory to share data between Seafowl and the WASM module exist for
several languages with significant differences in module size and performance.

| Language       | WASM module size                                                                              |
| -------------- | --------------------------------------------------------------------------------------------- |
| Rust           | 2Mb but [this may be improved](https://rustwasm.github.io/docs/book/reference/code-size.html) |
| C              | 200Kb                                                                                         |
| AssemblyScript | 11 Kb                                                                                         |

Fortunately, Seafowl has no problem registering even the multi-Mb Rust module
UDF, no query size limit prevented this so far.

Decoding input arguments and encoding results can be abstracted away from UDF
authors. The interface available to them is the MessagePack library of their
programming language.

The following timed queries use the `tripdata` table from the Seafowl demo,
which has > 2.4 million rows. I used a `release` build of seafowl and averaged
the processing time of 50 queries for each row in the table below.

| test                                                              | avg time |
| ----------------------------------------------------------------- | -------- |
| [35 sec demo](https://seafowl.io/docs/getting-started/quickstart) | 66 ms    |
| iterate through all rows with `SELECT 1`                          | 682 ms   |
| builtin `concat()`                                                | 1091 ms  |
| AssemblyScript WASM UDF                                           | 5985 ms  |
| C WASM UDF                                                        | 10241 ms |
| Rust WASM UDF                                                     | 13900 ms |

## Error messages, status codes

As mentined earlier, WASI's `stderr` is an enormous help in debugging UDFs.
Currently, there's no implemented way for a UDF to signal errors, although
writing no output when the return type of the UDF is not `NULL` should trigger
an error. Some possibilities:

- In addition to the MessagePack bytestream, UDFs should also return an integer
  exit code, similar to C programs.
- We could wrap the response in a MessagePack "envelope", eg:
  `{data?: any, error?: {message: string, code: i32, stack?: string}}`.
  Successfully computed results would be written to `data`, while erroronous
  execution would lead to the `error` field being present.

It would be useful to have standard error codes for common errors:

- Bad input: malformed input
  `expected an array of elements, found empty input, object or primitive` or
  `input could not be parsed as MessagePack`
- Bad input: wrong input arity, `expected N arguments, received X`
- Bad input: wrong input type, `expected type T for argument X, received U`
- Bad input: wrong input value, `DIV(): division by zero is forbidden`

These errors may also arise when Seafowl reads the UDFs result.

## Testing UDFs outside of Seafowl

It can be convenient to test UDFs outside of Seafowl during development. This
requires a harness host which instantiates and invokes the WASM function.

## Feature wishlist

These changes are not strictly necessary for supporting complex datatypes in
UDFs, but would go a long way for the overall UDF experience.

- The ability to replace (`CREATE OR REPLACE FUNCTION ...`) and delete
  (`DROP FUNCTION ...`) UDFs. Not only is it wasteful to store old functions'
  WASM modules indefinitely, it also forces users to rewrite queries which
  depend on these functions when the function is modified, since the new version
  will also need to have a new name.
- The ability to load UDF WASM Modules from an HTTP URL instead of providing it
  b64-ed as part of the JSON UDF metadata.
- A first class `CREATE FUNCTION` statement syntax which doesn't pass arguments
  as JSON, eg:

      CREATE [OR REPLACE] FUNCTION concat2
      RETURNING TEXT
      [LANGUAGE 'wasmMessagePack']
      MODULE 'https://...'
      ENTRY 'concat2';

  The currently required `input_type` field may be omitted since Seafowl cannot
  reliably verify UDF arguments prior to invocation (it's up to the UDF to
  verify input arguments).

- Planned support for UDAFs require multiple entrypoints to be specified in a
  single `CREATE` statement, so the entry's type may need to be optionally
  specified, e.g.:

      CREATE [OR REPLACE] AGGREGATE FUNCTION avg
      RETURNING BIGINT
      [LANGUAGE wasmMessagePack]
      MODULE 'https://...'
      ENTRY (
          ('row_accumulator', 'avg_rowacc'),
          ('merge_batch', 'avg_mergebatch'));

## Key open decisions

- How many `language` types do we want to support simultaneously? Should we
  eventually deprecate `Wasm` if we introduce `wasmMessagePack`?
- Which WASM languages should receive first-class support? Rust and C are pretty
  obvious, but eg: C++ programmers may prefer a different MessagePack library
  than C devs.
- How do Seafowl users receive warning messages emitted by UDFs if a result
  value can be computed? Some options are leaving such warning unsupported (any
  warning is an error), or displaying them only in the seafowl access log.

# References

- [Intro to WASM linear memory](https://radu-matei.com/blog/practical-guide-to-wasm-memory/) -
  well written, but includes some bugs in the code samples.
- [Rust UDF code sample](https://wasmedge.org/book/en/sdk/go/memory.html)
- [Blog post on using WASI to pass args to WASM functions](https://blog.scottlogic.com/2022/04/16/wasm-faas.html)
