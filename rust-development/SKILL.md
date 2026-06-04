---
name: rust-development
description: >
  Comprehensive Rust development skill for LLM agents. Covers idiomatic Rust,
  type-driven design (newtype, typestate, trait-as-typeclass), builder pattern,
  error handling, ownership & borrowing, async/await, modularity, tooling, and
  documentation. Apply when writing, reviewing, or refactoring any `.rs` file
  or Cargo workspace.

triggers:
  - Writing or reviewing Rust source files (*.rs)
  - Designing public crate APIs or library traits
  - Implementing newtype, builder, typestate, or error-handling patterns
  - Refactoring ownership, lifetimes, or async code
  - Organizing Cargo workspaces or module hierarchies
---

# SKILLS.md — Idiomatic Rust Development for LLM Agents

> **Golden Rule**: Every guideline exists for a reason. Understand the *why*
> before applying the *what*. Never blindly follow a rule that clearly violates
> its own intent.

---

## 0. Priority Order

Rules are grouped by impact. When two rules conflict, higher-priority rules win.

| Priority | Category                        | Impact         | Prefix    |
|----------|---------------------------------|----------------|-----------|
| 1        | Ownership & Borrowing           | CRITICAL       | `own-`    |
| 2        | Type Safety & Newtype Patterns  | CRITICAL       | `type-`   |
| 3        | Error Handling                  | CRITICAL       | `err-`    |
| 4        | API Design & Traits             | HIGH           | `api-`    |
| 5        | Async / Await                   | HIGH           | `async-`  |
| 6        | Module & Visibility             | MEDIUM-HIGH    | `mod-`    |
| 7        | Idiomatic Patterns              | MEDIUM         | `idiom-`  |
| 8        | Performance                     | MEDIUM         | `perf-`   |
| 9        | Documentation                   | MEDIUM         | `doc-`    |
| 10       | Testing                         | MEDIUM         | `test-`   |
| 11       | Tooling & Static Verification   | MEDIUM         | `tool-`   |
| 12       | Anti-patterns                   | REFERENCE      | `anti-`   |

---

## 1. Ownership & Borrowing (`own-`)

### `own-borrow-over-clone`
Prefer `&T` over `.clone()`. Clone only when the callee genuinely needs to own the value.

```rust
// Bad
fn print_name(name: String) { println!("{name}"); }
// Good
fn print_name(name: &str) { println!("{name}"); }
```

### `own-slice-over-vec`
Accept `&[T]` not `&Vec<T>`, accept `&str` not `&String`. Wider types admit more callers.

```rust
// Bad
fn sum(v: &Vec<i32>) -> i32 { v.iter().sum() }
// Good
fn sum(v: &[i32]) -> i32 { v.iter().sum() }
```

### `own-cow-conditional`
Use `Cow<'a, T>` when a value is sometimes borrowed, sometimes owned to avoid unconditional allocation.

### `own-arc-shared`
Use `Arc<T>` for shared ownership across threads; `Rc<T>` for single-threaded sharing.

### `own-refcell-mutex`
Use `RefCell<T>` for interior mutability in a single thread. Use `Mutex<T>` (or `RwLock<T>` when reads
dominate) for multi-threaded interior mutability.

### `own-copy-small`
Derive `Copy` for small, trivially-copyable types (scalars, small structs).
Do not derive `Copy` on types that hold heap resources.

### `own-move-large`
Move large data instead of cloning it. If the caller must retain ownership, use `Arc`.

### `own-lifetime-elision`
Rely on lifetime elision whenever possible. Add explicit lifetime annotations only when the compiler
cannot infer them.

### `own-impl-asref`
Accept `impl AsRef<T>` (or `impl Into<T>`) in public APIs to give callers flexibility without forcing
allocations.

```rust
// Flexible: accepts &str, String, Path, &Path
fn open_file(path: impl AsRef<std::path::Path>) { }
```

---

## 2. Type Safety & Newtype Patterns (`type-`)

### `type-newtype-units`
Wrap primitive types in newtypes to distinguish domain concepts at compile time with zero runtime cost.

```rust
struct Meters(f64);
struct Seconds(f64);
// Cannot accidentally pass Meters where Seconds is expected.
```

### `type-newtype-invariants`
Use validated newtypes (e.g., via the `nutype` crate) to enforce domain invariants at the boundary.
Never allow construction of an invalid value; check once, trust forever.

