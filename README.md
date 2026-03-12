# Zig CBOR

A fast & flexible [CBOR (RFC 8949)](https://cbor.io/) encoding, decoding, and
pattern-matching library for Zig 0.15+.

---

## Features

- **Zero-allocation encoding** - write directly into a caller-supplied buffer or
  use the streaming writer API
- **Expressive pattern matching** - match CBOR values against rich patterns
  including type sentinels, wildcards, and nested structures
- **Typed extraction** - pull values out of a CBOR stream directly into Zig
  variables, structs, unions, enums, and optionals
- **JSON interop** - convert CBOR <-> JSON with both allocating and
  non-allocating variants
- **Custom encode/decode hooks** - any type can implement `cborEncode` /
  `cborExtract` to control its own serialisation
- **Full type coverage** - integers (all widths + signs), floats (f16/f32/f64),
  booleans, null, strings, byte strings, arrays, maps, tagged unions, structs,
  enums, optionals, slices, vectors, error sets

---

## Installation

Add to your `build.zig.zon`:

```zig
.dependencies = .{
    .cbor = .{
        .url = "git+https://github.com/neurocyte/cbor?ref=master#<commit>",
        .hash = "...",
    },
},
```

Then in `build.zig`:

```zig
const cbor = b.dependency("cbor", .{});
my_module.addImport("cbor", cbor.module("cbor"));
```

---

## Encoding

### `fmt` - encode to a stack buffer

The simplest API. Encodes any value into a caller-supplied buffer and returns
the written slice. Panics if the buffer is too small.

```zig
const cbor = @import("cbor");

var buf: [64]u8 = undefined;

// Tuple -> CBOR array
const msg = cbor.fmt(&buf, .{ "exit", "normal" });

// Struct -> CBOR map
const point = cbor.fmt(&buf, .{ .x = 10, .y = 20 });

// Nested
const nested = cbor.fmt(&buf, .{ "pos", .{ .x = 1, .y = 2 }, "score", 42 });
```

Use `fmtBuf` when you need to handle overflow gracefully:

```zig
const msg = try cbor.fmtBuf(&buf, .{ "event", "click", "button", 3 });
// returns error.NoSpaceLeft if buf is too small
```

### Streaming writer

Build CBOR incrementally when the structure isn't known at compile time:

```zig
var buf: [256]u8 = undefined;
var writer: std.Io.Writer = .fixed(&buf);

try cbor.writeArrayHeader(&writer, 3);
try cbor.writeValue(&writer, "hello");
try cbor.writeValue(&writer, @as(i64, 42));
try cbor.writeValue(&writer, true);

const result = writer.buffered(); // ["hello", 42, true]
```

Maps work the same way - write key/value pairs after `writeMapHeader`:

```zig
try cbor.writeMapHeader(&writer, 2);
try cbor.writeValue(&writer, "name");
try cbor.writeValue(&writer, "Alice");
try cbor.writeValue(&writer, "age");
try cbor.writeValue(&writer, @as(i64, 30));
```

---

## Pattern matching

`match` tests whether a CBOR buffer conforms to a pattern. Patterns can be
exact values, type sentinels, wildcards, or nested structures.

### Type sentinels

```zig
const cbor = @import("cbor");

// Test the type of a value without caring about its content
try cbor.match(buf, cbor.string);  // any string
try cbor.match(buf, cbor.number);  // any integer
try cbor.match(buf, cbor.array);   // any array
try cbor.match(buf, cbor.map);     // any map
try cbor.match(buf, cbor.boolean); // any boolean
try cbor.match(buf, cbor.any);     // anything at all
```

### Exact values and mixed patterns

```zig
var buf: [64]u8 = undefined;
const msg = cbor.fmt(&buf, .{ "click", 3, true });

// Match exact values
_ = try cbor.match(msg, .{ "click", 3, true });   // true
_ = try cbor.match(msg, .{ "click", 4, true });   // false

// Mix exact values with type sentinels
_ = try cbor.match(msg, .{ cbor.string, cbor.number, cbor.any }); // true
_ = try cbor.match(msg, .{ "click", cbor.number, cbor.any });     // true
```

### `more` - match a prefix

`more` at the end of a pattern matches any remaining elements:

```zig
const msg = cbor.fmt(&buf, .{ "event", "click", "x", 10, "y", 20 });

_ = try cbor.match(msg, .{ "event", "click", cbor.more }); // true - ignores trailing fields
```

### Nested patterns

```zig
const msg = cbor.fmt(&buf, .{ "move", .{ 10, 20 } });

_ = try cbor.match(msg, .{ "move", .{ cbor.number, cbor.number } }); // true
_ = try cbor.match(msg, .{ "move", .{ 10, 20 } });                   // true
```

---

## Extraction

Extract values from a CBOR stream directly into Zig variables using
`extract` inside a pattern.

### Basic extraction

```zig
var buf: [64]u8 = undefined;
const msg = cbor.fmt(&buf, .{ "resize", 800, 600 });

var width: i64 = undefined;
var height: i64 = undefined;
if (try cbor.match(msg, .{ "resize", cbor.extract(&width), cbor.extract(&height) })) {
    // width == 800, height == 600
}
```

### Extracting strings

```zig
var name: []const u8 = undefined;
_ = try cbor.match(msg, .{ cbor.extract(&name), cbor.any });
// name is a slice into the original CBOR buffer - zero copy
```

### Extracting structs, unions, and enums

`extract` handles arbitrary Zig types automatically:

```zig
const Point = struct { x: f32, y: f32 };
var pos: Point = undefined;
_ = try cbor.match(msg, .{ "pos", cbor.extract(&pos) });
// pos.x and pos.y are populated from the CBOR map {"x":...,"y":...}

const Color = enum { red, green, blue };
var color: Color = undefined;
_ = try cbor.match(msg, cbor.extract(&color));

const Shape = union(enum) { circle: f32, rect: struct { w: f32, h: f32 } };
var shape: Shape = undefined;
_ = try cbor.match(msg, cbor.extract(&shape));
```

### `extractAlloc` - for heap-allocated types

Use `extractAlloc` when the extracted value contains slices or other types
that require allocation (nested unions with slice payloads, `[]T` fields, etc.):

```zig
var arena = std.heap.ArenaAllocator.init(allocator);
defer arena.deinit();

var tags: []const []const u8 = undefined;
_ = try cbor.match(msg, .{ "tags", cbor.extractAlloc(&tags, arena.allocator()) });
```

### `extract_cbor` - extract a raw sub-value

Capture a nested CBOR value as a raw byte slice for deferred decoding:

```zig
var payload: []const u8 = undefined;
if (try cbor.match(msg, .{ "type", cbor.extract_cbor(&payload) })) {
    // payload contains the raw CBOR bytes of the second element
    _ = try cbor.match(payload, ...);
}
```

---

## JSON interop

### CBOR -> JSON

```zig
var json_buf: [256]u8 = undefined;
const cbor_data = cbor.fmt(&buf, .{ .name = "Alice", .age = @as(i64, 30) });

const json = try cbor.toJson(cbor_data, &json_buf);
// json == ~{"name":"Alice","age":30}~

// Pretty-printed
const pretty = try cbor.toJsonPretty(cbor_data, &json_buf);

// Allocating variants
const json_owned = try cbor.toJsonAlloc(allocator, cbor_data);
defer allocator.free(json_owned);
```

### JSON -> CBOR

```zig
var cbor_buf: [256]u8 = undefined;
const json =
    \\{"items":[1,2,3],"meta":{"count":3}}
;

const cbor_data = try cbor.fromJson(json, &cbor_buf);

// Allocating variant
const cbor_owned = try cbor.fromJsonAlloc(allocator, json);
defer allocator.free(cbor_owned);
```

---

## Custom encode/decode

Implement `cborEncode` and/or `cborExtract` on any type to control its
serialisation:

```zig
const Timestamp = struct {
    seconds: i64,

    pub fn cborEncode(self: Timestamp, writer: *std.Io.Writer) !void {
        // encode as a single integer
        try cbor.writeValue(writer, self.seconds);
    }

    pub fn cborExtract(self: *Timestamp, iter: *[]const u8) cbor.Error!bool {
        return cbor.matchInt(i64, iter, &self.seconds);
    }
};
```

---

## API reference

| Function                         | Description                                |
| -------------------------------- | ------------------------------------------ |
| `fmt(buf, value)`                | Encode to buffer, panic on overflow        |
| `fmtBuf(buf, value)`             | Encode to buffer, return error on overflow |
| `writeValue(writer, value)`      | Stream-encode a single value               |
| `writeArrayHeader(writer, n)`    | Write an array header for `n` elements     |
| `writeMapHeader(writer, n)`      | Write a map header for `n` key-value pairs |
| `match(buf, pattern)`            | Test a CBOR buffer against a pattern       |
| `matchValue(iter, pattern)`      | Match and advance an iterator              |
| `extract(ptr)`                   | Extractor for use inside match patterns    |
| `extractAlloc(ptr, allocator)`   | Allocating extractor for heap types        |
| `extract_cbor(ptr)`              | Capture raw CBOR bytes of a value          |
| `toJson(cbor, buf)`              | Convert CBOR to JSON (non-allocating)      |
| `toJsonAlloc(allocator, cbor)`   | Convert CBOR to JSON (allocating)          |
| `toJsonPretty(cbor, buf)`        | Convert CBOR to indented JSON              |
| `fromJson(json, buf)`            | Convert JSON to CBOR (non-allocating)      |
| `fromJsonAlloc(allocator, json)` | Convert JSON to CBOR (allocating)          |
| `decodeType(iter)`               | Decode the type header of the next value   |
| `skipValue(iter)`                | Advance iterator past the next value       |
| `isNull(buf)`                    | Test whether a buffer contains a CBOR null |
