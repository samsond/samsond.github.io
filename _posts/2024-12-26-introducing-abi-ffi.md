---
layout: post
title:  "Understanding ABI and FFI"
date:   2025-01-03 12:00:00 -0800
categories: interoperability  
description: "A Comprehensive Guide to Interoperability in Software Development"
tags: [rust, java, ffi, abi, interoperability]
---

## Part 1: Introduction to ABI and FFI

### Introduction

Imagine you are working on a critical project that requires incorporating a powerful library to enhance its functionality. However, you soon discover that this library is written in a different programming language than the one you are using for your project. Rewriting the entire library from scratch in your language would be time-consuming and error-prone. 

Let's think about making bindings that let your project call functions from the library without having to rewrite it. This approach saves time and effort while making sure the library works well.

For simplicity, let's choose a binary search function as our example. This will help us touch on critical issues like memory layout, FFI safety, compatibility checks, and using the modern preview Java Panama and Rust.


### Understanding ABI and FFI

Application Binary Interface (ABI) defines the low-level interface between two program modules, typically between a library and an application. It ensures that compiled code can work together, regardless of the compiler or language used. ABI is crucial for software development as it allows different components to interact at the binary level.

Foreign Function Interface (FFI) enables a program written in one language to call functions or use services written in another language, allowing developers to leverage existing libraries and tools across different programming languages with out having to write the library themselves. FFI depends on the ABI to ensure that data and function calls are compatible between the languages.


### Data Alignment and Memory Layout

> **Inspired by Eric S. Raymond's article on [Structure Packing](http://www.catb.org/esr/structure-packing/)**

A crucial aspect of ABI is the calling convention, which dictates how registers are used when a function is called. It specifies how input and output parameters are handled, which registers are used for parameters and return values, and which registers can be modified by the function.

Basic C data types have alignment requirements: chars can start on any byte address, but other types like short, int, float, long, and double must start on addresses divisible by their size.

In C, the layout of variables in memory is influenced by data alignment and padding. 

Now, let's see how this translates to Rust. We can use the memoffset crate to determine the offsets of the fields in a Rust struct. By using the #[repr(C)] attribute, we tell the Rust compiler to lay out the struct in memory the same way C would, ensuring the same order, size, and alignment of the fields. Here is an equivalent Rust code:

```rust
use memoffset::offset_of;

#[repr(C)]
struct Foo1 {
    p: *mut u8, // Equivalent to `char *` in C
    c: u8,      // Equivalent to `char` in C
    x: i32,     // Equivalent to `int` in C (4 bytes)
}

fn main() {
    println!("Offset of p: {}", offset_of!(Foo1, p));
    println!("Offset of c: {}", offset_of!(Foo1, c));
    println!("Offset of x: {}", offset_of!(Foo1, x));
}
```
When you run this Rust code, you might get output similar to the following:

```bash
Offset of p: 0
Offset of c: 8
Offset of x: 12
```

This code might result in the following memory layout:

| Address | Variable | Description |
|---------|----------|-------------|
| 0-7     | p        | Pointer 8 byte|
| 8       | c        | Character 1 byte|
| 9-11     | Padding  |Added for alignment|
|12-15     | x        | Integer 4 bytes|


The exact offsets may vary depending on the machine you run it on, but this example demonstrates how data alignment is applied in Rust using the offset_of macro. This helps ensure that the memory layout is consistent with the alignment requirements, optimizing access speed and maintaining compatibility with C.



### Java and FFI

Java provides the Foreign Function & Memory API (FFM API) to facilitate interoperability with native code written in another languages like C, C++, and Rust.

From the Java docs: 


> In Java, characters are typically encoded in **UTF-16**, which is the format used by char arrays, which we normally get when we use String and StringBuffer objects.


#### Foreign Function & Memory API (FFM API)

The FFM API is a modern approach introduced as part of the Panama Project. It provides a straightforward and safer way to interact with native code by using Java's type system and memory management features. The FFM API resides in the `java.lang.foreign` package of the `java.base` module.