```rust
use nutype::nutype;

#[nutype(validate(not_empty, len_char_max = 200), derive(Debug, Clone, Display))]
struct Username(String);
```

### `type-newtype-derive`
Use `derive_more` or `nutype` to avoid boilerplate when exposing underlying-type traits on newtypes
(`Display`, `Debug`, `From`, `Into`, `Deref`, `AsRef`, `PartialEq`, `Hash`).

### `type-newtype-hide`
Do not expose the inner field of a newtype as part of the public API (use private tuple struct fields).
Implement explicit accessor methods only if needed.

### `type-avoid-bool-args`
Replace `bool` function arguments with descriptive newtypes or enums. A `bool` at a call site is
unreadable.

```rust
// Bad
fn connect(secure: bool) { }
// Good
enum Security { Tls, Plaintext }
fn connect(mode: Security) { }
```

### `type-typestate`
Use the typestate pattern (phantom-type-parameterized structs) to encode valid operation sequences in
the type system. Consume `self` on transitions.

```rust
use std::marker::PhantomData;

struct TrafficLight<S> { _state: PhantomData<S> }
struct Green; struct Yellow; struct Red;

impl TrafficLight<Green> {
    pub fn new() -> Self { TrafficLight { _state: PhantomData } }
    pub fn advance(self) -> TrafficLight<Yellow> { TrafficLight { _state: PhantomData } }
}
impl TrafficLight<Yellow> {
    pub fn advance(self) -> TrafficLight<Red> { TrafficLight { _state: PhantomData } }
}
```

Use `statum` or manual `PhantomData` for complex workflows. Reserve typestate for APIs where invalid
transitions must be statically impossible.

### `type-bitflags`
Use `bitflags!` for flag sets, not enums. Use enums for mutually exclusive variants.

### `type-strong-types`
Avoid "primitive obsession". Types like `UserId`, `OrderId`, `EmailAddress` prevent entire categories
of bugs and serve as self-documenting contracts for AI agents.

---

## 3. Error Handling (`err-`)

### `err-thiserror-lib`
Library crates **must** define their own error types using `thiserror`. Never expose `anyhow::Error`
or `Box<dyn Error>` in a public library API.

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum ParseError {
    #[error("invalid UTF-8 at byte {0}")]
    InvalidUtf8(usize),
    #[error("unexpected end of input")]
    UnexpectedEof,
}
```

### `err-anyhow-app`
Application crates (binaries, integration tests, CLI tools) **should** use `anyhow` or `eyre` for
unified, ergonomic error propagation.

```rust
use anyhow::{Context, Result};

fn run() -> Result<()> {
    let cfg = std::fs::read_to_string("config.toml")
        .context("failed to read config.toml")?;
    Ok(())
}
```

### `err-question-mark`
Use the `?` operator for propagation. Never use the deprecated `try!()` macro.

### `err-no-unwrap-prod`
Never use `.unwrap()` in production code paths. Use `.expect("reason")` only for conditions that
represent programming bugs (impossible at runtime if the code is correct).

### `err-result-over-panic`
Return `Result<T, E>` for expected failure conditions. Panic only for detected programming bugs,
violated invariants, or const contexts.

### `err-context-chain`
Add context at every abstraction boundary using `.context()` or `.with_context(|| ...)`.

```rust
open_db(path).with_context(|| format!("opening database at {path}"))?;
```

### `err-lowercase-msg`
Error messages must start with lowercase and must not end with punctuation. They will be composed
into chains.

### `err-doc-errors`
Every public function that returns `Result` must document its error conditions in a `# Errors`
rustdoc section.

### `err-from-impl`
Use `#[from]` to auto-generate `From` impls for wrapped error variants. Use `#[source]` to chain
underlying errors.

### `err-panic-is-stop`
A panic means "stop the program". Do not use panics for control flow or to communicate errors to
callers. Never assume panics will be caught.

---

## 4. API Design & Traits (`api-`)

### `api-naming-rfc430`
Follow RFC 430 casing:
- Types, traits, enums: `UpperCamelCase`
- Functions, methods, modules, fields: `snake_case`
- Constants, statics: `SCREAMING_SNAKE_CASE`
- Lifetimes: short lowercase (`'a`, `'de`)
- No `get_` prefix on getter methods (use `value()`, not `get_value()`)

### `api-conv-prefixes`
Conversion method naming conventions:

