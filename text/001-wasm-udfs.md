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

Seafowl already has the ability to introduce
[custom UDFs](https://seafowl.io/docs/guides/custom-udf-wasm). These functions
are capable of receiving a tuple of primitive values and returning a primitive.
The currently available WASM primitives are integer and floating point numbers:
`i32`, `i64`, `f32` and `f64`.

While the opportunity to extend Seafowl is exciting, the current implementation
falls short in several important ways:

1. Seafowl's [data types](https://seafowl.io/docs/reference/types) are are much
   richer than what can be trivially represented with scalar numeric primitives
   (eg: `TEXT` and `DATE`). Most data types cannot currently be used as UDF
   arguments or return values.
1. Currently unsupported, future user-defined aggregate functions may receive an
   array of tuples, which are also not easily represented with the numeric types
   currently available (even if the tuples themselves contain only numbers).
1. Aso currently unsupported user-defined table functions (the opposite of
   aggregate functions) may return a list of tuples, which is also not possible
   with the current limitations.
1. Not a priority on the short term, but scalar functions with variable number
   of arguments, eg: `CONCAT('prefix', table1.col1, 'suffix')` and those which
   accept multiple types, eg: `MAX(0.5, 1)` need more flexibility than what can
   be currently described in the `CREATE FUNCTION` expression.

# Terminology

- UDF - A _User Defined Function_ which can be used in `SELECT` statements just
  like built-in functions.

- WASI - WebAssembly System Interface - a standard set of functions which
  essentially act as a `libc` for `wasm` programs.

# Calling conventions

A calling convention refers to the rules of invoking a function. In this case,
it describes the following:

1. _The semantics of function arguments and return values_. In order to receive
   or return complex data types such as strings or arrays of tuples, the WASM
   function must use its linear memory to communicate with the host process.
   There are lots well-known calling conventions, for example, writing the
   arguments to a segment of allocated WASM memory and passing the starting
   pointer and length in bytes of the segment to the function as arguments.
   Unfortunately, only a single value may be returned by functions, so this
   method only works for passing input.
1. _Data serialization format_. Complex data types such as `DATE` or `TEXT`
   values, tuples or sets must be serialized into a bytestream before they can
   be written to the WASM function's linear memory. `JSON`, `MessagePack`,
   `Protocol Buffers`, etc. are all potential solutions.
1. _Error messages and status codes_. In addition to function results, a UDF may
   need to communicate warnings or error messages in case of failure.

Ideally the calling convention for UDFs should be:

- _efficient_, requiring no more work copying or (de)serializing objects in
  memory than necessary.
- _simple to implement_, since UDFs may be written in a number of languages
  which compile to WASM and each will need its own implementation of the calling
  convention logic.
- _flexible_ scalar UDFs' argument and return value data types are not known in
  advance when Seafowl is compiled, the calling convention shouldn't care about
  the specific types involved.

## Serialization format

`JSON` is so commonly used as a serialization format, that it's become the
default, especially on the web. The problem with JSON in this case is that it
has a single numeric type: `f64`. Seafowl's set of supported numeric types is
much richer, including `SMALLINT` (aka `i16`), `CHAR`, `VARCHAR`, `TEXT`,
`DECIMAL`, `BOOLEAN` and `TIMESTAMP`. Representing integers with floats may lead
to unexpected rounding / representational issues. For this reason projects like
[`node-pg`](https://node-postgres.com/) default to encoding 64-bit integers as
strings. That said,
[this proof-of-concept](https://github.com/splitgraph/rust-wasm-udf-demo/tree/main/experiments/wasi-streams-json-2)
demonstrates that JSON can be used as a serialization format if representational
issues are addressed.

Serialization formats with an explicit schema such as
[Google's Protocol Buffers](https://developers.google.com/protocol-buffers)
often prioritize development workflows which use the schema for code generation.
In this case, the UDFs' signature is not known at the time Seafowl is compiled,
so we need a solution which does not require compiling the schema.

[MessagePack](https://msgpack.org/index.html) is an efficient binary format
similar to JSON with support for many programming languages. MessagePack has
first class support for the numeric types offered by Seafowl as well as strings,
maps and arrays. Any tuple which can be currently `SELECT`-ed in Seafowl has a
trivial mapping to a MessagePack-encoded object. WASM MessagePack
implementations include:

- [msgpack-rust](https://github.com/3Hren/msgpack-rust) for Rust.
- [MPack](https://github.com/dsyer/mpack-wasm) for C and C++.
- [AssemblyScript](https://github.com/wapc/as-msgpack) support by the
  [waPC project](https://wapc.io/).
- [TinyGo](https://github.com/wapc/tinygo-msgpack) support also by waPC.

## WASI as a standard for invocation

While it's certainly
[possible](https://github.com/splitgraph/rust-wasm-udf-demo/blob/b6f5ea574e0221af0f34d0f495bcaf052e99c591/experiments/raw-string-passing/src/main.rs#L90-L123)
to define our own calling convention, it requires an implementation for any
language a UDF may be written in. Using an existing solution which already
supports a variety of languages gives Seafowl users options and saves us from
reimplementing the wheel. If during development UDF authors need a way to
execute their functions outside of Seafowl, we need to create this harness as
well.

The proclaimed "ultimate solution" for WASM function invocation is WebAssembly
Interface Types (WIT), which includes its own IDL to describe function
signatures. Unfortunately, WIT is still half-baked, using it in production is
currently not recommended. Another drawback of WIT is that we can't use the WIT
code generation tooling as intended, since Seafowl doesn't know the UDFs'
signatures at compile time. If UDF input and output is MessagePack-encoded, all
that needs to be passed is a stream of bytes. For that something like WIT seems
like overkill: we just need to read and write bytestreams, the language-specific
MessagePack library takes care of the rest.

While Webassembly as an instruction set is a stable standard, supported by a
number of runtimes both in browsers and on servers, it doesn't offer any builtin
stream IO functions. To write to stdout, interact with the filesystem, read
environment variables, etc, a host function exposing these capabilities must be
registered with the WASM runtime which can be called by WASM functions.

The need for such functions led to the creation of WASI, which provides a
standard implementation of these functions often mimicking the POSIX C standard
library's function names. Unix concepts such as `stdin`, `stdout` and `stderr`,
exit codes and file handles may be used in programs which compile to WASM as
long as the runtime provides a WASI implementation. WASI's goal is to make WASM
a viable platform for server-side applications.
[wasmer-js](https://github.com/wasmerio/wasmer-js) provides a solution for
running WASI-consuming WASM programs in the browser, making a WASI a promising
"run anywhere" standard.

A very convient way to pass the streams to / from our WASM UDF is to read from
`stdin` and write to `stdout`. This too is is implemented by storing the
bytestream in WASM linear memory, but the interface hides these details behind
the standard posix APIs.

Unfortunately, unlinke WASM, WASI is still in it's infancy. Currently, there are
two versions, `wasi_unstable` and `wasi_snapshot_preview1`. The version names
are the name of modules housing WASI functions. Due to this namespacing, both
WASI version modules may be present in the runtime simultaneously.

## WASI module types

First and foremost, WASI is a WASM module containing a set of
[standard functions](https://github.com/WebAssembly/WASI/blob/main/phases/snapshot/witx/wasi_snapshot_preview1.witx)
providing a POSIX-like API for server-side WASM applications, but a basic notion
of application lifecycle is necessary in many situations.

There are two major types of WASI programs:

- `Commands` Programs like commandline apps which are invoked, do some
  processing, and release all resources when the process terminates. When
  running commands, the runtime tries to execute the `_start` function (if no
  other export is implicitly specified).
- `Reactors` Programs like FaaS endpoints on Lambda / GCP Cloud Functions, etc,
  where a function is initialized and then is invoked over and over. The
  `_initialize` export is invoked when the module is loaded, after which any
  export may be invoked any number of times. AFAIK, there is no standard
  'cleanup' or destructor function, although Seafowl can certainly designate an
  optional cleanup export name for UDFs.

For more information on `Commands` vs `Reactors`, see:

- https://github.com/WebAssembly/WASI/issues/13
- https://github.com/bytecodealliance/wasmtime/issues/3474#issuecomment-951734116

UDFs definitely seem like they should be `Reactors`, although the semantics of
the `_initialize` function are not trivial. Once option is to ignore
`_initialize` / require it to be a `noop`. If UDFs need such an initializer, it
would be logical to run `_initialize` once for each transaction (not sure how
hard it is to implement in DataFusion) or each `SELECT` statement (this is more
in line with the current implementation).

## UDF function definition

DataFusion includes a `CREATE FUNCTION` statement which accepts
[arbitary data](https://github.com/splitgraph/seafowl/blob/27f105eda4a3076d95334a754c571c7fd147d4a7/src/context.rs#L1142)
which Seafowl uses to encode the UDF's properties as JSON, eg:

```sql
CREATE FUNCTION my_udf AS '
{
  "entrypoint": "wasm_function_name",
  "language": "wasiMessagePack",
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

Proof-of-concept WASM+WASI+MessagePack implementations of a
`concat(string, string) -> string` function exist for several languages, with
module sizes differing by orders of magnitude:

- Rust: currently 2Mb, but
  [this may be improved](https://rustwasm.github.io/docs/book/reference/code-size.html).
- C: 200 Kb
- AssemblyScript: 12Kb

Fortunately, Seafowl has no problem registering even the multi-Mb Rust module
UDF, no query size limit prevented this so far.

The interface between Seafowl and UDF is the programming language's MessagePack
implementation (plus a thin wrapper which reads `stdin` and writes `stdout`).

## Error messages, status codes

In addition to `stdout`, WASI also offers `stderr`, which can be used by the UDF
to pass error messages back to Seafowl. Currently this is unimplemented, but
returning an integer from the UDF could indicate successful execution of the
function (`0` for success, otherwise an error code). Unstructured UTF-8 error
messages should be permitted via `stderr`, but structured error data (like the
JS `Error` object consisting of `name`, `message`, `stack`) could also be
returned via `stderr` (or `stdout` with certain error codes).

## Testing UDFs outside of Seafowl

A big advantage of using WASI is that it's easy to invoke UDFs from the command
line. The Messagepack-encoded input can be written to a file, for example with
nodejs:

```javascript
// usage: node encode_args.js > input
const msgpack = require("@msgpack/msgpack");
const args = ["foo", "bar"];
process.stdout.write(msgpack.encode(args));
```

The UDF may then be executed using `wasmtime` or `wasirun` with the arguments
passed in from `stdin` and return values received via `stdout`.

```bash
wasmtime --dir . build/release.wasm --invoke udf < input > out
```

Output will also be messagepack-encoded, to decode we can use:

```javascript
// usage: node decode_output.js output
const fs = require("fs");
const msgpack = require("@msgpack/msgpack");
console.log(msgpack.decode(fs.readFileSync(process.argv[2])));
```

## Feature wishlist

These changes are not strictly necessary for supporting complex datatypes in
UDFs, but would go a long way for the overall UDF experience.

- The ability to replace (`CREATE OR REPLACE FUNCTION ...`) and delete (`DROP FUNCTION ...`) UDFs. Not only is it wasteful to
  store old functions' WASM modules indefinitely, it also forces users to
  rewrite queries which depend on these functions when the function is modified,
  since the new version will also need to have a new name.
- The ability to load UDF WASM Modules from an HTTP URL instead of providing it
  b64-ed as part of the JSON UDF metadata.
- A first class `CREATE FUNCTION` statement syntax which doesn't pass
  arguments as JSON, eg:
   
    CREATE [OR REPLACE] FUNCTION concat2
    RETURNING TEXT
    [LANGUAGE wasiMessagePack]
    MODULE 'https://...'
    ENTRY 'concat2'`

  The currently required `input_type` field may be omitted if Seafowl no longer
  attempts to verify UDF arguments prior to invocation.

## Key open decisions

- Are we happy with the approach recommended in this RFC of replacing primitive
  on-stack arguments and return values with serialized input and output for
  UDFs? If so, are we happy with MessagePack as a serialization format?
- Who is responsible for verifying the UDF type signatures against the type of
  the selected tuple and expected response? With the bytestream-passing
  approach, there is no guarantee that the UDF will return what it promises to
  return in the `CREATE FUNCTION` statement.
- How many `language` types do we want to support simultaneously? Should we
  eventually deprecate `Wasm` if we introduce `wasiMessagePack`?
- Which WASM languages should receive first-class support? Rust is pretty
  obvious, but eg: C++ programmers may prefer a different MessagePack library
  than C devs.
- Performance considerations: How expensive is it to use WASI `stdin` and
  `stdout`? If we used raw memory or WASI env vars, would that be faster? What's
  the right performance vs simple dev experience tradeoff here?
- UDF lifecycle: should modules' `_initialize` function be called before the
  exported UDF function is invoked during `SELECT`s? If so, when? My take is
  that we shouldn't do this: UDFs should be considered stateless.
- Using MessagePack for encoding input and output opens the door to user-defined
  aggregate functions, table functions and scalar functions with variable
  argument count. Are these things features we want to leave open or even
  support in our calling convention or should be ignore these use cases entirely
  for now and introduce a new `language` when users need these things.
- How do Seafowl users receive warning messages emitted by UDFs if a result
  value can be computed? Some options are leaving such warning unsupported (any
  warning is an error), or displaying them only in the seafowl access log.
