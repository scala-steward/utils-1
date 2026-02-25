# Kinoplan Utils — AI Agent Context

> This file provides context for AI assistants working with this codebase.
>
> Developers can edit this and/or related files to improve AI assistant behavior within the project.

---

## 1. Project Overview

**Kinoplan Utils** is an open-source library monorepo providing reusable Scala utilities for the Kinoplan ecosystem and beyond. Published to Maven Central under `io.kinoplan`.

- **Language:** Scala 2.12.x / 2.13.x
- **Build:** SBT with `sbt-projectmatrix` for cross-building (JVM + Scala.js)
- **Core stack:** ZIO 2.x, Circe, Tapir, ReactiveMongo, Redisson, Play Framework (2.x & 3.x)
- **Architecture:** modular library monorepo — each module is an independently published artifact

**Key references:**
- Project description, module list, usage — [README.md](README.md)
- Formatting, code style, workflow — [CONTRIBUTING.md](CONTRIBUTING.md)
- Dependencies and versions — [project/Dependencies.scala](project/Dependencies.scala)
- Build settings and profiles — [project/ProjectSettings.scala](project/ProjectSettings.scala)
- Module definitions — [project/ModulesCommon.scala](project/ModulesCommon.scala), [project/ModulesImplicits.scala](project/ModulesImplicits.scala), [project/ModulesPlay.scala](project/ModulesPlay.scala), [project/ModulesZio.scala](project/ModulesZio.scala)

### Module Groups

| Group | Directory | Description |
|-------|-----------|-------------|
| Common | `common/` | Core utilities: Circe/BSON codecs, ReactiveMongo, Redisson, Nullable, date, logging, http4s, tapir |
| Implicits | `implicits/` | Implicit enrichments for collections, booleans, dates, ZIO, ZIO Prelude |
| Play | `play/` | Play Framework integrations (error handlers, logging filters, ReactiveMongo) for Play 2.x and 3.x |
| ZIO | `zio/` | ZIO-specific modules: ReactiveMongo, Redisson, Tapir server, OpenTelemetry, healthcheck, monitoring |

---

## 2. Code Patterns

### ZIO + ZLayer

ZIO modules expose a trait (service interface) and a `Live` case class implementation, wired via `ZLayer`:

```scala
trait ReactiveMongoApi {
  def database: Task[DB]
}

case class ReactiveMongoApiLive(...) extends ReactiveMongoApi

object ReactiveMongoApi {
  val live: ZLayer[..., Throwable, ReactiveMongoApi] = ZLayer.scoped(make)
}
```

### ADT-based Designs

Sealed trait hierarchies are used for sum types (e.g. `Nullable[A]` with `Null`, `Absent`, `NonNull`). Pattern-match exhaustiveness is relied upon.

### Implicit Enrichments

The `implicits/` modules add extension methods via implicit classes. They are thin, pure, and well-tested.

### Cross-building (JVM / Scala.js)

Many modules are cross-compiled using `sbt-projectmatrix`:
- `.jvmPlatform(...)` and `.jsPlatform(...)` in `build.sbt`
- Platform-specific sources go in `scala-2.12/` or `scala-2.13/` directories

### Play Framework Dual Support

Play modules support both Play 2.x and Play 3.x via `customRow` with custom axes (`play2Axis`, `play3Axis`).

### Dependency Shading

Some dependencies (e.g. `zio-config`) are shaded using `coursier.ShadingPlugin` to avoid classpath conflicts. Shading config is in `project/Dependencies.scala` (`Shades` object).

---

## 3. AI Agent Rules

> Everything below is instructions for AI assistants.
>
> Developers can edit these rules to control AI behavior when working with this code.

### 3.1. Principles

1. **Library code** — this is a published library, not an application. API stability, binary compatibility, and minimal dependencies matter.
2. **Functional style** — immutability, pure functions, composition. No `var`, `null`, or mutable collections.
3. **Follow existing patterns** — study similar modules before writing new code.
4. **Cross-build awareness** — changes must compile on all supported Scala versions and platforms (JVM/JS).
5. **Minimal dependencies** — use only libraries already in `project/Dependencies.scala`. Do not add new ones without explicit approval.
6. **Formatting** — always run `sbt format` (or `sbt fix` + `sbt fmt`) before finishing.

### 3.2. Code Analysis

- Understand that this is a **library** — every public API is a contract.
- Check `project/Dependencies.scala` for available library versions before suggesting upgrades.
- Module build profiles are defined in `project/ModulesCommon.scala`, `ModulesImplicits.scala`, `ModulesPlay.scala`, `ModulesZio.scala`.
- Pay attention to cross-platform constraints — not all APIs are available on Scala.js.
- Look for existing utilities across modules before writing duplicate code.
- Implicit instances (codecs, type classes) are usually in companion objects.