| Prefix  | Meaning                                                 |
|---------|---------------------------------------------------------|
| `as_`   | Cheap, borrowing reference conversion                   |
| `to_`   | Expensive or by-value conversion, no ownership transfer |
| `into_` | Consuming conversion, takes ownership                   |
| `from_` | Consuming conversion on the target type (`From::from`)  |

### `api-common-traits`
Public types must eagerly implement applicable standard traits:
`Debug`, `Clone`, `PartialEq`, `Eq`, `PartialOrd`, `Ord`, `Hash`, `Default`, `Display`
(where semantically meaningful), `Send`, `Sync`.

### `api-builder-pattern`
Use the builder pattern (C-BUILDER) for structs with more than ~3 fields, optional fields, or complex
construction logic.

**Preferred: owning builder with method chaining.**

```rust
let request = Request::builder()
    .url("https://example.com")
    .timeout(Duration::from_secs(30))
    .header("Accept", "application/json")
    .build()?;
```

Use the `bon` crate for zero-boilerplate, typestate-verified builders that produce compile errors when
required fields are missing:

```rust
use bon::Builder;

#[derive(Builder)]
pub struct Config {
    #[builder(into)]
    pub host: String,
    pub port: u16,
    pub timeout: Option<Duration>,   // optional field
}
```

`bon`-generated builders use the typestate pattern internally: `build()` is only callable after all
required fields are set. For conditional chaining:

```rust
let mut b = Builder::new().name("foo");
if some_condition {
    b = b.extra("bar");
}
let result = b.build();
```

### `api-constructors`
Provide a `Type::new(...)` inherent constructor even when `Default` is implemented. Constructors are
static, inherent methods.

### `api-traits-as-typeclass`
Simulate Haskell-style typeclasses by defining narrow, composable traits. A trait is a *capability*.
Favour many small traits over one god trait.

```rust
pub trait Serialize { fn serialize(&self) -> Vec<u8>; }
pub trait Deserialize: Sized { fn deserialize(bytes: &[u8]) -> Result<Self, DecodeError>; }
// Consumers bound over `T: Serialize + Deserialize`
```

Implement external traits for newtype wrappers as the canonical orphan-rule workaround.

### `api-generics-vs-dyn`
Default to generics (`impl Trait` / `T: Trait`). Use `dyn Trait` when:
- The concrete type is unknown at compile time (heterogeneous collections, plugin systems)
- Compile time or binary size is a concern and the call site is not performance-critical
- The trait must be object-safe and you need `Box<dyn Trait>`

Generics monomorphize (faster, larger binary); `dyn` uses vtable dispatch (slightly slower, smaller
binary, erases type).

### `api-no-out-params`
Functions must not take out-parameters. Return values or tuples instead.

### `api-object-safety`
If a trait may be used as a trait object (`dyn Trait`), ensure it is object-safe. Avoid methods with
`Self: Sized` bounds or generic return types unless behind a sealed sub-trait.

### `api-sealed-traits`
Seal traits that must not be implemented by downstream users:

```rust
mod private { pub trait Sealed {} }

pub trait MyTrait: private::Sealed {
    fn method(&self);
}
```

### `api-no-weasel-names`
Avoid vague type names: `Manager`, `Service`, `Factory`, `Helper`, `Util`. Name things by what they
*do* or *represent*. A builder is a `Builder`; a dispatcher is a `Dispatcher`.

### `api-regular-fn-over-assoc`
Do not attach functions to `impl` blocks unless they operate on `self`. Free functions are first-class
citizens in Rust.

### `api-send-sync`
Public types must implement `Send` and `Sync` where structurally sound. Mark explicitly `!Send` or
`!Sync` only when thread safety cannot be guaranteed, and document why.

---

## 5. Async / Await (`async-`)

### `async-send-bounds`
Futures used with Tokio's multi-threaded scheduler must be `Send`. When writing generic async
functions that spawn tasks, add associated return type bounds:

```rust
async fn do_work<T>(task: T)
where
    T: WorkTask + Send + 'static,
    T::run(): Send,
{
    tokio::task::spawn(async move { task.run().await });
}
```

### `async-yield-points`
Long-running CPU-bound async tasks must call `tokio::task::yield_now().await` at regular intervals
(~10–100 microseconds of compute between yields) to cooperatively yield the scheduler.

