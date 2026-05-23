# Rust Coding Guidelines

## Formatting & Tooling

- `rustfmt` defaults — don't customize. 4-space indent.
- `clippy` with pedantic lints enabled (see Linting section)
- Run `cargo fmt --check` and `clippy --workspace -- -D warnings` in CI

## Naming

- `snake_case` for functions, variables, modules, fields
- `PascalCase` for types, traits, enums
- `SCREAMING_SNAKE_CASE` for constants
- No `get_` prefix on getters — just `fn name(&self)`
- Conversion methods: `as_` (cheap/borrowed), `to_` (expensive/cloned), `into_` (consumes self)
- Iterator methods: `iter()`, `iter_mut()`, `into_iter()`

## Project Layout

```text
src/
  lib.rs                # library root — curated public API via pub use
  main.rs               # thin binary entry point
  config.rs             # modern style: file-per-module
  config/               # sub-modules in directory
    parser.rs
tests/                  # integration tests (each file is a separate crate)
crates/                 # workspace members for larger projects
```

- Keep `main.rs` thin — parse args, wire deps, call `run()`
- Modern module style: `src/foo.rs` not `src/foo/mod.rs`
- `lib.rs` re-exports public API with `pub use`, internals stay private
- Use workspaces (`crates/`) when the project grows, split by domain

## Error Handling

- **Libraries**: `thiserror` for typed error enums — callers get exhaustive matching
- **Applications**: `anyhow` for ergonomic propagation, or `thiserror` everywhere for consistency
- Use `?` operator for propagation — replaces Go's `if err != nil` pattern
- Error messages: lowercase, no trailing punctuation
- Preserve error chains with `#[from]` or `#[source]`
- Sentinel errors as enum variants, not string matching

```rust
// Library error type
#[derive(Debug, thiserror::Error)]
pub enum ConfigError {
    #[error("file not found: {path}")]
    NotFound { path: PathBuf },
    #[error("parse error")]
    Parse(#[from] serde_json::Error),
}
```

## Common Patterns

### Newtype

Wrap primitives for type safety. Zero-cost abstraction.

```rust
struct UserId(u64);

impl From<u64> for UserId {
    fn from(id: u64) -> Self { Self(id) }
}
```

### Builder

Use for structs with many optional fields. Replaces Go's functional options.

```rust
let server = ServerBuilder::new("0.0.0.0:8080")
    .max_connections(100)
    .timeout(Duration::from_secs(30))
    .build()?;
```

### From / Into

Implement `From<A> for B` and get `Into<B> for A` free. Accept `impl Into<T>` in function params.

### Standard Trait Derivations

Eagerly derive on all data types:

```rust
#[derive(Debug, Clone, PartialEq, Default, serde::Serialize, serde::Deserialize)]
```

## Interfaces (Traits)

Traits are Rust's primary extensibility mechanism. Use them for pluggable backends, not just testing.

- Accept traits, return concrete types (same principle as Go)
- Define traits at the consumer, not the provider
- Keep traits small — 1-3 methods
- Explicit `impl Trait for Type` — no implicit satisfaction like Go
- Extract traits at every infrastructure boundary — storage, messaging, external APIs, clocks

### Design for Pluggability

The same trait should support production backends, local development, and testing — all as first-class implementations.

Native async traits (Rust 2024 edition) remove the need for `#[async_trait]` in most cases. Use `async_trait` only
when you need `dyn Trait` dispatch.

```rust
// src/storage/mod.rs — trait defined by the consumer
#[async_trait]
pub trait BlobStore: Send + Sync {
    async fn put(&self, key: &str, data: &[u8]) -> Result<()>;
    async fn get(&self, key: &str) -> Result<Vec<u8>>;
    async fn delete(&self, key: &str) -> Result<()>;
}

// src/storage/s3.rs — production
pub struct S3Store { client: aws_sdk_s3::Client, bucket: String }

// src/storage/fs.rs — on-prem / PVC
pub struct FsStore { base_path: PathBuf }

// src/storage/memory.rs — dev / testing
pub struct MemoryStore { data: RwLock<HashMap<String, Vec<u8>>> }
```

