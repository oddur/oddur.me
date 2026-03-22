+++
date = '2026-03-22T11:41:03+01:00'
draft = false
title = 'Agent Guardrails with Rust'
ShowToc = true
TocOpen = true
+++
I've been spending most of my time in agent-driven development lately, and the biggest insight I keep coming back to is this: the person in the driving seat matters more, not less.

The barrier to generating code with agents is basically zero. But agents will optimize for the path of least resistance, and the result will be mediocre. The engineers who have internalized best practices over years of building real systems, who know what good looks like, are the ones who can direct agents past that. The experience to know when code is merely working versus actually well-built is what separates useful agent output from slop.

That experience includes knowing which engineering practices to bring forward into the agentic age. TDD, static analysis, CI pipelines, type-driven design, code review, architectural patterns. These aren't relics. They're guardrails. The scaffolding we already have should act as the boundaries agents work within, where feedback is immediate and correctness is verifiable without human review of every line.

## The Edit-Compile-Test Loop

An agent is a heuristic system. The edit-compile-test loop is where we bring determinism to it. Static analysis, compile-time checking, and expressive type systems give agents something that prompt engineering can't: a hard, unambiguous signal about whether their output is correct. The more validation we can move out of the probabilistic LLM and into deterministic tooling, the better the results.

That principle is language-agnostic. But language choice determines how far you can take it. I've been enjoying studying and practicing programming in Rust, specifically with agents, and I want to call out where I see Rust bringing specific advantages to this workflow.

## Table Stakes: Testing, Linting, Formatting, CI