```rust
async fn crunch(items: &[Item]) {
    for chunk in items.chunks(256) {
        process(chunk);
        tokio::task::yield_now().await;
    }
}
```

### `async-fn-in-trait`
As of Rust 1.75+, `async fn` is stable in traits. Use it directly. For object-safe async traits
requiring `dyn`, use the erased-trait pattern or the `async-trait` crate.

### `async-no-block`
Never call blocking code (`std::thread::sleep`, `std::fs::read_to_string`, holding a mutex across
`.await`) inside an async context. Use `tokio::task::spawn_blocking` to run synchronous work off the
async thread pool.

### `async-timeout`
Always set timeouts for I/O operations. Use `tokio::time::timeout(duration, future)`.

### `async-avoid-arc-mutex-hot`
Minimize `Arc<Mutex<T>>` on hot async paths. Prefer message-passing (`tokio::sync::mpsc`) or
`tokio::sync::RwLock` (for read-heavy state) to reduce contention.

---

## 6. Module & Visibility (`mod-`)

### `mod-start-flat`
Start with a single module. Split into submodules only when a file grows large enough to navigate
poorly, or when clear encapsulation boundaries emerge.

### `mod-semantic-cohesion`
Group code by *semantic area* (domain concepts), not by kind (structs in one file, traits in another).
Related structs, traits, enums, and functions belong together.

### `mod-private-fields`
Struct fields are private by default. Expose fields only through explicit accessor methods. This
enables backward-compatible refactoring (C-STRUCT-PRIVATE).

### `mod-no-glob-reexport`
Do not re-export items with `pub use foo::*` in library crates. Re-export explicitly.

### `mod-pub-use-doc-inline`
Use `#[doc(inline)]` to integrate re-exported items visually with their siblings:

```rust
#[doc(inline)]
pub use internal::FooBar;
```

### `mod-crate-split`
Extract a submodule into a separate crate when:
- It can be used independently by other projects.
- Splitting reduces recompilation scope meaningfully.
- It requires its own distinct set of dependencies.

Use a `crates/` flat layout for workspaces (virtual manifest at root, one directory per crate named
identically to its crate name).

### `mod-features-vs-crates`
Crates are for items that can stand alone. Feature flags unlock extra functionality that depends on the
core crate. Do not use features as a substitute for proper crate splits when compile time or dependency
isolation matters.

---

## 7. Idiomatic Patterns (`idiom-`)

### `idiom-use-iterators`
Prefer iterator chains over manual `for` loops when expressing transformations.

```rust
let evens: Vec<i32> = (0..100).filter(|x| x % 2 == 0).collect();
```

### `idiom-if-let-while-let`
Use `if let` and `while let` for single-variant destructuring instead of `match`.

### `idiom-question-mark`
Use `?` for error propagation, not `match` + manual `Err(e) => return Err(e)`.

### `idiom-immutability-default`
Declare variables as immutable (`let x = ...`) unless mutation is explicitly required
(`let mut x = ...`).

### `idiom-entry-api`
Use the `HashMap::entry` API instead of paired `contains_key`/`insert` calls to avoid double-hashing.

```rust
map.entry(key).or_insert_with(|| expensive_default());
```

### `idiom-from-into`
Implement `From<A> for B`; `Into<A>` is then auto-derived. Always implement `From` on the destination
type.

### `idiom-display-not-to-string`
Implement `Display` for user-facing string conversion. Use `to_string()` (which calls `Display`) only
at output boundaries.

### `idiom-default-derive`
Derive `Default` when a sensible zero-value exists. Provide `Type::new()` for explicit construction.

### `idiom-derive-eagerly`
Eagerly derive `#[derive(Debug, Clone, PartialEq, Eq, Hash, Default)]` on data types wherever
structurally sound. Fewer manual impls = fewer bugs.

### `idiom-phantom-data`
Use `PhantomData<T>` when a type logically *owns* or *borrows* `T` without a direct field, to ensure
correct variance and `Drop` checking.

### `idiom-raii`
Encode resource lifetimes in the type system with RAII. Use `Drop` for cleanup. Never manually leak
resources.

### `idiom-oncellock`
Use `std::sync::OnceLock` (stable since Rust 1.70) for lazy static initialization. The `lazy_static!`
macro is deprecated.

```rust
static CONFIG: std::sync::OnceLock<Config> = std::sync::OnceLock::new();
```

---

## 8. Performance (`perf-`)

