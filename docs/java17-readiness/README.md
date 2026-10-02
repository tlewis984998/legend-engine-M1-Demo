# Java 17 readiness assessment: legend-engine fork

**Question:** can the root `pom.xml` move from `maven.compiler.release=8` to `17`? CI already builds on Zulu JDK 17.
**Status:** read-only assessment. No source or POM changes were made.
**Baseline:** `master` @ `7a87b75794`. 610 `pom.xml` files.

**Verdict: AMBER for every group.** Nothing in the code needs a redesign. The flip is mostly blocked by build tooling: the root build plugins and the server's container image. When `release=17` was tried, the build broke in the shared `maven-dependency-plugin` 2.10. Once `mdep.analyze.skip` bypassed that plugin, the test-compiles succeeded. Expect about one sprint of build hygiene (Phase 0), then a single flip PR (Phase 1). The large upgrade work, the javax to Jakarta server stack, is optional and separate (Phase 3).

Group reports, with every finding listed by file and line:
- [group1-core.md](group1-core.md): `legend-engine-core`, including the Pure compiler and runtime (128 POMs)
- [group2-relational-sql.md](group2-relational-sql.md): `legend-engine-xts-relationalStore` + `legend-engine-xts-sql` (148 + 18 POMs)
- [group3-xts-other-server-config.md](group3-xts-other-server-config.md): every other `legend-engine-xts-*`, `legend-engine-config`, `legend-engine-application-query`, `legend-engine-shared-mongo`, `legend-engine-docs` and the root POM (316 POMs)

Method: static search (sources, POMs, launch scripts, CI workflows), `jdeps --jdk-internals` on third-party jars, class-file version checks, and targeted `mvn -pl <module> -am` builds with `-Dmaven.compiler.release=17` on Temurin 17.0.20.1 / Maven 3.9.16. No full-reactor build was run.

---

## 1. Readiness by module group

| Group | POMs | Rating | Why | Release-17 evidence | Blocking items (see §2) |
|---|---|---|---|---|---|
| 1. Core (incl. Pure compiler/runtime) | 128 | AMBER | Main code has no `sun.*`, `jdk.internal.*` or `Unsafe` imports and no POM `--add-opens`. One test fails on JDK 17. Runtime Java codegen is pinned to Java 7/8. Two `getSecurityManager()` removal warnings. | `legend-engine-shared-javaCompiler` and `legend-engine-identity-core` compiled at release 17 (classfile 61). 11/11 javaCompiler tests passed. The Pure compiler and execution-plan reactors (57-60 modules each) were **not** compiled at 17 by this group. Some core modules were covered by group 2's `-am` reactor. | #1, #2, #5, #6, #7, #9 |
| 2. relationalStore + SQL | 166 | AMBER | All JDK-internal reflection is either in Arrow (needs `java.nio` opened) or is a legacy `sun.*` reference inside the Snowflake JDBC driver. No SecurityManager use. JEP 411 Kerberos APIs produce warnings only. | `test-compile` of the `executionPlan-connection` reactor passed at release 17: **78 modules, BUILD SUCCESS** (with `-Dmdep.analyze.skip=true`). H2 2.1.214 execution compiled at 17, then failed in dependency-plugin 2.10. SQL, most DB-specific execution modules and PCT modules were not built. | #1, #2, #4, #5, #8, #9 |
| 3. Other xts + server/config + root | 316 | AMBER | Code is clean: no Nashorn, JAXB, CORBA, `finalize` or `Thread.stop`. All `setAccessible` calls target project classes. The server image is JDK 11. Arrow flags are missing from the server and REPL launchers. Six JEP 411 removal warnings in Kerberos. The javax server stack runs on 17 and is not a blocker. | `legend-engine-xt-identity-kerberos` reactor compiled at release 17 (classfile 61, 6 removal warnings). | #1, #2, #3, #4, #5, #8, #9, #10 |
| Cross-cutting: root build/tooling | (root `pom.xml`) | **RED** until Phase 0 is done | Flipping only `maven.compiler.release` hard-fails the build (`maven-dependency-plugin` 2.10 cannot read classfile 61). Enforcer settings are still written for Java 8/11. | Reproduced independently by groups 1 and 2. | #1, #2, #5, #10 |

