---
language: zig
contributors:
    - ["Marc Gozlan", "http://github.com/mouvedia"]
lang: en-us
---

This article applies to the latest version of Zig which is, at the time of the writing, [0.7.1](https://ziglang.org/download/0.7.1/release-notes.html).

Every code snippets assumes

```zig
const std = @import("std");
const assert = std.debug.assert;
const mem_eql = std.mem.eql;
```

## Table of Contents

- [Comments](#comments)
- [Control Flow](#control-flow)
  - [`if`](#if)
    - [Ternary](#ternary)
    - [Captures](#captures)
      - [Optional](#Optional)
        - [single capture](#single-capture)
        - [multiple captures](#multiple-captures)
        - [pointer](#pointer)
      - [Error Union](#error-union)
        - [mandatory `else`](#mandatory-else)
        - [value unwrapping](#value-unwrapping)
      - [Unwrapping Order](#unwrapping-order)
  - [Loops](#loops)
    - [`for`](#for)
    - [`while`](#while)
    - [`break`](#break)
    - [`continue`](#continue)
    - [`inline`](#inline)
  - [`switch`](#switch)
    - [inclusive range](#inclusive-range)
    - [enum shorthand](#enum-shorthand)
    - [exhaustiveness](#exhaustiveness)
    - [`_` prong](#_-prong)
  - [Labels](#labels)
- [Data Structures](#data-structures)
  - [`struct`](#struct)
    - [tuples](#tuples)
  - [`enum`](#enum)
  - [`union`](#union)
  - [arrays](#arrays)
  - [slices](#slices)
- [Types](#types)

## Comments

```zig
//! top-level documentation

/// plain documentation comment

// single line comment
```

<small>Note: there are no multiline comments</small>

## Control Flow

### `if`

```zig
fn compare(a: f64, b: f64) i2 {
  if (a > b) {
    return 1;
  } else if (a < b) {
    return -1;
  } else {
    return 0;
  }
}

assert(compare(5, 4) == 1);
```

<small>Note: no implicit cast to `bool` is ever performed</small>

#### Ternary

```zig
const answer = if (true) 42 else unreachable;
```

#### Captures

##### Optional

###### single capture

```zig
const bar: ?u32 = 0;

// unwraps the value if bar is not null
if (bar) |value| {
  assert(value == 0);
} else {
  unreachable;
}

// which is equivalent to
{
  const value = bar orelse unreachable;
  assert(value == 0);
}

// .? shorthand
{
  const value = bar.?;
  assert(value == 0);
}
```

###### multiple captures

```zig
const one: ?u32 = 1;
const two: ?u32 = 2;
if (one) |foo| if (two) |bar| {
  assert(one.? == foo);
  assert(two.? == bar);
};
```

###### pointer

```zig
var foo: ?u32 = 3;
// permits to access the value by reference
if (foo) |*ptr| {
  ptr.* = 2;
  assert(foo.? == 2);
}
```

##### Error Union

###### mandatory `else`

```zig
const zero: anyerror!u7 = 0;
if (zero) |value| {
  assert(value == 0);
} else |_| {}
```

###### value unwrapping

```zig
const bar: anyerror!u7 = error.Foo;
if (bar) |_| {} else |value| {
  assert(value == error.Foo);
}

// which is equivalent to

bar catch |value| {
  assert(value == error.Foo);
};
```

##### Unwrapping Order

```zig
var foo: anyerror!?u32 = 1;
if (foo) |*optional_ptr| {
  if (optional_ptr.*) |*ptr| {
    ptr.* = 2;
  }
} else |_| unreachable;

assert((try foo).? == 2);
```

<small>Note: error unions are always checked first</small>

### Loops

#### for

`for (iterable) |*item, index| statement else statement`

```zig
var array: [3]i32 = undefined;

for (array) |*item, i| {
  // i is of type usize
  item.* = @intCast(i32, i);
}

assert(array[0] == 0);
assert(array[1] == 1);
assert(array[2] == 2);
```

<small>Note: only [arrays](#arrays), [slices](#slices) and [tuples](#tuples) can be iterated over</small>

#### while

`while (condition) |capture| : (statement) statement else statement`

```zig
var uppercaseLetters: [26]u8 = undefined;
var i: usize = 0;

while (i < 26) : (i += 1) {
  uppercaseLetters[i] = @intCast(u8, i) + 65;
}

assert(uppercaseLetters[25] == @intCast(u8, 'Z'));
```

#### break

```zig
var i: u32 = 0;
const result = while (true) {
  i += 1;
  if (i < 10) continue;
  break "break";
} else "else";

assert(mem_eql(u8, result, "break"));
```

#### continue

```zig
// evaluates to &[_:0]u8{ 'f', 'o', 'o' };
const str = "foo";

// the index capture is optional
const result = for (str) |_| {
  continue;
} else "else";

assert(mem_eql(u8, result, "else"));
```

#### inline

An inline loop will be unrolled at compile time.

```zig
const tuple = .{ true, 42, "foo" };
comptime const foo = struct {
  var @"0": bool = undefined;
  var @"1": i32 = undefined;
  var @"2": []const u8 = undefined;
};

@setEvalBranchQuota(std.math.maxInt(u32));
inline for (tuple) |*item, i| {
  comptime const char = std.fmt.comptimePrint("{}", .{i});
  @field(foo, char) = tuple[i];
}

assert(mem_eql(u8, @field(foo, "2"), "foo"));
```

### `switch`

#### inclusive range

```zig
const foo: i32 = 0;
const bar = switch (foo) {
  0 => "zero",
  1...std.math.maxInt(i32) => "positive",
  else => "negative",
};

assert(mem_eql(u8, bar, "zero"));
```

<small>Note: the order of the cases doesn't matter hence overlapping ought to be avoided to ward off undefined behaviours</small>

#### enum shorthand

```zig
const cardinalDirections = enum { North, South, East, West };
const direction: cardinalDirections = .North;
const bool = switch (direction) {
  // shorthand of cardinalDirections.North => true,
  .North => true,
  else => false
};

assert(cardinalDirections.North == .North);
assert(bool);
```

#### exhaustiveness

```zig
const bar: error{ Foo, Qux }!u7 = error.Foo;
bar catch |err| switch (err) {
  error.Foo => {},
  error.Qux => unreachable,
};
```

<small>Note: if a switch statement does not have an `else` prong, it must exhaustively list all the possible values</small>

#### `_` prong

```zig
const Elements = enum(u7) {
  hydrogen = 1,
  helium = 2,
  ununennium = 119,
  unbinilium = 120,
  _
};

const element = Elements.hydrogen;
const is_hydrogen = switch (element) {
  .hydrogen => true,
  .helium,
  .ununennium,
  .unbinilium => false,
  // only available when switching on non-exhaustive enums
  _ => false
};

assert(is_hydrogen);
```

<small>Note: a compile error will be thrown if all the _currently_ named tags are not handled by the switch</small>

### Labels

```
```

## Data Structures

### `struct`

```zig
const Foo = struct {
  const bar = "bar";

  a: bool,
  b: bool,
  c: bool = true,
};

assert(mem_eql(u8, @field(Foo, "bar"), "bar"));

// instantiation
var foo = Foo {
  .a = false,
  .b = undefined,
};

foo.b = true;

assert(foo.a == false);
assert(foo.b == true);
assert(foo.c == true);
```

<small>Notes:

- `@field` [currently](https://github.com/ziglang/zig/issues/3839) returns fields _and_ declarations.
- the order of the fields cannot be garanteed to be preserved
- the fields' alignment is ABI dependent
- `@bitSizeOf` and `@sizeOf` cannot be relied on for plain structs
</small>

#### tuples

Tuples are anonymous struct literals that are analogous to [arrays](#arrays) in various ways:

- support array-specific operators (`**` and `++`)
- have a `len` field
- may be indexed by a comptime index
- must use `inline for` to be iterated over

```zig
const foo = .{
  "bar",
  true,
  42
} ++ .{ undefined };

assert(foo.len == 4);
assert(foo[2] == foo.@"2");
```

### `enum`

```zig
const Result = enum(u1) {
  const Self = @This();

  ok = 1,
  ko = 0,

  fn toString(self: Self) []const u8 {
    return @tagName(self);
  }

  fn fromString(str: []const u8) !Self {
    return std.meta.stringToEnum(Result, str) orelse error.InvalidEnumTag;
  }
};

try std.testing.expectError(error.InvalidEnumTag, Result.fromString("bar"));
```

### `union`

```
```

### arrays

```
```

### slices

```
```

## Types

| Name                         | Description                                                                  |
|------------------------------|------------------------------------------------------------------------------|
| i0–i65535, u0–u65535         | integer                                                                      |
| isize, usize                 | pointer sized integer                                                        |
| f16, f32, f64, f128          | floating point IEEE-754-2008                                                 |
| c_short, c_ushort<br>c_int, c_uint<br>c_long, c_ulong<br>c_longlong, c_ulonglong<br>c_longdouble<br>c_void  | for ABI compatibility with C
| void                         | 0 bit type                                                                   |
| bool                         | true or false                                                                |
| noreturn                     | the type of `break`, `continue`, `return`, `unreachable`, and `while (true) {}` |
| type                         | the type of types                                                            |
| anyerror                     | global error set                                                             |
| comptime_int, comptime_float | comptime-known literals                                                      |

Notes

- characters are of type `comptime_int` but strings are of type `[]const u8`
- [`c_char` doesn't exist yet](https://github.com/ziglang/zig/issues/875)
- bfloat16 (AKA `f16b`) and tf32 (AKA `f19b`) are missing
