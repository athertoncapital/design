# Binary Encoding

This document describes the [portable](Portability.md) binary encoding of the
[Abstract Syntax Tree](AstSemantics.md) nodes.

The binary encoding is designed to allow fast startup, which includes reducing
download size and allow for quick decoding. For more information, see the 
[rationale document](Rationale.md#why-a-binary-encoding)

Reducing download size, is achieved through three layers:

 * The **raw** binary encoding itself, natively decoded by the browser, and to
   be standardized in the [MVP](MVP.md).
 * **Specific** compression to the binary encoding, that is unreasonable to
   expect a generic compression algorithm like gzip to achieve.
   * This is not meant to be standardized, at least not initially, as it can be
     done with a downloaded decompressor that runs as web content on the client,
     and in particular can be implemented in a [polyfill](Polyfill.md).
 * **Generic** compression, such as gzip, already supported in browsers. Other
   compression algorithms being considered and which might be standardized
   include: LZMA, [LZHAM](https://github.com/richgel999/lzham_codec),
    [Brotli](https://datatracker.ietf.org/doc/draft-alakuijala-brotli/).

## Variable-length integers
 * [Polyfill prototype](https://github.com/WebAssembly/polyfill-prototype-1) shows significant size savings before (31%) and after (7%) compression.
 * [LEB128](https://en.wikipedia.org/wiki/LEB128) except limited to uint32_t payloads.

## Global structure

* A module contains (in this order):
  - A header, containing:
    + The [magic number](https://en.wikipedia.org/wiki/Magic_number_%28programming%29)
    + Other data TBD
  - A table (sorted by offset) containing, for each section:
    + A string literal section type name
    + 64-bit offset within the module
  - A sequence of sections
* A section contains:
  - A header followed by
  - The section contents (specific to the section type)
* A `definitions` section contains (in this order):
  - The generic section header
  - A table (sorted by offset) containing, for each type which has operators:
    + A standardized string literal [type name](AstSemantics.md#expression-types).
      The index of a type name in this table is referred to as a type ID
    + 64-bit offset of its operator table within the section
  - A sequence of operator tables
  - An operator table contains:
    + A sequence of standardized string literal [operator names](AstSemantics.md),
      where order determines operator index
* A `code` section contains (in this order):
  - The generic section header
  - A table (sorted by offset) containing, for each function:
     + Signature
     + Function attributes, valid attributes TBD (could include hot/cold, optimization level, noreturn, read/write/pure, ...)
     + 64-bit offset within the section
  - A sequence of functions
  - A function contains:
    + A table containing, for each type ID that has [locals](AstSemantics.md#local-variables):
      * Type ID
      * Count of locals
    + The serialized AST
* A `data` section contains (in this order):
   - The generic section header
   - A sequence of byte ranges within the binary and corresponding addresses in the linear memory


All strings are encoded as null-terminated UTF8. Data segments represent
initialized data that is loaded directly from the binary into the linear memory
when the program starts (see [modules](Modules.md#linear-memory-section)).

## Serialized AST

* Use a preorder encoding of the AST
  * Efficient single-pass validation+compilation and polyfill
* The data of a node (if there is any), is written immediately after the operator and before child nodes
  * The operator statically determines what follows, so no generic metadata is necessary.
* Examples
  * Given a simple AST node: `struct I32Add { AstNode *left, *right; }`
    * First write the operator of `I32Add` (1 byte)
    * Then recursively write the left and right nodes.
  * Given a call AST node: `struct Call { uint32_t callee; vector<AstNode*> args; }`
    * First write the operator of `Call` (1 byte)
    * Then write the (variable-length) integer `Call::callee` (1-5 bytes)
    * Then recursively write each arg node (arity is determined by looking up `callee` in table of signatures)

## Backwards Compatibility

As explained above, for size- and decode-efficiency, the binary format will serialize AST nodes,
their contents and children using dense integer indices and without any kind of embedded metadata 
or tagging. This raises the question of how to reconcile the efficient encoding with the 
backwards-compatibility goals.

Specifically, we'd like to avoid the situation where a future version of WebAssembly has features 
F1 and F2 and vendor V1 implements F1, assigning the next logical operator indices to F1's new
operators, and V2 implements F2, assigning the same next logical operator indices to F2's new operators 
and now a single binary has ambiguous semantics if it tries to use either F1 or F2. This type of 
non-linear feature addition is commonplace in JavaScript and Web APIs and is guarded against by 
having unique names for unique features (and associated [conventions](https://hsivonen.fi/vendor-prefixes/)).

The current proposal is to maintain both the efficiency of indices in the [serialized AST](BinaryEncoding.md#serialized-ast) and the established
conflict-avoidance practices surrounding string names:
  * The WebAssembly spec doesn't define any global index spaces
    * So, as a general rule, no magic numbers in the spec (other than the literal [magic number](https://en.wikipedia.org/wiki/Magic_number_%28programming%29)).
  * Instead, a module defines its *own* local index spaces of operators by providing tables *of names*. 
    * So what the spec *would* define is a set of names and their associated semantics.
  * To avoid (over time) large index-space declaration sections that are largely the same
    between modules, finalized versions of standards would define named baseline index spaces
    that modules could optionally use as a starting point to further refine.
    * For example, to use all of [the MVP](MVP.md) plus
      [SIMD](PostMVP.md#fixed-width-simd) the declaration could be "base"
      followed by the list of SIMD operators used.
    * This feature would also be most useful for people handwriting the [text format](TextFormat.md).
    * However, such a version declaration does not establish a global "version" for the module
      or affect anything outside of the initialization of the index spaces; decoders would
      remain versionless and simply add cases for new *names* (as with current JavaScript parsers).

## Proposals

The native prototype built for [V8](https://github.com/WebAssembly/v8-native-prototype)
implements a binary format that embodies most, but not all of the ideas in this document.
It is described in detail in a [public design doc](https://docs.google.com/a/google.com/document/d/1761v1AfhFM5kE8NArF_PyXcl-iVh0Dx3InOrmcyIoiI/pub) and a [copy of the original](https://docs.google.com/document/d/1-G11CnMA0My20KI9D7dBR6ZCPOBCRD0oCH6SHCPFGx0/edit?usp=sharing).
