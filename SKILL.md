---
name: performance-optimization
description: This skill should be used when the user asks to "optimize this code", "make this faster", "reduce allocations", "improve performance", "review for performance issues", "why is this slow", "reduce GC pressure", "profile this", or asks about algorithmic complexity, memory allocation patterns, batch API design, or performance tradeoffs generally — including Kotlin/JVM-specific concerns like boxing, inline functions, coroutine overhead, or value classes.
version: 0.1.0
---

# Performance Optimization

Apply Google's internal performance-engineering principles (Jeff Dean and Sanjay
Ghemawat's "Performance Tips," published via Abseil at
https://abseil.io/fast/hints.html) to code review, design, and optimization work.
The core thesis: "premature optimization is the root of all evil" applies to
roughly 97% of code, but the remaining 3% warrants deliberate attention during
initial design, not just after profiling flags it.

For JVM/Kotlin codebases, translate these principles using
`references/kotlin-jvm.md` — several techniques (view types, memory layout)
work differently on the JVM, and Kotlin introduces its own hazards (boxing,
lambda capture, coroutine state machines) that the original hardware-level
advice doesn't cover.

## Why think about performance early

Three reasons to consider performance during initial design rather than
deferring entirely to post-hoc profiling:

1. Ignoring performance broadly produces flat profiles with no obvious single
   hotspot — nothing to point a profiler at, just death by a thousand cuts.
2. Callers of library code often cannot optimize around a slow internal
   implementation; the cost is paid by everyone who depends on it.
3. Design changes get harder to make once a system is in heavy production use
   — data formats, APIs, and storage layouts calcify.

## Estimate before implementing

Know approximate operation costs well enough to reason about design
tradeoffs without writing code first:

| Operation | Approx. cost |
|---|---|
| L1 cache reference | 0.5 ns |
| Main memory reference | 50 ns |
| Disk seek | 5,000,000 ns |
| Network round trip (same datacenter) | 50,000 ns |

Use these for back-of-the-envelope calculations when comparing design
alternatives — e.g., "this adds one disk seek per request" is often decisive
without needing to prototype both options.

## Profile before micro-optimizing

Measure before guessing which part is slow:

- **pprof** (or equivalent sampling profiler) for high-level hotspot location.
- **perf** (or hardware counters generally) for cache-miss, branch-misprediction,
  and instruction-level detail.
- **Microbenchmarks** for tracking regressions and turnaround time on a
  specific hot function — not a substitute for end-to-end profiling.

If a profile comes back flat (no single hotspot), that itself is a signal:
look for many small ~1% costs to accumulate improvements on, inspect loops
higher in the call stack, check memory allocation/GC patterns, and pull
hardware performance-counter profiles for cache and branch behavior.

## Design APIs for performance

- **Batch operations.** Bulk APIs amortize per-call overhead (locking,
  virtual dispatch, boundary crossings) and open up algorithmic improvements
  that a one-at-a-time API structurally forecloses.
- **View types for arguments.** Accept read-only views (`std::string_view`,
  `absl::Span<T>`) rather than forcing copies into a specific container type;
  let the caller choose their own representation. On the JVM this needs
  adaptation — see `references/kotlin-jvm.md`.
- **Thread-compatible by default.** Most general-purpose types should be
  thread-*compatible* (safe under external synchronization) rather than
  internally thread-safe. Forcing synchronization on every caller taxes the
  common single-threaded case to support the rare concurrent one.

## Algorithms first

The largest wins usually come from algorithmic change, not tuning:

- Replace O(N²) with O(N log N) or O(N) where the data structure allows it.
- Eliminate accidentally exponential behavior (e.g., repeated re-computation
  without memoization).
- Swap data structures for the access pattern actually used — hash tables
  instead of sorted lists for lookup-heavy code, for instance.
- Batch instead of incrementally updating when updates are frequent and the
  incremental cost compounds (e.g., adding graph nodes in reverse
  post-order to eliminate expensive per-edge cycle-detection work).

## Choose compact, flat memory representations

- Order struct/class fields to reduce padding; use smaller numeric types
  when the range allows; separate hot fields from cold ones so hot data
  packs into fewer cache lines.