```rust
// src/db/mod.rs
#[async_trait]
pub trait Repository: Send + Sync {
    async fn find_by_id(&self, id: &str) -> Result<Order>;
    async fn save(&self, order: &Order) -> Result<()>;
}

// src/db/postgres.rs — production
// src/db/sqlite.rs — local dev, CI
// src/db/memory.rs — unit tests
```

Each implementation is a real module — not a mock. Tests use in-memory. Local dev uses SQLite or filesystem. Production
uses Postgres or S3. Same trait, same service code.

### Generics vs Trait Objects

- **Generics** (`impl Trait` / `<T: Trait>`) — zero-cost, monomorphized. Use by default.
- **Trait objects** (`dyn Trait`) — dynamic dispatch, runtime cost. Use when you need heterogeneous collections or
  plugin-style extensibility.

```rust
// Generics — compile-time dispatch (preferred)
fn process(store: &impl Store) -> Result<()> { ... }

// Trait object — runtime dispatch (when needed)
fn process(store: &dyn Store) -> Result<()> { ... }

// Heterogeneous collection requires trait objects
let handlers: Vec<Box<dyn Handler>> = vec![...];
```

### Trait Composition

Compose larger traits from smaller ones using supertraits:

```rust
trait Reader { fn read(&self, id: &str) -> Result<Item>; }
trait Writer { fn write(&self, item: &Item) -> Result<()>; }
trait ReadWriter: Reader + Writer {}

// Blanket impl — any type implementing both gets ReadWriter for free
impl<T: Reader + Writer> ReadWriter for T {}
```

### Constructor Injection

Pass trait implementations through constructors:

```rust
impl<S: Store, C: Clock> Service<S, C> {
    pub fn new(store: S, clock: C) -> Self {
        Self { store, clock }
    }
}
```

### Testing with Traits

Mock via simple test structs — no framework needed:

```rust
#[cfg(test)]
mod tests {
    struct MockStore { items: Vec<Order> }

    impl Store for MockStore {
        async fn save(&self, order: &Order) -> Result<(), StoreError> { Ok(()) }
        async fn find_by_id(&self, id: &str) -> Result<Order, StoreError> {
            self.items.iter().find(|o| o.id == id)
                .cloned().ok_or(StoreError::NotFound)
        }
    }
}
```

## Testing

### Unit Tests

Inline `#[cfg(test)]` module in the same file. Can access private functions.

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parse_valid_config() {
        let config = Config::parse("key=value").unwrap();
        assert_eq!(config.key, "value");
    }
}
```

### Integration Tests

Each file in `tests/` is a separate crate. Only tests `pub` API.

```rust
// tests/api_test.rs
use myapp::Client;

#[test]
fn client_connects() { ... }
```

### Fuzz Testing

Use `cargo-fuzz` for parsers, validators, and anything handling untrusted input.

```rust
// fuzz/fuzz_targets/parse_config.rs
#![no_main]
use libfuzzer_sys::fuzz_target;

fuzz_target!(|data: &[u8]| {
    let _ = myapp::Config::parse(data);
});
```

### Tooling

- `cargo-nextest` for faster parallel test execution
- `proptest` for property-based testing
- `tarpaulin` for coverage
- No benchmarks

## Linting

Configure in `Cargo.toml`, not scattered `#[allow]` attributes:

```toml
[lints.clippy]
pedantic = { level = "warn", priority = -1 }
module_name_repetitions = "allow"
must_use_candidate = "allow"
dbg_macro = "warn"
unwrap_used = "warn"
undocumented_unsafe_blocks = "warn"
```

## Dependencies

- Minimal third-party deps — justify every addition
- Use `workspace.dependencies` for version consistency across crates
- Use `cargo-machete` to detect unused deps
- Set `edition = "2024"` and `resolver = "3"` in workspace root
- Always fill in `description`, `license`, `repository` in `Cargo.toml`
