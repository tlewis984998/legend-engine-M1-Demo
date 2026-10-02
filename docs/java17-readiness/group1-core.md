# Java 17 readiness: Group 1, `legend-engine-core` (including the Pure compiler/runtime)

**Repo:** tlewis984998/legend-engine-M1-Demo @ `master` (`7a87b75794`) · **Scope:** 128 `pom.xml` files under `legend-engine-core/` · **Toolchain used:** Temurin JDK 17.0.20.1, Maven 3.9.16
**Mode:** read-only. I made no commits or pushes, and no POM or source edits. Experiments only used command-line `-D` overrides or out-of-tree copies under `/tmp`. `git status --porcelain` is empty.

## Summary and rating: **AMBER**

Core is close to compiling at `--release 17`. No main code in the group uses `sun.*`, `jdk.internal.*`, or `Unsafe`, or reflects into `java.*` internals. No core POM adds `--add-opens`.

Two modules were actually compiled at release 17 with no errors and no deprecation or removal warnings:
- `legend-engine-shared-javaCompiler`, whose 11 tests also pass on JDK 17;
- `legend-engine-identity-core`.

The work that remains is targeted, not architectural:

1. **The runtime Java code generators are hard-wired to Java 7/8.**
   - `EngineJavaCompiler` is used by execution-plan Java code. It only supports `JavaVersion.JAVA_7` and `JAVA_8`, and `JavaHelper` asks for `JAVA_8`, so it compiles with `--release 8`.
   - Pure's external `PureJavaCompiler` (legend-pure 5.105.0) defaults to `-source 7 -target 7` on JDK < 20.

   This still works on JDK 17. I confirmed by experiment that generated code compiled at release 8 or source 7 links and runs against classes built at release 17. However, generated code cannot use Java 9+ APIs, and it would break if engine classes exposed Java 9+ types in signatures that generated code calls. So this is a design pin to lift, not a break.
2. **The root Enforcer `enforceBytecodeVersion` is set to `maxJdkVersion 1.8`.** It will reject the reactor's own release-17 artifacts as dependencies.
3. **One test (`IdentityTest`) hacks `java.lang.reflect.Field.modifiers`.** It is confirmed to fail on JDK 17 with `NoSuchFieldException: modifiers`.
4. **Warnings and cleanups:**
   - two `System.getSecurityManager()` calls (deprecated for removal, JEP 411);
   - `Class.newInstance()`;
   - Java 8 hard-codes in the root compiler and Javadoc settings.

The library stack (Dropwizard 1.3 / Jetty 9.4 / Jersey 2.25 / pac4j 4 / javax.*) is old but runs on JDK 17; CI already runs on JDK 17. Upgrading it is *not* needed to set `release=17`. Upgrading it to current majors would be a large javax→jakarta migration, so I count it as follow-on work, not a blocker. It is the main reason this group is not GREEN in the long run.

## 1. Reflection / JDK internals