For more information, refer to the [OpenJDK JEP 424](https://openjdk.org/jeps/424).

### Key Features of the FFM API

1. **Allocate Foreign Memory**: This involves allocating memory that is not part of the Java heap. Classes like `MemorySegment`, `MemoryAddress`, and `SegmentAllocator` facilitate this process.

2. **Manipulate and Access Structured Foreign Memory**: You can define the layout of the foreign memory (such as structs or arrays) using `MemoryLayout` and access individual elements with `VarHandle`.

3. **Control the Allocation and Deallocation of Foreign Memory**: `MemorySession` helps manage the lifecycle of the foreign memory, ensuring it is properly allocated and freed when needed.

4. **Call Foreign Functions**: You can call functions from non-Java code using classes like `Linker`, `FunctionDescriptor`, and `SymbolLookup`.

### Translating Memory layout with Java Using the FFM API 

Circling back to our Rust example, let's see how this translates to Java. We need to use the Foreign Function & Memory API (FFM API). In Java, we must manually specify the padding (reorder the fields) to ensure proper alignment. This highlights the importance of understanding memory layout and ABI to maintain compatibility and optimize performance.

Here is the equivalent Java code using the FFM API:

```java
import java.lang.foreign.MemoryLayout;
import java.lang.foreign.MemoryLayout.PathElement;
import static java.lang.foreign.ValueLayout.JAVA_BYTE;
import static java.lang.foreign.ValueLayout.JAVA_INT;
import static java.lang.foreign.ValueLayout.ADDRESS;
public class Main {
    public static void main(String[] args) {
        // Define the layout of the struct
        MemoryLayout foo1Layout = MemoryLayout.structLayout(
                ADDRESS.withName("p"), // Equivalent to `char *` in C
                JAVA_BYTE.withName("c"), // Equivalent to `char` in C
                MemoryLayout.paddingLayout(3), // Padding to align the next field
                JAVA_INT.withName("x")   // Equivalent to `int` in C (4 bytes)
        );

        // Get the offsets of the fields
        long offsetOfP = foo1Layout.byteOffset(PathElement.groupElement("p"));
        long offsetOfC = foo1Layout.byteOffset(PathElement.groupElement("c"));
        long offsetOfX = foo1Layout.byteOffset(PathElement.groupElement("x"));

        // Print the offsets
        System.out.println("Offset of p: " + offsetOfP);
        System.out.println("Offset of c: " + offsetOfC);
        System.out.println("Offset of x: " + offsetOfX);
    }
}
```
When we run the above coe we would get the following similar to Rust example:

```bash
Offset of p: 0
Offset of c: 8
Offset of x: 12
```

### Rust and FFI

Rust provides robust support for FFI, allowing it to interface with other languages like C, C++, and Java. When handling strings in FFI, Rust represents owned strings with the `String` type and borrowed slices of strings with the `str` primitive.

> Rust strings are encoded in **UTF-8**, which is the standard format for string types like String and str.

However, C strings differ from Rust strings, so we need to consider the following as described in Rust FFI docs:

- **Encodings**: Since different languages may use different encodings, it's important to verify which encoding is used in each case.
- **Character Size**: C strings rely on a nul terminator at the end to determine the string's size, whereas in Rust, the size of the string is included explicitly within the string itself. 
- **Internal Nul Characters**: Rust strings can have nul characters in the middle, but C strings typically cannot.
```rust
fn main() {
    let rust_string = "Hello\0world";
    let bytes = rust_string.as_bytes();
    println!("{:?}", bytes); // Will show the nul character inside the string
}
```
output
```bash
[72, 101, 108, 108, 111, 0, 119, 111, 114, 108, 100]
```

### Handling Potential Mismatch with CString

To know early on if there are a potential mismatch between C and Rust, we need to use `CString`. For example, let's try the above using `CString`:
```rust
use std::ffi::CString;

fn main() {
    let rust_string = "Hello\0world";

    // Convert Rust string to CString
    // this line will panic due if there is \0 within the string
    let c_string = CString::new(rust_string).expect("CString::new failed");

    // Now you can pass c_string.as_ptr() to a C function.
    let bytes = c_string.as_bytes_with_nul();
    println!("{:?}", bytes);  // Now the internal \0 will not cause issues in C
}
```
As expected we get error at compile time:
```bash
thread 'main' panicked at src/main.rs:13:46:
CString::new failed: NulError(5, [72, 101, 108, 108, 111, 0, 119, 111, 114, 108, 100])
```

Let's try one more example with the following:

```rust
#[no_mangle]
pub extern "C" fn binary_search(slice: &[i32], target: i32, start_index: usize) -> Option<i64> {
    match slice[start_index..].binary_search(&target) {
        Ok(index) => Some((index + start_index) as i64),
        Err(_) => None,
    }
}
```

When you compile the above you would get the following warning:


```bash
warning: `extern` fn uses type `[i32]`, which is not FFI-safe
  --> src/lib.rs:16:40
   |
16 | pub extern "C" fn binary_search(slice: &[i32], target: i32, start_index: usize) -> Option<i6...
   |                                        ^^^^^^ not FFI-safe
   |
   = help: consider using a raw pointer instead
   = note: slices have no C equivalent
   = note: `#[warn(improper_ctypes_definitions)]` on by default

warning: `extern` fn uses type `Option<i64>`, which is not FFI-safe
  --> src/lib.rs:16:84
   |
16 | ...et: i32, start_index: usize) -> Option<i64> {
   |                                    ^^^^^^^^^^^ not FFI-safe
   |
   = help: consider adding a `#[repr(C)]`, `#[repr(transparent)]`, or integer `#[repr(...)]` attribute to this enum
   = note: enum has no representation hint

warning: `binary_search_lib` (lib) generated 2 warnings
```

Two warnings are printed. The type `&[i32]` is not FFI-safe because slices do not have a direct equivalent in C. 
Instead, use a raw pointer and a length to represent the array.

The warning also indicates that the `Option<i64>` type is not FFI-safe. This means that `Option<i64>` does not 
have a guaranteed representation that is compatible with the C ABI. To make the function FFI-safe, you need 
to use a type that has a well-defined representation in C.


To make the function more compliant with the C ABI, we can use an `enum` to represent the result. and the other one Fix
Replace the slice with a raw pointer and a length.
This way, we avoid returning Rust-specific types like Option, which may not be directly compatible with C.

## Part 2: Practical Example with Rust and Java

In this part, we will demonstrate how to call Rust functions from Java using FFI. This example will highlight the importance of ABI in ensuring compatibility between different languages.
> To follow along with the code examples in this section, you can clone the repository from Github:
[abi-ffi-examples](https://github.com/samsond/abi-ffi-examples.git)

>`Prerequisites`
- Java 19 or above
- Rust
- Maven or Gradle
- Docker



### Creating the Rust Library
First, let's create a new Rust library. Open your terminal and run the following:

```bash
cargo new binary_search_lib --lib
```

This will create a new directory named binary_search_lib with the necessary files for a Rust library.

Next, navigate to the src directory and open the lib.rs file. Replace its contents with the following code:

```rust
#[no_mangle]
pub extern "C" fn binary_search(arr: *const i32, len: usize, target: i32) -> i32 {
    let slice = unsafe { std::slice::from_raw_parts(arr, len) };
    match slice.binary_search(&target) {
        Ok(index) => index as i32,
        Err(_) => -1,
    }
}
```

Build the Rust library

```bash
cargo build --release
```
### Creating the Java Project
Next, let's create a new Java project using Maven. Open your terminal and run the following command:

```bash
mkdir demo-ffi
gradle init --type java-application --package org.example --project-name demo-ffi
```
This will create a new directory named demo-ffi with the necessary files for a Java project.

Navigate to the src/main/java/org/example directory and open the App.java file. Replace its contents with the following code:
 
```java
public static void main(String[] args) {

        String libraryPath = "/path/to/your/libbinary_search_lib.dylib";

        if (libraryPath == null) {
            System.err.println("Library path not set!");
            return;
        }
        
        try (Arena arena = Arena.ofConfined()) {
            // Load the library
            SymbolLookup lib = SymbolLookup.libraryLookup(libraryPath, arena);

            // Find the binary_search function
            MemorySegment binarySearchSymbol = lib.find("binary_search").orElseThrow();

            // Prepare the function descriptor
            FunctionDescriptor descriptor = FunctionDescriptor.of(
                    ValueLayout.JAVA_INT,   // return type
                    ValueLayout.ADDRESS,    // arr type
                    ValueLayout.JAVA_INT,   // len type
                    ValueLayout.JAVA_INT    // target type
            );

            // Create a MethodHandle for calling the binary_search function
            MethodHandle methodHandle = Linker.nativeLinker().downcallHandle(binarySearchSymbol, descriptor);

            // Prepare arguments
            int[] arr = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11};
            int len = arr.length;
            int target = 7;

            // Allocate off-heap memory and copy the array data
            MemorySegment arrSegment = arena.allocateArray(ValueLayout.JAVA_INT, arr);

            // Call the binary_search function
            int result = (int) methodHandle.invokeExact(arrSegment, len, target);

            System.out.println("Result: " + result);
        } catch (UnsatisfiedLinkError e) {
            System.err.println("Error loading library: " + e.getMessage());
            e.printStackTrace();
        } catch (Throwable t) {
            t.printStackTrace();
        }
    }