### `perf-profile-first`
Never optimize without measurement. Use `criterion` or `divan` for micro-benchmarks. Enable debug
symbols in benchmarks:

```toml
[profile.bench]
debug = 1
```

### `perf-with-capacity`
Pre-allocate collections when the final size is known:

```rust
let mut v = Vec::with_capacity(expected_len);
let mut s = String::with_capacity(64);
let mut map = HashMap::with_capacity(expected_len);
```

### `perf-avoid-string-format-hot`
String formatting via `format!` allocates. In hot paths, use structured logging or write directly to
a `String` with `write!`.

### `perf-mimalloc`
Application binaries should set `mimalloc` as the global allocator. Benchmarks show up to 25%
throughput improvements on allocation-heavy paths:

```toml
[dependencies]
mimalloc = "0.1"
```

```rust
use mimalloc::MiMalloc;
#[global_allocator]
static GLOBAL: MiMalloc = MiMalloc;
```

### `perf-batch-operations`
Design APIs for batched operations. Avoid per-item processing loops when bulk APIs exist.

### `perf-cache-locality`
Prefer `Vec<T>` over linked structures. Keep hot data in contiguous memory.

### `perf-default-hasher`
Replace `std`'s default SipHash with `rustc-hash` (`FxHashMap`) or `ahash` for non-cryptographic,
performance-sensitive maps.

---

## 9. Documentation (`doc-`)

### `doc-module-level`
Every public module must have `//!` documentation:

```rust
//! Provides request-building utilities for the HTTP client.
//!
//! See [`RequestBuilder`] for the primary entry point.
```

### `doc-item-canonical`
Public items must include:
1. **Summary sentence** — 15 words max, on one line (appears in module index).
2. **Extended description** — free-form, as needed.
3. `# Examples` — at least one runnable example.
4. `# Errors` — when the function returns `Result`.
5. `# Panics` — when the function may panic.
6. `# Safety` — mandatory for `unsafe` functions.

```rust
/// Opens a file at the given path for reading.
///
/// # Errors
/// Returns [`IoError::NotFound`] if the path does not exist.
///
/// # Examples
/// ```
/// let f = open_file("data.txt")?;
/// ```
pub fn open_file(path: &str) -> Result<File, IoError> { todo!() }
```

### `doc-no-params-table`
Do not document parameters in a `# Parameters` table. Explain them in prose inline. Rust doc
convention is prose, not JavaDoc.

### `doc-examples-use-question-mark`
Examples in doc-tests must use `?` (not `unwrap()` or `try!`). Wrap in
`fn main() -> Result<(), Box<dyn Error>>` if needed.

### `doc-first-sentence`
The first doc sentence is extracted as the item's tooltip/summary. Keep it 15 words or fewer and
self-contained.

---

## 10. Testing (`test-`)

### `test-unit-inline`
Write unit tests in a `#[cfg(test)] mod tests { ... }` block within the same file as the code under
test. Tests in the same module can access private items.

### `test-integration-tests-dir`
Place integration tests in `tests/` at the crate root. Each file is a separate crate.

### `test-doc-tests`
Write doc-tests for every public function. They serve as both documentation and regression tests.

### `test-assert-messages`
Use `assert_eq!(actual, expected, "context: {detail}")` and `assert!(cond, "context")` to produce
useful failure messages.

### `test-no-unwrap-in-tests`
Even in tests, prefer `.expect("reason")` over `.unwrap()` to make failures self-explanatory.

### `test-property-based`
Consider `proptest` or `quickcheck` for functions with complex invariants (parsers, serializers,
mathematical operations).

### `test-mockable`
Design APIs to be testable. Accept traits (`impl Trait`) instead of concrete types at boundaries that
need stubbing in tests.

---

## 11. Tooling & Static Verification (`tool-`)

### `tool-rustfmt`
All code must be formatted with `rustfmt` using default settings (4-space indent, 100-column limit).
Run on every commit:

```sh
cargo fmt --all
```

### `tool-clippy`
Enable all major Clippy lint categories in `Cargo.toml`:

```toml
[lints.clippy]
cargo       = { level = "warn", priority = -1 }
complexity  = { level = "warn", priority = -1 }
correctness = { level = "warn", priority = -1 }
pedantic    = { level = "warn", priority = -1 }
perf        = { level = "warn", priority = -1 }
style       = { level = "warn", priority = -1 }
suspicious  = { level = "warn", priority = -1 }
# High-value restriction group rules
allow_attributes_without_reason = "warn"
undocumented_unsafe_blocks       = "warn"
unused_result_ok                 = "warn"
clone_on_ref_ptr                 = "warn"
```

