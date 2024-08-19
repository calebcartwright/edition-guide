# Rustdoc combined tests

🚧 The 2024 Edition has not yet been released and hence this section is still "under construction".
More information may be found in the tracking issue at <https://github.com/rust-lang/rust/issues/124853>.

## Summary

- [Doctests] are now combined into a single binary which should result in a significant performance improvement.

## Details

Prior the the 2024 Edition, rustdoc's "test" mode would compile each code block in your documentation as a separate executable. Although this was relatively simple to implement, it resulted in a significant performance burden when there are a large number of documentation tests. Starting with the 2024 Edition, rustdoc will attempt to combine documentation tests into a single binary, significantly reducing the overhead for compiling doctests.

```rust
/// Adds two numbers
///
/// ```
/// assert_eq!(add(1, 1), 2);
/// ```
pub fn add(left: u64, right: u64) -> u64 {
    left + right
}

/// Subtracts two numbers
///
/// ```
/// assert_eq!(subtract(2, 1), 1);
/// ```
pub fn subtract(left: u64, right: u64) -> u64 {
    left - right
}
```

In this example, the two doctests will now be compiled in a single executable. Rustdoc will essentially place each example in a separate function within a single binary. Rustdoc also adds a small `main` function which tells it which test to run. The tests still run in independent processes as they did before, so any global state (like global statics) should still continue to work correctly.

This change is only available in the 2024 Edition to avoid potential incompatibilities with existing doctests which may not work in a combined executable.

[doctests]: ../../rustdoc/write-documentation/documentation-tests.html
[libtest harness]: ../../rustc/tests/index.html

### Standalone tag

In some situations it is not possible for rustdoc to combine examples in a single executable. Rustdoc will attempt to automatically detect if this is not possible, for example:

* Uses the [`compile_fail`][tags] language tag, which indicates that the example fails to compile.
* Uses the [`edition`][tags], which indicates the edition of the example.[^edition-tag]
* Uses global attributes like the [`global_allocator`] attribute, which could potentially interfere with other tests.
* Defines any crate-wide attributes (like `#![deny(unused)]`).
* Defines a macro that uses `$crate`, because the `$crate` path will not work correctly.

However, rustdoc is not able to automatically determine *all* situations where an example cannot be combined with other examples. In these situations, you can add the `standalone` language tag to indicate that the example should be built as a separate executable.

```rust
//! ```
//! let location = std::panic::Location::caller();
//! assert_eq!(location.line(), 5);
//! ```
```

This example is sensitive to the code structure of how the example is compiled. This won't work with the "combined" approach because the line numbers will shift depending on how the doctests are combined. In these situations, you can add the `standalone` tag to force the example to be built separately just as it did in previous editions.

```rust
//! ```standalone
//! let location = std::panic::Location::caller();
//! assert_eq!(location.line(), 5);
//! ```
```

[tags]: ../../rustdoc/write-documentation/documentation-tests.html#attributes
[`global_allocator`]: ../../std/alloc/trait.GlobalAlloc.html

[^edition-tag]: Note that rustdoc will only combine tests if the entire crate is Edition 2024 or greater. Using `edition2024` tags in older editions will not result in those tests being combined.

## Migration

There is no automatic migration to determine which doctests need to be annotated with the `standalone` tag. You will need to update your crate to the 2024 Edition and then run your documentation tests and see if any of them fail. If they do fail, you will need to analyze whether the test can be rewritten to be compatible with the combined approach, or add the `standalone` tag to retain the previous behavior.
