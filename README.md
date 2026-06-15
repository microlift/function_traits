# Function Traits

Function Traits is a Rust library that extracts type information from functions using macro based type reflection. It can be used for projects that require type reflection in a low level context, such as high-level emulators.

## Type Traits

`ToFunctionArgumentType` maps Rust primitive types to the `FunctionArgumentType` enum at compile time via an associated constant. This lets you query the ABI-level type of any supported Rust type without any runtime overhead.

```rust
use function_traits::{ToFunctionArgumentType, FunctionArgumentType};

// Each primitive type carries its ABI kind as a compile-time constant.
assert_eq!(u8::TYPE,  FunctionArgumentType::UnsignedByte);
assert_eq!(i32::TYPE, FunctionArgumentType::SignedWord);
assert_eq!(f64::TYPE, FunctionArgumentType::DoublePrecisionFp);

// Raw pointers, regardless of pointee, are always Pointer.
assert_eq!(<*const u8>::TYPE, FunctionArgumentType::Pointer);
assert_eq!(<*mut i64>::TYPE,  FunctionArgumentType::Pointer);
```

### Inspecting a function's signature at runtime

`DynamicSymbol` combines a function pointer with its fully-reflected type signature. Pass any function that implements `Function` (all `fn` and `unsafe extern "C"` pointers with up to 8 arguments are covered) and the types are inferred automatically:

```rust
use function_traits::{DynamicSymbol, FunctionArgumentType, trampoline_func};

unsafe extern "C" fn add(a: i32, b: i32) -> i64 {
    (a + b) as i64
}

// trampoline_func! casts the function to the expected extern "C" signature.
let sym = DynamicSymbol::new("add", trampoline_func!(add, i32, i32));

// arg_types[0] is always the return type; the rest are parameters in order.
assert_eq!(sym.arg_types[0], FunctionArgumentType::SignedDoubleword); // i64 return
assert_eq!(sym.arg_types[1], FunctionArgumentType::SignedWord);       // i32 a
assert_eq!(sym.arg_types[2], FunctionArgumentType::SignedWord);       // i32 b
assert!(!sym.is_ellipsis_function);
```

### Variadic (C-ellipsis) functions

Variadic `unsafe extern "C"` functions are also supported. The variadic slot is appended as `FunctionArgumentType::VariadicList`:

```rust
use function_traits::{DynamicSymbol, FunctionArgumentType, trampoline_func};

unsafe extern "C" fn my_printf(fmt: *const u8, ...) -> i32 { 0 }

let sym = DynamicSymbol::new("my_printf", trampoline_func!(my_printf, *const u8, ...));

assert_eq!(sym.arg_types[0], FunctionArgumentType::SignedWord);    // i32 return
assert_eq!(sym.arg_types[1], FunctionArgumentType::Pointer);       // fmt
assert_eq!(sym.arg_types[2], FunctionArgumentType::VariadicList);  // ...
assert!(sym.is_ellipsis_function);
```