### `tool-compiler-lints`
Enable additional compiler lints in `Cargo.toml`:

```toml
[lints.rust]
missing_debug_implementations = "warn"
unsafe_op_in_unsafe_fn         = "warn"
unused_lifetimes               = "warn"
redundant_lifetimes            = "warn"
```

### `tool-expect-over-allow`
Use `#[expect(lint, reason = "...")]` over `#[allow(lint)]` for per-item lint overrides. `#[expect]`
warns if the suppressed lint no longer fires, preventing stale suppressions.

### `tool-cargo-audit`
Run `cargo audit` in CI to check for known security vulnerabilities in dependencies.

### `tool-cargo-hack`
Run `cargo hack --feature-powerset check` to validate that all feature combinations compile correctly.

### `tool-cargo-udeps`
Run `cargo udeps` to detect unused dependencies.

### `tool-miri`
Run `miri` on all `unsafe` code to detect undefined behaviour. Miri is mandatory before merging any
`unsafe` block.

### `tool-structured-logging`
Use `tracing` (preferred) or `log` for structured, levelled logging. Never use `println!` for
observability. Follow OpenTelemetry semantic conventions for attribute names:

```rust
tracing::event!(
    name: "request.completed",
    Level::INFO,
    http.request.method = "GET",
    http.response.status_code = 200,
    "request {http.request.method} completed with {http.response.status_code}"
);
```

Redact sensitive data (email, tokens, PII) before logging.

---

## 12. Unsafe Code (`unsafe-`)

### `unsafe-valid-reasons`
`unsafe` is justified only for:
- Novel safe abstractions (new smart pointer, allocator, lock-free structure)
- Performance-critical operations where profiling shows a measurable gain
- FFI and platform/kernel calls

Never use `unsafe` to bypass `Send`/`Sync` bounds, shorten code, or circumvent lifetime requirements
ad-hoc.

### `unsafe-implies-ub`
Mark a function `unsafe` only if misuse can lead to **undefined behaviour (UB)**. Do not mark
merely-dangerous functions `unsafe`.

### `unsafe-document-invariants`
Every `unsafe` block must carry a `// SAFETY:` comment explaining what invariants are upheld and why.

```rust
// SAFETY: `ptr` is non-null and valid for reads of `len` bytes,
// as guaranteed by the caller per the function contract.
let slice = unsafe { std::slice::from_raw_parts(ptr, len) };
```

### `unsafe-miri`
All `unsafe` code must pass `cargo miri test`, including adversarial test cases (closures that panic,
malicious `Deref`/`Drop` impls).

### `unsafe-no-soundness-exceptions`
Unsound code is never acceptable. If safe encapsulation is impossible, expose `unsafe fn` instead of
a safe wrapper over UB. No exceptions.

---

## 13. Anti-patterns (`anti-`)

| Anti-pattern                                      | Prefer instead                                    |
|---------------------------------------------------|---------------------------------------------------|
| Returning `Box<dyn Error>` from libraries         | `thiserror`-defined error enum                    |
| Using `.unwrap()` in production code              | `?` operator or `.expect("invariant reason")`     |
| Implementing `Deref<Target=T>` on non-smart-ptrs  | Explicit accessor methods                         |
| Using `bool` arguments for binary choices         | Dedicated enum or newtype                         |
| Deep nested `if let` chains                       | `?`, `and_then`, `map_or`, `match`                |
| `Vec<Box<dyn Trait>>` when types are known        | Generic collections or enums                      |
| `panic!` in library code for expected errors      | Return `Result<T, E>`                             |
| Blocking syscalls inside `async fn`               | `tokio::task::spawn_blocking`                     |
| Over-cloning to satisfy the borrow checker        | Restructure lifetimes, use `Cow` or `Arc`         |
| `unsafe impl Send/Sync` without proof             | Genuine `Send`/`Sync` types or documented inv.    |
| God traits with 20+ methods                       | Multiple narrow, composable traits                |
| `String` for filesystem paths                     | `std::path::Path` / `PathBuf`                     |
| `HashMap::contains_key` + `insert`                | `Entry` API                                       |
| Silent error discard `let _ = f()`                | Propagate or log with context                     |
| `lazy_static!` (deprecated)                       | `std::sync::OnceLock` (stable since Rust 1.70)    |

