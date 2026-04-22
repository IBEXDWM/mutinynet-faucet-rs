# CLAUDE.md

## Project overview

Rust project. Uses Cargo for build/dependency management.

## Common commands

```bash
cargo build              # Build the project
cargo run                # Build and run
cargo test               # Run tests
cargo fmt                # Format code
cargo clippy             # Lint
cargo clippy -- -D warnings  # Treat warnings as errors
```

## Code conventions

- **Formatting**: `cargo fmt` — enforced, no manual style variation
- **Linting**: `cargo clippy` — fix all warnings before merging
- **Naming**: `snake_case` for functions/variables/modules, `CamelCase` for types/traits, `SCREAMING_SNAKE_CASE` for constants
- **Error handling**: prefer `Result<T, E>` over panics in library code; use `thiserror` for error types
- **Unsafe**: avoid `unsafe` unless strictly necessary; document every `unsafe` block with the invariants it upholds
- **Modules**: one concept per module; use `mod.rs` or file-per-module as appropriate

## Git conventions

- **Commit messages**: conventional commits — `feat(scope): message`, `fix(scope):`, etc.
- **Branch naming**: `<jira-ticket>-<slug>`
- **Main branch**: `master`

## What to watch out for

- Never commit `.env` or secrets
- Review `Cargo.lock` changes carefully — unexpected dependency bumps can break builds
- Pay attention to lifetimes and ownership — prefer borrowing over cloning when possible
- Use `#[must_use]` on functions returning `Result` or important values
