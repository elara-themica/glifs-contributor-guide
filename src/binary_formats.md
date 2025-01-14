# Binary Formats

<!-- toc -->

In general, the _glyph_ format uses little-endian byte ordering, as this is the
de-facto standard for most modern processors. For consistency, and for clarity
when bit fields span multiple bytes, this means that bit order within bytes is
also presented with the least significant bit first.

## The Glyph Header

```mermaid
---
title: "Glyph Header (bits)"
---
packet-beta
0-2: "Padding Bytes"
3: "L"
4-7: "Glyphs Version"
8-15: "Reserved"
16-31: "Glyph Type"
32-63: "Length (8-byte Words) or Short Content"
```

- __Version__. A 4-bit unsigned integer describing the version of GLIFS.
  Currently, these bit must all be set to zero.
- __Long Bit__. If this bit is set to zero, the content length field is
  ignored, and 4 bytes of data can be stored in that field instead. If the bit
  is set to one, then the content length field is interpreted normally (see
  below).
- __Padding__. A 3-bit unsigned integer describing the amount of zero-padding
  at the end of a glyph. As the length field specifies the length of the
  content section in increments of 8 bytes, glyphs with content requiring a
  specific byte length are required to be zero-padded. For example, if storing
  a 13-byte UTF string, the content length field would be equal to 2 (16 bytes)
  and the padding bits would be set to `011` (3).
- __Glyph Type__. A little-endian 16-bit unsigned integer describing the type
  of data stored in the content section. In the experimental version of GLIFS,
  this list is stored in the `enum GlyphType` in `glyph.rs`.
- __Content Length (or Data)__. The interpretation of this field depends on the
  value of the long bit. If the long bit is set to 1, this indicates that the
  field contains the length of the content section of the glyph, in multiples
  of 8 bytes, stored as an unsigned, little-endian, 32-bit integer.

Note that glyphs are always a multiple of 8-bytes long, so that we can guarantee
alignment (up to 8 bytes) when necessary.

## Unit Glyphs

```mermaid
---
title: "Unit Glyph (Short Glyph Header, bits)"
---
packet-beta
0-2: "Padding Bytes"
3: "0"
4-7: "Glyphs Version"
8-15: "Reserved"
16-31: "GlyphType::Unit"
32-63: "UnitTypes (a u32 enum, in little-endian format)"
```

