---
layout: post
title:  "Understanding Strings in Rust: Reversing and Handling Graphemes"
date:   2024-11-29 15:05:00 -0800
categories: rust programming
description: "An in-depth look at strings in Rust, including how to reverse a string and handle graphemes."
---

Rust is a systems programming language that provides powerful features for memory safety and performance. One of the essential aspects of any programming language is its handling of strings. In this article, we will explore how Rust manages strings, and we will provide an example of reversing a string while correctly handling graphemes.

## Strings in Rust

Rust has two main types for handling strings:

1. **String**: A growable, heap-allocated data structure that is used when you need an owned string.
2. **&str**: A string slice that is a reference to a sequence of UTF-8 bytes.

### Creating Strings

You can create a `String` using the `String::from` method or the `to_string` method:

```rust
let s1 = String::from("Hello, Rust!");
let s2 = "Hello, Rust!".to_string();
```

### String slices

```rust
let s = String::from("Hello, Rust!");
let slice = &s[0..5]; // "Hello"
```

### Reversing a String
Reversing a string in Rust can be tricky because of its UTF-8 encoding. Simply reversing the bytes can lead to invalid UTF-8 sequences. Instead, we need to reverse the characters.

Here is an example of reversing a string:

```rust
fn reverse_string(s: &str) -> String {
    s.chars().rev().collect()
}

fn main() {
    let original = "Hello, Rust!";
    let reversed = reverse_string(original);
    println!("Original: {}", original);
    println!("Reversed: {}", reversed);
}
```

### Example with Combining Characters
Let's consider a more complex example with combining characters. The string "cöde 👋" contains a combining diaeresis character. 

The string **"cöde"** consists of the following Unicode scalar values:
- `c`
- `o`(base character) +  ̈ (combining diaeresis)
- `d`
- `e`

When we reverse this string using the chars() method, the combining diaeresis is treated as a separate character:

```rust
fn main() {
    let original = "cöde 👋";
    let reversed = reverse(original);
    println!("Original: {}", original);
    println!("Reversed: {}", reversed);
}
```

Output:

```
Original: cöde 👋
Reversed: 👋 ed̈oc
```

As you can see, the combining **diaeresis** is not correctly handled.


### Handling Graphemes
To correctly handle graphemes, we need to use the unicode-segmentation crate. This crate provides functionality to work with grapheme clusters, which are user-perceived characters that may consist of multiple Unicode code points.

First, add the unicode-segmentation crate to your Cargo.toml:

```
[dependencies]
unicode-segmentation = "1.8.0"
```

### Updating the Reverse Function
We can update the reverse function to use the unicode-segmentation crate when the grapheme feature is enabled. This allows us to handle graphemes correctly:

```rust
#[cfg(feature = "grapheme")]
use unicode_segmentation::UnicodeSegmentation;

pub fn reverse(input: &str) -> String {
    #[cfg(feature = "grapheme")]
    {
        // Reverse based on grapheme clusters
        input.graphemes(true).rev().collect()
    }

    #[cfg(not(feature = "grapheme"))]
    {
        // Default to reversing based on Unicode scalar values
        input.chars().rev().collect()
    }
}

fn main() {
    let original = "cöde 👋";
    let reversed = reverse(original);
    assert_ne!("👋 ed̈oc", reversed);
    println!("Original: {}", original);
    println!("Reversed: {}", reversed);
}
```

### Running the Program with Grapheme Support
To run the program with the grapheme feature enabled, use the following command:

```
cargo run --features grapheme
```

Output:

```
Original: cöde 👋
Reversed: 👋 edöc
```

As you can see, the combining diaeresis is now correctly handled.

### Conclusion
In this article, we explored how Rust handles strings and provided examples of reversing a string and handling graphemes. We demonstrated how reversing a string with combining characters can lead to incorrect results and showed how to fix this using the unicode-segmentation crate.

This article was inspired by **exercism** from [Exercism](https://exercism.org/).

Happy coding!