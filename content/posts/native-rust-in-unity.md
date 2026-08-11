+++
date = '2026-08-10T12:00:00+02:00'
draft = false
title = 'Using Rust Code from Unity for High Performance'
description = "Unity gives you several ways to write fast code, and one more over the same pointers Burst uses. Measured across every Unity runtime, the speed gap turns out to depend more on how old the runtime is than on which language you picked, and most of what looks like a language difference is effort spent unevenly."
tags = ['rust', 'unity', 'ffi', 'performance', 'benchmarks']
ShowToc = true
TocOpen = true
+++
Unity gives you several ways to turn C# into machine code. This post is about one it does not ship: a Rust library, called from C# over the same pointers Burst already uses.

The interop is about a dozen lines. On the Mono that Unity desktop games ship by default, a game AI workload runs 6.4 times faster in Rust. Against IL2CPP it is 4.7 times faster, and against Unity's experimental CoreCLR backend and plain .NET 10 about 2.3 times.

The more durable argument is memory. In every run, on every runtime, the Rust engine allocated zero bytes and the collector never touched it. A collector cannot walk memory it cannot see, and that stays true however good the runtime gets.

## How your C# becomes machine code

Your C# does not run as C#. Something has to turn it into machine code first, and Unity offers several options.

- **Mono** compiles your code while the game runs, a piece at a time, the first time each piece is needed.
- **IL2CPP** compiles ahead of time: it converts your C# into C++ and hands that to a normal C++ compiler.
- **CoreCLR** is the runtime modern .NET uses, a JIT twenty years younger than Mono, and Unity has an experimental backend for it.
- **Burst** compiles a restricted flavor of C# into native code through LLVM, the same compiler machinery behind Rust and Clang.
- **Rust** is the one this post adds: native code before you ship, and no restrictions on what you can write.

The first three are *managed*, which means a garbage collector owns your memory. You never free anything. Instead, every so often, the collector walks through everything you have allocated, works out what is still in use, and throws away the rest. While it works, your game waits.

The last two have no collector. Burst avoids one by forbidding you to allocate the kind of memory that needs it. Rust avoids one because its memory belongs to Rust, and Unity's collector never learns it exists.

{{< animsvg src="/images/posts/rust-unity/landscape.svg" alt="Columns comparing Mono, IL2CPP, CoreCLR, Burst and Rust: how each turns source into machine code, whether garbage is collected, and what you are allowed to write" >}}

The managed runtimes let you write anything and attach a collector. Burst removes the collector and takes away most of the language. Rust removes the collector and keeps the language.

## What Burst makes you rewrite