```

> **From Java Docs** 
> 
> `ValueLayout and Memory Alignment`
> 
> `ValueLayout.ADDRESS` in Java defines the layout of a machine address, adhering to the size, alignment, and byte order of `size_t` on the underlying platform. These properties ensure compatibility with platform-specific conventions and are crucial when working with low-level memory or native code.
> 
> Key attributes of `ValueLayout.ADDRESS`:
> 
> - **Size and Alignment:** Matches `sizeof(size_t)` to ensure proper memory alignment, a fundamental requirement for efficient and correct execution.
> - **Byte Order:** Uses `ByteOrder.nativeOrder()` to match the hardware’s native endianness for consistent interpretation of multi-byte data.
> 
> This approach applies broadly to other `ValueLayout` types, where alignment and layout properties are critical for ABI (Application Binary Interface) compliance.

```bash
# Compile the Java program
 javac --enable-preview --release 21 -d target/classes src/main/java/org/example/App.java

# Run the Java program 
 java --enable-preview -cp target/classes org.example.App
```

## Part 3: Ensuring ABI Compatibility: A Critical Requirement for Clients

Imagine you are a software developer working on a critical library used by various clients in different applications. This library provides essential functions, including a binary search algorithm, which is widely used for efficient data retrieval. Your clients rely on the stability and compatibility of this library to ensure their applications run smoothly.

### The Problem

One day, you receive a request to enhance the binary search function to allow specifying a starting index for the search. This change will provide more flexibility and improve performance for certain use cases. However, you realize that modifying the function's signature could potentially break ABI (Application Binary Interface) compatibility, causing existing client applications to fail.



### The Importance of ABI Compatibility

ABI compatibility is crucial because it ensures that different versions of a library can be used interchangeably without causing runtime errors. When a library's ABI changes, applications that depend on the previous version may encounter issues such as incorrect function calls, memory corruption, or crashes. Therefore, maintaining ABI compatibility is essential for providing a stable and reliable library for your clients.

### Introducing abi-dumper and abi-compliance-checker

To address this challenge, you decide to use `abi-dumper` and `abi-compliance-checker`, powerful tools for checking ABI compatibility between different versions of a library. These tools help you identify changes in the ABI and ensure that your modifications do not break compatibility with existing client applications.

### Installing abi-dumper and abi-compliance-checker

First, you need to install `abi-dumper` and `abi-compliance-checker` on your development environment. Here are the instructions for installing them on different platforms:

### Compiling for the Linux Platform

First let's compile for linux platform:

Let's consider this version for our base to compare when changing our code

```rust