### 3.3. Making Changes

- Study an existing module in the same group before creating a new one.
- New modules must follow the same `sbt-projectmatrix` pattern (see `build.sbt`).
- Keep public API surface minimal — prefer `private[package]` for internal helpers.
- Use sealed traits for ADTs, case classes for data.
- Use `ZLayer` for dependency injection in ZIO modules.

### 3.4. Adding Dependencies

1. Check existing libraries in `project/Dependencies.scala`.
2. Reuse existing version variables (e.g. `circeV`, `zioV`, `tapirV`).
3. Verify compatibility with both Scala 2.12 and 2.13.
4. For cross-platform modules, use `Def.setting(... %%% ...)` syntax.
5. Add new entries to `Dependencies.Libraries` and wire them in the corresponding `Modules*.scala` profile.

### 3.5. Common Pitfalls

- **Cross-build breakage** — Scala 2.12 lacks some 2.13 collection APIs. Use `scala-collection-compat` when needed.
- **Scala.js limitations** — no JVM-only APIs (blocking I/O, Java reflection, JodaTime on JS).
- **Implicit conflicts** — when Circe codecs conflict, use semiauto derivation.
- **Shaded dependencies** — `zio-config` is shaded; importing the wrong package will fail.
- **Play version matrix** — Play 2.x and 3.x have different package namespaces (`com.typesafe.play` vs `org.playframework`).
- **BSON codecs** — ReactiveMongo requires implicit `BSONDocumentReader`/`BSONDocumentWriter` instances, usually in companion objects.

### 3.6. Testing

- Test framework: **ScalaTest** (primary), **ZIO Test** (for ZIO modules).
- Assertions use `assert` with `===`.
- Integration tests use **Testcontainers** (MongoDB, Redis).
- Run tests for a specific module: `sbt "project <moduleName>" test`.
- Run scoped tests: `sbt "testScoped 2.13 JVM"`.
- `Test / fork := true` is the default (except Scala.js where it is `false`).

### 3.7. Planning

**Create a plan for:**
- New modules (new artifact to publish)
- Cross-cutting changes affecting multiple modules
- Public API changes (additions, modifications, deprecations)
- Integration of new external libraries

**No plan needed for:**
- Bug fixes within a single module
- Internal refactors that don't change public API
- Formatting, documentation, or typo fixes
- Adding tests for existing code

**Planning process:**
1. Research existing modules for similar patterns
2. Create plan (specify module group, files, public API changes)
3. Get confirmation for non-obvious decisions
4. Execute with TODO tracking
5. If something goes wrong — stop and re-plan

### 3.8. Checklist Before Finishing

1. **Self-check** — would a senior library maintainer approve this on code review?
2. **Compilation** — `sbt "compileScoped 2.13 JVM"` and `sbt "compileScoped 2.12 JVM"` pass without errors
3. **Scala.js** — if applicable, `sbt "compileScoped 2.13 JS"` passes
4. **Formatting** — `sbt format` executed
5. **Tests** — `sbt "testScoped 2.13 JVM"` passes (at minimum for affected modules)
6. **API review** — no unintended public API surface exposed

### 3.9. Quality Standards

**Code:**
- Functional style — immutability, pure functions, composition
- Minimal and well-documented public API
- Sealed trait hierarchies for sum types
- No `var`, `null`, mutable collections, or side effects outside ZIO

**Simplicity:**
- Each change should be minimal in scope
- Don't over-abstract — this is a utils library, not a framework
- Prefer straightforward implementations over clever ones
- If a solution feels like a hack — reconsider the approach

**Errors:**
- Use typed errors in ZIO modules (`ZIO[R, E, A]`)
- Don't swallow errors (no `.catchAll(_ => ZIO.unit)`)
- Provide meaningful error messages
- Document error cases in scaladoc for public APIs

### 3.10. Debugging

**Investigation:**
1. Compile errors — read messages carefully, especially implicit resolution failures
2. Cross-build failures — check Scala version-specific source directories (`scala-2.12/`, `scala-2.13/`)
3. Scala.js failures — verify no JVM-only APIs are used
4. Test failures — check Testcontainers setup (Docker must be running for integration tests)
5. Missing implicits — look in companion objects and imported packages

**Root Cause Analysis:**
- Don't fix symptoms — find the real cause
- Check how similar cases are handled in other modules
- Understand why the code was written that way before changing it