| module | file:line | finding | Java 17 impact | suggested fix |
|---|---|---|---|---|
| legend-engine-identity-core (test) | `legend-engine-core/legend-engine-core-identity/legend-engine-identity-core/src/test/java/org/finos/legend/engine/shared/core/identity/IdentityTest.java:83-86` | `Field.class.getDeclaredField("modifiers")` + `setAccessible(true)` to strip `final` from `Identity.FACTORIES`. **(a) JDK-internal** | **runtime_failure.** Confirmed: 3/3 tests fail with `NoSuchFieldException: modifiers` at line 85 (JDK 12+ filters this field). `--add-opens` does not help. | Add a package-private/test hook in `Identity` to replace factories, or make `FACTORIES` non-final behind a setter. Don't rewrite `final` fields. |
| legend-engine-language-pure-compiler | `legend-engine-core/legend-engine-core-base/legend-engine-core-language-pure/legend-engine-language-pure-compiler/src/main/java/org/finos/legend/engine/language/pure/compiler/toPureGraph/PureModel.java:488` | `Package_Impl.class.getDeclaredField("classifier").setAccessible(true)`. **(b) Pure-generated class** | none | none required |
| legend-engine-executionPlan-execution | `legend-engine-core/legend-engine-core-base/legend-engine-core-executionPlan-execution/legend-engine-executionPlan-execution/src/main/java/org/finos/legend/engine/plan/execution/result/ResultNormalizer.java:85` | `setAccessible(true)` on declared fields of arbitrary result objects. (b) as long as the objects are engine/generated classes | none expected; `unknown` if JDK types (e.g. `java.time.*`) reach this path | Guard with `canAccess`/`trySetAccessible()`, or skip `java.*` classes |
| legend-engine-test-mft | `legend-engine-core/legend-engine-core-testable/legend-engine-test-mft/src/main/java/org/finos/legend/engine/test/mft/MFTTestSuitBuilder.java:103` | Reflects into OpenTracing `GlobalTracer.tracer` (third-party) | none | optional: `GlobalTracer.registerIfAbsent` / test util |
| legend-engine-pure-functions-unclassified-pure (test) | `legend-engine-core/legend-engine-core-pure/legend-engine-pure-code-functions-unclassified/legend-engine-pure-functions-unclassified-pure/src/test/java/org/finos/legend/engine/pure/code/core/functions/unclassified/base/tracing/AbstractTestTraceSpan.java:176` | Same `GlobalTracer.tracer` reflection | none | same |
| legend-engine-shared-javaCompiler | `legend-engine-core/legend-engine-core-shared/legend-engine-shared-javaCompiler/src/main/java/org/finos/legend/engine/shared/javaCompiler/MemoryClassLoader.java:34` | Custom `ClassLoader.defineClass` from in-memory bytes (public API) | none | none |
| legend-engine-executionPlan-dependencies | `legend-engine-core/legend-engine-core-base/legend-engine-core-executionPlan-execution/legend-engine-executionPlan-dependencies/src/main/java/org/finos/legend/engine/plan/compilation/GeneratePureConfig.java:246-263` | Own `defineClass` helper (application code, not `ClassLoader`) | none | none |
| all core | (searched all `legend-engine-core/**/*.java`) | **No** `sun.*`, `com.sun.*` internals, `jdk.internal.*` or `sun.misc.Unsafe` imports | none | none |
| all core POMs/scripts | (searched `legend-engine-core/**/pom.xml`, `*.sh`, `Dockerfile`, `*.json`) | **No** `--add-opens/--add-exports/--illegal-access/argLine` in core | none | none |
| root (affects core) | `pom.xml:154` | `surefire.vm.params` has `--add-opens=java.base/java.nio=ALL-UNNAMED` (for Arrow). CI uses `--add-opens=java.base/java.nio=org.apache.arrow.memory.core,ALL-UNNAMED` (`.github/workflows/build.yml`, `release.yml`) | none (already JDK-17 style) | keep it; core itself has no Arrow imports |

## 2. SecurityManager

| module | file:line | finding | impact | fix |
|---|---|---|---|---|
| legend-engine-pure-runtime-java-extension-interpreted-functions-unclassified | `legend-engine-core/legend-engine-core-pure/legend-engine-pure-code-functions-unclassified/legend-engine-pure-runtime-java-extension-interpreted-functions-unclassified/src/main/java/org/finos/legend/pure/runtime/java/extension/functions/interpreted/natives/tracing/TraceSpan.java:59` | `System.getSecurityManager()` (thread-group selection, typical copy of the `DefaultThreadFactory` idiom) | warning_only (`[removal]` under release 17; throws `UnsupportedOperationException` only from `setSecurityManager`, not the getter) | Use `Thread.currentThread().getThreadGroup()` directly |
| legend-engine-pure-runtime-java-extension-compiled-functions-unclassified | `legend-engine-core/legend-engine-core-pure/legend-engine-pure-code-functions-unclassified/legend-engine-pure-runtime-java-extension-compiled-functions-unclassified/src/main/java/org/finos/legend/pure/runtime/java/extension/functions/compiled/FunctionsHelper.java:549-551` | Same idiom | warning_only | same |
| all core | — | No `setSecurityManager`, `AccessController`, `AccessControlContext`, `doPrivileged`, `Policy`, `SecurityManager` subclasses or `java.security.manager` properties | none | — |