Unit glyphs are used to represent unit types, that is, types that hold only one
value and thus no information. In other words, these are types for which we are
only interested in knowing the specific type but do not need to store any data
apart from the type identifier.
Seethe [wikipedia article](https://en.wikipedia.org/wiki/Unit_type) for a short
description.

One of the most common use of the unit type is probably as the value for
`Option::None`, used but other kinds of unit types can also be useful.

## Boolean Glyphs

```mermaid
---
title: "Boolean Glyph (Short Glyph Header, bits)"
---
packet-beta
0-2: "Padding Bytes"
3: "0"
4-7: "Glyphs Version"
8-15: "Reserved"
16-31: "GlyphType::Boolean"
32-63: "All zeros means false, any other value means true"
```

A single true/false value. As a short glyph, the "long bit" is set to zero and
the contents are placed in the length field. When read, a value of all zeros
means `false`, and any other value means `true`. Can be found in `basic.rs` as
the
type `BoolG`.

### Bit Vector Glyphs

An array of bits. Could be used, e.g., for storing bloom filter data. Can be
found in `basic.rs` as the type `BitVecG`.

- The first byte of the contents section is used to store an adjustment
  to the number of bits in the vector, so the length can be exact and
  not just a multiple of 8.
- The rest of the bytes in the content section contain the bit vector
  itself, with the first bit index in every byte starting at the most
  significant bit.
- An exact number of bits can be specified by taking the number of bytes
  in the unpadded content section, subtracting one, multiplying by eight,
  and subtracting the three least significant bits of the first byte of the
  contents section.
- Alternatively, the contents section can be completely empty, which is
  interpreted as an empty bit vector.

```mermaid
---
title: "Bit Vector Glyph (bits)"
---
packet-beta
0-2: "Padding Bytes"
3: "L"
4-7: "Glyphs Version"
8-15: "Reserved"
16-31: "GlyphType::VecBool"
32-63: "Length of content, in 8-byte words"
64-66: "Bits Padding"
67-71: "Reserved (0)"
72-95: "Data..."
```

## Numeric Glyphs

Currently, numeric types are spearated into three types--signed, unsigned, and
floating point. Different lengths are supported based on the length of the glyph
itself, with 32-, 64-, and 128-bit versions supported for integer types, and 32-
and 64-bit versions are supported for floating point types. See
`basic::IntGlyph`, `basic::UIntGlyph` and `basic::FloatGlyph` for more details.

Additionally, as the overhead of a glyph header and type checking would be
unreasonable when dealing with numeric arrays, more specific arrays can
currently be used with a generic zero-copy vector for simple fixed-length types.
See `basic::ZcVecG` for more information. Feedback is requested on the
possibility of a better way of presenting this to users as an API.

Example: 32-bit Unsigned Integer.

```mermaid
---
title: "32-bit Unsigned Integer Glyph (bits)"
---
packet-beta
0-2: "Padding Bytes"
3: "0"
4-7: "Glyphs Version"
8-15: "Reserved"
16-31: "GlyphType::UnsignedInt"
32-63: "Unsigned 32-bit Integer (Little Endian)"
```

Example: 128-bit Signed Integer

```mermaid
---
title: "128-bit Signed Integer Glyph (bits)"
---
packet-beta
0-2: "Padding Bytes"
3: "1"
4-7: "Glyphs Version"
8-15: "Reserved"
16-31: "GlyphType::SignedInt"
32-63: "2"
64-191: "Signed 128-bit Integer (Little Endian)"
```

Example: 64-bit Floating Point

```mermaid
---
title: "64-bit Floating Point Glyph (bits)"
---
packet-beta
0-2: "Padding Bytes"
3: "1"
4-7: "Glyphs Version"
8-15: "Reserved"
16-31: "GlyphType::FloatingPoint"
32-63: "1"
64-127: "64-bit Floating Point IEEE-754 (Little Endian)"
```

Ideas for additional numeric data types in the future include currency,
dimensioned tensors, imaginary numbers, large decimals, and so on.

## Language Types

### String Glyphs

```mermaid
---
title: "String Glyph (bits)"
---
packet-beta
0-2: "Padding Bytes"
3: "L"
4-7: "Glyphs Version"
8-15: "Reserved"
16-31: "GlyphType::String"
32-63: "Length of content, in 8-byte words"
64-71: "Language"
72-77: "Reserved"
78-79: "Direction"
80-121: "UTF-8 Data"
122-127: "Zero Padding"

```

Strings are encoded in UTF-8, with additional metadata for language and text
direction. (These choices were informed by a
2024 [W3C working group paper](https://www.w3.org/TR/string-meta/).)  Feedback
is requested on whether encodings other than `UTF-8` should be supported, the
proper collation method, and any discussion of additional metadata that should
be included.

The default collation method for the `str` type in rust is language-unaware
bytewise comparison, but unicode sorting using the default locale seems like it
would be preferable. Currently, string collation is performed with the
`icu_collcation` crate using the default collation options and locale.

The value of the language field corresponds with `enum Language` in `lang.rs`.
The list of languages there was taken from the Wikipedia's list
of [ISO 639 language codes](https://en.wikipedia.org/wiki/List_of_ISO_639_language_codes)
as of 2025.01.10.

### Character Glyphs

Character glyphs currently reflect the rust `char` type, which the documentation
referrs to as a "Unicode Scalar Value".

```mermaid
---
title: "Character Glyph"
---
packet-beta
0-2: "Padding Bytes"
3: "L"
4-7: "Glyphs Version"
8-15: "Reserved"
16-31: "GlyphType::UnicodeChar"
32-63: "Data (Unicode Scalar Value, LE)"
```

## Collections

Currently, four types of collations are provided. Tuples, vectors, _basic_
vectors, and maps. Plans for future development include a copy-on-write
LSM/B-Tree hybrid based on a Merkle DAG, for creating indexes that can be shared
and updated directly by cliets / peers without performing collation.

### Tuple Glyphs

Tuple Glyphs can contain an abitrary number of item glyphs, though they
arenintended to be used (1) with a relatively small number and (2) where all
items will be decoded, as there is no random access to the individual items. For
the raw glyph type, see `TupleGlyph`. Encoding and decoding is currently
provided for rust tuples up to length 6.

Glyphs that are either (1) short or (2) long but with a content length of zero
are interpreted as tuples of length zero.

```mermaid
---
title: "Tuple Glyph (bytes)"
---
packet-beta
0-7: "Glyph Header"
8-47: "Item #0..."
48-55: "Item #1..."
56-63: "Item #2..."

```

### Vector Glyphs

Vector glyphs differ from Tuple glyphs only in that an offsets table is provided
to allow for random access within the vector. This allows users to decode
individual glyphs without also decoding every preceeding glyph.

Offsets are in the same units as the length field in the `Glyph` header, a `u32`
that specifies a number of 8-byte words relative to the start of the glyph's
content. The number of items in the vector is specified with a `u32` at the
start of the content section, followed by a vector of `u32` offsets. If
necessary (i.e., there are an even number of items), zero padding must be added
at the end of the offsets table so that the first glyph starts on an 8-byte
boundary.

Glyphs that are either (1) short or (2) with a content length of zero are
interpreted as a vectors of length zero. See the `VecGlyph` type for more
details.

```mermaid
---
title: "Vector Glyph (bytes)"
---
packet-beta
0-7: "Glyph Header"
8-11: "# of Items (u32)"
12-15: "Offset of Item #0"
16-19: "Offset of Item #1"
20-23: "Zero Padding"
24-47: "Glyph #0 ..."
48-63: "Glyph #1 ..."
```

### Basic Vector Glyphs

Basic vector glyphs are intended for use when storing a large number of items of
the same type and fixed length, so the storage and processing overhead of a
glyph header for each item can be avoided. When possible (they can use a
fixed-endian format and require an alignment of 8 bytes or less), they use
common in-memory formats and can be referenced as simply a `&[T]`.

Basic vector glyphs can also optionally store rank and dimensionality for
tensors of arbitrary length. For details, see the `BasicVecGlyph` type. Feedback
is welcome on whether this facility is useful.

```mermaid
---
title: "Basic Vector Glyph (Bytes)"
---
packet-beta
0-7: "Glyph Header"
8-9: "Type ID"
10-11: "Rank"
12-15: "Size_0"
16-19: "Size_1"
20-23: "Padding (opt)"
24-43: "Array of [T]..."
44-47: "Padding"
```

### Map Glyphs

Map glyphs are similar to vector glyphs, but for the fact that both a key and
value are stored at each offset.

Ideally, maps should be written so that keys are in sorted order and therefore
items can be found by key value with a binary search in `log(2)` time. However, this requires a consitent sort order for glyphs.  Additionally, this may interact with practical encoding issues for some sources of map data (e.g., `HashMap`s, etc...). Contributions are currently
welcome along these lines.

See `MapGlyph` for more info.

```mermaid
---
title: "Map Glyph (bytes)"
---
packet-beta
0-7: "Glyph Header (GlyphType::Map)"
8-11: "# of Items (u32)"
12-15: "Reserved"
16-19: "Offset of K/V Pair #0"
20-23: "Offset of K/V Pair #0"
24-31: "Zero Padding"
32-39: "Key #0 ..."
40-47: "Value #0 ..."
48-55: "Key #1 ..."
56-63: "Value #1 ..."
```

## Structured Types

### Object Glyphs

Object glyphs are for labeled, semi-structured data with arbitrary field names
and values. The labels may include zero or more of the following:

- A cryptographic hash referring to a glyph where further documentation on
  the type can be found.
- A (Utf-8) type name
- A variant name (e.g., for sum types like rust's `enums`)
- An (integer) variant ID (like above).

These identifiers start at bit 128, and their presence is determined by the
`flags field` described below. If present, they are stored in the order listed
above. If either a type name or variant is specified, their binary
representation starts with a single `u8` byte describing the length of the
string, the UTF-8 data, and zero-padding to the nearest 8-byte boundary.

Following these identifiers, there is an offsets table for each field, similar
to those describe in previous types. The length of this table is equal to
the number of fields as identified in the header, zero-padded to the nearest
8-byte boundary.

```mermaid
---
title: "Object Glyph"
---
packet-beta
0-3: "Version"
4-7: "Reserved"
8-10: "0"
11: "1"
12-15: "Reserved"
16-31: "GlyphType::Object"
32-63: "Length (in 8-byte words)"
64-71: "Version"
72-79: "Flags"
80-95: "# of Fields"
96-127: "Reserved"
128-191: "Data ... (See Above)"
```

The `flags` field contains the following flags, starting from the least
significant bit:

| Bit | Name                 |
|-----|----------------------|
| 0   | Type Hash Present    |
| 1   | Type Name Present    |
| 2   | Variant Name Present |
| 3   | Variant ID Present   |
| 4   | Field Names Present  |
| 5   | Fields Are Ordered   |
| 6   | Reserved / Unused    |
| 7   | Reserved / Unused    |

If the "Field Names Present" is set, then each field (i.e., the offset at
which the field offsets table points) contains the UTF-8 field name in the
same format as described above for the type and variant names--i.e., a
single `u8` byte describing the length of the string, the UTF-8 data, and
zero-padding to the nearest 8-byte boundary.

Finally, the "Fields are Ordered" bit describes whether, if the fields are
named, if they are also stored in sorted order.

See the `ObjG` structure in `obj.rs` for more information. Feedback is
requested on whether this type can or should be simplified.

### Document Glyphs

Document glyphs are for semi-structured information typical in
a [document-oriented database](https://en.wikipedia.org/wiki/Document-oriented_database).
Document glyphs are
a [conflict-free replicated data type](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type) (
CRDT), or, a bit more accurately, provide a way to track and manage these
conflicts to be resolved at the application layer.

When multiple documents are written to a collection with the same ID, because
collections are content-addressed there is no conflict between the two at the
storage layer. However, at the index layer, we can take note of the existence of
two objects with the same ID and present them to the application layer to
resolve the conflict. Conflict resolution takes the form of a new document
listing the hashes of the previous documents that it is replacing and the
transaction in which it was committed to the collection.

(This is to ensure consistent operation with unordered updates that occur when
data is updated in different partitions. See the chapter on documents for more
details.)

The data section includes an array of these previous versions of the specified
length, followed immediately by two glyphs--the (1) identifier (2) body.

```mermaid
---
title: "Document Glyph"
---
packet-beta
0-3: "Version"
4-7: "Reserved"
8-10: "0"
11: "Long"
12-15: "Reserved"
16-31: "GlyphType::Document"
32-63: "Length (in 8-byte words)"
64-79: "Reserved (0)"
80-95: "# of Previous Versions (u16)"
96-127: "Reserved (0)"
128-191: "Data ... (See Above)"
```