Ratings: GREEN means flip the property and it builds and runs. AMBER means only bounded, known fixes are needed, with no redesign. RED means a hard failure or unbounded work.

---

## 2. Top 10 blockers (ranked)

"Confirmed" = reproduced by a build or test run. "Inferred" = found by static analysis or from library documentation.

| # | Blocker | Where | Evidence | Fix | Effort |
|---|---|---|---|---|---|
| 1 | **`maven-dependency-plugin` 2.10 crashes on Java 17 class files.** Its ASM throws `IllegalArgumentException` in `ClassReader` on major version 61. The `analyze-only` execution runs in every module with `failOnWarning=true`. | `pom.xml:249` (`maven.dependency.plugin.version`), execution at `pom.xml:555-571` | **Confirmed** (core/identity and relational/H2) | Upgrade to 3.8.x. Then triage any new used-undeclared or unused-declared warnings, which the newer analyzer reports more accurately. | S-M (the warning triage is the unknown part) |
| 2 | **Enforcer is configured for Java 8/11.** `enforceBytecodeVersion maxJdkVersion=1.8` will reject the fork's own Java 17 jars once they are consumed as packaged artifacts. `requireJavaVersion` is `[11.0.10,12),[17,18)`, which blocks JDK 18+ and still allows JDK 11, which cannot build release 17. The Enforcer version property is declared twice (`3.0.0-M1` and `3.5.0`). | `pom.xml:592`, `pom.xml:151`, `pom.xml:250` + `:258` | Inferred. The 78-module `test-compile` passed with the rule active, probably because in-reactor dependencies resolve to `target/classes` there, not jars. Not yet tested with `package`/`install`. | Set `maxJdkVersion` to 17 and prune the Java 11 `excludes` that are no longer needed. Set `requireJavaVersion` to `[17,)`. Remove the duplicate property. | S |
| 3 | **The server container runs JDK 11.** Release-17 classes would fail with `UnsupportedClassVersionError` at startup. | `legend-engine-config/legend-engine-server/legend-engine-server-http-server/pom.xml:39` (`eclipse-temurin:11.0.17_8-jdk-jammy`) | Inferred (certain from JVM semantics) | Change to `eclipse-temurin:17-jre-jammy`, pinned to a patch version. Ship it in Phase 0, before the flip: the release-8 bytecode already runs on 17 in CI. | S |
| 4 | **Arrow 18 needs `java.nio` opened on JDK 16+, and only CI/Surefire sets that flag.** The server Jib `jvmFlags` and the REPL launcher do not, so Arrow paths fail with `InaccessibleObjectException` once those launchers run on JDK 17. The Snowflake PCT `argLine` override only sets `ALL-UNNAMED`. | Missing: `…/legend-engine-server-http-server/pom.xml:51-56`, `legend-engine-config/legend-engine-repl/legend-engine-repl-app-assembly/assemble/repl.sh:55`. Present: `pom.xml:154`, `.github/workflows/build.yml:271,318`, `release.yml:212,256`, `…/legend-engine-xt-relationalStore-snowflake-PCT/pom.xml:125` | Inferred (Arrow's documented requirement, plus the existing CI workaround) | Add `--add-opens=java.base/java.nio=org.apache.arrow.memory.core,ALL-UNNAMED` to every launcher. Do it together with #3. | S |
| 5 | **Compiler and Javadoc settings are hard-coded to Java 8 or 11.** | `pom.xml:148-150` (source/target 1.8, release 8), `pom.xml:498-499` (plugin passes `<source>/<target>`), `pom.xml:312` (Javadoc `<source>8</source>`), `legend-engine-xts-persistence/legend-engine-xt-persistence-component/pom.xml:57` (Javadoc `<source>8</source>`), `legend-engine-xts-relationalStore/legend-engine-xt-relationalStore-dbExtension/legend-engine-xt-relationalStore-h2/legend-engine-xt-relationalStore-h2-execution-2.1.214/pom.xml:33-34` (source/target 11) | Confirmed by inspection | Set `release=17` and use `<release>${maven.compiler.release}</release>` in the plugin. Delete source/target, the Javadoc hard-codes and the module overrides. Optionally bump `maven-compiler-plugin` from 3.8.0 to 3.13.x. | S |
| 6 | **Runtime Java code generation is pinned to Java 7/8.** `EngineJavaCompiler` only knows `JAVA_7`/`JAVA_8` and uses `--release 8` on JDK 9+. Execution plans always compile with `JAVA_8`. External `legend-pure` 5.105.0 (`PureJavaCompiler`) defaults to `-source 7 -target 7` in the non-modular paths. Not a JDK 17 failure, but generated code cannot use Java 9+ APIs, and JDK 20+ `javac` dropped `--release 7`. | `legend-engine-core/legend-engine-core-shared/legend-engine-shared-javaCompiler/src/main/java/org/finos/legend/engine/shared/javaCompiler/EngineJavaCompiler.java:163-178`, `…/JavaVersion.java`, `legend-engine-core/legend-engine-core-base/legend-engine-core-executionPlan-execution/legend-engine-executionPlan-execution/src/main/java/org/finos/legend/engine/plan/execution/nodes/helpers/platform/JavaHelper.java:91` | Confirmed by inspection. Not a blocker for the flip. | Add `JAVA_11`/`JAVA_17` and switch the plan default to `JAVA_17` after the flip. Raise the `legend-pure` default upstream. Re-run plan-generation and PCT suites. | M |
| 7 | **`IdentityTest` fails on JDK 17.** It rewrites a `static final` field through `Field.class.getDeclaredField("modifiers")`, which JDK 12+ filters out (`NoSuchFieldException`; `--add-opens` cannot fix this). Surefire also reports **0 tests** for this JUnit 5 class, so CI never sees the failure. | `legend-engine-core/legend-engine-core-identity/legend-engine-identity-core/src/test/java/org/finos/legend/engine/shared/core/identity/IdentityTest.java:83-86` | **Confirmed** (3/3 failures via JUnit Platform console) | Add a package-private test hook for replacing `Identity.FACTORIES`. Fix the JUnit 5 provider wiring for this module and check other modules for the same 0-test symptom. | S |
| 8 | **The test stack is pinned by Java 8 and is internally inconsistent.** `mockito-core` 4.4.0 vs `mockito-inline` 5.2.0, `byte-buddy` 1.11.20, `wiremock-jre8` 2.35.2 (uses non-exported `sun.security.x509` for HTTPS), Surefire 2.22.2. | `pom.xml:219-220`, `pom.xml:174`, `pom.xml:236`, `pom.xml:262` (Surefire) | Inferred (jdeps + version analysis) | After the flip: Mockito 5.x + byte-buddy 1.15+, `org.wiremock:wiremock` 3.x, Surefire 3.5.x. | M |
| 9 | **JEP 411 APIs (SecurityManager / AccessController / `Subject.getSubject`).** Removal warnings on 17. They become hard failures on newer JDKs: `Subject.getSubject` throws on JDK 23+ unless `-Djava.security.manager=allow` is set. Not a blocker for release 17. | Kerberos: `legend-engine-xts-identity/legend-engine-xt-identity-kerberos/src/main/java/org/finos/legend/engine/shared/core/kerberos/ExecSubject.java:28`, `…/SubjectTools.java:117-120`. Relational: `InteractiveAuthenticationStrategy.java:32`, `AuthenticationStrategy.java:81`. Core: `…/natives/tracing/TraceSpan.java:59`, `…/compiled/FunctionsHelper.java:549-551`. `Subject.doAs` in `KerberosUtils.java:28,34`, `ExecSubject.java:46`, `HostedServiceDeploymentManager.java:97,133`, `GraphQL.java:61,75`, `ServicePostValidationRunner.java:178` and the Postgres wire-protocol classes (`PostgresWireProtocol`, `CatalogManager`, `LegendPreparedStatement`, `LegendStatement`) | **Confirmed** warnings (6 in kerberos, 2 in relational auth) | For 17: accept the warnings or add `@SuppressWarnings("removal")`. Before any JDK 21+ runtime: move to `Subject.current()` / `Subject.callAs()` (needs JDK 18+ API, so shim or wait for release 21), and drop the `getSecurityManager()` checks in Pure tracing/functions. | M |
| 10 | **Checkstyle 8.25 cannot parse Java 14-17 syntax** (records, switch expressions, text blocks, `instanceof` patterns). The flip still works, but the first PR that uses Java 17 language features will fail the build. | `pom.xml:256` (Checkstyle 8.25), `pom.xml:247` (plugin 3.1.1) | Inferred (Checkstyle release history) | Checkstyle 10.x + `maven-checkstyle-plugin` 3.5.x. Re-check the rule set for renamed checks. | S |

Watch items, not counted as blockers:
- **Third-party `sun.*` references.** Snowflake JDBC 4.3.2 (group 2) and Snowpark 1.16.0 (`legend-engine-xts-snowflake/legend-engine-xt-snowflake-m2mudf-plan-executor/pom.xml:114-115`, `sun.security.util.Der*`) reference non-exported internals. These already run on JDK 17 in CI. Whether the paths are reachable was not verified.
- **`com.sun.*` packages that are exported, and fine.** `com.sun.security.jgss.GSSUtil` (Postgres GSS) and `com.sun.net.httpserver` (REPL data-cube) both compiled at release 17.
- **Deprecated APIs.** `Class.newInstance()` at `legend-engine-core/…/ExecutionNodeExecutor.java:251`, and boxing constructors in group 2/3 test code. Warnings only.
- **Immutables 2.8.2** uses `com.sun.tools.javac.code.*` with a fallback (`legend-engine-xts-persistence/legend-engine-xt-persistence-component/pom.xml:44`). It works on 17. Bump to 2.10.x anyway.

---

## 3. Dependency upgrades unlocked by release 17

These upgrades are blocked today only because their newer majors need Java 11 or 17 bytecode. Both the compiler release and the Enforcer `maxJdkVersion` (#2) forbid that today. Minimum Java versions come from upstream release notes as the workers knew them. They were not re-checked online, because this session is network-restricted. Entries marked (unverified) are uncertain.

### 3a. Unlocked with no namespace change (Phase 2 candidates)

| Library | Current | Defined in | Target | Min Java | Notes |
|---|---|---|---|---|---|
| Mockito (`mockito-core` / `-inline`) | 4.4.0 / 5.2.0 | `pom.xml:219-220` | 5.x (inline is the default) | 11 | Also fixes the core/inline mismatch. Bump byte-buddy 1.11.20 to 1.15+ (`pom.xml:174`) |
| WireMock | `wiremock-jre8` 2.35.2 | `pom.xml:236` | `org.wiremock:wiremock` 3.x | 11 | Removes the `sun.security.x509` usage. Test scope only |
| JUnit | 4.13.1 / Jupiter 5.11.0 | `pom.xml:213-214` | JUnit 6 | 17 | Optional. 5.x stays supported |
| Testcontainers | 1.21.4 | `pom.xml:234` | 2.x | 17 (unverified) | Used by many DB/integration tests |
| JsonUnit | 2.17.0 | `pom.xml:212` | 3.x / 4.x | 17 (unverified) | Tests |
| Eclipse Collections | 10.2.0 | `pom.xml:186` | 12 / 13 | 11 | Can go to 11.1 (Java 8) first. Very widely used, so do it as its own PR |
| H2 | 2.1.214 | root `h2.version`; `…/legend-engine-xt-relationalStore-h2-execution-2.1.214/pom.xml:73-74` | 2.2 / 2.3 | 11 | Also delete the module's source/target-11 override (#5) |
| mssql-jdbc | `9.4.1.jre8` | `pom.xml:4264` | 12.x `.jre11` | 11 | Classifier change |
| Oracle JDBC | `ojdbc11` | relational (see group 2) | `ojdbc17` | 17 | |
| Apache Iceberg | 1.3.0 | `legend-engine-xts-iceberg/pom.xml` (`iceberg.version`) | 1.7+ | 11 | Iceberg 1.7 dropped Java 8 |
| ANTLR | 4.8-1 | `pom.xml:171` | 4.13.x | tool 11, runtime 8 | Regenerates every grammar (new ATN format). Medium risk |
| Logback / SLF4J | 1.2.3 / 1.7.36 | `pom.xml:217`, `:230` | 1.5.x / 2.0.x | 11 | CVE fixes. Check compatibility with Dropwizard 1.3's logging bootstrap; 1.3.x (Java 8) is the fallback |
| Checkstyle | 8.25 / plugin 3.1.1 | `pom.xml:256`, `:247` | 10.x / 3.5.x | 11 | Needed before using Java 17 syntax (#10) |
| Maven plugins | compiler 3.8.0, Surefire 2.22.2, JaCoCo 0.8.10, dependency 2.10 | root `pom.xml` | compiler 3.13, Surefire 3.5, JaCoCo 0.8.12+, dependency 3.8 | (JDK 17 runtime) | These run in the build, not in the product. Do them in Phase 0 |
| pac4j | 4.5.8 / jersey-pac4j 4.0.0 | `pom.xml:225-227` | 5.x (still javax) | 11 | Coupled to `legend-shared` 0.37.0 (`pom.xml:126`), which owns the auth glue |
| Jetty | 9.4.44 (EOL) | `pom.xml:209` | 10.x (still javax) | 11 | Only possible if Dropwizard/Jersey allow it. Usually done as part of 3b instead |

### 3b. Unlocked but requires the javax to Jakarta migration (Phase 3, separate program)

| Library | Current | Target | Min Java |
|---|---|---|---|
| Dropwizard | 1.3.29 (`pom.xml:185`) | 4.x / 5.x | 11 / 17 |
| Jetty | 9.4.44 | 12.x | 17 |
| Jersey / JAX-RS | 2.25.1 / `javax.ws.rs-api` 2.0.1 (`pom.xml:207-208`) | Jersey 3.1 / `jakarta.ws.rs` 3.1 | 11 |
| Servlet | `javax.servlet-api` 3.1.0 (`pom.xml:206`) | `jakarta.servlet` 6.0 | 11 |
| HK2 | 2.5.0-b32 (`pom.xml:197`) | 3.x | 11 |
| pac4j | 4.5.8 | 6.x | 17 |
| Hibernate Validator | 5.4.3 (`pom.xml:196`) | 8.x / 9.x | 11 / 17 |
| Swagger | 1.5.20 / dropwizard-swagger 1.3.17-1 | swagger-core v3 `-jakarta` | 11 |
| Jackson | 2.10.5 (`pom.xml:202-203`) | 2.18+ now (Java 8), then 3.x | 8, then 17 for 3.x (new `tools.jackson` packages) |

Scale: 73 files across 18 top-level directories import `javax.ws.rs`. 16 import `javax.servlet`. `legend-shared` must move first.

### 3c. Already Java 11 bytecode (shows that Java 8 compatibility is already broken at runtime)

Arrow 18.0.0, HikariCP 7.0.2, Databricks JDBC 3.4.1, `ojdbc11`, Deephaven 0.40.7, mongo-java-server 1.46.0 and jline 3.26.3 are all excluded from the bytecode rule today. With release 17, those excludes can be deleted.

---

## 4. Phased plan

### Phase 0: Tooling and runtime prep (release stays 8; each item is an independent PR that is safe on 8)
1. `maven-dependency-plugin` 2.10 → 3.8.x. Triage the analyzer warnings (#1).
2. Enforcer: `requireJavaVersion=[17,)`, remove the duplicate version property (#2).
3. Server Jib image → `eclipse-temurin:17-jre`. Add the Arrow `--add-opens` to the Jib `jvmFlags` and `repl.sh` (#3, #4). Smoke-test the server and the REPL data-cube.
4. Fix `IdentityTest`, and fix JUnit 5 discovery in `legend-engine-identity-core`. Scan the module test reports for other 0-test modules (#7).
5. Bump Surefire to 3.5.x, compiler plugin to 3.13.x, JaCoCo to 0.8.12+, Checkstyle to 10.x (#8 partial, #10).

Exit: CI is green on JDK 17 with release 8, and the server image runs on 17.

### Phase 1: The flip (one PR)
1. `maven.compiler.release=17`. Remove `maven.compiler.source/target` and the plugin `<source>/<target>`. Use `<release>` (#5).
2. Remove the Javadoc `<source>8</source>` hard-codes (`pom.xml:312`, persistence-component `pom.xml:57`) and the H2 module's source/target-11 override.
3. Enforcer `maxJdkVersion` → 17. Delete the Java 11 dependency excludes (#2).
4. Full reactor `mvn clean install` in CI, then the full test, PCT, Snowflake/Arrow and Kerberos/GSS suites. Add `@SuppressWarnings("removal")` at the JEP 411 sites, or accept the warnings (#9).

Exit: green full build and tests. The published artifacts are classfile 61. The server image boots.
Communicate: downstream consumers of the published engine jars must run on JDK 17+.

### Phase 2: Upgrades unlocked by 17, no namespace change
- Test stack: Mockito 5 + byte-buddy, WireMock 3, Testcontainers 2, JsonUnit 3, optionally JUnit 6 (#8).
- Runtime codegen: add `JavaVersion.JAVA_11`/`JAVA_17`, switch `JavaHelper` to `JAVA_17`, and raise the `legend-pure` `PureJavaCompiler` default upstream (#6).
- Libraries: Eclipse Collections 12/13, H2 2.3, mssql `.jre11`, `ojdbc17`, Iceberg 1.7+, ANTLR 4.13 (regenerate the grammars), Logback 1.5/SLF4J 2 (if Dropwizard 1.3 allows), Jackson 2.18.
- Replace `Class.newInstance()` (`ExecutionNodeExecutor.java:251`).

Exit: one PR per library family, each with a green full build.

### Phase 3: Jakarta and newer-JDK program (separate initiative)
- Upgrade `legend-shared` first, then Dropwizard 4/5, Jetty 12, Jersey 3.1, HK2 3, pac4j 6, Hibernate Validator 8/9, Swagger v3. Rewrite the `javax.ws.rs`/`javax.servlet` imports with OpenRewrite.
- JEP 411 clean-up before any JDK 21/23 runtime: `Subject.current()`/`callAs()`, and remove the `AccessController`/`SubjectDomainCombiner` usage in Kerberos (#9).
- Optionally Jackson 3.x.

---

## 5. Evidence and confidence

Targeted builds that were run (all `-Dmaven.compiler.release=17`, JDK 17.0.20.1, no changes to the working tree; full commands are in the group reports):

| Group | Reactor | Result |
|---|---|---|
| 1 | `legend-engine-identity-core` `-am` | Compiled. Failed in `maven-dependency-plugin:2.10:analyze-only` (ASM, classfile 61). `IdentityTest` 3/3 failures (`NoSuchFieldException: modifiers`) |
| 1 | `legend-engine-shared-javaCompiler` (out of tree) | Compiled at 17. 11/11 tests passed |
| 2 | `…-executionPlan-connection` `-am test-compile -Dmdep.analyze.skip=true` | **BUILD SUCCESS, 78 modules**. 2 removal warnings (auth) |
| 2 | `…-h2-2.1.214-execution` `-am` | Compiled. Failed in dependency-plugin 2.10 (same ASM error) |
| 3 | `legend-engine-xt-identity-kerberos` `-am` | Compiled (classfile 61). 6 removal warnings |

Not verified:
- A full reactor build or full test suite at release 17 (out of scope for a targeted assessment).
- The Pure compiler and execution-plan reactors at release 17 as standalone builds.
- The SQL modules, most DB-specific execution modules, PCT suites, and the server image booting on JDK 17.
- The exact set of warnings the newer dependency plugin will report.
- Whether Enforcer `maxJdkVersion=1.8` actually fails during `install` (strongly expected).
- Whether the `sun.*` paths in the Snowflake JDBC and Snowpark drivers are reachable at runtime.
- Minimum Java versions marked (unverified) in §3.

Worker sessions:
[core](https://app.devin.ai/sessions/011bec92a128434fb70c49d18673cf0c) ·
[relational + SQL](https://app.devin.ai/sessions/d031440c39354253a5d1148bde2bbd2a) ·
[other xts + server/config](https://app.devin.ai/sessions/16e82e7fee814b63ae5a3b6589323ae5)
