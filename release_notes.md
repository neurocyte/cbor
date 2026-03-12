# Release Notes — v1.1.0
2026-03-12

## Features

- **Custom encoder hooks**: Types can implement `cborEncode(writer: *Io.Writer) !void`
  to control their own serialisation.
- **Custom extractor hooks**: Types can implement `cborExtract(iter: *[]const u8) !bool`
  to control their own deserialisation. Enum types are
  now also supported via `cborExtract`.
- **`extractAlloc`**: New allocating extractor for types containing slices
  or heap-allocated payloads. Use `extractAlloc(&dest, allocator)` inside a
  `match` pattern.
- **`extractAlloc` arrays**: `extractAlloc` now handles `[]T` slices,
  allocating and populating each element.
- **Struct and union extraction**: `extract` and `extractAlloc` now handle
  structs (decoded from CBOR maps) and tagged unions (decoded from
  two-element CBOR arrays of `[tag, payload]`). Enum extraction is also
  supported.
- **`fmtBuf`**: New `fmtBuf(buf, value) ![]const u8` — like `fmt` but
  returns `error.NoSpaceLeft` instead of panicking on overflow.
- **`fmt` explicit panic**: `fmt` now calls `@panic("cbor.fmt: buffer too small")`
  on overflow rather than `catch unreachable`.
- **Non-u8 slice matching**: `match` now supports patterns of the form
  `&[_]T{...}` for slices and arrays where the element type is not `u8`.
- **`writeJsonValue` arrays and objects**: `writeJsonValue` now handles
  `json.Value.array` and `json.Value.object`.
- **More explicit error handling**: Errors that were previously collapsed
  are now surfaced with distinct error values.

## Bug Fixes

- **`matchArray` non-match**: `matchArray` was incorrectly returning an
  error on a type mismatch instead of `false`.
- **`decodeMapHeader` error type**: Corrected the error return type to
  include all possible errors.
- **Missing `NotAnObject` error**: Added missing error variant to the error
  set.
- **`extract` optional null**: Extracting a null CBOR value into a
  non-allocating optional now correctly sets the destination to `null`.
- **`extractAlloc` optional null**: Same fix for the allocating extractor
  path.
- **`extractAlloc` `json.Value` arrays/objects**: Fell through to
  union-match logic and failed. Now handled via `matchJsonValue` with the
  real allocator.
- **`decodeNInt` overflow**: Decoding large negative integers (magnitude >
  `maxInt(i64)`) would panic. Now returns `error.IntegerTooSmall`.
- **`skipValue` truncated floats**: Calling `skipValue` on a truncated
  float16/32/64 would panic. Now returns `error.TooShort`.
- **`matchJsonValue` floats**: Float values were silently dropped during
  CBOR→`json.Value` extraction. All three widths now decoded correctly.
- **`writeValue` large unsigned integers**: Unsigned integers wider than
  `i64` were encoded incorrectly due to signed overflow.
- **`matchStructScalar` / `matchStructAlloc`**: A short read returned
  `false` instead of propagating `error.TooShort`, masking corruption as a
  non-match.

## Performance

- **`writeTypedVal`**: Single `writer.write` call instead of multiple
  single-byte writes.
- **Float encoding**: `writeF16`/`writeF32`/`writeF64` each issue a single
  write using `std.mem.writeInt` (also fixes a latent big-endian bug in the
  previous manual byte-swap code).
- **`decodeUIntLength`**: Replaced recursive implementation with an
  iterative one.
- **`fromJson` / `fromJsonAlloc`**: A single allocator is now threaded
  through the entire JSON parse instead of creating a new arena per nested
  container.

## Build

- Minimum Zig version is now **0.15.2**.
- Updated APIs for Zig 0.15 compatibility.