## 3. Deprecated-for-removal / removed APIs

| module | file:line | finding | impact | fix |
|---|---|---|---|---|
| legend-engine-executionPlan-execution | `legend-engine-core/legend-engine-core-base/legend-engine-core-executionPlan-execution/legend-engine-executionPlan-execution/src/main/java/org/finos/legend/engine/plan/execution/nodes/ExecutionNodeExecutor.java:251` | `clazz.newInstance()` (deprecated since 9, *not* for removal) | warning_only | `clazz.getDeclaredConstructor().newInstance()` |
| legend-engine-pure-ide-light-http-server | `legend-engine-core/legend-engine-core-pure/legend-engine-pure-ide/legend-engine-pure-ide-light-http-server/src/main/java/org/finos/legend/engine/ide/session/PureSession.java:361` | `TestRunner.stop()`. This is the application's own type, **not** `Thread.stop` | none | — |
| (2 SecurityManager getters above) | see §2 | `@Deprecated(forRemoval=true, since="17")` | warning_only | see §2 |
| all core | — | None found: `finalize()` overrides, `runFinalization`, `Thread.stop/suspend/resume`, boxing constructors (`new Integer(` etc.), Applet, RMI activation, `javax.security.cert`, Nashorn/`ScriptEngineManager`, `javax.xml.bind`, `javax.activation`, `javax.annotation`, JAX-WS, CORBA, Pack200, `sun.misc.BASE64*` | none | — |
| serialization/caching | `legend-engine-core/legend-engine-core-base/legend-engine-core-executionPlan-execution/legend-engine-executionPlan-execution/src/main/java/org/finos/legend/engine/plan/execution/cache/ExecutionCacheBuilder.java`, `.../cache/executionPlan/ExecutionPlanCacheBuilder.java` | Caching is Guava `com.google.common.cache` (Guava 33.4.6-jre). **No** Kryo, Caffeine, or `ObjectInput/OutputStream` in core. Jackson is used in 269 files | none | — |

## 4. Runtime Java code generation / compilation (a focus area)

| module | file:line | finding | impact | fix |
|---|---|---|---|---|
| legend-engine-shared-javaCompiler | `legend-engine-core/legend-engine-core-shared/legend-engine-shared-javaCompiler/src/main/java/org/finos/legend/engine/shared/javaCompiler/EngineJavaCompiler.java:163-178` | `ToolProvider.getSystemJavaCompiler()` + `MemoryFileManager`/`MemoryClassLoader`. Options: `JAVA_7` → `-source 7` only (on JDK 17 this emits **v61** classes plus "obsolete source 7" warnings); `JAVA_8` → `--release 8` on JDK ≥ 9. Default when null: `JAVA_7` | warning_only / design pin (generated plan code limited to Java 8 APIs; `-source 7` will hard-fail on JDK ≥ 20, where javac drops source 7) | Add `JAVA_11`/`JAVA_17` (or `Runtime.version().feature()`) to `JavaVersion`. Default to the running/compile release, and drop `JAVA_7` |
| legend-engine-shared-javaCompiler | `legend-engine-core/legend-engine-core-shared/legend-engine-shared-javaCompiler/src/main/java/org/finos/legend/engine/shared/javaCompiler/JavaVersion.java` | Enum is only `JAVA_7`, `JAVA_8` | design pin | as above |
| legend-engine-executionPlan-execution | `legend-engine-core/legend-engine-core-base/legend-engine-core-executionPlan-execution/legend-engine-executionPlan-execution/src/main/java/org/finos/legend/engine/plan/execution/nodes/helpers/platform/JavaHelper.java:91` | `new EngineJavaCompiler(JavaVersion.JAVA_8, ClassPathFilters.any(...))` for execution-plan Java nodes | works on 17 (verified by experiment: `--release 8` code compiled against v61 libs runs). Risk: compile errors if engine API signatures used by generated code adopt Java 9+ types | Switch to `JAVA_17` once the enum supports it |
| legend-engine-shared-javaCompiler | `.../javaCompiler/SingleFileCompiler.java:45` | Janino 3.1.0 `Parser`/`UnitCompiler`, used for single-file compilation | none (tests pass on 17) | optional: bump to Janino 3.1.12 |
| external `org.finos.legend.pure:legend-pure-runtime-java-engine-compiled:5.105.0` | `PureJavaCompiler` (bytecode, cached jar) | `ToolProvider.getSystemJavaCompiler()`. The 3-arg `compile(JavaCompiler, Iterable, JavaFileManager)` passes `release=null`. On JDK < 20 (`SourceVersion.latest().ordinal()` check) it uses `-source 7 -target 7` → v51 classes; otherwise `--release`. Used by `GenerateAndCompile`, `JavaStandaloneLibraryGenerator`, `CompiledSupport` (runtime dynamic compile), `FunctionExecutionCompiled` | warning_only on 17 (javac prints obsolete-option warnings). Not controllable from engine POMs | Upstream: pass an explicit release, or upgrade legend-pure to a line that does |
| Pure modules (37 core POMs use `legend-pure-maven-*`) | e.g. `legend-engine-core/legend-engine-core-pure/legend-engine-pure-code-compiled-core/pom.xml:98-104` | `legend-pure-maven-generation-java` `build-pure-compiled-jar` with `generationType=modular`, `preventJavaCompilation=true`. Generated Java is then compiled by maven-compiler, so it **follows `maven.compiler.release`** | none | — |
| external Pure jars | `~/.m2/.../org/finos/legend/pure/*/5.105.0/*.jar` | All Pure runtime/plugin jars are Java 8 class files (major 52). Fine as dependencies on 17 | none (but see the Enforcer row in §6) | — |

