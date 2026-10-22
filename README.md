# pi-rust-style-guide

A [pi](https://pi.dev) skill that turns [Google's Rust Style Guide](https://google.github.io/styleguide/rustguide.html) into a compact set of rules an agent can apply while writing, reviewing, or refactoring Rust code.

Covers code organization and imports, ownership (`Rc`/`Arc`, `.clone()`), function parameter and return types, error reporting, types and traits, concurrency, macros, naming, docs, `unsafe`, accessors, `#[non_exhaustive]`, `#[must_use]`, type inference, iterator combinators, and assertions — including the guide's codebase-specific exceptions.

## Install

```bash
pi install git:github.com/xiaoshaoning/pi-rust-style-guide
```

Or copy `skills/rust-style-guide/` into `~/.agents/skills/` (user) or `.agents/skills/` (project).

## Use

The skill loads automatically when a Rust style/API-design task matches. To force it:

```text
/skill:rust-style-guide review this module
```

## Contents

- `skills/rust-style-guide/SKILL.md` — the skill
- `docs/rust_guide.txt` — verbatim source text the skill was distilled from

## License

[CC BY 4.0](LICENSE). Derived from Google's Rust Style Guide, © Google, licensed under CC BY 4.0; `docs/rust_guide.txt` is reproduced unmodified.