#[no_mangle]
pub extern "C" fn binary_search(arr: *const i32, len: usize, target: i32) -> i32 {
    let slice = unsafe { std::slice::from_raw_parts(arr, len) };
    match slice.binary_search(&target) {
        Ok(index) => index as i32,
        Err(_) => -1,
    }
}
```
Save to place where you can refere library as V1. 

```bash
cargo build --release --target x86_64-unknown-linux-gnu
```

Let's have the second version as follows:


```rust
#[repr(C)]
pub enum SearchResult {
    Found(i64),
    NotFound,
}

#[no_mangle]
pub extern "C" fn binary_search(array: *const i32, len: usize, target: i32, start_index: usize) -> SearchResult {
    let slice = unsafe { std::slice::from_raw_parts(array, len) };
    match slice[start_index..].binary_search(&target) {
        Ok(index) => SearchResult::Found((index + start_index) as i64),
        Err(_) => SearchResult::NotFound,
    }
}
```
Same as before cargo build save the library bundle to a location that you can refer as V2.


### Using Dockerfile
To set up the environment and install abi-dumper and abi-compliance-checker, we will use Docker. The Dockerfile for this setup can be found in the GitHub repository [abi-ffi-examples/Dockerfile](https://github.com/samsond/abi-ffi-examples/blob/main/Dockerfile); adapt it to your library location and mount point.

Steps to build and run: 

```bash
docker build -t abi-tools -f Dockerfile .