## 5. Libraries (versions from the root `pom.xml` `<properties>` / `dependencyManagement`; core POMs do not override them)

The "min Java" column for newer majors comes from upstream release notes I know of. I did **not** re-check it live in this session (see *Not verified*).

| artifact | current | defined in | newer major → min Java | pinned b/c Java 8? | broken on 17 now? | notes |
|---|---|---|---|---|---|---|
| io.dropwizard:dropwizard-* | 1.3.29 | `pom.xml:185` (also via `legend-shared-server` 0.37.0, `pom.xml:126`) | 2.1 (Java 8) / 3.x (Java 11) / 4.x (Java 11, **jakarta**) / 5.x (Java 17) | likely | no | 6 core files import `io.dropwizard`. Upgrading drags in Jetty/Jersey/HV |
| org.eclipse.jetty:* | 9.4.44.v20210927 | `pom.xml:209` | 10 (Java 11, javax) / 11 (Java 11, **jakarta**) / 12 (Java 17) | likely | no | 9.4 is EOL |
| org.glassfish.jersey:* | 2.25.1 | `pom.xml:208` | 2.4x (Java 8) / 3.x (Java 11, **jakarta**) | likely | no | `javax.ws.rs` used in 56 core files |
| javax.servlet:javax.servlet-api | 3.1.0 | `pom.xml:206` | jakarta.servlet-api 5/6 (Java 11, **namespace change**) | tied to Jetty 9 | no | `javax.servlet` used in 37 core files |
| org.pac4j:pac4j-* | 4.5.8 (jax-rs/jersey 4.0.0) | `pom.xml:225-227` | 5.x (Java 11) / 6.x (Java 17, **jakarta**) | likely | no | 16 core files |
| com.fasterxml.jackson.core:jackson-databind | 2.10.5.1 (core/annotations 2.10.5) | `pom.xml:202-203` | 2.18+ (Java 8); 3.x (Java 17, new `tools.jackson` packages) | no (2.1x is Java 8) | no | 2.10 has **no record support** (needs 2.12+). Upgrade within 2.x if records are adopted. Used in 269 files |
| org.eclipse.collections:eclipse-collections(-api) | 10.2.0 | `pom.xml:186` | 11.x (Java 8) / 12–13 (Java 11) | possibly | no | tests pass on 17 |
| org.antlr:antlr4 / antlr4-runtime / antlr4-maven-plugin | 4.8-1 | `pom.xml:171`, plugin `pom.xml:398` | 4.10+ (ATN format change, regenerate grammars) / 4.13 (Java 11) | possibly | no | 38 core files import `org.antlr.v4`. Grammar module compile at 17 not run (57-module `-am`) |
| org.codehaus.janino:janino, commons-compiler | 3.1.0 | `pom.xml:204` | 3.1.12 (Java 8, no major) | no | no | verified by `TestSingleFileJavaCompiler` on 17 |
| io.github.classgraph:classgraph | 4.8.25 | root `pom.xml` properties | 4.8.17x (Java 7+) | no | no | JDK 17 smoke test passed (scans `jrt:`) |
| com.google.guava:guava | 33.4.6-jre | `pom.xml:194` | current | no | no | used for caches |
| org.mockito:mockito-core / mockito-inline | **4.4.0 / 5.2.0** | `pom.xml:219-220`, mgmt `pom.xml:3683-3694` | 5.x (Java 11) | **yes (core held at 4.x)** | no | Inconsistent: inline 5.2.0 expects core 5.x. Used in 4 core test files |
| net.bytebuddy:byte-buddy | 1.11.20 | `pom.xml:174`, mgmt `pom.xml:3708-3710` | 1.14/1.15+ (Java 5+ runtime) | no | not observed | it overrides the Byte Buddy version Mockito brings in. Bump to ≥1.12 to be safe with v61 test classes (unverified) |
| ch.qos.logback:logback-* | 1.2.3 | `pom.xml:217` | 1.3 (Java 8) / 1.4–1.5 (Java 11) | likely (1.4 = jakarta servlet) | no | 1.2.3 has known CVEs |
| org.slf4j:* | 1.7.36 | `pom.xml:230` | 2.0 (Java 8) | no | no | |
| io.opentracing:* | 0.32.0 (contrib 0.3.0) | `pom.xml:223-224` | project EOL → OpenTelemetry (Java 8) | no | no | 40 core files |
| junit:junit / org.junit.jupiter | 4.13.1 / 5.11.0 | `pom.xml:213-214` | JUnit 6 (Java 17) | no | no | |
| org.freemarker:freemarker | 2.3.34 | `pom.xml:189` | current | no | no | |
| org.yaml:snakeyaml | 1.33 | `pom.xml:231` | 2.x (Java 8) | no | no | |
| org.apache.httpcomponents:httpclient | 4.5.13 | `pom.xml:199` | httpclient5 (Java 8) | no | no | |
| commons-io:commons-io | 2.7 | `pom.xml:178` | 2.1x (Java 8) | no | no | |
| org.finos.legend.pure:* (+ `legend-pure-maven-*` plugins) | 5.105.0 | `pom.xml:125` | n/a (same-org) | — | no | Java 8 class files; `PureJavaCompiler` defaults to source/target 7 (§4) |
| org.finos.legend.shared:legend-shared-server | 0.37.0 | `pom.xml:126` | not checked | likely (brings in Dropwizard 1.3) | no | must move together with Dropwizard |

