# Google protobuf support for Lua

[![Build Status](https://travis-ci.org/starwing/lua-protobuf.svg?branch=master)](https://travis-ci.org/starwing/lua-protobuf)[![Coverage Status](https://coveralls.io/repos/github/starwing/lua-protobuf/badge.svg?branch=master)](https://coveralls.io/github/starwing/lua-protobuf?branch=master)

中文使用说明：https://zhuanlan.zhihu.com/p/26014103

This project offers a simple C library for basic protobuf wire format encode/decode, it splits to several modules:
  - `pb.slice`: a wire format decode module.
  - `pb.buffer`: a buffer implement that use to encode basic types into protobuf's wire format.  It also can be used to support streaming decode protobuf data.
  - `pb.conv`: a module to convert between integers in protobuf.
  - `pb.io`: a module to support binary mode read/write to stdin/stdout.

It also has a high level Lua module to direct decode protobuf data to Lua table, or encode Lua table to protobuf data.

To support new protobuf type, you can use pb.load()/pb.loadfile() to load a compiled protobuf schema file, generated by Google protobuf's protoc compiler.  And if you don't want include google's huge library, it includes a pure Lua module named protoc to translate text protobuf schema file into *.pb format.

After load schema into `pb` module, you could use `pb.encode()` or `pb.decode()` routines to produce protobuf data.

## Install

To install this Lua module, you could just use `luarocks`:

```shell
luarocks install lua-protobuf
```

If you want build it from source, just clone repo and use luarocks:

```shell
git clone https://github.com/starwing/lua-protobuf
luarocks make rockspecs/lua-protobuf-scm-1.rockspec
```

or you can build it by hand, it only have a pure Lua module and a pair of C file: pb.h and pb.c

to build it on macOS, use your favor compiler:

```shell
gcc -O2 -shared -undefined dynamic_lookup pb.c -o pb.so
```

on Linux, use the nearly same command:

```shell
gcc -O2 -shared -fPIC pb.c -o pb.so
```

on Windows, you could use MinGW or MSVC, create a sln project or build it on command line:

```shell
cl /O2 /LD /Fepb.dll /I Lua53\include /DLUA_BUILD_AS_DLL pb.c Lua53\lib\lua53.lib
```

## Example

```lua
local pb = require "pb"
local protoc = require "protoc"

assert(protoc:load [[
   message Phone {
      optional string name        = 1;
      optional int64  phonenumber = 2;
   }
   message Person {
      optional string name     = 1;
      optional int32  age      = 2;
      optional string address  = 3;
      repeated Phone  contacts = 4;
   } ]])

local data = {
   name = "ilse",
   age  = 18,
   contacts = {
      { name = "alice", phonenumber = 12312341234 },
      { name = "bob",   phonenumber = 45645674567 }
   }
}

local bytes = assert(pb.encode("Person", data))
print(pb.tohex(bytes))

local data2 = assert(pb.decode("Person", bytes))
print(require "serpent".block(data2))

```

## License

It has the same license as the [Lua license](https://www.lua.org/license.html).

## Usage

### `protoc` Module

| Function                | Returns       | Descriptions                             |
| ----------------------- | ------------- | ---------------------------------------- |
| `protoc.new()`          | Proroc object | create a new compiler instance           |
| `protoc.reload()`       | true          | reload all google standard messages into `pb` module |
| `p:parse(string)`       | table         | transform schema to `DescriptorProto` table |
| `p:parsefile(string)`   | table         | like `p:parse()`, but accept filename    |
| `p:compile(string)`     | string        | transform schema to binary *.pb format data |
| `p:compilefile(string)` | string        | like `p:compile()`, but accept filename  |
| `p:load(string)`        | true          | load schema into `pb` module             |
| `p:loadfile(string)`    | true          | like `pb:loadfile()`, but accept filename |
| `p.loaded`              | table         | contains all parsed `DescriptorProto` table |
| `p.paths`               | table         | a table contains import search directories |
| `p.unknown_module`      | see below     | handle schema import error               |
| `p.unknown_type`        | see below     | handle unknown type in schema            |

To parse a text schema file, you should create a compiler instance first:

```lua
local p = protoc.new()
```

Then, you can set some options to compiler, e.g. the search path, the unknown handlers, etc.

```lua
p.paths[#p.paths+1] = "whatever/folder/hold/.proto/files"
p.unknown_module = function(self, module_name) ... end
p.unknown_type = function(self, type_name) ... end
```

The `unknwon_module` and `unknown_type` handle could be `true`, string or a function.  If set it to `true`, means all non-exists module or type are given a default value and do not trigger a error.  If set it to a string, that string will be a Lua pattern that indicate whether a unknown module or type should produce a error, e.g.

```lua
p.unknown_type = "Foo.*"
```

means all Types prefixed by Foo will treat as exists type and not trigger errors.

If these handlers are set to functions, the unknown type or module name will passed to function, for module handler, it should return a `DescriptorProto` Table produced by `p:load[file]()` functions, for type handler, it should return a type name and type, such as `message` or `enum`, e.g.

```lua
function p:unknown_module(name)
  -- if can not find "foo.proto", load "my_foo.proto" instead
  return p:load("my_"..name)
end

function p:unknown_type(name)
  -- if can not find "Type", treat it as ".MyType" and is a message type
  return ".My"..name, "message"
end
```

After setting options, use `load[file]()` or `compile[file]()` or `parse[file]()` function to get result.

### `pb` Module

`pb` module have the high level routines to maniplate protobuf messages.

in below functions, we have several types that has special means:

- `type`: a string that indicate a protobuf message type, `".Foo"` means a type in a proto file that have not `package` statement declared.  `"foo.Foo"` means a type in a proto file that declared `package foo;`

- `data`: could be string, `pb.Slice` value or `pb.Buffer` value.

- `iterator`: a function that can used in Lua `for in` statement, e.g.

  ```lua
  for name in pb.types() do
    print(name)
  end
  ```


all functions returns `nil, errmsg` when meet errors.

| Function                    | Returns   | Description                              |
| --------------------------- | --------- | ---------------------------------------- |
| `pb.clear()`                | None      | clear all types                          |
| `pb.clear(type)`            | None      | delete specific type                     |
| `pb.load(data)`             | true      | load a binary schema data into `pb` module |
| `pb.loadfile(string)`       | true      | same as `pb.load()`, but accept file name |
| `pb.encode(type, table)`    | string    | encode a message table into binary form  |
| `pb.decode(type, data)`     | table     | decode a binary message into Lua tabl    |
| `pb.pack(fmt, ...)`         | string    | same as `buffer.pack()` but return string |
| `pb.unpack(data, fmt, ...)` | values... | same as `slice.unpack()` but accept data |
| `pb.types()`                | iterator  | iterate all types in `pb` module         |
| `pb.type(type)`             | see below | return informations for specific type    |
| `pb.fields(type)`           | iterator  | iterate all fields in a message          |
| `pb.field(type, string)     | see below | return informations for specific field of type |
| `pb.enum(type, string)`     | number    | get the value of a enum by name          |
| `pb.enum(type, number)`     | string    | get the name of a enum by value          |

You can use `pb.(type|field)[s]()` functions to retrieve type informations for loaded messages.  

`pb.type()` returns multiple informations for specified type:

- name : the full qualitier name of type, e.g. ".package.TypeName"
- basename: the type name without package prefix, e.g. "TypeName"
- "map" | "enum" | "message": whether the type is a map_entry type, enum type or message type.

`pb.types()` returns a iterators, behavior like call `pb.type()` on every types of all messages.

```lua
print(pb.type "MyType")

-- list all types that loaded into pb
for name, basename, type in pb.types() do
  print(name, basename, type)
end
```

`pb.field()` returns informations of the specified field for one type:

- name: the name of field
- number: number of field in schema
- type: field type
- default value: if no default value, nil
- "packed"|"repeated"| "optional": label of field, optional or repeated, required is not supported
- [oneof_name, oneof_index] : if this is a oneof field, this is the oneof name and index

And `pb.fields()` iterates all fields in a message:

```lua
print(pb.field("MyType", "the_first_field"))

-- notice that you needn't receive all return values from iterator
for name, number, type in pb.fields "MyType" do
  print(name, number, type)
end
```

`pb.enum()` maps from enum name and value:

```lua
protoc:load [[
enum Color { Red = 1; Green = 2; Blue = 3 }
]]
print(pb.enum("Color", "Red")) --> 1
print(pb.enum("Color", 2)) --> "Green"
```



### `pb.io` Module

`pb.io` module are used to read binary data from file or `stdin`/`stdout`, `pb.io.read()` is used to read binary data from file, if no file name given as the first parameter, it reads binary data from `stdin`.

`pb.io.write()` and `pb.io.dump()` all same as Lua's `io.write()` but they write binary data.  The first one write data to `stdout`, and latter write data to a file, specific by the first parameter, the file name.

All these functions return true value when success, and return `nil, errmsg` when error occurs.

| Function                | Returns | Description                         |
| ----------------------- | ------- | ----------------------------------- |
| `io.read()`             | string  | read all binary data from `stdin`   |
| `io.read(string)`       | string  | read all binary data from file name |
| `io.write(string, ...)` | string  | write binary data to file name      |
| `io.dump(...)`          | true    | write binary data to `stdout`       |



### `pb.conv` Module

`pb.conv` have functions to convert between numbers.

| Encode Function        | Decode Function        |
| ---------------------- | ---------------------- |
| `conv.encode_int32()`  | `conv.decode_int32()`  |
| `conv.encode_uint32()` | `conv.decode_uint32()` |
| `conv.encode_sint32()` | `conv.decode_sint32()` |
| `conv.encode_sint64()` | `conv.decode_sint64()` |
| `conv.encode_float()`  | `conv.decode_float()`  |
| `conv.encode_double()` | `conv.decode_double()` |



### `pb.slice` Module

Slice module used to parse binary protobuf data in low level way.  Use `slice.new()` to create a slice object, you can specific offset `i` and `j` to just access a sub part of the original data (named a view).

A slice object itself has a stack, use `s:enter(i, j)` to save current position and enter next level which the view is specific by the offset `i` and `j`, just as `slice.new()`.  Use `s:leave()` to restore previous view.  You can use `s:level()` to get the current level, and use `s:level(n)` to get the current position, the start and the end position informations of the `n`th level.  You can also use `s:enter()` without parameter to read a length delimited type value from slice and enter the view in readed value.  To get the count of bytes remains in current view to read, use `#s`.

To read values from slice, use `slice.unpack()`, it use a format string to control how to read into a slice (same format character are also used in `buffer.pack()`):

| Format | Description                              |
| ------ | ---------------------------------------- |
| v      | variable Int value                       |
| d      | 4 bytes fixed32 value                    |
| q      | 8 bytes fixed64 value                    |
| s      | length delimited value, usually a `string`, `bytes` or `message` in protobuf. |
| c      | receive a extra number parameter `count` after the format, and reads `count` bytes in slice. |
| b      | variable int value as a Lua `boolean` value. |
| f      | 4 bytes `fixed32` value as floating point `number` value. |
| F      | 8 bytes `fixed64` value as floating point `number` value. |
| i      | variable int value as signed int value, i.e. `int32` |
| j      | variable int value as zig-zad encoded signed int value, i.e.`sint32` |
| u      | variable int value as unsigned int value, i.e. `uint32` |
| x      | 4 bytes fixed32 value as unsigned fixed32 value, i.e.`fixed32` |
| y      | 4 bytes fixed32 value as signed fixed32 value, i.e. `sfixed32` |
| I      | variable int value as signed int value, i.e.`int64` |
| J      | variable int value as zig-zad encoded signed int value, i.e. `sint64` |
| U      | variable int value and treat it as `uint64` |
| X      | 8 bytes fixed64 value as unsigned fixed64 value, i.e. `fixed64` |
| Y      | 8 bytes fixed64 value as signed fixed64 value, i.e. `sfixed64` |

And you can use extra format to control the read cursor in one `slice.unpack()` process:

| Format | Description                              |
| ------ | ---------------------------------------- |
| @      | returns current cursor position in slice, related with the begining of current view. |
| *      | set the current cursor position to the extra parameter after format string. |
| +      | set the relate cursor position, i.e. add the extra parameter to current position. |

e.g. If you want to read a varint value twice, you can write it as:

```lua
local v1, v2 = s:unpack("v*v", 1)
-- v: reads a varint value
-- *: receive the second parameter 1 and set it to current cursor position, i.e. restore cursor to the head of view
-- v: reads the first varint value again
```



| Function                  | Returns      | Description                              |
| ------------------------- | ------------ | ---------------------------------------- |
| `slice.new(data[,i[,j]])` | Slice object | create a new slice object                |
| `s:delete()`              | none         | same as `s:reset()`, free it's content   |
| `tostring(s)`             | string       | return the string repr of the object     |
| `#s`                      | number       | returns the count of bytes can read in current view |
| `s:reset([...])`          | self         | reset object to another data             |
| `s:level()`               | number       | returns the count of stored state        |
| `s:level(number)`         | p, i, j      | returns the informations of the `n`th stored state |
| `s:enter()`               | self         | reads a bytes value, and enter it's view |
| `s:enter(i[, j])`         | self         | enter a view start at `i` and ends at `j`, includes |
| `s:leave([number])`       | self, n      | leave the number count of level (default 1) and return current level |
| `s:unpack(fmt, ...)`      | values...    | reads values of current view from slice  |



### `pb.buffer` Module

Buffer module used to construct a protobuf data format stream in low level way, it's just a bytes data buffer, you can add values by `buffer.pack()`, and use `buffer.result()` get the encoded raw data, or use `buffer.tohex()` to get human readable hexadigit value of data.

the `buffer.pack()` use the same format syntax with `slice.unpack()`, and support '()' format, if you use a pair of parenthesis, it means the inner value will be encoded as a length delimited value, i.e. a message value encoded format.

e.g.

```lua
b:pack("(vvv)", 1, 2, 3) -- get a bytes value that contains three varint value.
```

parenthesis could be nested.

| Function            | Returns       | Description                              |
| ------------------- | ------------- | ---------------------------------------- |
| `buffer.new([...])` | Buffer object | create a new buffer object, extra args will passed to `b:reset()` |
| `b:delete()`        | none          | same as `b:reset()`, free it's content   |
| `tostring(b)`       | string        | returns the string repr of the object    |
| `#b`                | number        | returns the encoded count of bytes in buffer |
| `b:reset()`         | self          | free buffer content, reset it to a empty buffer |
| `b:reset([...])`    | self          | resets the buffer and set its content as the concat of it's args |
| `b:tohex([i[, j]])` | string        | return the string of hexadigit represent of the data, `i` and `j` are ranges in encoded data, includes. Omit it means the whole range |
| `b:result([i[,j]])` | string        | return the raw data, `i` and `j` are ranges in encoded data, includes,. Omit it means the whole range |
| `b:pack(fmt, ...)`  | self          | encode the values passed to `b:pack()`, use `fmt` to indicate how to encode value |

