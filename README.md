# Zig Language Cheat Sheet (v0.15.1)

## Table of Contents

1. [Basic Syntax & Types](#1-basic-syntax--types)
2. [Control Flow](#2-control-flow)
3. [Functions & Generics](#3-functions--generics)
4. [Comptime](#4-comptime)
5. [Error Handling](#5-error-handling)
6. [Memory Management](#6-memory-management)
7. [Build System](#7-build-system)
8. [Standard Library Highlights](#8-standard-library-highlights)
9. [Testing](#9-testing)
10. [Builtin Functions Reference](#10-builtin-functions-reference)
11. [Common Patterns & Gotchas](#11-common-patterns--gotchas)

---

## 1. Basic Syntax & Types

### Variables

```zig
// Immutable (default preference)
const x: i32 = 10;
const y = 42;              // type inferred

// Mutable
var z: u8 = 200;
z += 1;

// Shadowing allowed
const width: u32 = 100;
const width: f64 = 100.0;  // different type, same name

// Undefined (no initialization, must set before read)
var buf: [64]u8 = undefined;
```

### Primitive Types

```zig
// Integers — arbitrary bit width
const a: u8  = 255;        // unsigned, 0..255
const b: i8  = -128;       // signed, -128..127
const c: u16 = 65535;
const d: i32 = -42;
const e: usize = 0;        // pointer-sized unsigned (array index type)
const f: isize = 0;        // pointer-sized signed

// Floats
const g: f16 = 1.0;        // half precision
const h: f32 = 3.14;       // single precision
const i: f64 = 2.71828;    // double precision
const j: f128 = 0.0;       // quad precision

// Boolean
const ok: bool = true;
const err: bool = false;

// comptime-only types (cannot appear at runtime)
comptime {
    const ci: comptime_int = 999;
    const cf: comptime_float = 3.14;
}

// noreturn — function never returns
fn panic() noreturn { while (true) {} }

// void — absence of value (like unit type)
const nothing: void = {};

// type — the type of types
const T: type = u32;

// anyopaque — opaque pointer (used for C interop)
const ptr: *anyopaque = @ptrFromInt(0x1000);
```

### Optional Types (`?T`)

```zig
const maybe_int: ?i32 = null;       // no value
const has_int: ?i32 = 42;           // value present

// Unwrap with orelse (provide default)
const val: i32 = has_int orelse 0;  // 42

// Unwrap with if capture
if (has_int) |n| {
    // n is i32 here
}

// Unwrap with while capture
var opt: ?u8 = 100;
while (opt) |n| {
    opt = null;  // exit after one iteration
}

// Optional pointer (nullable pointer, single null byte)
const opt_ptr: ?*u32 = null;
```

### Error Union Types (`!T`)

```zig
// Function returning error union
fn mightFail() !i32 {
    return error.SomethingBroke;
}

// Handle with catch
const result: i32 = mightFail() catch 0;

// Handle with try (propagates error upward)
const result = try mightFail();

// Handle with if (in assignment)
if (mightFail()) |value| {
    // value is i32
} else |err| {
    // err is the error
}
```

### Arrays

```zig
// Fixed-size array
const arr: [3]u8 = .{ 1, 2, 3 };
const first = arr[0];             // 1
const len = arr.len;              // 3 (comptime-known)

// Infer length with _
const inferred: [_]u8 = .{ 10, 20, 30 };

// Sentinel-terminated array (C string compatible)
const cstr: [5:0]u8 = .{ 'h', 'e', 'l', 'l', 'o' };
// len = 5, but guaranteed null terminator at index 5

// Multidimensional
const matrix: [2][3]i32 = .{ .{ 1, 2, 3 }, .{ 4, 5, 6 } };

// Comptime array init with repeating value
const zeroes: [100]u8 = .{0} ** 100;
// ** is the array multiplication operator
```

### Slices

```zig
const arr = [_]u8{ 1, 2, 3, 4, 5 };

// Slice — pointer + length (fat pointer)
const slice: []const u8 = &arr;          // entire array
const slice2: []const u8 = arr[1..4];    // { 2, 3, 4 }
const slice3: []const u8 = arr[0..3];    // { 1, 2, 3 }

// Mutable slice
var mutable_arr = [_]u8{ 1, 2, 3 };
const mut_slice: []u8 = &mutable_arr;
mut_slice[0] = 99;

// Runtime-known length
const runtime_len: usize = 3;
const dyn_slice = arr[0..runtime_len];

// Sentinel-terminated slice
const sent_slice: [:0]const u8 = "hello";  // len=5, sentinel=0
```

### Slices — Key Distinction from Arrays

```zig
const arr: [3]u8 = .{ 1, 2, 3 };    // value type, copied on assign
const s: []const u8 = &arr;          // reference type (ptr+len)

// arr is 3 bytes on stack
// s is 16 bytes on 64-bit (ptr 8 + len 8)

// const slice — element type is const
// Cannot modify elements even if underlying array is mutable
var m_arr = [_]u8{ 1, 2, 3 };
const cs: []const u8 = &m_arr;
// cs[0] = 9;  // compile error

const ms: []u8 = &m_arr;
ms[0] = 9;     // ok
```

### Pointers

```zig
var x: u32 = 42;

// Single-item pointer
const px: *u32 = &x;          // pointer to one u32
const pcx: *const u32 = &x;   // pointer to const u32
px.* = 100;                    // dereference to mutate

// Const pointer (pointer itself is const, not value)
const cpx: *const u32 = &x;
// cpx = &other;  // compile error — pointer is const

// Many-item pointer (C style, unknown length)
const mptr: [*]u32 = @ptrCast(&x);
// mptr[0] = 10;  // ok, but no bounds checking
// mptr.len;      // compile error — no length

// Sentinel many-item pointer
const sptr: [*:0]const u8 = "hello";

// Address-of syntax
const addr: usize = @intFromPtr(&x);

// Optional pointer (C nullable pointer)
const opt: ?*u32 = null;
```

### Structs

```zig
// Named struct
const Point = struct {
    x: f64 = 0.0,    // default value
    y: f64 = 0.0,

    // method (takes self pointer)
    pub fn length(self: Point) f64 {
        return @sqrt(self.x * self.x + self.y * self.y);
    }
};

// Create with named fields
const p1 = Point{ .x = 3.0, .y = 4.0 };
const p2: Point = .{ .x = 1.0, .y = 2.0 };
_ = p1.length();

// Anonymous struct (inferred type)
const anon = .{ .name = "Alice", .age = 30 };
// anon is of type struct { name: []const u8, age: i32 }

// Anonymous struct literal as function argument
fn draw(pos: struct { x: f64, y: f64 }) void { _ = pos; }
draw(.{ .x = 10.0, .y = 20.0 });

// Packed struct — bit-exact layout, backs integers
const Flags = packed struct(u8) {
    a: bool = false,
    b: bool = false,
    c: bool = false,
    _: u5 = 0,        // padding
};
const f: Flags = .{ .a = true };
_ = @as(u8, @bitCast(f));  // 0b00000001

// Extern struct — C ABI compatible
const CStruct = extern struct {
    a: i32,
    b: f64,
};
```

### Enums

```zig
// Simple enum
const Color = enum {
    red,
    green,
    blue,
};
const c: Color = .red;

// Enum with explicit tag values
const Status = enum(u8) {
    ok = 200,
    not_found = 404,
    server_error = 500,
    _,
};

// Non-exhaustive enum (forces else in switch)
const Animal = enum(u2) {
    cat,
    dog,
    _,
};

// @tagName — get name as string
const name: []const u8 = @tagName(Color.red);  // "red"

// @intFromEnum / @enumFromInt
const n: u8 = @intFromEnum(Status.ok);
const s: Status = @enumFromInt(200);

// Enum literal — comptime-only, coerces to any enum
const lit: Color = .red;
```

### Unions

```zig
// Plain union — manual tag management
const Value = union {
    int: i32,
    float: f64,
    bool: bool,
};
var v = Value{ .int = 42 };
v.float = 3.14;  // overwrites int

// Tagged union — compiler tracks which field is active
const Tagged = union(enum) {
    int: i32,
    float: f64,
    text: []const u8,
};
var tv = Tagged{ .int = 42 };
// Accessing tv.float would be safety-checked panic

switch (tv) {
    .int => |val| _ = val,       // val is i32
    .float => |val| _ = val,     // val is f64
    .text => |val| _ = val,      // val is []const u8
}
```

### Vectors

```zig
// SIMD vector — operates on multiple values simultaneously
const v1: @Vector(4, i32) = .{ 1, 2, 3, 4 };
const v2: @Vector(4, i32) = .{ 5, 6, 7, 8 };
const sum = v1 + v2;  // element-wise: { 6, 8, 10, 12 }

// Boolean vector
const mask: @Vector(4, bool) = v1 > v2;  // { false, false, false, false }

// Shuffle
const shuffled = @shuffle(i32, v1, v2, .{ 0, 4, 1, 5 });
```

### Type Coercion & Casting

```zig
// @as — explicit type coercion (only safe, comptime-checked)
const a: u16 = @as(u16, 42);
const b: f64 = @as(f64, 3.14);

// @intCast — between same-sign integers (runtime check in safe modes)
const small: u8 = @intCast(255);    // from comptime_int
const big: u32 = 1000;
const back: u8 = @intCast(big);     // panic if out of range

// @floatCast — between float types
const f: f32 = @floatCast(3.14);    // from comptime_float
const f64_val: f64 = 1.0;
const f32_back: f32 = @floatCast(f64_val);

// @floatFromInt, @intFromFloat — cross-kind conversion
const fi: f64 = @floatFromInt(42);
const nt: i32 = @intFromFloat(3.9);  // truncates to 3

// @truncate — narrow integer, discard high bits
const wide: u32 = 0x12345678;
const narrow: u16 = @truncate(wide);  // 0x5678

// @bitCast — reinterpret bits (same size)
const bits: u32 = @bitCast(@as(f32, 1.0));
const opaque_ptr: *anyopaque = @ptrFromInt(0x1000);
const typed_ptr: *u8 = @ptrCast(opaque_ptr);

// @intFromPtr / @ptrFromInt
const addr: usize = @intFromPtr(&a);
const back_ptr: *u16 = @ptrFromInt(addr);
```

---

## 2. Control Flow

### `if` / `else`

```zig
// if is an expression — both branches must be compatible types
const result = if (true) 1 else 0;  // i32

// if without else produces void
if (x > 10) {
    // do something
}

// Optional capture
const opt: ?i32 = 42;
if (opt) |value| {
    _ = value;  // value is i32
} else {
    // opt was null
}

// Error union capture
if (mightFail()) |value| {
    _ = value;
} else |err| {
    _ = err;
}

// Chained if-else
if (cond1) {
} else if (cond2) {
} else {
}
```

### `switch`

```zig
// Switch on enum
const color: Color = .red;
const name = switch (color) {
    .red   => "red",
    .green => "green",
    .blue  => "blue",
};

// Switch on integer with ranges
const age: u8 = 25;
const category = switch (age) {
    0..=12  => "child",
    13..=19 => "teen",
    20..=59 => "adult",
    else    => "senior",
};

// Switch with capture (on tagged union)
const tv = Tagged{ .int = 42 };
switch (tv) {
    .int   => |val| _ = val,
    .float => |val| _ = val,
    .text  => |val| _ = val,
}

// Inline switch — generates code per branch
fn printValue(value: anytype) void {
    switch (@TypeOf(value)) {
        i32, i64 => { _ = value; },
        f32, f64 => { _ = value; },
        else => @compileError("unsupported type"),
    }
}

// Inline with else
inline for (.{ u8, u16, u32 }) |T| {
    _ = T;
}
```

### `while`

```zig
// Basic while
var i: u32 = 0;
while (i < 10) : (i += 1) {
    _ = i;  // 0..9
}

// while-else (runs if condition was never true)
while (false) {
    unreachable;
} else {
    // executed
}

// While with optional capture
var it: ?u8 = 100;
while (it) |val| : (it = null) {
    _ = val;  // runs once
}

// While with error capture
while (next()) |item| {
    _ = item;
} else |err| {
    // error from next() or null
}

// Labeled while with continue
var j: u32 = 0;
outer: while (j < 3) : (j += 1) {
    var k: u32 = 0;
    while (k < 3) : (k += 1) {
        if (k == 1) continue :outer;  // continue outer loop
    }
}

// break from labeled loop
outer: while (true) {
    while (true) {
        break :outer;  // breaks both loops
    }
}
```

### `for`

```zig
// For over slice (index only)
const items = [_]u8{ 10, 20, 30 };
for (items) |item| {
    _ = item;  // 10, 20, 30
}

// For with index
for (items, 0..) |item, idx| {
    _ = idx;   // 0, 1, 2
    _ = item;
}

// For over multiple slices (zip-like)
const a = [_]u8{ 1, 2, 3 };
const b = [_]u8{ 4, 5, 6 };
for (a, b) |x, y| {
    _ = x + y;  // 5, 7, 9
}

// For with pointer (avoids copy)
for (&items) |*item| {
    item.* *= 2;  // mutates in place
}

// For over range literal
for (0..5) |i| {
    _ = i;  // 0, 1, 2, 3, 4
}

// Labeled for with break
outer: for (items) |item| {
    for (items) |_| {
        break :outer;
    }
}
```

### `blk` — Break from Block

```zig
// Labeled block — break with a value
const result = blk: {
    if (some_condition) {
        break :blk 10;
    }
    break :blk 20;
};

// Nested blk
const val = outer: {
    const tmp = inner: {
        break :inner 5;
    };
    break :outer tmp * 2;
};
// val = 10
```

### `defer` & `errdefer`

```zig
// defer — runs at end of current scope (LIFO order)
{
    defer _ = cleanup1();  // runs third
    defer _ = cleanup2();  // runs second
    defer _ = cleanup3();  // runs first
    // ... work ...
}

// errdefer — runs only if scope exits with error
fn doWork(alloc: std.mem.Allocator) !void {
    const buf = try alloc.alloc(u8, 100);
    errdefer alloc.free(buf);  // only free if we return error
    try mightFail();            // if this fails, buf is freed
    // success path — caller manages buf
}

fn cleanup1() void {}
fn cleanup2() void {}
fn cleanup3() void {}
fn some_condition() bool { return true; }
fn next() ?u8 { return null; }
const std = @import("std");
```

---

## 3. Functions & Generics

### Function Basics

```zig
// Basic function
fn add(a: i32, b: i32) i32 {
    return a + b;
}

// Void return
fn noop() void {}

// Error return
fn mightFail() !void {
    return error.SomethingWentWrong;
}

// noreturn
fn abort() noreturn {
    while (true) {}
}

// Multiple return values via struct
fn divide(a: f64, b: f64) struct { quotient: f64, remainder: f64 } {
    return .{ .quotient = a / b, .remainder = @mod(a, b) };
}
```

### Generic Functions

```zig
// Comptime type parameter
fn max(comptime T: type, a: T, b: T) T {
    return if (a > b) a else b;
}
const m = max(i32, 10, 20);  // 20

// anytype parameter (type inferred at call site)
fn print(thing: anytype) void {
    const T = @TypeOf(thing);
    _ = T;
}

// Return type dependent on parameter
fn makeSlice(comptime T: type, ptr: [*]T, len: usize) []T {
    return ptr[0..len];
}

// Comptime value parameter
fn Array(comptime N: usize) type {
    return [N]u8;
}
const Buf = Array(100);
var buf: Buf = undefined;
```

### Inline Functions

```zig
// Inline function — body is inserted at call site
inline fn add(a: i32, b: i32) i32 {
    return a + b;
}

// Inline with comptime parameters creates specialized versions
inline fn dispatch(comptime T: type, value: T) void {
    switch (T) {
        i32 => _ = value + 1,
        f64 => _ = value * 2.0,
        else => @compileError("unsupported"),
    }
}
```

### Function Pointers

```zig
fn add(a: i32, b: i32) i32 { return a + b; }
fn sub(a: i32, b: i32) i32 { return a - b; }

const BinOp = *const fn (i32, i32) i32;
const op: BinOp = if (some_condition()) &add else &sub;
const result: i32 = op(10, 5);

fn some_condition() bool { return true; }
```

### Public / Private

```zig
// Functions are private by default
fn privateFunc() void {}

// pub makes them accessible outside the file
pub fn publicFunc() void {}
```

---

## 4. Comptime

### Comptime Expressions & Blocks

```zig
// comptime keyword forces evaluation at compile time
const square = comptime blk: {
    const x: i32 = 7;
    break :blk x * x;
};  // 49, embedded in binary

// Comptime variable — not accessible at runtime
comptime var counter: u32 = 0;
comptime counter += 1;

// Compile-time computation in function
fn factorial(comptime n: u32) u32 {
    var result: u32 = 1;
    for (1..n + 1) |i| result *= i;
    return result;
}
const f10 = comptime factorial(10);  // computed at compile time
```

### Type Reflection

```zig
// @typeInfo — get detailed type information
const S = struct { x: i32, y: f64 };
comptime {
    const info = @typeInfo(S);
    // info is std.builtin.Type{ .Struct = ... }
    for (info.Struct.fields) |field| {
        _ = field.name;   // "x", "y"
        _ = field.type;   // i32, f64
    }
}

// Type-info for enums
const E = enum { a, b, c };
comptime {
    const info = @typeInfo(E);
    _ = info.Enum.fields.len;  // 3
}

// @TypeOf — get type of any expression
const v: i32 = 42;
const T = @TypeOf(v);          // i32
const U = @TypeOf(v + 1.0);    // f64 (coerced)

// @typeName — string name of type
const name = @typeName(i32);   // "i32"

// @tagName — string name of enum field
const ename = @tagName(E.a);   // "a"

// @hasDecl — check if type has declaration
comptime {
    if (@hasDecl(S, "x")) { _ = @compileLog("S has x"); }
}

// @hasField — check if struct has field
comptime {
    if (@hasField(S, "y")) { _ = @compileLog("S has y"); }
}
```

### Comptime Strings

```zig
// Comptime string concatenation with ++
comptime {
    const msg = "Hello " ++ "World";  // "Hello World"
    // NOTE: must use consistent whitespace around ++
}

// @embedFile — embed file contents at compile time
const shader_source = @embedFile("shader.glsl");
```

### Comptime Diagnostics

```zig
// @compileLog — print values at compile time
comptime {
    @compileLog("sizeof(i32) = ", @sizeOf(i32));
}

// @compileError — intentional compile error with message
fn assert(ok: bool) void {
    if (!ok) @compileError("assertion failed");
}

// @setEvalBranchQuota — increase comptime loop limit
comptime {
    @setEvalBranchQuota(10000);
    for (0..5000) |i| { _ = i; }
}
```

### Generic Types via Comptime

```zig
// Generic struct
fn Vec(comptime T: type, comptime N: usize) type {
    return struct {
        data: [N]T,

        pub fn len(self: @This()) usize {
            return self.data.len;
        }
    };
}

const Vec3f = Vec(f32, 3);
var v = Vec3f{ .data = .{ 1.0, 2.0, 3.0 } };
_ = v.len();  // 3

// @This() — refers to innermost type being defined
const Node = struct {
    next: ?*@This() = null,
    value: i32,
};
```

---

## 5. Error Handling

### Error Sets

```zig
// Named error set
const MyError = error{
    OutOfMemory,
    InvalidInput,
    NotFound,
};

// Merged error sets — inferred at compile time
fn a() error{A, B}!void { return error.A; }
fn b() error{B, C}!void { return error.C; }
// fn c() !void — inferred error set is {A, B, C}

// Empty error set — never fails
const NeverError = error{};

// Global error set (anyerror)
fn catchAll() anyerror!void {}
```

### `try` / `catch`

```zig
// try — propagate error upward
fn doStuff() !void {
    try mightFail1();
    try mightFail2();  // only reached if mightFail1 succeeds
}

// catch — handle error, provide fallback
fn getOrDefault() i32 {
    const val = mightFail() catch 0;
    return val;
}

// catch with error capture
fn handle() void {
    const result = mightFail() catch |err| {
        switch (err) {
            error.InvalidInput => return,
            else => {},
        }
    };
    _ = result;
}
```

### `errdefer`

```zig
fn allocateAndWork(alloc: std.mem.Allocator) ![]u8 {
    const buf = try alloc.alloc(u8, 1024);
    errdefer alloc.free(buf);   // free on error
    try doWork(buf);             // if this fails, errdefer runs
    return buf;                  // success — errdefer does NOT run
}

fn doWork(buf: []u8) !void { _ = buf; }
fn mightFail1() !void { return error.A; }
fn mightFail2() !void { return error.B; }
const std = @import("std");

// Key difference from defer:
// defer always runs (success or error)
// errdefer only runs on error
```

### Error Union & Optional Combined

```zig
// !?T — can be error, null, or value
fn find() !?i32 {
    if (some_condition()) return null;
    return 42;
}

const result = find() catch null;  // flatten to ?i32
```

### Error Return Trace

```zig
// Error return traces show the path errors propagated
// Enabled in Debug/ReleaseSafe, disabled in ReleaseFast/ReleaseSmall
fn a() !void { try b(); }
fn b() !void { try c(); }
fn c() !void { return error.Bottom; }

// @errorReturnTrace — access the stack trace
pub fn main() !void {
    try a();
}
```

---

## 6. Memory Management

### The Allocator Interface

```zig
// Allocator — vtable-based interface
const alloc: std.mem.Allocator = std.heap.page_allocator;

// Core methods in Allocator:
// alloc(T, n)          — allocate n items of type T, return []T
// free(slice)          — free slice
// create(T)            — allocate single T, return *T
// destroy(ptr)         — free single T
// allocSentinel(T, n, s) — allocate with sentinel
// realloc(old, new_n)  — grow/shrink allocation
// dupe(T, m, value)    — duplicate a value

const slice = try alloc.alloc(u8, 100);
defer alloc.free(slice);

const ptr = try alloc.create(u32);
defer alloc.destroy(ptr);
ptr.* = 42;

const duped = try alloc.dupe(u8, "hello");  // owned copy
defer alloc.free(duped);
```

### Built-in Allocators

```zig
// Page allocator — directly from OS, no deinit needed
const pa = std.heap.page_allocator;

// GeneralPurposeAllocator — recommended for most programs
// Detects leaks and double-frees in debug modes
var gpa = std.heap.GeneralPurposeAllocator(.{}){};
defer _ = gpa.deinit();
const gpa_alloc = gpa.allocator();

// Arena allocator — free all at once, no individual frees
var arena = std.heap.ArenaAllocator.init(std.heap.page_allocator);
defer arena.deinit();
const arena_alloc = arena.allocator();
// You can omit individual frees; arena.deinit() frees everything

// FixedBufferAllocator — from a stack buffer, no heap
var buf: [4096]u8 = undefined;
var fba = std.heap.FixedBufferAllocator.init(&buf);
const fba_alloc = fba.allocator();

// Testing allocator — detects leaks in tests
const test_alloc = std.testing.allocator;

// C allocator — wraps malloc/free
const c_alloc = std.heap.c_allocator;

// SMP allocator — thread-safe wrapper
// var smp = std.heap.smpAllocator(gpa_alloc);
```

### `ArrayList` (v0.15.1)

```zig
// Initialization — use .empty, NOT .init(allocator)
var list: std.ArrayList(u8) = .empty;
defer list.deinit(alloc);

// append takes allocator as first argument
try list.append(alloc, 42);
try list.append(alloc, 99);

// Access
_ = list.items;        // slice of all items
_ = list.items[0];     // 42
_ = list.items.len;    // 2
_ = list.getLast();    // 99

// Pop
const last = list.pop();  // 99

// Insert
try list.insert(alloc, 0, 100);

// Append slice
try list.appendSlice(alloc, &.{ 1, 2, 3 });

// Capacity
try list.ensureTotalCapacity(alloc, 32);

// Writer
const writer = list.writer(alloc);
try writer.print("Hello {s}", .{"World"});
```

### Ownership Patterns

```zig
// Pattern 1: Accept allocator, return owned slice
fn makeString(alloc: std.mem.Allocator) ![]const u8 {
    return try std.fmt.allocPrint(alloc, "{d}", .{42});
}

// Pattern 2: Caller owns, takes buffer
fn fillBuffer(buf: []u8) usize {
    const text = "hello";
    @memcpy(buf[0..text.len], text);
    return text.len;
}

// Pattern 3: Arena for short-lived allocations
fn process() !void {
    var arena = std.heap.ArenaAllocator.init(std.heap.page_allocator);
    defer arena.deinit();
    const temp = try arena.allocator().alloc(u8, 1024);
    _ = temp;
    // no need to free temp individually
}
```

---

## 7. Build System

### `build.zig` Structure

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    // Target and optimization
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // Executable from a single file
    const exe = b.addExecutable(.{
        .name = "myapp",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    b.installArtifact(exe);

    // Run step
    const run_cmd = b.addRunArtifact(exe);
    run_cmd.step.dependOn(b.getInstallStep());
    if (b.args) |args| run_cmd.addArgs(args);
    const run_step = b.step("run", "Run the app");
    run_step.dependOn(&run_cmd.step);

    // Tests
    const exe_tests = b.addTest(.{
        .root_module = b.createModule(.{
            .root_source_file = b.path("src/main.zig"),
            .target = target,
            .optimize = optimize,
        }),
    });
    const test_step = b.step("test", "Run tests");
    test_step.dependOn(&b.addRunArtifact(exe_tests).step);
}
```

### Modules & Imports (v0.15.1)

```zig
// Creating a module
const my_mod = b.createModule(.{
    .root_source_file = b.path("src/my_mod.zig"),
    .target = target,
    .optimize = optimize,
    .imports = &.{
        .{ .name = "other", .module = other_mod },
    },
});

// Adding module as import to executable
const exe = b.addExecutable(.{
    .name = "myapp",
    .root_module = b.createModule(.{
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
        .imports = &.{
            .{ .name = "my_mod", .module = my_mod },
        },
    }),
});

// Adding to an existing module
my_mod.addImport("another", another_mod);

// Top-level package module
_ = b.addModule("pkg_name", .{
    .root_source_file = b.path("src/root.zig"),
    .target = target,
});
```

### Library

```zig
// Shared library
const lib = b.addSharedLibrary(.{
    .name = "mylib",
    .root_source_file = b.path("src/lib.zig"),
    .target = target,
    .optimize = optimize,
});
b.installArtifact(lib);

// Static library
const static_lib = b.addStaticLibrary(.{ ... });
b.installArtifact(static_lib);
```

### `build.zig.zon` (Package Manifest)

```zig
.{
    .name = .myapp,
    .version = "0.1.0",
    .fingerprint = 0x...,
    .minimum_zig_version = "0.15.1",
    .dependencies = .{
        .somelib = .{
            .url = "https://example.com/somelib.tar.gz",
            .hash = "1220...",
        },
        .locallib = .{
            .path = "libs/locallib",
        },
    },
    .paths = .{
        "build.zig",
        "build.zig.zon",
        "src",
    },
}
```

### Common Build Patterns

```zig
// Compile-time options via b.options
const options = b.addOptions();
options.addOption(bool, "enable_feature", true);

exe.root_module.addOptions("config", options);

// In source: const config = @import("config");
// const features: bool = config.enable_feature;

// Conditional compilation
const target = b.standardTargetOptions(.{});
exe.root_module.target = target;  // accessible via builtin

// Install additional files
const install_step = b.addInstallDirectory(.{
    .source_dir = b.path("assets"),
    .install_dir = .{ .custom = "share/myapp" },
    .install_subdir = "",
});
b.getInstallStep().dependOn(&install_step.step);

// Lazy dependencies
// .dependencies = .{ .foo = .{ .path = "lib/foo", .lazy = true } }
```

---

## 8. Standard Library Highlights

### `std.fmt` — Formatting

```zig
// bufPrint — format into a stack buffer (returns slice of buf)
var buf: [256]u8 = undefined;
const s = try std.fmt.bufPrint(&buf, "{s}: {d}", .{ "count", 42 });
// s is a slice of buf

// allocPrint — format into a heap-allocated string (must free)
const heap_str = try std.fmt.allocPrint(alloc, "{s} {d}", .{ "age", 30 });
defer alloc.free(heap_str);

// Format specifiers
// {s}  — string ([]const u8 or [:0]const u8)
// {d}  — decimal integer
// {d:.2} — float with 2 decimal places
// {x}  — hex
// {b}  — binary
// {any} — default format for any type
// {?}  — pointer address
// {}   — default formatting for the type

// parseInt / parseFloat
const n = try std.fmt.parseInt(i32, "42", 10);  // base 10
const f = try std.fmt.parseFloat(f64, "3.14");
```

### `std.mem` — Memory Operations

```zig
// Comparison
_ = std.mem.eql(u8, "abc", "abc");            // true (exact)
_ = std.mem.startsWith(u8, "hello", "he");     // true
_ = std.mem.endsWith(u8, "hello", "lo");       // true

// Search
_ = std.mem.indexOfScalar(u8, "hello", 'e');    // ?usize = 1
_ = std.mem.indexOf(u8, "hello world", "lo");   // ?usize = 3
_ = std.mem.lastIndexOf(u8, "ababa", "ba");     // ?usize = 3
_ = std.mem.containsAtLeast(u8, "hello", 1, "l"); // true

// Trim & split
_ = std.mem.trim(u8, "  hello  ", " ");         // "hello"
_ = std.mem.trimLeft(u8, "  hello  ", " ");     // "hello  "
_ = std.mem.trimRight(u8, "  hello  ", " ");    // "  hello"

var iter = std.mem.splitSequence(u8, "a,b,c", ",");
while (iter.next()) |part| { _ = part; }         // "a", "b", "c"

var iter2 = std.mem.splitScalar(u8, "a b c", ' ');
while (iter2.next()) |part| { _ = part; }

var iter3 = std.mem.tokenize(u8, "  a  b  ", " ");
while (iter3.next()) |part| { _ = part; }       // "a", "b"

// Slice to sentinel (null terminator)
const c_ptr: [*:0]const u8 = "hello";
const slice: []const u8 = std.mem.sliceTo(c_ptr, 0);  // "hello"

// Counting
_ = std.mem.count(u8, "hello", "l");  // 2
_ = std.mem.replacementSize(u8, "hello", "l", "x");  // 5

// Zero init
var data: [100]u8 = undefined;
@memset(&data, 0);

// Copy
@memcpy(dest[0..src.len], src);

// Alignment
_ = std.mem.alignForward(usize, 7, 4);  // 8

// Valid UTF-8 check
_ = std.unicode.utf8ValidateSlice("hello");  // true
```

### `std.fs` — File System

```zig
// Open file (absolute path)
const file = try std.fs.openFileAbsolute("/etc/hostname", .{});
defer file.close();
const contents = try file.readToEndAlloc(alloc, 1024 * 1024);
defer alloc.free(contents);

// Open file (relative to CWD)
const cwd = try std.fs.cwd().openFile("data.txt", .{});
defer cwd.close();

// Read all into existing buffer
var buf: [1024]u8 = undefined;
const bytes_read = try file.readAll(&buf);
const data = buf[0..bytes_read];

// Write file
const out = try std.fs.cwd().createFile("out.txt", .{});
defer out.close();
try out.writeAll("Hello, world!");

// File operations
_ = try file.getEndPos();  // file size
try file.seekTo(0);        // rewind

// Directory listings
var dir = try std.fs.cwd().openDir("src", .{ .iterate = true });
defer dir.close();
var walker = dir.iterate();
while (try walker.next()) |entry| {
    _ = entry.name;
    _ = entry.kind;  // .file, .directory, etc.
}

// Path manipulation
_ = std.fs.path.basename("/usr/bin/bash");      // "bash"
_ = std.fs.path.dirname("/usr/bin/bash");        // "/usr/bin"
_ = std.fs.path.extension("file.txt");           // ".txt"
_ = std.fs.path.stem("file.txt");                // "file"
_ = std.fs.path.join(alloc, &.{ "/usr", "bin" }); // "/usr/bin"
```

### `std.process` — Process & Environment

```zig
// Run external command
const result = try std.process.Child.run(.{
    .allocator = alloc,
    .argv = &.{ "uname", "-r" },
});
defer alloc.free(result.stdout);
defer alloc.free(result.stderr);
_ = result.stdout;  // command output
_ = result.term;    // exit status

// Env vars (returns ?[*:0]u8 — use sliceTo for []const u8)
if (std.posix.getenv("HOME")) |home| {
    const home_str: []const u8 = std.mem.sliceTo(home, 0);
    _ = home_str;
}

// Current directory
var cwd_buf: [std.posix.PATH_MAX]u8 = undefined;
const cwd = std.posix.getcwd(&cwd_buf);
_ = cwd;  // ?[]const u8 (from cwd_buf)

// Command-line arguments
const args = try std.process.argsAlloc(alloc);
defer std.process.argsFree(alloc, args);
_ = args;  // []const []const u8
```

### `std.posix` — POSIX System Calls

```zig
// uname — returns a struct (v0.15.1: returns value, not takes pointer)
const uts = std.posix.uname();
const sysname = std.mem.sliceTo(&uts.sysname, 0);    // "Linux"
const release = std.mem.sliceTo(&uts.release, 0);    // "6.8.0-..."
const machine = std.mem.sliceTo(&uts.machine, 0);    // "x86_64"
const nodename = std.mem.sliceTo(&uts.nodename, 0);  // hostname

// Environment variable (returns ?[*:0]u8)
if (std.posix.getenv("PATH")) |path| {
    const path_str = std.mem.sliceTo(path, 0);
    _ = path_str;
}

// Current working directory
var buf: [std.posix.PATH_MAX]u8 = undefined;
const cwd = std.posix.getcwd(&buf);  // ?[]const u8

// PATH_MAX constant
_ = std.posix.PATH_MAX;  // typically 4096
```

### `std.builtin` — Compile-time Constants (via `@import("builtin")`)

```zig
const builtin = @import("builtin");

_ = builtin.os.tag;           // .linux, .macos, .windows, ...
_ = builtin.cpu.arch;         // .x86_64, .aarch64, ...
_ = builtin.mode;             // .Debug, .ReleaseSafe, .ReleaseFast, .ReleaseSmall
_ = builtin.target;           // full target info
_ = builtin.zig_version;      // .{ .major = 0, .minor = 15, .patch = 1 }
_ = builtin.link_libc;        // true if linking libc
_ = @tagName(builtin.cpu.arch);  // "x86_64" (string for display)
```

---

## 9. Testing

### Test Blocks

```zig
// Any test block in any file is discovered
test "basic arithmetic" {
    try std.testing.expect(1 + 1 == 2);
}

test "expect equal" {
    try std.testing.expectEqual(@as(i32, 42), 42);
}

test "expect strings" {
    try std.testing.expectEqualStrings("hello", "hello");
}

test "expect error" {
    try std.testing.expectError(error.InvalidInput, mightFail());
}

test "expect approx" {
    try std.testing.expectApproxEqRel(1.0, 1.0001, 0.01);
}

test "allocator in test" {
    const alloc = std.testing.allocator;  // detecting leaks
    const buf = try alloc.alloc(u8, 10);
    defer alloc.free(buf);
}

// Reference declarations from the same file
test "refAllDecls" {
    std.testing.refAllDecls(@This());
}
```

### Run Tests

```bash
zig build test           # run all tests via build.zig
zig test src/main.zig    # run tests in a single file
```

---

## 10. Builtin Functions Reference

### Type Conversion

| Builtin | Signature | Purpose |
|---------|-----------|---------|
| `@as(T, value)` | `fn (type, anytype) T` | Safe type coercion (no-op at runtime) |
| `@intCast(value)` | `fn (anytype) T` | Integer-to-integer (same sign) |
| `@floatCast(value)` | `fn (anytype) T` | Float-to-float |
| `@floatFromInt(value)` | `fn (anytype) T` | Integer to float |
| `@intFromFloat(value)` | `fn (anytype) T` | Float to integer (truncates) |
| `@truncate(value)` | `fn (anytype) T` | Narrow integer (discard high bits) |
| `@bitCast(value)` | `fn (anytype) T` | Reinterpret bits (same size) |
| `@ptrCast(value)` | `fn (anytype) T` | Pointer type conversion |
| `@ptrFromInt(value)` | `fn (usize) *T` | Integer to pointer |
| `@intFromPtr(value)` | `fn (*T) usize` | Pointer to integer |
| `@enumFromInt(value)` | `fn (anytype) T` | Integer to enum |

### Type Inspection

| Builtin | Purpose |
|---------|---------|
| `@TypeOf(expr)` | Returns the type of an expression |
| `@typeInfo(T)` | Returns `std.builtin.Type` union |
| `@typeName(T)` | String name of type (e.g., "i32") |
| `@tagName(enum_field)` | String name of enum field |
| `@sizeOf(T)` | Byte size of a type |
| `@alignOf(T)` | Alignment of a type |
| `@bitSizeOf(T)` | Bit size of a type |
| `@hasDecl(T, name)` | Check if type has declaration |
| `@hasField(T, name)` | Check if struct/union has field |
| `@field(L, field_name)` | Access struct field by comptime string |
| `@This()` | Type of innermost struct/enum/union |
| `@Vector(N, T)` | SIMD vector type |

### Control

| Builtin | Purpose |
|---------|---------|
| `@call(modifier, func, args)` | Call function with calling convention |
| `@compileError(msg)` | Emit compile error |
| `@compileLog(values...)` | Print values at compile time |
| `@setEvalBranchQuota(n)` | Increase comptime loop limit |
| `@select(a, b, pred)` | Conditional select (like ternary for SIMD) |
| `@branchHint(prob)` | Hint branch probability |
| `@panic(msg)` | Panic with message |
| `@trap()` | Platform-specific trap (crash) |
| `@breakpoint()` | Platform debug breakpoint |

### Memory & Data

| Builtin | Purpose |
|---------|---------|
| `@memcpy(dest, src)` | Copy bytes (no overlap) |
| `@memset(dest, value)` | Fill bytes with value |
| `@memcpy(dest_noalias, src_noalias)` | Copy non-overlapping |
| `@atomicLoad(ptr, order)` | Atomic load |
| `@atomicStore(ptr, value, order)` | Atomic store |
| `@cmpxchgStrong/Weak(ptr, expected, new, succ, fail)` | CAS |
| `@fence(order)` | Memory fence |

### Misc

| Builtin | Purpose |
|---------|---------|
| `@src()` | Source location (file, line, column) |
| `@returnAddress()` | Return address of current function |
| `@frameAddress()` | Frame address |
| `@embedFile(path)` | Embed file as comptime string |
| `@cImport(c_code)` | Parse C headers |
| `@cInclude(header)` | Include C header |
| `@export(target, linkage)` | Export symbol |
| `@extern(T, options)` | Reference external symbol |
| `@errorReturnTrace()` | Get error return trace |
| `@errorName(err)` | String name of error |
| `@shuffle(T, a, b, mask)` | SIMD vector shuffle |
| `@splat(scalar)` | Broadcast scalar to SIMD vector |
| `@reduce(op, vec)` | Horizontal SIMD reduction |
| `@popCount(value)` | Count set bits |
| `@clz(value)` | Count leading zeroes |
| `@ctz(value)` | Count trailing zeroes |
| `@byteSwap(value)` | Endian swap |
| `@bitReverse(value)` | Reverse bit order |
| `@mulAdd(a, b, c)` | Fused multiply-add |

---

## 11. Common Patterns & Gotchas

### Initialization

```zig
// Default struct/array values: use .{ }
var sys: SystemInfo = .{};                      // all defaults
const arr: [3]u8 = .{ 1, 2, 3 };
const pt = Point{ .x = 1.0, .y = 2.0 };

// With explicit type annotation, omit struct name:
const pt2: Point = .{ .x = 3.0, .y = 4.0 };
// But NOT: const pt2 = .{ .x = 3.0, .y = 4.0 };  // anonymous!
```

### Stack Buffers vs Heap

```zig
// Stack buffer — fast, fixed size, no allocator
var buf: [256]u8 = undefined;
const formatted = try std.fmt.bufPrint(&buf, "{d}", .{42});

// Heap buffer — dynamic size, requires allocator
const heap_buf = try allocator.alloc(u8, n);
defer allocator.free(heap_buf);
```

### Optional Unwrapping Patterns

```zig
// Default value
const val = optional orelse defaultValue;

// Propagate null (must return optional or sendinel error)
const val = optional orelse return null;
const val = optional orelse return error.Missing;

// Execute side effect on null
const val = optional orelse {
    // handle null
    return defaultValue;
};

// Chain with and
if (a != null and b != null) {
    const x = a.?;
    const y = b.?;
}
// Better:
if (a) |x| {
    if (b) |y| {
        // use x and y
    }
}
```

### Error Propagation Patterns

```zig
// Propagate with try
const result = try mightFail();

// Catch and convert error
const result = mightFail() catch |err| {
    return error.Wrapped;
};

// Catch and log
const result = mightFail() catch |err| {
    std.debug.print("Failed: {!}\n", .{err});
    return error.Wrapped;
};

// Ignore error (discouraged but sometimes needed)
mightFail() catch {};
```

### `errdefer` Gotcha

```zig
fn example(alloc: std.mem.Allocator) !void {
    var a = try alloc.alloc(u8, 100);
    errdefer alloc.free(a);
    var b = try alloc.alloc(u8, 200);
    errdefer alloc.free(b);  // this runs BEFORE the first errdefer
    // errdefers run in LIFO order on error path
    try mightFail();  // if fails: free b, then free a
}
```

### Comptime String `++` Gotcha

```zig
// OK: consistent whitespace
_ = "Hello " ++ "World";

// Compile error — inconsistent spacing
// _ = "Hello "++"World";
// _ = "Hello " ++"World";

// Must use consistent whitespace around ++ for comptime strings
```

### `builtin.cpu.arch` not `std.Target.current.cpu.arch`

```zig
const builtin = @import("builtin");
const arch = builtin.cpu.arch;  // correct in 0.15.1
// Do NOT use: std.Target.current.cpu.arch
```

### ArrayList v0.15.1 Changes

```zig
// 0.14: var list = std.ArrayList(u8).init(allocator);
// 0.15: var list: std.ArrayList(u8) = .empty;

// 0.14: try list.append(42);
// 0.15: try list.append(allocator, 42);

// 0.14: list.deinit();
// 0.15: list.deinit(allocator);
```

### `uname()` v0.15.1

```zig
// 0.14: var uts: std.posix.utsname = undefined; std.posix.uname(&uts);
// 0.15: const uts = std.posix.uname();  // returns by value
```

### `getenv()` Returns `?[*:0]u8`

```zig
// Must convert to []const u8 with sliceTo
if (std.posix.getenv("HOME")) |raw| {
    const home: []const u8 = std.mem.sliceTo(raw, 0);
    // use home
}
```

### Writer Pattern (v0.15.1)

```zig
// For stdout
var out_buf: [8192]u8 = undefined;
var out = std.fs.File.stdout().writer(&out_buf);
try out.interface.writeAll("Hello");
try std.Io.Writer.flush(&out.interface);

// For ArrayList
var list: std.ArrayList(u8) = .empty;
defer list.deinit(alloc);
const writer = list.writer(alloc);
try writer.print("{s}: {d}\n", .{ "count", 42 });
```

### `packed struct` for Bit Flags

```zig
const Flags = packed struct(u8) {
    read: bool = false,
    write: bool = false,
    execute: bool = false,
    _: u5 = 0,
};
// sizeof = 1 byte
// @bitCast to u8 gives numeric representation
```

### `for` over Ranges (v0.15.1)

```zig
// 0.14: var i: usize = 0; while (i < n) : (i += 1) {}
// 0.15: for (0..n) |i| {}  // preferred
```

### `defer` with Error Returns

```zig
// defer can't capture error returns — use errdefer instead
fn writeFile() !void {
    const file = try std.fs.cwd().createFile("out.txt", .{});
    defer file.close();    // always close (even on error after this)
    errdefer deleteFile(); // only if we return error
    try doWork(file);
}
```

---

## Quick Reference Card

### File Template

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // Your code here...
    std.debug.print("Hello, {s}!\n", .{"World"});
}
```

### Common Imports

```zig
const std = @import("std");
const builtin = @import("builtin");
const Allocator = std.mem.Allocator;
```

### Print Debugging

```zig
std.debug.print("int: {d}, str: {s}, any: {any}\n", .{ 42, "hi", some_struct });
```

### TODO / Unreachable

```zig
_ = try unrealized();  // error: TODO implement
unreachable;          // marks unreachable code
```

### Namespacing

```zig
// Files are structs, pub items are accessible
const math = @import("math.zig");
_ = math.add(1, 2);

// Using declarations (namespacing using namespace)
const w = struct {
    usingnamespace @import("other.zig");
};
```

### Naming Conventions

```
camelCase       — local variables, function parameters
snake_case      — function names, type names (structs, enums)
PascalCase      — only for type names that have methods
SCREAMING_CASE  — constants
```