## 6. Build/tooling (root POM items that affect core)

| item | file:line | issue | impact | fix |
|---|---|---|---|---|
| Enforcer `enforceBytecodeVersion` | `pom.xml:591-592` | `maxJdkVersion 1.8` | **compile_break (build failure)**: rejects release-17 reactor dependencies | set to 17 |
| compiler properties | `pom.xml:148-150` | `source/target 1.8`, `release 8` | change needed | set `maven.compiler.release=17`; remove source/target |
| maven-compiler-plugin | `pom.xml:248` (3.8.0), config `pom.xml:335-336` | passes `<source>/<target>` explicitly; works with `release` | warning_only | bump to 3.11+ and configure `<release>` |
| Javadoc | `pom.xml:252` (3.3.1), `pom.xml:312` | `<source>8</source>` hard-coded | warning/inconsistency | `<release>17</release>` or remove; bump plugin |
| Surefire | `pom.xml:262` (2.22.2) | works on 17; the VM params already use `--add-opens` style | none | bump to 3.x recommended (JUnit 5 detection: see the note below) |
| JaCoCo | `pom.xml:245` (0.8.10) | supports class files up to Java 20 | none | — |
| ANTLR plugin | `pom.xml:398` (`${antlr.version}` 4.8-1) | Java 8 tool runs on 17 | none | — |
| `legend-pure-maven-*` plugins (37 core POMs) | e.g. `legend-engine-core/legend-engine-core-pure/legend-engine-pure-code-compiled-core/pom.xml:98-104` | modular generation leaves compilation to maven-compiler (follows release). Non-modular / runtime paths use `PureJavaCompiler`'s source/target 7 | warning_only | upstream change (§4) |
| Toolchains / Animal Sniffer / Checkstyle / SpotBugs | — | no toolchains or animal-sniffer in core. Checkstyle was skipped in the experiments | none observed | — |
| Java-version assertions in tests | — | none (the only `Runtime.version()` hit is a comment at `EngineJavaCompiler.java:168`) | none | — |

