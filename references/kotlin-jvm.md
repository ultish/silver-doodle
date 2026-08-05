# Kotlin/JVM Translation of the Performance Principles

The hardware-level reasoning in SKILL.md (cache/memory/disk/network costs,
algorithms-first, avoid-unnecessary-work) is unchanged on the JVM — same
hardware underneath. What differs is the toolchain and a handful of
JVM/Kotlin-specific costs the original C++-oriented advice has no equivalent
for. Apply this file as an overlay on top of SKILL.md, not a replacement.

## Profiling tools (JVM equivalents)

| Original (C++) | JVM/Kotlin equivalent |
|---|---|
| pprof | async-profiler, or a sampling profiler in a JVM APM tool |
| perf / hardware counters | async-profiler with `-e cache-misses`, or `perf` attached to the JVM process directly |
| Microbenchmarks | **JMH** (Java Microbenchmark Harness) — do not hand-roll timing loops; JIT warmup, dead-code elimination, and constant-folding will produce misleading numbers without it |
| — | **JFR** (Java Flight Recorder) — low-overhead, always-on profiling built into the JVM; good first stop for production hotspot and allocation-rate data |

Flat-profile advice still applies: if nothing stands out, look at allocation
rate (JFR's allocation profiling view) and GC pause frequency/duration before
assuming there's no easy win.

## API design

- **Batch operations** — unchanged; still the highest-leverage API-level
  change (JNI boundary crossings, JIT-friendly loops, amortized dispatch).
- **View types** — the weak spot. `String` in the JVM is always a full
  heap object; there's no zero-copy `string_view`. Use `CharSequence` as a
  parameter type where possible to avoid forcing a `String` allocation at
  the call site. `List.subList()` is view-like (backed by the original list)
  but easy to leak or accidentally copy — document which behavior a function
  needs. Kotlin's `Array`/`List` slicing (`slice()`, `take()`) *copies*;
  it is not a view.
- **Thread-compatible by default** — same principle, extended to coroutines:
  default to dispatcher-confined state rather than reaching for `Mutex` or
  `synchronized`. Structured concurrency (a `CoroutineScope` per component)
  is usually a substitute for synchronization, not an addition to it.

## Memory representation

- JVM object headers are 12–16 bytes regardless of field layout, so
  hand-ordering fields to minimize padding (as in C++) has much less payoff.
  Don't spend effort there.
- **Boxing is the primary JVM-specific tax**, with no analogue in the
  original doc. Generic collections box primitives: `List<Int>` allocates a
  boxed `Integer` per element; `IntArray`/`LongArray`/`DoubleArray`/etc. do
  not. In hot paths — large collections, tight loops — prefer primitive
  arrays over `List<Int>`/`List<Long>`/`List<Double>`.
- **`@JvmInline value class`** is the closest JVM equivalent to a
  zero-cost abstraction: wraps a single primitive or `String` with no
  runtime allocation in the common case (the wrapper is erased at compile
  time when possible). Use it for the "specialized code without giving up
  type safety" pattern instead of a full wrapper class.
- Compound-key maps (`Map<Pair<A, B>, C>` or a dedicated key data class)
  still beat nested `Map<A, Map<B, C>>` for the same reasons as the
  original doc — fewer allocations, fewer lookups, better locality.
- No arena allocators in idiomatic Kotlin/JVM. Object pooling can substitute
  in specific hot paths, but pools add complexity (lifecycle bugs, GC
  interaction) — only use them where profiling justifies it.
- Flat vs. pointer-rich: `ArrayList`/`ArrayDeque` over `LinkedList`, in
  general — `LinkedList` in Kotlin/Java has poor cache locality and rarely
  wins even where it looks like the "right" data structure.

## Reducing allocations

This section needs the most Kotlin-specific expansion — the JVM's GC makes
allocation rate a first-class performance metric, not just an allocator-time
cost.

- **Lambda capture allocates.** A lambda that captures a variable becomes a
  heap-allocated closure object unless the enclosing function is `inline`.
  Marking a higher-order function `inline fun foo(block: () -> T)` eliminates
  both the call overhead *and* the closure allocation — this single Kotlin
  feature covers what the original doc treats as two separate techniques
  ("avoid function calls" + "reduce allocations"). Prefer `inline` for
  small, hot higher-order functions; use `noinline`/`crossinline` only where
  the lambda must escape or cross a non-local-return boundary.
- **Eager collection chains allocate an intermediate collection per stage.**
  `list.map { }.filter { }.map { }` allocates three lists. `asSequence()`
  makes the chain lazy and single-pass (matches the "deferral" principle),
  at the cost of per-element dispatch overhead — worth it for large
  collections or when a chain short-circuits (`first { }`, `take(n)`);
  not worth it for small, fixed-size collections where the eager version is
  simpler and the JIT handles it fine.
- **`data class copy()` and generated `equals`/`hashCode`** on large data
  classes can be unexpectedly expensive in a hot path — same "reuse instead
  of recreate" principle as the original doc's temporary-object advice.
- **Coroutines are not free.** Every `suspend` function compiles to a state
  machine; a real suspension can allocate a `Continuation`. This matters for
  both "reduce allocations" and "help the compiler" — don't mark a function
  `suspend` reflexively in a hot path if it never actually suspends.
- Avoid `vararg` parameters in hot call sites — each call allocates an array
  to hold the arguments.
- String concatenation with `+` in a loop allocates a new `String` per
  iteration; use `StringBuilder` (or `buildString { }`) instead — the same
  "hoist and reuse" principle as the original doc.

## Compiler/JIT assistance

- Kotlin classes are **`final` by default** (the opposite of Java) — this is
  a direct, free win for the "help the compiler devirtualize calls"
  principle. Only mark a class `open` when subclassing is actually needed;
  every unnecessary `open` reintroduces a virtual-dispatch site the JIT
  can't easily monomorphize.
  ​
- `inline fun` is the direct tool for "avoid function calls in hot loops" —
  see Reducing Allocations above; it solves both problems from the original
  doc simultaneously.
- Watch for megamorphic call sites (an interface implemented by many classes,
  called from one hot loop) — these defeat JIT inline caching the same way
  they defeat devirtualization in C++.

## Stats and logging

Same advice as the original doc, with a Kotlin-specific implementation:
logging libraries built for Kotlin (e.g., kotlin-logging) commonly expose
`inline fun debug(msg: () -> String)` — the message-building lambda is
inlined and only evaluated if the log level is active, so a disabled debug
log costs a single boolean check, not a string build. Prefer this pattern
(or manually guard with `if (logger.isDebugEnabled)`) over
`logger.debug("$expensiveValue")`, which always builds the string regardless
of whether it gets logged.

## Quick checklist for a Kotlin performance pass

- [ ] Any `List<Int>`/`List<Long>`/`List<Double>` etc. in a hot path that
      could be a primitive array instead?
- [ ] Any hot higher-order function that could be `inline`?
- [ ] Any eager `.map{}.filter{}` chain on a large collection that could be
      `asSequence()`, or vice versa (an unnecessary sequence on a small
      collection)?
- [ ] Any `suspend fun` in a hot path that never actually suspends?
- [ ] Any class marked `open` without an actual subclass?
- [ ] Any string concatenation in a loop that should be `buildString { }`?
- [ ] Any log call building a string eagerly instead of behind a lambda/level
      check?
- [ ] Any microbenchmark written as a hand-rolled timing loop instead of JMH?
