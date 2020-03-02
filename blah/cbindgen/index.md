% cbindgen

Alexis Beingessner

[<img src="icon.png" width="250" style="display:inline; box-shadow:none;"></img>](http://cglab.ca/~abeinges)
[<img src="rust.png" width="250" style="display:inline; box-shadow:none;"></img>](https://rust-lang.org)
<img src="firefox-nightly.png" width="250" style="display:inline; box-shadow:none;"></img>

slides: https://github.com/Gankra/tutorial-cbindgen/



# What is cbindgen?

Creates C(++) headers for Rust libraries!

github.com/eqrion/cbindgen

```
cargo install cbindgen
```




# **Why** is cbindgen?

Webrender!

* Rewrite of firefox's graphics in Rust
* Lots of C++ calling into Rust
* Managing bindings by hand too error-prone




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
enum class MyEnum: uint32_t { A, B, C }
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
  enum class Tag : uint32_t { Some, None, };
  struct Some_Body { Tag tag; uint32_t _0; };

  struct { Tag tag; };
  Some_Body some;
};
```

# Fieldful Enum Conveniences

```cpp
auto val = COptionU32::Some(12);
if (val.IsSome()) {
  printf("%d\n", val.some._0);
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





# How Does it Work?

* grab all `#[no_mangle] pub extern fns`
* grab all types those reference
  * and types those reference
    * and ...


(note: hacky parser)



# How Does it Work?

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
* as a cli tool




# Let's Look at a Simple Codebase

https://github.com/Gankra/tutorial-cbindgen

* `rust-bindings/`
  * `cbindgen.toml`
  * `Cargo.toml`
  * `src/lib.rs`
* `build.sh`
* `main.cpp`




# Pitfalls

* Doesn't respect namespacing
* Doesn't support non-POD types (destructors)
* Doesn't work if there's any `repr(rust)` types
