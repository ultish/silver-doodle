# performance-skill

A Claude Code skill for performance optimization work, based on Google's
internal "Performance Tips" (Jeff Dean and Sanjay Ghemawat), published via
Abseil at [abseil.io/fast/hints.html](https://abseil.io/fast/hints.html).

## What's here

- **[`SKILL.md`](SKILL.md)** — the core skill. Covers cost estimation,
  profiling, API design (batch operations, view types, thread-compatibility),
  algorithmic improvements, memory representation, reducing allocations,
  avoiding unnecessary work, compiler assistance, and stats/logging overhead.
- **[`references/kotlin-jvm.md`](references/kotlin-jvm.md)** — a Kotlin/JVM
  translation of the same principles: boxing, `inline` lambdas, coroutine
  state-machine overhead, `value class`, sequences vs. eager collections,
  `final`-by-default dispatch, and JVM profiling tools (JFR, async-profiler,
  JMH).

## Usage

Drop this repo's contents into a project's `.claude/skills/` directory (or
install it as a plugin skill) so Claude Code can load it when asked to
optimize code, review for performance issues, or reduce allocations. The
skill triggers on requests like "make this faster," "reduce allocations,"
"why is this slow," or "review for performance issues," and pulls in the
Kotlin/JVM reference automatically for Kotlin code.
