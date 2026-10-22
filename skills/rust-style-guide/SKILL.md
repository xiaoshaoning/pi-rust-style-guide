---
name: rust-style-guide
description: Apply Google's Rust Style Guide while writing, reviewing, or refactoring Rust code. Covers code organization, imports, ownership (Rc/Arc, clone), function parameter and return types, error reporting, types, traits, concurrency, macros, naming, docs, unsafe, accessors, #[non_exhaustive], #[must_use], type inference, iterator combinators, assertions, and per-codebase exceptions. Use when the user mentions Google Rust style, rustguide, Rust API design review, or asks how to shape a Rust API, module layout, trait, or error type.
---

# Google Rust Style Guide

Full text: `docs/rust_guide.txt` (upstream: <https://google.github.io/styleguide/rustguide.html>).

"Style" here covers API and implementation design, not just formatting.
Also follow the official Rust Style Guide and Rust API Guidelines unless they conflict with these rules.
API-design rules are **binding for library code**, **strong recommendations for application code** — in app code favor simplicity and directness over reusability; no premature abstraction.
Tags `[chrome]`, `[Fuchsia]` mark codebase-specific rules; skip them outside that codebase.

## Code organization

- Prefer small crates, ideally one file focused on one abstraction `[chrome]`. Split when you can; large crates only when trait coherence forces types together or it is a foundational library.
- Flat file layout per crate `[chrome]`: one directory per `rust_library`, no subdirectories. Needing subdirectories means the crate is too big.
- Name leaf modules and source files after their primary item: `mod foo` → `foo.rs`.
- Non-leaf modules hold only `mod` declarations and re-exports — no implementation code.
- No glob imports. Allowed: `use super::*` in inline modules; `use enum::*` inside enum methods; `pub use submodule::*` in non-leaf modules; merging C++ namespaces into a **private** module; third-party preludes at reviewer discretion. Don't define preludes.
- One module per `use`: no `::` inside `{..}` (`imports_granularity = Module`). Prefer `use foo::Foo; use foo::bar::Bar;`.
- Import traits inside the function body unless used file-wide.
- Qualify top-level functions with their module: `mem::transmute(y)`, not an imported `transmute`. Exceptions: self-disambiguating C++ functions, small single-purpose first-party crates `[chrome]`, collision-prone module names.
- Import and use types unqualified: `Regex` + `use regex::Regex`. Exceptions: ambiguity (`io::Error` or `use std::io::Error as IoError`), large type families, macros/generated code, huge dependency sets.
- `[chrome]` use `chromium::import!()`.

## Formatting

- Use the codebase formatter. Target rustfmt config: `max_width = 100`, `hex_literal_case = Lower`, `format_macro_matchers = true`, escapes (`\u{200D}`) for invisible characters.
- `#[rustfmt::skip]` only for manual alignment or generated code — never personal style.

## Ownership

- Avoid `Rc`/`Arc`: prefer unique ownership and borrowing. Use them only when the borrow structure would be unreasonably complex.
- Prefer borrows over `.clone()`; clone when you truly need a deep copy, when borrows become excessively complex, or when performance allows. Cloning `Rc`/`Arc` is not a deep copy and is fine.

## Function inputs and outputs

- Immutable borrowed input: `&T` not `&Box<T>`, `&[T]` not `&Vec<T>`, `&str` not `&String`, `Option<&T>` not `&Option<T>`. (Exception: non-linear text like Rope/Cord.)
- Library return types: prefer iterators (`impl Iterator`) over `Vec<T>`. Return `Vec` when most callers need one, you build one anyway, random indexing is expected, a `for` loop reads better, or you return an existing vector. App code may return `Vec` freely.
- Library parameters: accept `impl IntoIterator`/iterators when you only iterate; `&[T]` is fine in app code.
- `impl Into<T>` parameters only for library ergonomics when call-site convenience matters; weigh code bloat and inference cost. App code almost never.
- Prefer an enum over a `bool` parameter whose meaning isn't obvious; if you keep the `bool`, comment it.

## Reporting errors

- `Option` — obvious, detail-free failure (missing value, parse mismatch).
- `Result` — recoverable failure where the caller needs to distinguish causes (I/O, network).
- `panic!` — bugs and violated invariants; fix the code, don't handle at runtime. Dev-facing binaries and tests may panic.

## Types

- Newtype over type alias when semantics differ, the type deserves its own API, the underlying type may change, or values could get mixed up (`Day`/`Month`/`Year`).
- Named fields over tuples and multi-field tuple structs (newtypes and very local code excepted).

## Traits

- Application code: add impls when needed. Low-level libraries: anticipate users' needs.
- Implement the std trait instead of a look-alike method (`impl FromStr for Foo`, not `fn from_str`); keeping both for discoverability is fine. Use `#[derive()]` when possible.
- Library parameters: prefer generics (`T: Trait`, `impl Trait`) over `dyn Trait` (speed, inlining, dyn compatibility); use `dyn` when code size matters.
- Define a trait only to enable generics, to accept/return `impl Trait`/`dyn Trait`, or as an extension trait. Otherwise use an inherent `impl`. Never use traits merely to organize code.
- Extension trait = adding methods to a foreign type; suffix `Ext`, owned by the type's owner. Not for generics or dispatch.
- Error types: `Copy` when possible. Most types: `Clone` (unless they wrap an external resource). `Default` when there is a `new()` or a natural default. Prefer `PartialEq` over `Eq` and `PartialOrd` over `Ord`; implement `Eq`/`Ord` only when reflexivity/total order truly hold. Likely map keys: `Hash + Eq` (+ `Ord` if a clear total order exists).
- Pair an important `From` impl with an inherent constructor (`new`, `new_from_foo`, `from_foo`).
- All-public "options" structs implement `Default` (enables struct update syntax).
- Implement `len()` and `is_empty()` together.

## Concurrency

- Never read data out of a poisoned mutex.
- `[chrome]` no `async`/`.await`; no threads from std — use sequences.
- `[Fuchsia]` use `fuchsia_sync::Mutex`, not `std::sync` or `parking_lot`.

## Macros

Preference order: plain functions/types/traits/modules → `macro_rules!` → other code generation → `proc_macro`. Macros break tooling, create dialects, and make LSCs expensive — justify the cost.

## Naming

- Follow std method naming; `as_*`/`to_*`/`into_*` with the matching `self` convention.
- Lifetimes: descriptive for type lifetimes; `'_` is fine; one or two single letters only for simple input/output relations or `for<'a>`; otherwise spell them out.
- Generic params: short but indicative, ideally from the key bound. `T`/`U` for unconstrained APIs like `Vec<T>`.
- Extension traits: `~Ext`.
- Constructors: `new`, `new_with_foo`, `try_new` when fallible; conversions `from_foo` + `From`.
- No abbreviations or deleted letters unless widely known (`IpAddr` is fine); don't abbreviate by default.

## Documentation

- Document public APIs (consider the `missing_docs` lint). Include examples somewhere for the crate; always document panics from malformed input; document invariants of `unsafe` functions/traits; use capitalization and punctuation.
- Don't explain the language or restate the name — omit a comment that says nothing. Individual enum variants can go undocumented.
- `///` for items, `//!` at the top of the module/crate; Markdown; `//` for implementation notes, TODOs, commented-out code. Mixing doc and ordinary comments is encouraged. Prefer hand-written docs over macro-generated ones.

## Other features

- `unsafe`: only when no safe implementation exists. Encapsulate behind a safe abstraction, isolate from business logic (ideally its own crate). Add a `SAFETY:` comment and a technical justification answering "why not safe code?" and, if performance, "where are the benchmarks?".
- No unstable features (codebase-approved exceptions only).
- Accessors: `foo()`, `foo_mut()`, `take_foo()`, `set_foo()`, `replace_foo()`; closure accessors `with_foo()`/`with_mut_foo()` for interior mutability, expensive-to-clone values, or post-change validation. Don't leak `MutexGuard`s when a closure accessor would hide the detail.
- Treat foreign `#[doc(hidden)]` items as private.
- `const fn` only if it will stay `const` (un-consting is breaking).
- No operator overloading as syntax sugar.
- `#[non_exhaustive]` on public cross-team enums and public-field structs. Exceptions: never-changing ontologies (`Color`), two-state bool-like enums, committed LSC fixes, and structs needing functional record update.
- `#[must_use]` on functions whose return value can't be ignored (pure functions, error/critical results) and on the types they return (error types, lazy computations).
- No code before `main()` (e.g. `ctor`).

## Implementation concerns

- Don't overuse inference: annotate variable declarations for `.into()`/`.parse()` results and complex iterator chains; avoid turbofish (a turbofish with `_` is acceptable when forced).
- Prefer pattern matching over check-then-`unwrap`; prefer `expect("reason")` over `unwrap`.
- Matching `#[non_exhaustive]` enums: list known variants, not bare `_`. Use `Shape::Triangle | _ => {}` to keep exhaustiveness checked, or `if let` / `let ... else` when you only care about some variants. Consider the `non_exhaustive_omitted_patterns` lint.
- Assertions: pass a format-string context saying why it holds, what failed, or that the caller misused the API — not a restatement of the condition.
- Iterators: avoid long/complex chains, never build a chain in a `for` header (assign it to a named variable), avoid `Iterator::for_each`; prefer real combinators when they encode the logic (`any`, `filter_map`) over hand-rolled loops with breaks.
- No `panic::catch_unwind` (defense-in-depth only, `panic=unwind` codebases, never relied upon).
- `format!`/`write!` over `+` for 3+ strings and over `str::replace` for substitutions on literals.
- Use the codebase's test framework: `[chrome]` `rust_gtest_interop`, `[Fuchsia]` `#[fuchsia::test]`.