Side observation (not caused by Java 17): `mvn surefire:test` on `legend-engine-identity-core` reported **0 tests** for the JUnit 5 `IdentityTest`. That would explain why its JDK-17 failure has not shown up in CI. Worth checking the JUnit Platform provider wiring.

## 7. Top blockers (ranked by effort × risk)

1. **Root Enforcer bytecode rule caps at Java 8.** `pom.xml:591-592`. Fix: `maxJdkVersion` → 17 (keep the rule). *Effort S, risk low. This is a hard build break once release=17.*
2. **`IdentityTest` rewrites `Field.modifiers`.** `legend-engine-core/legend-engine-core-identity/legend-engine-identity-core/src/test/java/org/finos/legend/engine/shared/core/identity/IdentityTest.java:83-86`. Fix: add a test hook in `Identity` to swap `FACTORIES`. *S, low. Confirmed runtime failure on JDK 17.*
3. **Execution-plan runtime Java compilation is pinned to Java 7/8.** `EngineJavaCompiler.java:163-178`, `JavaVersion.java`, `JavaHelper.java:91`. Fix: add `JAVA_17` (or derive from `Runtime.version().feature()`), default to it, and drop `JAVA_7`, so that `-source 7` doesn't break on JDK 20+. Regression-test the generated plans. *M, medium.*
4. **Pure external compiler (legend-pure 5.105.0) defaults to source/target 7** for runtime and non-modular compilation (`PureJavaCompiler`). Fix: upstream change, or a legend-pure upgrade that passes an explicit release. Until then it works on 17, with warnings. *M, medium (depends on upstream).*
5. **Root compiler and Javadoc hard-codes.** `pom.xml:148-150`, `pom.xml:248`, `pom.xml:312`, `pom.xml:335-336`. Fix: `release=17`, compiler plugin ≥ 3.11, Javadoc `<release>17</release>`. *S, low.*
6. **Mockito 4.4.0 core vs 5.2.0 inline, with Byte Buddy pinned at 1.11.20.** `pom.xml:174,219-220,3683-3710`. Fix: align on Mockito 5.x (Java 11+ is fine once on 17) with its matching Byte Buddy, or at least bump Byte Buddy. *S–M, medium (test-only).*
7. **SecurityManager getters (JEP 411).** `TraceSpan.java:59`, `FunctionsHelper.java:549-551`. Fix: use `Thread.currentThread().getThreadGroup()`. *S, low (warning only).*
8. **`ResultNormalizer` blanket `setAccessible(true)`.** `ResultNormalizer.java:85`. Fix: `trySetAccessible()` and skip `java.*`. *S, low–medium (only if JDK objects reach it).*
9. **Deprecated `Class.newInstance()`.** `ExecutionNodeExecutor.java:251`. *S, low.*
10. **(Follow-on, not needed for release=17) The javax-era stack** (Dropwizard 1.3.29, Jetty 9.4, Jersey 2.25.1, pac4j 4.5.8, javax.servlet/ws.rs in ~90 core files, Logback 1.2.3, Jackson 2.10), with `legend-shared-server` 0.37.0. Fix: a staged upgrade; the jakarta majors need Java 11/17. *XL, high.*

## 8. Builds and commands run (with outcomes)

