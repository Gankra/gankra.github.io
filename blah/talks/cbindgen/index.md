% You Should Use cbindgen

Alexis Beingessner

<img src="icon.png" width="250" style="display:inline; box-shadow:none;"></img>
<img src="rust.png" width="250" style="display:inline; box-shadow:none;"></img>
<img src="firefox-nightly.png" width="250" style="display:inline; box-shadow:none;"></img>

slides: https://github.com/Gankra/tutorial-cbindgen/



# What is cbindgen?

Creates C/C++11 headers for a Rust lib's C APIs

github.com/eqrion/cbindgen/blob/master/docs.md

```
cargo install cbindgen
```




# **Why** is cbindgen?

Webrender!

* Lots of C++ calling into Rust
* Otherwise, too error-prone
* Otherwise, too time-consuming




# What Can cbindgen Do?

Let's see!



# Functions

```rust
#[no_mangle]
pub extern fn double(a: u32,
                     b: &mut u32,
                     c: Option<&mut u32>);
```

Generated:
---------

```cpp
extern "C" {
  void double(uint32_t a,
              uint32_t* b,
              uint32_t* c);
}
```


# Structs/Unions

```rust
#[repr(C)]
pub struct MyStruct { a: u32, b: bool }
```

Generated:
---------

```cpp
struct MyStruct { uint32_t a; bool b; };
```



# Tuple Structs

```rust
#[repr(C)]
pub struct MyTuple(u8, MyStruct);
```

Generated:
---------

```cpp
struct MyTuple { uint8_t _0; MyStruct _1; };
```


# Typedefs

```rust
pub type Weight = f32;
```

Generated:
---------

```cpp
using Weight = float;
```


# Fieldless Enums


```rust
#[repr(u32)]
pub enum MyEnum { A, B, C }
```

Generated:
---------

```cpp
enum class MyEnum: uint32_t { A, B, C };
```



# Fieldful Enums

```rust
#[repr(u32)]
pub enum COptionU32 { Some(u32), None, }
```

Generated:
---------

```cpp
union COptionU32 {
  enum class Tag : uint32_t { Some, None };
  struct Some_Body { Tag tag; uint32_t _0; };

  struct { Tag tag; };
  Some_Body some;
};
```




# Fieldful Enum Conveniences

```cpp
auto val = COptionU32::Some(12);
if (val.IsSome()) {
  printf("%d\n", val.AsSome());
}
```




# Generic Structs (templated)

```rust
#[repr(C)]
pub struct MyGenericStruct<T> {
    vals: [T; 16],
}
```

Generated:
---------

```cpp
template<typename T>
struct MyGenericStruct {
  T vals[16];
};
```





# Generic Structs (instantiated)

```rust
#[no_mangle]
pub extern fn process(input: MyGenericStruct<u8>)
```

Generated:
---------

```c
typedef struct {
  uint8_t vals[16];
} MyGenericStruct_u8;

extern "C" {
  void process(MyGenericStruct_u8 input);
}
```





# Opaque Types

```rust
#[no_mangle]
pub extern fn process(input: &Vec<u32>);
```

Generated:
---------

```c
template<typename T>
struct Vec;

extern "C" {
  void process(Vec<u32>* input);
}
```





# Bitflags

```rust
bitflags! {
  #[repr(C)] pub struct Flags: u8 {
    const A = 1; const B = 2;
} }
```

Generated:
---------

```cpp
struct Flags { uint8_t bits; };
// ... a bunch of operator overloads ...
static const Flags Flags_A = (Flags){ .bits = 1 };
static const Flags Flags_B = (Flags){ .bits = 2 };
```


# Platform-Specific Definitions

```rust
#[cfg(target_os = "freebsd")]
struct MyStruct(f32);
#[cfg(target_os = "macos")]
struct MyStruct(u32, u32);
```

Generated:
----------

```cpp
#if defined(PLATFORM_FREEBSD)
struct MyStruct { float _0; };
#endif
#if defined(PLATFORM_MACOS)
struct MyStruct { uint32_t _0; uint32_t _1; };
#endif
```




# How Does It Work?

Find all:
* `#[no_mangle] pub extern fn` (functions)
* `#[no_mangle] pub static` (globals)
* `pub const` (constants)



# How Does It Work?

Find all types those reference

And that those types reference

And that THOSE types reference...



# How Does It Work?

```rust
#[no_mangle]
pub extern fn process(val: Input) -> bool { ... }

#[repr(C)]
pub struct Input {
  data: Data,
}

#[repr(u32)]
pub enum Data { A, B, C }
```



# How To Use It

* as a lib in build.rs
* as a cli tool (currently better errors)


# Common Pitfalls

* Doesn't respect namespacing
* Can't properly support destructors (but tries)
* Easily falls over if there's `repr(Rust)` types
* Need to opt-in to parsing deps (see `[parse]`)




# Things cbindgen Can't Support

* generic functions (no symbols)
* tuples (no defined repr)
* wide pointers (no defined repr) -- `&[T]`, `&Trait`
* empty top-level types (inconsistent ABI) -- `()`



# Let's Look at Webrender!

* [Webrender's Native Rust API](https://dev.searchfox.org/mozilla-central/source/gfx/wr/webrender_api/src/display_list.rs)
* [Webrender's C API](https://dev.searchfox.org/mozilla-central/source/gfx/webrender_bindings/src/bindings.rs)
* [cbindgen.toml](https://dev.searchfox.org/mozilla-central/source/gfx/webrender_bindings/cbindgen.toml)
* [Generated Header](https://dev.searchfox.org/mozilla-central/source/__GENERATED__/gfx/webrender_bindings/webrender_ffi_generated.h)




# Read The Docs!

[github.com/eqrion/cbindgen/blob/master/docs.md](https://github.com/eqrion/cbindgen/blob/master/docs.md
)