Burst comes with restrictions: your data has to be flat arrays of [blittable](https://learn.microsoft.com/en-us/dotnet/standard/native-interop/blittable-and-non-blittable-types) values: types laid out the same way in managed and native memory, so integers, floats and structs of them. Nothing can grow while a parallel job runs. Containers cannot hold other containers.

For a loop that multiplies a million floats, none of that is a problem, and Burst will beat anything you write by hand. The trouble starts with code whose natural shape is not a flat array.

Take the system this post benchmarks, a utility AI, the pattern game characters use to pick what to do next. Every candidate action gets a score from a set of scorers, and each scorer reads one input and pushes it through a response curve. The natural C# is an interface with a class per curve, and adding a curve means adding a class. Burst rejects the interface, every class, the array that holds them, and even the plain `float[]` holding a character's stats. To get in, the polymorphism has to be re-encoded as a byte tag and a switch, which is a different program that happens to compute the same answer.

{{< animsvg src="/images/posts/rust-unity/burst-rewrite.svg" alt="Left: the natural C#, an IScorer interface with four curve classes under it and an IScorer array, where a new curve is one new class, and Burst rejects all of it. Right: what Burst accepts, a flat struct with a byte curve tag and a switch over four cases, where a new curve is a new case in every switch" >}}

That rewrite is the price of entry, and the restriction follows every helper the job calls. It also removes the extension point: a new curve was a new class, and becomes a new case in a switch every caller pays for.

## A dozen lines of interop

Burst is fast partly *because* it works on raw pointers into plain native memory. A `NativeArray` is a block of ordinary memory with a safety wrapper around it, and Burst compiles down to code that reads and writes that block directly by address.

C# can pass those same addresses to a native library through P/Invoke. The only requirement is that the data is blittable, the same constraint Burst already puts on anything you hand a job.

This is the dozen lines from the intro, the entire C# side of the boundary:

```csharp
[DllImport("libutility_ai", CallingConvention = CallingConvention.Cdecl)]
static extern int ai_score(
    ref float needs,
    ref Character characters, int characterCount,
    ref ActionDef actions, int actionCount,
    ref Scorer scorers, int scorerCount,
    ref uint outAction, ref float outScore);

// the whole call: first-element addresses, nothing copied
Native.ai_score(ref needs[0], ref chars[0], chars.Length,
                ref actions[0], actions.Length,
                ref scorers[0], scorers.Length,
                ref outAction[0], ref outScore[0]);
```

You do not need `NativeArray` for this. A plain managed array of blittable values is the same flat block of memory, just on the C# heap, and a reference to its first element pins it in place for the duration of the call. Every number in this post comes from ordinary `float[]` and struct arrays crossing the boundary that way.

The limit is in that phrase: the duration of the call. These calls are synchronous, so the pin ends when the call returns and nothing more is needed. If you copy Burst's async pattern instead, schedule the work, do something else, complete it later, the pointer has to stay valid the whole time, and a compacting collector is free to move an unpinned array whenever it likes. Unity's current Boehm collector [never moves objects](https://docs.unity3d.com/6000.1/Documentation/Manual/performance-garbage-collector.html), so this rarely bites today, but CoreCLR compacts. Then you pin explicitly with `GCHandle`, allocate long-lived buffers on the [Pinned Object Heap](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/poh) so the pins do not fragment the heap, or keep the buffer on the Rust side of the boundary, which is the simplest answer. Rust's own memory is invisible to the collector; the managed buffers you lend it are not.

{{< animsvg src="/images/posts/rust-unity/ffi-boundary.svg" alt="C# passes three addresses across the P/Invoke boundary. The managed heap is drawn as one contiguous block of memory with the needs, scorers and results arrays as ranges inside it, and Rust's three slices each point at the start of their range. Rust writes the results range in place and returns a single status code" >}}

One call goes out carrying a few addresses. Rust wraps them as slices, spreads the work across every core, and writes the answers into the buffers C# already owns. The buffers never move, and C# reads its results out of the same memory it handed over.

The receiving side is the mirror image, lightly trimmed from the repository:

```rust
#[unsafe(no_mangle)]
pub unsafe extern "C" fn ai_score(
    needs: *const f32,
    characters: *const Character, character_count: i32,
    /* actions, scorers, and the two out pointers */
) -> i32 {
    let nc = character_count as usize;
    let needs = slice::from_raw_parts(needs, nc * INPUTS);
    let chars = slice::from_raw_parts(characters, nc);
    // ...the rest wrapped the same way, then rayon fans out per character:
    out_action.par_iter_mut().zip(out_score.par_iter_mut())
        .zip(chars).zip(needs.par_chunks_exact(INPUTS))
        .for_each(|(((a, s), ch), n)| {
            let (best, score) = best_action(n, ch, actions, scorers);
            *a = best; *s = score;
        });
    0
}
```

One rule applies at this edge: a Rust panic must not reach it, because a panic crossing `extern "C"` aborts the whole player with no Unity error log. Catch it at the boundary with `std::panic::catch_unwind` and turn it into the error code C# already checks.

The call itself costs tens of nanoseconds. When the work behind it takes microseconds, the boundary is a rounding error. That fixed cost is also why you ask for a lot at once: the benchmark scores all two hundred characters in one call rather than making two hundred calls.

None of this is specific to Rust. Any language that can build a C-compatible library can stand on the other side of the boundary, and C++ works the same way. This post uses Rust because the point of leaving managed code is taking manual control of memory, and Rust lets you do that without a new class of crashes.

It is not a new pattern either. In the browser it is Rust compiled to WebAssembly: JavaScript keeps the orchestration and a compiled module takes the hot loop. That is how Mozilla [sped up its source-map library](https://hacks.mozilla.org/2018/01/oxidizing-source-maps-with-rust-and-webassembly/), how Prime Video [runs its UI engine on low-powered devices](https://www.amazon.science/blog/how-prime-video-updates-its-app-for-more-than-8-000-device-types), and how 1Password [ships its core inside a browser extension](https://1password.com/blog/1password-8-the-story-so-far). The Unity version gets a cheaper boundary, though. WebAssembly runs in its own linear memory, so the JavaScript side usually pays a copy on the way in. P/Invoke hands over addresses in the same address space, and Rust reads the heap in place.

## The benchmark: a utility AI

The workload is the utility AI from earlier: two hundred characters, a thousand candidate actions each, six scorers per action.

{{< animsvg src="/images/posts/rust-unity/utility-anatomy.svg" alt="One character with its needs as bars feeds one action holding six scorers. Each scorer shows its response curve shape, linear, quadratic, logistic or gaussian, its input and its weight. The scorer outputs multiply into one score, one of a thousand, and the highest wins" >}}

That is 1,200,000 scorer evaluations per tick against 160 KB of scorer and action data, just past the test machine's 128 KB first-level cache and well inside the second. Both sides split the characters across the same six threads, Rust through rayon and C# through `Parallel.For`.

Both sides stay in safe code: no `unsafe` in the Rust, no `Unsafe.*` in the C#. The C# is an interface with a class per curve, what you find in a real codebase:

```csharp
public interface IScorer
{
    float Score(CharacterObj ch, in ActionDef action);
}

public sealed class LinearScorer : IScorer
{
    public int Input; public float M, B, C, Weight;
    public float Score(CharacterObj ch, in ActionDef a)
    {
        float x = Engines.ReadInput(Input, ch, in a);
        return Math.Clamp(M * (x - C) + B, 0f, 1f) * Weight;
    }
}
// three more classes: Quadratic, Logistic, Gaussian
```

The Rust is a data-carrying enum matched directly, which is what a Rust programmer writes when the variants are known up front. Each variant carries only the fields its curve reads, and the type system stops anyone touching the others:

```rust
pub enum Curve {
    Linear { m: f32, c: f32, b: f32 },
    Quadratic { m: f32, c: f32, b: f32 },
    Logistic { k: f32, c: f32 },
    Gaussian { k: f32, c: f32 },
}

fn curve_enum(c: &Curve, x: f32) -> f32 {
    let v = match *c {
        Curve::Linear { m, c, b } => m * (x - c) + b,
        Curve::Quadratic { m, c, b } => { let d = x - c; m * d * d + b }
        Curve::Logistic { k, c } => 1.0 / (1.0 + (-k * (x - c)).exp()),
        Curve::Gaussian { k, c } => { let d = x - c; (-k * d * d).exp() }
    };
    v.clamp(0.0, 1.0)
}
```

Neither is a translation of the other, which is the point: each language does this job the way its own practitioners write it.

The idiomatic C# is also not the slow choice. On Unity's CoreCLR it beats a hand-flattened version with a switch statement, 1.158 milliseconds against 1.327, because the JIT watches which implementation turns up at each call site and compiles the indirection away. On Mono and IL2CPP the switch wins by a lot. That is a property of the newer JIT rather than of C#. Know which runtime you are on before you hand-flatten anything.

## Making the comparison fair

A benchmark where one side is tuned and the other is not measures the author, not the languages. The C# engine in this benchmark got 35% faster after it received the same flat data layout the Rust side already had, and none of that 35% had anything to do with the language. The rest was measurement error: the machine, the warm-up, a stray counter in the timed loop.

Both implementations are held to the same standard. They must do provably identical work: 200 characters × 1,000 actions × 6 scorers, every time, by construction. Every run checks that both languages pick the same action and score for all two hundred characters before it reports a time. Any optimization that wins on one side is only a hypothesis for the other until the other side tries it. The full protocol is in [the repository](https://github.com/oddur/blog-unityrust).

Three measurement details matter. This chip has six performance cores and twelve efficiency ones, so work spread across all eighteen swung by a factor of four between runs, and everything here is capped to six threads. Managed code needs a warm-up: the tiered JIT runs up to 25% slow over the first couple of hundred batches while it recompiles the hot code. Every number is therefore the median of warm back-to-back batches, with the early ones discarded. And nothing is compared across processes: Rust and C# are timed in the same program on the same data, with the identical native library landing within 2% across all six managed hosts as the control.

## The results, runtime by runtime

{{< animsvg src="/images/posts/rust-unity/utility-results.svg" alt="Bar chart of the utility AI scorer across six runtimes: Rust 0.49 ms on every runtime, Mono C# 3.15 at 6.4x, Mono incremental 3.20, IL2CPP 2.30 at 4.7x, IL2CPP incremental 2.32, Unity CoreCLR 1.16 at 2.3x, standalone .NET 10 1.13 at 2.3x" >}}

Read the chart downward: the gap tracks runtime age as much as language. Mono is a twenty-year-old JIT and loses by 6.4x. IL2CPP compiles ahead of time and loses by 4.7x. CoreCLR, a modern JIT with profile-guided optimization, loses by 2.3x, and plain .NET 10, the closest thing to Unity's future, matches it.

The gap shrinks as the runtime modernizes and stops at 2.3x: that is what remains after the runtime has caught up.

## Beating Burst with safe SIMD

Everything above compares Rust against C#. Unity's own answer for hot code is Burst. It is free, it ships with the engine, and it is fast: the same scorer in a Burst job runs at 0.147 ms against the plain C#'s 2.30.

So the question is whether Rust can match that. It can, in safe code, and it comes out ahead.

Burst's speed comes from SIMD, so first, what that is. A processor normally works on one number at a time. SIMD applies one instruction to several numbers at once: four floats multiplied by four others in one register, in roughly the time one multiply takes.

{{< animsvg src="/images/posts/burst/simd-wide.svg" alt="Top: four multiplies done one after another in four steps. Bottom: the same four operands packed into three registers and multiplied in one step" >}}

There are conditions. The four numbers have to sit together and they all have to want the same operation. The moment the code asks a question about one and not the others, it falls back to scalar.

So the scorer was measured four ways, in the same IL2CPP player configuration on the same six threads. Each language appears twice: once written normally, one score at a time, and once with the four-at-a-time version written by hand. The Burst rows use `Unity.Mathematics` and its `float4` type. The Rust rows are the enum engine and a four-wide rewrite of it.

{{< animsvg src="/images/posts/rust-unity/simd-results.svg" alt="Bar chart in two groups. Written normally: Rust scalar 0.491 ms, Burst scalar 0.703. Hand-written four wide: Burst float4 0.147 at 4.8x its own scalar, Rust four wide safe 0.135 at 3.6x its own scalar with no unsafe" >}}

The first thing that chart says is that **neither compiler vectorized anything on its own**. Burst's pitch is that it finds the loops and widens them for you. Here it did not: written as ordinary scalar code inside a Burst job, it ran at 0.703 ms. Rust was no better, with five vector instructions in the whole scoring function and all of them register moves. The four-way branch on the response curve is what stops both.

That matches my experience with Burst beyond this benchmark. The promise is auto-vectorization out of the box, and in practice it delivers inconsistently. You write the code, check the Burst Inspector to see what the compiler produced, adjust, and check again, until the vectorizer emits the SIMD you were after. You end up wrestling an abstraction layer that sits between you and instructions you already know you want, and you reach the goal by trial and error rather than by stating it. In Rust, with the right crate, you state it: the SIMD operations compile to the instructions they name, exactly where you put them, with no translation layer to persuade.

The four-wide versions had to be written by hand on both sides, and the trick is not the obvious one. Four scorers in a register fails, because each can be a different curve, so every lane computes all four kinds and discards three. What works is four *characters*, who share one scorer and therefore one curve and one set of constants.

{{< animsvg src="/images/posts/burst/simd-lanes.svg" alt="Animated comparison scoring the same eight characters with the same scorer. The scalar side fills one character per pass and takes eight passes. The four-wide side fills a register of four characters per pass and is done in two, because the lanes share one scorer and its curve and constants" >}}

Once both sides are written that way, Rust comes out about 8% ahead, and every character still picks the same action as the scalar engines.

The Rust version is also **safe code**. Stable Rust does not ship `std::simd` yet, but the [`fearless_simd`](https://crates.io/crates/fearless_simd) crate reaches the real instructions safely, with no `unsafe` anywhere in the engine. And it is recognizably the `curve_enum` from the benchmark section, gone four wide:

```rust
fn curve_fs<S: Simd>(s: S, cv: &Curve, x: f32x4<S>) -> f32x4<S> {
    let v = match *cv {
        Curve::Linear { m, c, b } => {
            s.splat_f32x4(m) * (x - s.splat_f32x4(c)) + s.splat_f32x4(b)
        }
        Curve::Gaussian { k, c } => {
            let d = x - s.splat_f32x4(c);
            exp_fs(s, -s.splat_f32x4(k) * d * d)
        }
        // ...
    };
    s.min_f32x4(s.max_f32x4(v, s.splat_f32x4(0.0)), s.splat_f32x4(1.0))
}
```

That is an ordinary `match` on a data-carrying enum. It runs once and serves all four lanes. Around it sit a `&[ScorerEnum]` slice and rayon on the outer loop, none of which can exist inside a Burst job.

SIMD in Rust is a type you reach for in one expression. Burst is a mode you enter, and entering it means the rewrite from earlier. It also means no allocation: Burst code runs outside the runtime's control, so the managed heap is out of reach.

If you have already laid your data out flat to feed a Burst job, you have done the work needed to hand it to Rust. That layout is not a Burst tax or a Rust tax. It is the cost of caring about performance at all, and the hand-flattened C# ended up with the same flat arrays without either compiler asking. What differs is what you are allowed to write around it.

## Zero allocations, zero collections

Over a thousand batches on every runtime, the Rust engine allocated zero bytes and triggered zero collections. That is not a tuning result, it is structural: Rust's memory is invisible to Unity's collector, so there is nothing for it to walk.

The C# side allocates 4 to 13 kilobytes per batch. The scoring engine is not responsible. It is the `Parallel.For` machinery around it, and it is still enough to trigger real collections.

{{< animsvg src="/images/posts/rust-unity/gc-tails.svg" alt="Collections over 1,000 batches: Rust 0 on every runtime with zero bytes allocated, Unity CoreCLR C# 0, Mono C# 4, IL2CPP C# 6, Mono incremental 13, IL2CPP incremental 24" >}}

Incremental collection does what it promises, three to four times as many collections, each smaller. Here it moved the median by 1 to 7% and barely touched the tail, because the workload does not allocate enough for the collector to become the tail. Systems that allocate more per frame will see a different trade.

This workload was built to be allocation-light, so it is close to the best case for the managed side. The worst case is the one every Unity developer already knows: a system that allocates per entity per frame, and a collection that arrives in the middle of one.

## Shipping it to every platform

The benchmark ran on one machine, but I have shipped this pattern to Windows, macOS, Linux, Android and iOS. The crate builds as a `cdylib`, which is a `.dll`, `.dylib` or `.so` depending on the platform, and the result goes into `Assets/Plugins`. iOS wants a `staticlib` linked into the player instead, with the `DllImport` name set to `__Internal`.

[csbindgen](https://github.com/Cysharp/csbindgen) generates the C# bindings from the Rust exports on every build, iOS conditional included, which removes the classic FFI hazard of the two sides drifting apart. Both crates in the repository work this way.

None of this is much work, but it is work, and it is the part Burst saves you.

## Living with it

Two things change about your day once a Rust library is in the project, and neither shows up in a benchmark.

**The editor holds on to the library.** Unity loads a native plugin on first use and never unloads it, so picking up a new Rust build usually means restarting the editor. Burst recompiles in place. This is the cost you feel most if you iterate on the native side all day. It pushes the work toward the crate's own tests: `cargo test` runs in seconds, and the editor round-trip is saved for integration.

**Rayon and Unity's job system do not know about each other.** Left alone, rayon sizes its pool to every core in the machine, and Unity's workers assume the same cores are theirs. Two schedulers fighting over eighteen cores is how you get a smooth benchmark and a stuttering game. Cap the pool once at startup, which is what the repository's `ai_init_threads` is for, and treat the thread count as part of your frame budget rather than a default.

Neither is hard, both are real, and Burst has no equivalent of either.

## What this adds up to

If a system eats your frame budget, the move to Rust costs about a dozen lines of interop and a build step. What you get back:

- **Speed on the runtimes games actually ship on.** 6.4x over Mono and 4.7x over IL2CPP today, and 2.3x once Unity's CoreCLR future arrives.
- **A higher ceiling when you need it.** Hand-written SIMD in safe Rust beats Burst by 8 to 13% on this workload, without giving up interfaces, collections or allocation to get there.
- **A system the garbage collector cannot touch.** Zero bytes allocated, zero collections, on every runtime, by construction rather than by discipline.
- **Code that outlives the engine.** The same library runs in the game, in a .NET service, and on a server with no engine at all, and its tests run without opening an editor.

Reach for Burst instead when the work is already flat numeric loops over arrays: it is free, it needs no boundary, and it is superb at exactly that. Reach for Rust when the system has real structure, interfaces, growing collections, code a designer reads, and you want it fast anyway.

The numbers here come from Unity 6 on Apple Silicon, six threads on the performance cores. The methodology, every implementation, and the raw results are in the companion repository: [github.com/oddur/blog-unityrust](https://github.com/oddur/blog-unityrust).

## AI disclosure

This was done as a research project with AI in the loop. A blend of Claude Opus and Claude Fable wrote the harnesses, ran the benchmarks, and chased down the wrong turns, and the article was written with their help as well. Every number went through the validation described above, and the code and raw results are public in the repository.