- `rg`/`grep` sweeps across `legend-engine-core/**/*.java` and `pom.xml` for every category above → findings as tabled.
- `find legend-engine-core -name pom.xml | wc -l` → 128.
- `javap -c -p` on `PureJavaCompiler` and its callers in the cached legend-pure 5.105.0 jars → `-source/-target/--release` logic confirmed; the 3-arg `compile` passes `release=null`.
- `javap -v` on cached Pure/engine jars → major version 52.
- `jdeps --jdk-internals` on cached engine jars → no JDK-internal usage reported.
- ClassGraph 4.8.25 JDK 17 smoke test → OK.
- **Experiment (out-of-tree, /tmp/jt):** a library compiled with `--release 17`. "Generated" code compiled against it with `--release 8` (v52), `-source 7 -target 7` (v51) and `-source 7` alone (v61, plus obsolete-option warnings) → compiles, and the `--release 8` variant ran. This shows the codegen pins are compatible with a release-17 engine.
- **Experiment (out-of-tree, /tmp/jcx):** javac `--release 17 -Xlint:deprecation,removal` on `legend-engine-shared-javaCompiler` main and test sources against the cached 10.2.0/3.1.0/4.8.25/2.10.5.1 jars. `MetricsHandler` was stubbed, because building `legend-engine-shared-core` needs a 57-module `-am`. → **compiles with 0 warnings; JUnit 4: `OK (11 tests)`** (`TestJavaCompiler`, `TestSingleFileJavaCompiler`, `TestJavaCompileClassPathFilter`) on JDK 17. Class files major 61.
- `mvn -B -pl <module> -am validate` for 5 targets, to size the reactors: javaCompiler 57, identity-core 7, language-pure-grammar 57, executionPlan-execution 60, language-pure-compiler 58. All later failed resolving the unbuilt `4.150.1-SNAPSHOT` siblings (expected, nothing had been installed).
- **Experiment:** `mvn -B clean test-compile -pl legend-engine-core/legend-engine-core-identity/legend-engine-identity-core -am -Dmaven.compiler.release=17 -Dcheckstyle.skip -Djacoco.skip=true -Dmaven.compiler.showDeprecation=true -Dmaven.compiler.showWarnings=true`
  - → failed first in `maven-dependency-plugin:2.10:analyze-only` (`IllegalArgumentException`, before javac) on `legend-engine-shared-structures`.
  - Retried with `-Denforcer.skip=true -Dmdep.analyze.skip=true` → **BUILD SUCCESS**, 0 compiler errors, 0 deprecation/removal warnings.
  - So dependency-plugin 2.10's analyzer cannot handle v61 classes either; add **maven-dependency-plugin** to the build/tooling bumps (3.6+).
- Installed the 7-module identity graph under an isolated `master-SNAPSHOT` coordinate, then ran `IdentityTest` with the JUnit Platform Console on JDK 17 → **3/3 failed, `NoSuchFieldException: modifiers` at `IdentityTest.java:85`.** Surefire reported 0 tests for the module.
- Logs: `/home/ubuntu/j17-exp/{a,b,c,d,e}.log`. Final `git status --porcelain` → empty.

## 9. Not verified

- Release-17 compile of `legend-engine-language-pure-grammar` (ANTLR), `legend-engine-language-pure-compiler`, `legend-engine-executionPlan-execution`, and all Pure (`legend-pure-maven-*`) modules. Not built: each needs a 57–60 module `-am` reactor, and the Pure modules need clean installs. The static sweeps found no code that would stop them compiling.
- Execution-plan generated-code tests (e.g. Java platform plan nodes) under JDK 17 / release 17.
- `jdeps --multi-release 17` on freshly built release-17 core jars (only cached jars were analysed).
- The `-Denforcer.skip=true` retry: I could not tell from the log alone whether Enforcer failed because of `maxJdkVersion 1.8` or because of the local artifact gap. The rule's behaviour on v61 artifacts is inferred from its documented semantics.
- Newer-major minimum Java versions and "pinned because of Java 8" status come from known upstream release notes and version patterns, not commit history. Same for Byte Buddy 1.11.20 handling v61 classes under Mockito.
- `ResultNormalizer` with JDK-typed result objects.
- `legend-shared-server` 0.37.0 internals.
- Surefire's 0-test result for the JUnit 5 identity tests.
