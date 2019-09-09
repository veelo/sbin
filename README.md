# Simple binary [de]serialize

[![Build Status](https://travis-ci.org/deviator/sbin.svg?branch=master)](https://travis-ci.org/deviator/sbin)
[![codecov](https://codecov.io/gh/deviator/sbin/branch/master/graph/badge.svg)](https://codecov.io/gh/deviator/sbin)
[![Dub](https://img.shields.io/dub/v/sbin.svg)](http://code.dlang.org/packages/sbin)
[![License](https://img.shields.io/dub/l/sbin.svg)](http://code.dlang.org/packages/sbin)

## Usage

Library provides functions for simple serialize and deserialize data:

You can serialize/deserialize numbers, arrays, enums, structs and combinations of those.

### Functions

#### `void sbinSerialize(R, Ts...)(ref R r, auto ref const Ts vals) if (isOutputRange!(R, ubyte) && Ts.length)`

Call `put(r, <data>)` for all fields in vals recursively.

Do not allocate memory if you use range with `@nogc` `put`.

#### `ubyte[] sbinSerialize(T)(auto ref const T val)`

Uses inner `appender!(ubyte[])`.

#### `void sbinDeserialize(R, Target...)(R range, ref Target target) if (isInputRange!R && is(Unqual!(ElementType!R) == ubyte))`

Fills `target` from `range` bytes

Can throw:

* `SBinDeserializeEmptyRangeException` then try read empty range
* `SBinDeserializeException` then after deserialize input range is not empty

Exception messages builds with gc (`~` is used for concatenate).

Allocate memory if deserialize:

* strings
* associative arrays
* dynamic array if target length is not equal length from input bytes

#### `Target sbinDeserialize(Target, R)(R range)`

Creates `Unqual!Target ret`, fill it and return.

### Key points

* all struct fields are serialized and deserialized by default (no `ignore` or `serializeOnly` features)
* only dynamic arrays (associative arrays too) has variable length, all other types has fixed size

### Example

```d
// All enums serialize as numbers
enum Color
{
    black = "#000000",
    red = "#ff0000",
    green = "#00ff00",
    blue = "#0000ff",
    white = "#ffffff"
}

struct Foo
{
    ulong a;
    float b, c;
    ushort d;
    string str;
    Color color;
}

const foo1 = Foo(10, 3.14, 2.17, 8, "s1", Color.red);

//                 a              b            c       d
const foo1Size = ulong.sizeof + float.sizeof * 2 + ushort.sizeof +
//                      str                      color
        (length_t.sizeof + foo1.str.length) + ubyte.sizeof;

// color is ubyte because [EnumMembers!Color].length < ubyte.max

const foo1Data = foo1.sbinSerialize; // used inner appender

assert(foo1Data.length == foo1Size);

// deserialization return instance of Foo
assert(foo1Data.sbinDeserialize!Foo == foo1);

const foo2 = Foo(2, 2.22, 2.22, 2, "str2", Color.green);

const foo2Size = ulong.sizeof + float.sizeof * 2 + ushort.sizeof +
        (length_t.sizeof + foo2.str.length) + ubyte.sizeof;

enum Level { low, medium, high }

struct Bar
{
    ulong a;
    float b;
    Level level;
    Foo[] foos;
}

auto bar = Bar(123, 3.14, Level.high, [ foo1, foo2 ]);

//                   a               b          level
const barSize = ulong.sizeof + float.sizeof + ubyte.sizeof +
//                                 foos
                (length_t.sizeof + foo1Size + foo2Size);

assert(bar.sbinSerialize.length == barSize);
```

### Custom [de]serialization algorithm

Add to your type `Foo`:

* `T sbinCustomRepr() @property const` where `T` is serializable representation
  of `Foo`, what can be used for full restore data
* `static Foo sbinFromCustomRepr(auto ref const T repr)` what returns is new
  instance for your deserialization type

### TaggedAlgebraic

See [example](example/app.d).

## Limitations

### Unions

Unions serializes/deserializes as static byte array without analyze elements (size of union is size of max element).

**If you want use arrays or strings in unions you must implement custom [de]serialize methods or use `TaggedAlgebraic`**

### std.variant

Not supported. See [TaggedAlgebraic](#taggedalgebraic) if you need variablic types.

### Max dynamic array length

By default uses `uint` as `length_t`, for using `ulong` use library cofiguration `ulong_length`.

### Code versions

If you want use sbin for message passing between applications you
must use strictly identical types (one source code), because struct fields are not marked 
(deserialization relies solely on information from type) and any change in code
(swap fields, change fields type, change enum values list) must be accompanied by
recompilation of all applications.

### Immutable and const

Deserialize works after initialization of object and const or immutable
fields are can't be setted.

### Inderect fields

If struct have two arrays

```d
struct Foo
{
    ubyte[] a, b;
}
```

and these arrays point to one memory

```d
auto arr = cast(ubyte[])[1,2,3,4,5,6];
auto foo = Foo(arr, arr[0,2]);
assert (foo.a.ptr == foo.b.ptr);
```

then they will be serialized separated and after deserialize will be
point to different memory parts

```d
auto foo2 = foo.sbinSerialize.sbinDeserialize!Foo;
assert (foo2.a.ptr != foo.b.ptr);
```

### Classes

Classes must have custom serialize methods, otherwise they can't be serialized.

```d
class Foo
{
    ulong id;
    this(ulong v) { id = v; }

    ulong sbinCustomRepr() const @property
    {
        return id;
    }

    // must be static
    static Foo sbinFromCustomRepr(ulong v)
    {
        return new Foo(v);
    }
}
```