**Testing.** TDD is non-negotiable. Unit tests, integration tests, end-to-end tests. All three, running constantly as part of the agent's workflow. Previously TDD was often aspirational in practice, just because of the extra effort involved. With agents, that effort is effectively free, so there's no excuse anymore not to work this way. Rust makes testing a first-class language concept with [inline unit tests](https://doc.rust-lang.org/book/ch11-01-writing-tests.html) and parallel execution by default.

**Linting.** Run the strictest configuration your language offers. In Rust, that's [Clippy](https://doc.rust-lang.org/clippy/) in pedantic mode. Feed your agent the most aggressive linting rules available. The more constraints you give it, the fewer bad decisions it makes.

**Formatting.** Languages like Go and Rust killed off the bikeshed arguments about formatting by making it [part of the toolchain](https://github.com/rust-lang/rustfmt). For agents, canonical formatting means less noise in diffs.

**Pre-commit hooks: the final gate.** This is the last step in the edit-compile-test loop, and it's the one that guarantees the agent never delivers work that doesn't pass your quality gates. If any step fails, the commit is rejected. The agent doesn't get to ship half-finished work. In Rust, that's `fmt`, `clippy`, [`cargo check`](https://doc.rust-lang.org/cargo/commands/cargo-check.html), and `cargo test`. Add [TruffleHog](https://github.com/trufflesecurity/trufflehog) to scan for secrets. Agents are especially prone to hardcoding credentials, API keys, and tokens, and you never want those committed. Add dependency auditing in CI too ([cargo-deny](https://github.com/EmbarkStudios/cargo-deny) for Rust) for license checks, vulnerability scanning, and banned dependency detection.

The key is trying to front load checks as much as possible, so they get incorporated into the agent's inner loop. Once it hits your CI checks it should have passed as many of those checks as possible locally.

## Where Rust Starts to Pull Ahead: The Borrow Checker

The borrow checker gives you compile-time guarantees about [ownership, lifetimes, and data race prevention](https://doc.rust-lang.org/book/ch04-00-understanding-ownership.html). Code that passes it has properties you can't get in most languages regardless of how many tests you write: no data races, no use-after-free, no dangling references. These hold whether the agent got it right on the first pass or the fifth.

What makes this particularly useful for agents is that every borrow checker error is specific, deterministic, and points to exactly what needs to change. It's structured feedback that keeps the agent on track throughout the edit-compile cycle.

## The Type System

### Enums and Exhaustive Matching

Rust requires every variant of an [enum](https://doc.rust-lang.org/book/ch06-00-enums.html) to be handled in [match statements](https://doc.rust-lang.org/book/ch06-02-match.html). Add a new variant, and the compiler flags every location in the codebase where that case isn't covered.

Say you have a `PaymentStatus` enum with `Pending`, `Completed`, and `Failed`. Your codebase has match statements handling those three cases in your API responses, your notification logic, your database updates, your audit logging. Now you add a `Refunded` variant. In a dynamically typed language, that's a grep and a prayer. In Rust, the compiler immediately gives you a list of every match statement that doesn't handle `Refunded`. The agent gets a precise, actionable list of locations to update, and the code won't compile until every one of them is addressed.

### Newtypes

Instead of passing raw `String` and `i32` around, wrap them in semantically rich [newtypes](https://doc.rust-lang.org/book/ch20-03-advanced-types.html#using-the-newtype-pattern-for-type-safety-and-abstraction). A newtype is a distinct type that wraps a primitive, so the compiler treats it as fundamentally different from every other string or integer, even though the underlying data is the same.

This catches real bugs at compile time. I've seen an agent mix up OAuth refresh tokens and JWT tokens in a token refresh flow. Both are strings, but semantically they're completely different. With newtypes, `JwtToken` and `RefreshToken` are distinct types and the compiler won't let you pass one where the other is expected. That bug never makes it to runtime.

Another example: reading files from disk. Is the content compressed or not? Wrap it. `CompressedFileContents` goes into the decompression function, `FileContents` comes out. You can never accidentally pass a compressed payload to a function expecting uncompressed data. Without newtypes, that's a subtle bug that might only surface under specific conditions in production.

Newtypes also make APIs self-documenting. A function that takes `JwtToken` instead of `String` is a clearer contract for both humans reading the code and agents generating it.

### Typestates

We often have code that should only run when the application is in a certain state, such as initialized or authenticated. Normally you'd check this at runtime with something like `if self.is_initialized()`. Rust lets you push this into the type system so the compiler enforces it for you, and the whole thing compiles away to zero runtime cost.

Take a database pool. You don't want anyone running queries before the pool is initialized. With typestates, you define each state as its own zero-sized marker struct and make the pool generic over it:

```rust
struct Uninitialized;
struct Ready;

struct DatabasePool<S> {
    connection_string: String,
    _state: PhantomData<S>,  // tells the compiler S matters, takes up zero space at runtime
}

impl DatabasePool<Uninitialized> {
    fn new(conn: &str) -> Self { /* ... */ }
    fn initialize(self) -> DatabasePool<Ready> { /* ... */ }
}

impl DatabasePool<Ready> {
    fn query(&self, sql: &str) -> Result<Rows> { /* ... */ }
}
```

There are a few things going on here. Each state gets its own `impl` block, so `.query()` simply doesn't exist on `DatabasePool<Uninitialized>`. If an agent tries to call it, the compiler rejects it as a compile error, not a runtime panic. And `initialize` takes `self` by value, consuming the `Uninitialized` version and returning a `Ready` one. After that call, the old value is gone. You can't accidentally keep using an uninitialized pool.

The states are separate structs rather than enum variants because that's what keeps this at compile time. An enum would mean the state is determined at runtime and you'd be back to matching on variants. With marker structs and [`PhantomData`](https://doc.rust-lang.org/std/marker/struct.PhantomData.html), the state parameter exists only for the type checker and compiles away to nothing.

Construction is controlled through factory methods and state transitions. There's no way to conjure a `DatabasePool<Ready>` out of thin air, you have to go through `.initialize()`.

The same pattern works at the business logic level. A `User<Unauthenticated>` can call `.login()`, which returns a `User<Authenticated>`. Only `User<Authenticated>` exposes `.view_dashboard()` or `.place_order()`. An agent cannot generate a handler that serves protected content to an unauthenticated user, because it won't compile.

This scales to anything with a state machine: HTTP connections that must be opened before sending, crypto contexts that need keys loaded before encrypting, payment flows that must be authorized before capturing. These are bugs that traditionally hide behind runtime checks and get discovered in production. Typestates surface them at compile time, and for agents, they encode exactly when specific operations are legal directly into the API.

The obvious tradeoff is compile times. Rust compiles slowly, and for an agent iterating rapidly, every compile cycle is latency. This is real, especially in early development when the agent is exploring solution space. Using `cargo check` instead of full compilation helps since it skips codegen, but it's still slower than something like TypeScript's `tsc`.

## The Core Argument

An agent-built codebase without deterministic guarantees is a half-baked product. If there's no compile-time type safety, no exhaustive matching, no static analysis gates, you're shipping the output of a probabilistic system with no verification that it's correct. Languages like TypeScript with strict mode can go a long way here, and for many projects that's enough.

Rust lets you go further. The borrow checker, ownership semantics, newtypes, typestates, and [fearless concurrency](https://doc.rust-lang.org/book/ch16-00-concurrency.html) push more of your application logic into compile-time checks, turning runtime errors into impossible states. The agent ends up generating code that the compiler has verified is correct, not just code that happens to work.