- Prefer `InlinedVector`-style types (small-buffer-optimized collections)
  for collections that are typically small.
- Replace nested containers with a compound key: a map keyed on `(a, b)` is
  usually better than a map of maps keyed on `a` then `b`.
- Use bit vectors or arrays instead of hash sets/maps when the domain is
  small or enum-like.
- Use arenas for complex structures with many sub-objects, when the target
  language supports them.
- Prefer flat/chunked structures (vectors, flat hash maps) over pointer-rich
  ones (linked structures, node-based maps/sets) — better cache locality,
  less allocator overhead, fewer pointer chases.

JVM-specific caveats (object headers, boxing, lack of arenas) are in
`references/kotlin-jvm.md`.

## Reduce allocations

Every allocation costs more than the allocator call itself: initialization,
destruction, and scattering related data across cache lines. Mitigate with:

- Pre-size containers (`resize()`/`reserve()` or language equivalent) when
  the final size is known or estimable.
- Hoist temporary objects out of loops instead of recreating them each
  iteration; reuse them across iterations.
- Avoid unnecessary copies; prefer moves where the language supports them.
- Use static/shared allocation for values needed repeatedly rather than
  re-deriving or re-allocating them.

## Avoid unnecessary work

- **Fast paths for common cases.** Structure code so the common, simple case
  is fast, without penalizing the rare, complex case — e.g., a specialized
  path for small inputs or ASCII-only strings.
- **Precomputation.** Compute expensive derived information once (e.g., a
  bit-flag summary of a node's type) instead of recomputing it on every use.
- **Deferral.** Delay expensive computation until it's actually needed rather
  than computing eagerly on the chance it might be.
- **Specialized code over generality.** Replace a general-purpose library
  call with a narrower, purpose-built implementation at performance-critical
  call sites that don't need the library's full generality.
- **Caching.** Store results of expensive computations; check the cache
  before recomputing.
- **Array-based fast rejection.** Use a cheap array/bitmap lookup (e.g.,
  filter by first byte) to quickly reject non-matches before running an
  expensive full comparison.

## Help the compiler help you

- Avoid function calls in the hottest inner loops where inlining matters.
- Move slow-path code into a separate, typically tail-called function so the
  hot path stays small and inlinable.
- Copy small pieces of data into local variables to make alias analysis
  easier for the optimizer.
- Hand-unroll only the very hottest loops, and only after measuring.
- Replace abstraction layers (e.g., span/view types) with raw pointers in the
  rare call site where the abstraction's overhead is proven to matter.

JVM/Kotlin equivalents (inlining, `final`-by-default classes, devirtualization)
are covered in `references/kotlin-jvm.md`.

## Stats and logging shouldn't tax the hot path

- **Sample instead of tracking every element** when maintaining statistics —
  a representative sample is usually sufficient and much cheaper.
- **Remove logging from hot paths entirely** rather than relying on a log
  level check alone — even a disabled log statement can cost real work if
  its arguments are still evaluated eagerly. Precompute whether a log level
  is active outside the loop rather than checking it per iteration.

## Applying this skill

When asked to optimize code or review it for performance:

1. Ask whether the bottleneck is known (profiled) or assumed. If assumed,
   profile first — don't guess.
2. Check for an algorithmic improvement before reaching for micro-optimizations;
   it's almost always the higher-leverage change.
3. Walk the sections above in order (API design → data structures →
   allocations → unnecessary work → compiler-level tuning) and note which
   apply to the code in question.
4. For Kotlin/JVM code, cross-check against `references/kotlin-jvm.md` for
   boxing, lambda-capture allocation, coroutine overhead, and other
   JVM-specific costs that don't appear in a language-agnostic pass.
5. Prefer the smallest change that fixes the actual measured cost; don't
   restructure unrelated code under the banner of performance.

## Additional resources

- **`references/kotlin-jvm.md`** — how each principle above translates to
  Kotlin/JVM: boxing, `inline` functions, value classes, coroutine state
  machines, sequences vs. eager collections, `final`-by-default dispatch,
  and JVM profiling tools (JFR, async-profiler, JMH).