docker run -it --rm --name abi-tools-container \
  -v /Users/sami/projects/rust-project/binary_search_lib/mnt/v1:/mnt/v1 \
  -v /Users/sami/projects/rust-project/binary_search_lib/mnt/v2:/mnt/v2 \
  abi-tools
```
### Checking ABI Compatibility

First, generate an ABI dump for the new version of the library:

```bash
abi-dumper /mnt/v1/libbinary_search_lib.so -o v1_abi.dump -lver 1
abi-dumper /mnt/v2/libbinary_search_lib.so -o v2_abi.dump -lver 2
```

Next, compare the ABI of the new version with the old version using abi-compliance-checker:

```bash
abi-compliance-checker -l binary_search_lib -old v1_abi.dump -new v2_abi.dump
```

Then, we will get a report at compat_reports/binary_search_lib/1_to_2/compat_report.html. 
The report highlights the following changes, making it easy to understand the modifications made 
to the binary_search function.

| Change | 
|--------|
| 1    Parameter `start_index` of type `usize` has been added to the calling stack.  | 
| 2    Type of return value has been changed from `i32` to `struct SearchResult` of different format. | 

By following these steps, you can ensure that your library modifications do not break ABI compatibility, providing a stable and reliable library for your clients.

### References

* [https://doc.rust-lang.org/std/ffi/index.html](https://doc.rust-lang.org/std/ffi/index.html)
* [https://docs.rust-embedded.org/book/interoperability/rust-with-c.html](https://docs.rust-embedded.org/book/interoperability/rust-with-c.html)
* [https://openjdk.org/jeps/424](https://openjdk.org/jeps/424)
* [http://www.catb.org/esr/structure-packing/](http://www.catb.org/esr/structure-packing/)
* [https://docs.oracle.com/javase/8/docs/api/java/lang/Character.html#unicode](https://docs.oracle.com/javase/8/docs/api/java/lang/Character.html#unicode)
* [https://www.unicode.org/glossary/#code_point](https://www.unicode.org/glossary/#code_point)