---

## 14. Cargo & Workspace Structure

### Workspace layout (for medium-to-large projects)
```
repo/
|-- Cargo.toml          # virtual manifest: no [package], only [workspace]
|-- crates/
|   |-- my-core/        # directory name == crate name
|   |   |-- Cargo.toml
|   |   `-- src/lib.rs
|   |-- my-api/
|   `-- my-cli/
`-- tests/              # integration tests
```

- Use a virtual manifest at the root (no `[package]`).
- Keep all crates one level deep under `crates/`.
- Internal-only crates use `version = "0.0.0"`.
- Automation tooling lives in an `xtask` crate.

### Dependency hygiene
- Pin exact versions for binaries; use SemVer ranges for libraries.
- Prefer permissive licenses (MIT / Apache-2.0 dual-license).
- Audit dependencies with `cargo audit` and `cargo udeps`.

---

## 15. AI-Agent-Specific Guidelines

These rules improve how LLM agents operate on Rust codebases:

1. **Use strong, explicit types.** Avoid `String`/`i64` where domain newtypes exist. The compiler
   becomes a second reviewer.
2. **Follow API naming conventions exactly.** AI-generated code that violates `as_`/`to_`/`into_`
   conventions introduces silent semantic errors.
3. **Never generate `.unwrap()` in non-test paths.** Default to `?` or `.expect()` with an
   explanatory string.
4. **Generate doc-comments for every public item.** Agents working downstream on the same codebase
   depend on docs.
5. **Prefer `#[derive(...)]` over manual trait impls** unless customization is required.
6. **Generate tests alongside implementations.** Unit tests in `#[cfg(test)]` blocks, doc-tests in
   doc-comments.
7. **Do not introduce `unsafe` without an explicit human-approved reason.** Default to safe
   abstractions.
8. **Never silence clippy/compiler warnings with `#[allow(...)]` unless `#[expect(...)]` with a
   reason is used.**
9. **Prefer `impl Trait` in function signatures** over concrete types to enable testability and
   flexibility.
10. **Generate structured error types (`thiserror`) for every new module** that performs fallible
    work.

---

## 16. Quick Reference: Crate Ecosystem

| Need                         | Crate(s)                                 |
|------------------------------|------------------------------------------|
| Validated newtypes           | `nutype`                                 |
| Builder pattern              | `bon`, `derive_builder`                  |
| Derive extra traits on types | `derive_more`                            |
| Error types (libraries)      | `thiserror`                              |
| Error handling (apps)        | `anyhow`, `eyre`                         |
| Async runtime                | `tokio`                                  |
| Structured logging           | `tracing`, `tracing-subscriber`          |
| Serialization                | `serde` + `serde_json` / `serde_yaml`    |
| Custom allocator             | `mimalloc`                               |
| Benchmarking                 | `criterion`, `divan`                     |
| Property-based testing       | `proptest`, `quickcheck`                 |
| Bit flags                    | `bitflags`                               |
| State machines (typestate)   | `statum` or manual PhantomData           |
| Hashing (non-crypto, fast)   | `rustc-hash` (FxHashMap), `ahash`        |
| UB detection                 | `cargo miri test`                        |
| Unused dep detection         | `cargo udeps`                            |
| Security audit               | `cargo audit`                            |
| Feature combinatorics        | `cargo hack`                             |

---

## 17. Sources & Further Reading

- [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/) — canonical upstream checklist
- [Microsoft Pragmatic Rust Guidelines](https://microsoft.github.io/rust-guidelines/) — production-scale rules
- [Rust Design Patterns](https://rust-unofficial.github.io/patterns/) — patterns book (newtype, builder, typestate, RAII)
- [Rust Style Guide](https://rustwiki.org/en/style-guide/) — canonical formatting reference
- [Comprehensive Rust (Google)](https://google.github.io/comprehensive-rust/) — error handling, ownership, async
- [Effective Rust](https://effective-rust.com/) — generics vs trait objects trade-offs
- [rust-unofficial/patterns](https://github.com/rust-unofficial/patterns) — community-maintained patterns catalogue

---

*Last synthesized: June 2026. Re-verify against current Rust edition (2024) and latest stable toolchain.*
