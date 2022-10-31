- Feature Name: WASM UDFs
- Start Date: 2022-10-31
- RFC PR: [001](https://github.com/splitgraph/seafowl-rfcs/issues/001)
- ClickUp task: [CU-3jv2jw7](https://app.clickup.com/t/3jv2jw7)
- GH issue: [106](https://github.com/splitgraph/seafowl/issues/106)

# Summary

This RFC is the response to [#106](https://github.com/splitgraph/seafowl/issues/106). It describes a way to extend the existing support for WASM UDFs allowing for non-primitive data types such as strings in arguments and return values.

# Table of contents

[[_TOC_]]

# Motivation

Seafowl already has the ability to introduce [custom UDFs](https://seafowl.io/docs/guides/custom-udf-wasm). These functions are capable of receiving a tuple of primitive values and returning a primitive. The currently available WASM primitives are integer and floating point numbers: `i32`, `i64`, `f32` and `f64`.

While the opportunity to extend Seafowl is exciting, the current implementation falls short in several important ways:
1. Seafowl's [data types](https://seafowl.io/docs/reference/types) are are much richer than what can be trivially represented with the numeric primitives (eg: `TEXT` and `DATE`). Most data types cannot currently be used as UDF arguments or return values.
1. Aggregate functions may receive an array of tuples, which are also not easily represented with the numeric types currently available (even if the tuples themselves contain only numbers).
1. Table functions (the opposite of aggregate functions) may return a list of tuples, which is also not possible with the current limitations.
1. Not a priority on the short term, but variadic functions, eg `CONCAT('prefix', table1.col1, 'suffix')` need to be flexible enough to support input with different signatures (not currently possible).


# Terminology

- UDF - A _User Defined Function_ which can be used in `SELECT` statements just like built-in functions.

- WASI - WebAssembly System Interface - a standard set of functions which essentially act as a `libc` for `wasm` programs.

# Calling conventions

A calling convention refers to the rules of invoking a function. In this case, it describes the following:

1. _The semantics of function arguments and return values_. In order to receive or return complex data types such as strings or arrays of tuples, the WASM function must resort to using its linear memory to communicate with the host process. There are lots well-known calling conventions, for example, writing the arguments to a segment of WASM memory and passing the starting pointer and length in bytes of the segment to the function as arguments.
1. _Data serialization format_. Complex data types such as `DATE` or `TEXT` values, tuples or sets must be serialized into a bytestream before they can be written to the WASM function's linear memory. `JSON`, `MessagePack`, `Protocol Buffers`, etc. are all potential solutions.
1. _Error messages and status codes_. In addition to function results, a UDF may need to communicate warnings or error messages in case of failure.

Ideally the calling convention for UDFs should be:
- *efficient*, requiring no more work copying or (de)serializing objects in memory than necessary.
- *simple to implement*, since UDFs may be written in a number of languages which compile to WASM (and each will need it's own implementation of the calling convention logic).
- *flexible* scalar UDFs' argument and return value data types are not known in advance when Seafowl is compiled, the calling convention shouldn't care about the specific types involved.

## Serialization format

`JSON` is so commonly used as a serialization format, that it's become the default, especially on the web. The problem with JSON in this case is that it has a single numeric type: `f64`. Seafowl's set of supported numeric types is much richer. Representing integers with floats may lead to unexpected rounding / representational issues. For this reason projects like [`node-pg`](https://node-postgres.com/) default to encoding 64-bit integers as strings. That said, [this proof-of-concept](https://github.com/splitgraph/rust-wasm-udf-demo/tree/main/experiments/wasi-streams-json-2) demonstrates that JSON can be used.

Serialization formats with an explicit schema such as [Google's Protocol Buffers](https://developers.google.com/protocol-buffers) often prioritize development workflows which use the schema for code generation. In this case, the UDFs' signature is not known at the time Seafowl is compiled, so we need a solution which does not require compiling a schema.

[MessagePack](https://msgpack.org/index.html) is an efficient binary format similar to JSON with support for many programming languages. MessagePack has first class support for the numeric types offered by Seafowl as well as strings, maps and arrays. Any tuple which can be currently `SELECT`-ed in Seafowl has a trivial mapping to a MessagePack-encoded object.

## WASI as a standard for invocation

While it's certainly [possible](https://github.com/splitgraph/rust-wasm-udf-demo/blob/b6f5ea574e0221af0f34d0f495bcaf052e99c591/experiments/raw-string-passing/src/main.rs#L90-L123) to define our own calling convention, it requires an implementation for any language a UDF may be written in. Using an existing solution which already supports a variety of languages gives Seafowl users options and saves us from reimplementing the wheel. If during development UDF authors need a way to execute their functions outside of Seafowl, we need to create this harness as well.

The proclaimed "ultimate solution" for WASM function invocation is WebAssembly Interface Types (WIT), which includes its own IDL to describe function signatures. Unfortunately, WIT is still half-baked, using it in production is currently not recommended. We can't use the WIT code generation tooling as intended, since Seafowl doesn't know the UDFs' signatures at compile time. If UDF input and output is MessagePack-encoded, all that needs to be passed is a stream of bytes. For that something like WIT seems like overkill: we just need to read and write bytestreams.

While Webassembly as an instruction set is a stable standard, supported by a number of runtimes both in browsers and on servers, it doesn't offer any builtin stream IO functions. To write to stdout, interact with the filesystem, read environment variables, etc, a host function exposing these capabilities must be registered with the WASM runtime which can be called by WASM functions.

The need for such functions led to the creation of WASI, which provides a standard implementation of these functions often mimicking the POSIX C standard library's function names. Unix concepts such as `stdin`, `stdout` and `stderr`, exit codes and file handles may be used in programs which compile to WASM as long as the runtime provides a WASI implementation.

A very convient way to pass the streams to / from our WASM UDF is to read from `stdin` and write to `stdout`. This too is is implemented by storing the bytestream in WASM linear memory, but the interface hides these details behind the standard posix APIs.

Unfortunately, unlinke WASM, WASI is still in it's infancy. Currently, there are two versions, `wasi_unstable` and `wasi_snapshot_preview1`. The version names are the name of modules housing WASI functions. Due to this namespacing, both WASI version modules may be present in the runtime simultaneously.

## WASI module types
First and foremost, WASI is a WASM module containing a set of [standard functions](https://github.com/WebAssembly/WASI/blob/main/phases/snapshot/witx/wasi_snapshot_preview1.witx) providing a POSIX-like API for server-side WASM applications, but a basic notion of application lifecycle is necessary in many situations.

There are two major types of WASI programs:
* `Commands` Programs like commandline apps which are invoked, do some processing, and release all resources when the process terminates. When running commands, the runtime tries to execute the `_start` function (if no other export is implicitly specified).
* `Reactors` Programs like FaaS endpoints on Lambda / GCP Cloud Functions, etc, where a function is initialized and then is invoked over and over. The `_initialize` export is invoked when the module is loaded, after which any export may be invoked any number of times. AFAIK, there is no standard 'cleanup' or destructor function, although Seafowl can certainly designate an optional cleanup export name for UDFs.

For more information on `Commands` vs `Reactors`, see:
* https://github.com/WebAssembly/WASI/issues/13
* https://github.com/bytecodealliance/wasmtime/issues/3474#issuecomment-951734116

UDFs definitely seem like they should be `Reactors`, although the semantics of the `_initialize` function are not trivial. It would be logical to run `_initialize` once for each transaction (not sure how hard it is to implement this in DataFusion) or each `SELECT` statement (this is more in line with the current implementation).

## UDF function definition

DataFusion includes a `CREATE FUNCTION` statement which accepts [arbitary data](https://github.com/splitgraph/seafowl/blob/27f105eda4a3076d95334a754c571c7fd147d4a7/src/context.rs#L1142) which Seafowl uses to encode the UDF's properties as JSON, eg:

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

* `entrypoint` - name of WASM module export to execute.
* `language` - A string determining the calling convention used by the WASM UDF. Currently supported values are `Wasm` and `WasiMessagePack`.
* `input_types` - An array of strings denoting the [Seafowl data types](https://seafowl.io/docs/reference/types) passed as arguments to the function. The WASM native types `i32`, `i64`, `f32`, `f64` may also be used).
* `return_type` - A single string determining return type.
* `data` - Base64 encoded bytecode for the UDF's WASM module.


## Error messages, status codes

In addition to `stdout`, WASI also offers `stderr`, which can be used by the UDF to pass error messages back to Seafowl.
At this moment this is unimplemented, but returning an integer from the UDF could indicate successful execution of the function (`0` for success, otherwise an error code).


