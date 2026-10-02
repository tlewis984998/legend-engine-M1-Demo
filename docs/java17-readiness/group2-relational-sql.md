# Java 17 readiness: Group 2, relational store and SQL (`relational-sql`)

Repo: `tlewis984998/legend-engine-M1-Demo` @ `master` (`7a87b75794`). Scope: 148 POMs under `legend-engine-xts-relationalStore/` and 18 under `legend-engine-xts-sql/`. Environment: Temurin 17.0.20.1, Maven 3.9.16.
This was a read-only assessment. No repo files were changed, committed, or pushed. All experiments used `-D` overrides on the command line, so no POM edits had to be reverted. `git status` stayed clean.

## Summary and rating: **AMBER**

The relational and SQL code is close to ready for `release=17`. Nothing in scope needs an architectural migration.

- **Compiles on 17:** an experimental `-Dmaven.compiler.release=17` `test-compile` of `legend-engine-xt-relationalStore-executionPlan-connection -am` succeeded across 78 reactor modules. The main and test sources of every in-scope module in that reactor compiled; the only new diagnostics were two removal warnings.
- **Fails on 17 (confirmed):** `maven-dependency-plugin:2.10`, configured in the root `pom.xml`, crashes on Java 17 class files (ASM `ClassReader` `IllegalArgumentException`). It runs at `test-compile` with `failOnWarning=true`, so it fails every in-scope module build. This was reproduced on an in-scope module.
- **Runtime risks:**
  - Arrow (Snowflake driver and `RelationalResultToArrowIPCSerializer`) needs `--add-opens java.base/java.nio`. The flag is only set for Surefire, not for deployed servers.
  - Kerberos and secure-connect auth uses `AccessController` / `Subject.getSubject(AccessControlContext)`, which are deprecated for removal (JEP 411). On 17 this is only a warning, but it breaks on JDK 23+.
- **Java 11 is already the real runtime floor.** HikariCP 7.0.2, Arrow 18, Databricks JDBC 3.4.1 and ojdbc11 ship Java 11 (class file 55) bytecode. The `release=8` setting therefore doesn't make the artifacts run on Java 8.
- **Not blockers for release 17:** the javax-era stack in the postgres/SQL server (Jetty 9.4, Dropwizard 1.3, Jersey 2.25, Servlet 3.1, Logback 1.2.3) works on 17. It is the next migration boundary (Jakarta) but does not block this change.

## 1. Reflection on JDK internals / `setAccessible`

| module | file:line | finding | Java 17 impact | suggested fix |
|---|---|---|---|---|
| relationalStore-executionPlan-connection-tests | legend-engine-xts-relationalStore/legend-engine-xt-relationalStore-execution/legend-engine-xt-relationalStore-executionPlan-connection-tests/src/test/java/org/finos/legend/engine/plan/execution/stores/relational/connection/test/utils/ReflectionUtils.java:37,45 | `field.setAccessible(true)` on project classes (test util) | none (category b) | none |
| relationalStore-executionPlan-connection | .../executionPlan-connection/src/test/java/.../ds/state/TestConnectionManagement.java:52 | setAccessible on project singleton field | none | none |
| relationalStore-executionPlan-connection | .../executionPlan-connection/src/test/java/.../ds/specifications/TestLocalH2ConcurrentConnectionAcquisition.java:182 | setAccessible on project/Hikari field | none | none |
| relationalStore-executionPlan-connection-authentication | .../executionPlan-connection-authentication/src/test/java/org/finos/legend/engine/authentication/TestDatabaseAuthenticationFlowProviderSelector.java:33 | setAccessible on project singleton | none | none |
| relationalStore-executionPlan | .../executionPlan/src/test/java/.../plugin/TestPreprocessedConnectionFlowE2E.java:202; TestPreprocessedConnectionFlowAtSeam.java:301,316; TestHelpers.java:70 | setAccessible on project classes | none | none |
| relationalStore-executionPlan | .../executionPlan/src/test/java/.../serialization/TestRelationalResultToArrowIPCSerializer.java:63 | setAccessible on public `RelationalResult` field | none | none |
| sql-e2e-tests | legend-engine-xts-sql/legend-engine-xt-sql-e2e-tests/src/test/java/org/finos/legend/engine/postgres/e2e/TestPostgresParity.java:738,752 | setAccessible on Dropwizard `ResourceTestRule.resource` (third-party, unnamed module) | none | none |
| sql-postgres-server | legend-engine-xts-sql/legend-engine-xt-sql-postgres-server/src/main/java/org/finos/legend/engine/postgres/protocol/wire/PostgresWireProtocol.java:25,763 | `com.sun.security.jgss.GSSUtil.createSubject` | none: `jdk.security.jgss` exports `com.sun.security.jgss`, and compiled under release 17 | keep (supported API); optionally isolate behind a helper |
| root (affects all) | pom.xml:154 | `surefire.vm.params` includes `--add-opens=java.base/java.nio=ALL-UNNAMED` | none for tests; documents Arrow requirement | also add to runtime launchers (see blocker 2) |
| relationalStore-snowflake-PCT | legend-engine-xts-relationalStore/legend-engine-xt-relationalStore-dbExtension/legend-engine-xt-relationalStore-snowflake/legend-engine-xt-relationalStore-snowflake-PCT/pom.xml:120-125 | duplicate argLine with comment: Arrow allocator reflects into java.nio, "JDK 16+ blocks that, and every query then fails inside the driver" | runtime failure without flag | keep; centralise; make sure it isn't dropped when argLine is overridden |
| third-party: Arrow 18 (`arrow-memory-core`) | pom.xml (arrow.version, ~line 172) | `MemoryUtil` uses `sun.misc.Unsafe` and reflects into `java.nio.Buffer.address`. Error text in the jar: ``You must start Java with `--add-opens=java.base/java.nio=org.apache.arrow.memory.core,ALL-UNNAMED` `` | runtime failure without flag | add flag to all JVM launch configs (server, Docker, IDE run configs) |
| third-party: Snowflake JDBC 4.3.2 | pom.xml (snowflake.version, ~line 164) | jdeps: references `sun.misc.SharedSecrets`, `sun.misc.JavaLangAccess` (absent since 9), `sun.util.calendar.ZoneInfo`, `sun.security.x509.AlgorithmId`; bundles Arrow | unknown: probably guarded/legacy paths, but the Arrow path needs add-opens | keep flag; run Snowflake PCT on 17 |
| third-party: Netty 4.1.133, Testcontainers 1.21.4, Trino 438, Databricks 3.4.1, DB2 JCC 11.5.9, Eclipse Collections 10.2.0 | pom.xml properties | jdeps: `sun.misc.Unsafe` (jdk.unsupported, still accessible on 17) | none / warning only | none |

**Static searches with no hits in scope (production or test):** `sun.misc.*` imports, `jdk.internal.*`, `sun.security.krb5` imports, `com.sun.security.auth` imports, `--add-exports`, `--illegal-access`, and `setAccessible` on `java.*` classes.

## 2. SecurityManager / JEP 411

| module | file:line | finding | Java 17 impact | suggested fix |
|---|---|---|---|---|
| relationalStore-executionPlan-connection | legend-engine-xts-relationalStore/legend-engine-xt-relationalStore-execution/legend-engine-xt-relationalStore-executionPlan-connection/src/main/java/org/finos/legend/engine/plan/execution/stores/relational/connection/authentication/strategy/InteractiveAuthenticationStrategy.java:24,32 | `Subject.getSubject(AccessController.getContext())`; javac on release 17 warns: "AccessController ... deprecated and marked for removal", "getSubject(AccessControlContext) ... deprecated and marked for removal" | warning only on 17. On JDK 23+ `getSubject` throws `UnsupportedOperationException` unless the SecurityManager is allowed | replace with `Subject.current()` (JDK 18+) through a small compat helper, or track the subject explicitly |
| relationalStore-executionPlan-connection | .../connection/authentication/AuthenticationStrategy.java:81 | `Subject.doAs(subject, PrivilegedExceptionAction)` | none on 17 (deprecated, not for removal, in 18+) | later migrate to `Subject.callAs` (18+) |
| sql-postgres-server | legend-engine-xts-sql/legend-engine-xt-sql-postgres-server/src/main/java/org/finos/legend/engine/postgres/protocol/wire/PostgresWireProtocol.java:749 | `Subject.doAs` for GSS acceptor credential | none on 17 | `Subject.callAs` later |
| sql-postgres-server | .../postgres/protocol/sql/handler/jdbc/catalog/CatalogManager.java:70,75 | `Subject.doAs` | none on 17 | same |
| sql-postgres-server | .../postgres/protocol/sql/handler/legend/statement/LegendPreparedStatement.java:66,72,123,128; LegendStatement.java:54,60 | `Subject.doAs` | none on 17 | same |
| sql-postgres-server | .../postgres/PostgresServerLauncher.java:81 | `System.setProperty("java.security.krb5.conf", ...)` | none | none |

**No hits in scope:** `System.setSecurityManager`, `getSecurityManager`, `doPrivileged`, `java.security.Policy`, custom `SecurityManager` subclasses, and `java.security.manager` properties.

## 3. Deprecated-for-removal / removed APIs

| module | file:line | finding | Java 17 impact | suggested fix |
|---|---|---|---|---|
| relationalStore-executionPlan-connection | InteractiveAuthenticationStrategy.java:32 (see §2) | the only removal warnings from the release-17 compile of in-scope code | warning only | see §2 |
| relationalStore-executionPlan | .../executionPlan/src/test/java/.../exploration/TestSchemaExploration.java:322 | `new Double()` is the project protocol type `...relational.model.datatype.Double` (import line 43), not `java.lang.Double` | none (false positive) | none |

**No hits in scope:** boxing constructors on `java.lang` types, `finalize()` overrides, `runFinalization`, `Thread.stop/suspend/resume`, Nashorn/`ScriptEngineManager`, JAXB `javax.xml.bind`, `javax.activation`, `javax.annotation.PostConstruct`, JAX-WS, CORBA, Pack200, `javax.security.cert`, `sun.misc.BASE64*`, and `Thread.getId`.

## 4. Libraries

Class file major version 52 = Java 8, 55 = Java 11. Versions were measured on the jars.

| artifact | current | defined in | class ver | newer major (min Java) | pinned by Java 8? | broken on 17 at current? | notes |
|---|---|---|---|---|---|---|---|
| com.zaxxer:HikariCP | 7.0.2 | pom.xml hikaricp.version | 55 | already latest | no | no | already needs Java 11 |
| org.apache.arrow:arrow-memory-core/vector/jdbc | 18.0.0 | pom.xml arrow.version | 55 | 18.x line (Java 11) | no | needs `--add-opens java.base/java.nio` | runtime flag required |
| net.snowflake:snowflake-jdbc | 4.3.2 | pom.xml snowflake.version | 52 | current | no | works with java.nio opens (per PCT comment); not run here | jdeps shows legacy `sun.*` refs |
| com.databricks:databricks-jdbc | 3.4.1 | pom.xml databricks.version | 55 | current | no | not observed | Java 11 bytecode |
| com.oracle.database.jdbc:ojdbc11 | 23.6.0.24.10 | relationalStore-oracle-execution/pom.xml:78-79 | 55 | ojdbc17 (Java 17) | no | no | could move to ojdbc17 after the flip |
| io.trino:trino-jdbc | 438 | relationalStore-trino-PCT/pom.xml:296-297 | 52 | newer releases (Java 11+ / 17+ in recent lines, unverified) | no | not observed | |
| com.google.cloud:google-cloud-spanner-jdbc | 2.35.0 | spanner-jdbc-shaded/pom.xml:21-22 | 52 | 2.x (Java 8) | no | not observed | shaded |
| software.amazon.jdbc:aws-advanced-jdbc-wrapper | 3.3.0 | aurora execution pom.xml:81-82 | 52 | current | no | not observed | |
| org.postgresql:postgresql | 42.7.4 | pom.xml postgres.version | 52 | 42.7.x | no | no | |
| com.amazon.redshift:redshift-jdbc42 | 2.0.0.3 | pom.xml redshiftJDBC.version | 52 | 2.1.x (Java 8) | no | not observed | old; bump for fixes |
| com.h2database:h2 | 2.1.214 | pom.xml h2.version; h2-execution-2.1.214/pom.xml:73-74 | 52 | 2.2/2.3 (Java 11) | partly (module compiles with source/target 11 at pom lines 33-34) | no | already on the 2.x line; `AlloyH2Server` uses `Server.createTcpServer` |
| org.duckdb:duckdb_jdbc | 1.3.0.0 | pom.xml duckdb.version | 52 | current | no | not observed | native lib |
| com.clickhouse:clickhouse-jdbc | 0.9.6 | pom.xml clickhouse.version | 52 | current | no | not observed | |
| com.ibm.db2:jcc | 11.5.9.0 | pom.xml db2jcc.version | 50 | 12.x | no | not observed | uses Unsafe |
| org.mariadb.jdbc:mariadb-java-client | 3.0.6 | pom.xml mariadb.version | 52 | 3.5.x | no | no | |
| com.microsoft.sqlserver:mssql-jdbc | 9.4.1.jre8 | pom.xml:4262-4264 | 52 | 12.x `.jre11`; `.jre17` exists in recent lines | yes (jre8 classifier) | no | switch to the jre11/jre17 artifact after the flip |
| io.netty:netty-* | 4.1.133.Final | pom.xml io.netty.version | 50 | 4.2.x | no | no | postgres wire server; epoll with NIO fallback |
| org.testcontainers:testcontainers | 1.21.4 | pom.xml testcontainers.version | 52 | 2.x (Java version unverified) | no | no | |
| org.eclipse.jetty:jetty-* | 9.4.44.v20210927 | pom.xml jetty.version | — | 10 (Java 11, javax), 11 (Java 11, jakarta), 12 (Java 17) | yes | no | used by sql-postgres-server metrics/HTTP |
| io.dropwizard:* | 1.3.29 | pom.xml dropwizard.version | — | 2.x (8), 3.x (11, javax), 4.x (11, jakarta) | yes | not observed (tests compiled) | Jakarta boundary |
| org.glassfish.jersey:* | 2.25.1 | pom.xml jersey.version | — | 3.x (jakarta) | yes | not observed | |
| javax.servlet:javax.servlet-api | 3.1.0 | pom.xml javax.servlet.version | — | jakarta.servlet 5/6 | yes | no | |
| ch.qos.logback:logback-classic | 1.2.3 | pom.xml logback.version | — | 1.4/1.5 (Java 11) | yes | no | |
| org.mockito:mockito-core / mockito-inline | 4.4.0 / 5.2.0 | pom.xml mockito-core.version / mockito-inline.version | 52 | 5.x (Java 11) | yes (core) | not observed; version skew | align both on 5.x after the flip |
| net.bytebuddy:byte-buddy | 1.11.20 | pom.xml byte-buddy.version | 49 | 1.15+ | no | supports Java 17 class files; old | bump with Mockito |
| org.antlr:antlr4 / antlr4-maven-plugin | 4.8-1 | pom.xml antlr.version | — | 4.13 (Java 11) | no (4.10+ changes the generated ATN format) | no | SQL grammar |
| org.eclipse.collections | 10.2.0 | pom.xml eclipsecollections.version | 52 | 11/12 (Java 8/11) | no | no | |
| Calcite | not used | — | — | — | — | — | no `calcite` dependency found in scope |
| BigQuery | no JDBC driver dependency in bigquery-execution POM | — | — | — | — | — | not resolved |

## 5. Build / tooling (root POM items that affect this group)

| module | file:line | finding | Java 17 impact | suggested fix |
|---|---|---|---|---|
| all in scope | pom.xml:249 (maven.dependency.plugin.version=2.10); execution at pom.xml:555-571 (`analyze-only`, phase test-compile, `failOnWarning=true`) | ASM inside 2.10 can't read class file 61. Reproduced: `legend-engine-xt-relationalStore-h2-execution-2.1.214` with `-Dmaven.compiler.release=17` gives `maven-dependency-plugin:2.10:analyze-only ... failed. IllegalArgumentException` | **compile break (build failure)** | bump to 3.6.1+ (verified working on a standalone JDK 17 project); then re-triage any new used/unused-declared warnings, since failOnWarning=true |
| all | pom.xml:148-150 | `source/target=1.8`, `release=8` | — | set `maven.compiler.release=17`. The compiler config passes only source/target, so also update or remove those |
| relationalStore-h2-execution-2.1.214 | .../h2-execution-2.1.214/pom.xml:33-34 | module overrides source/target to 11, but the inherited `maven.compiler.release=8` takes precedence in compiler 3.8.0 | none after the flip | remove the overrides when the root moves to 17 |
| all | pom.xml:248 compiler 3.8.0 | release 17 compile verified working | none | optional bump to 3.11+ |
| all | pom.xml:262 surefire 2.22.2 | works on 17; argLine carries add-opens | none | optional 3.x |
| all | pom.xml:245 jacoco 0.8.10 | supports Java 17 (≥0.8.7) | none | none |
| all | pom.xml:260 shade 3.4.1; pom.xml:247 checkstyle 3.1.1; pom.xml:252 javadoc 3.3.1 | no Java 17 issue observed (shade failed only because parent POMs weren't installed in the single-module experiment) | none observed | none |
| all Pure modules | pom.xml:125 legend.pure.version 5.105.0 | Pure compiler/generation plugins ran on JDK 17 in the `-am` reactor with `release=17` | none observed | none |
| SQL grammar | pom.xml:397-398 antlr4-maven-plugin 4.8-1 | plugin runs on 17 | none | none |

**Other tooling searches:** no in-scope `<source>8` / `<target>1.8` / `<release>8` hard-codes, no in-scope tests asserting `java.version`, and no in-scope `javax.tools` invocations with `-source 8`.

## 6. Top blockers (effort × risk)

1. **maven-dependency-plugin 2.10 can't analyse Java 17 class files.** Files: pom.xml:249, pom.xml:555-571. Fix: bump to 3.6.1+ and triage any new warnings (failOnWarning). Effort S, risk low.
2. **Arrow / Snowflake `java.nio` opens missing outside Surefire.** Files: pom.xml:154, snowflake-PCT/pom.xml:120-125, executionPlan/.../RelationalResultToArrowIPCSerializer.java:77. Fix: add `--add-opens=java.base/java.nio=org.apache.arrow.memory.core,ALL-UNNAMED` to every server launch (Docker / entrypoint / `JAVA_TOOL_OPTIONS`, owned by group 3), and keep it when modules override argLine. Effort S, risk medium.
3. **JEP 411 removal APIs in Kerberos / secure-connect auth.** Files: InteractiveAuthenticationStrategy.java:24,32. Fix: move to a `Subject.current()` compat helper (or pass the subject explicitly). Warning on 17, failure on 23+. Effort S, risk medium.
4. **mssql-jdbc `.jre8` classifier.** File: pom.xml:4262-4264. Fix: switch to the jre11/jre17 build (12.x). Effort S, risk low.
5. **Mockito core 4.4.0 vs inline 5.2.0 skew; Byte Buddy 1.11.20.** File: pom.xml (mockito/byte-buddy properties ~lines 174, 219-220). Fix: align on Mockito 5.x plus a current Byte Buddy once Java 11+ is the floor. Effort S, risk low.
6. **Run the Snowflake, Arrow and Kerberos test paths on 17.** jdeps shows legacy `sun.*` refs in Snowflake 4.3.2. Files: snowflake-PCT/pom.xml, PostgresWireProtocol.java:749-763. Fix: run the Snowflake PCT and GSS e2e tests on 17 with release 17. Effort M, risk medium.
7. **javax stack (Jetty 9.4 / Dropwizard 1.3 / Jersey 2.25 / Servlet 3.1 / Logback 1.2.3) in the SQL server.** Files: pom.xml jetty/dropwizard/jersey/logback properties; sql-postgres-server/pom.xml. Not a release-17 blocker; it's the next upgrade wave (Jakarta). Effort XL, risk high.
8. **Remove module-level source/target 11 overrides.** File: h2-execution-2.1.214/pom.xml:33-34. Effort S, risk low.
9. **Subject.doAs to callAs** (deprecated since 18). Files: AuthenticationStrategy.java:81, CatalogManager.java:70,75, LegendStatement.java:54,60, LegendPreparedStatement.java:66,72,123,128, PostgresWireProtocol.java:749. Effort S, risk low.

## 7. Commands run and outcome

- `java -version` / `mvn -version`: Temurin 17.0.20.1, Maven 3.9.16.
- `find legend-engine-xts-relationalStore legend-engine-xts-sql -name pom.xml | wc -l`: 148 / 18.
- `rg` searches for setAccessible, `sun.`, `com.sun.`, `jdk.internal`, Unsafe, add-opens/exports, SecurityManager/AccessController/doPrivileged/Subject, boxing constructors, finalize, Nashorn, JAXB, javax.annotation, etc. Results are in the tables above.
- `curl` to repo.maven.apache.org: HTTP 429 (rate-limited). Used the configured Google mirror (`maven-central.storage-download.googleapis.com`): HTTP 200.
- Downloaded driver/library jars. Measured class file versions (unzip + od) and ran `jdeps --jdk-internals --multi-release 17 --ignore-missing-deps`. For Arrow, the jar was unpacked to get past module-resolution errors.
- `javac --release 17 -Xlint:all` probe on InteractiveAuthenticationStrategy-style code: 2 removal warnings. `--release 21` is not supported by this JDK.
- Standalone Maven project with `release=17`: compiler 3.8.0 OK. dependency 2.10 `analyze-only` failed with ASM `IllegalArgumentException`. dependency 3.6.1 OK.
- `mvn clean install -DskipTests -Dmaven.compiler.release=17 -pl legend-engine-xts-relationalStore/.../legend-engine-xt-relationalStore-h2-execution-2.1.214`: **FAILED** in `maven-dependency-plugin:2.10:analyze-only` with `IllegalArgumentException`.
- Same with `-Dmdep.analyze.skip=true`: compile OK and class file major = 61. Then shade failed: "Non-resolvable parent POM" (parents not installed; not a Java 17 issue).
- `mvn test-compile -Dmaven.compiler.release=17 -Dmdep.analyze.skip=true -Dmaven.compiler.showWarnings=true -Dmaven.compiler.showDeprecation=true -pl legend-engine-xts-relationalStore/legend-engine-xt-relationalStore-execution/legend-engine-xt-relationalStore-executionPlan-connection -am`: **BUILD SUCCESS**, 78 modules, 8m23s. The only in-scope removal warnings were InteractiveAuthenticationStrategy.java:[32,34] and [32,46].
- `git status --short`: clean after all experiments.

## 8. Not verified

- No full build of the ~166 in-scope modules under release 17. Only the connection reactor (`h2-execution`, `relationalStore-protocol`, `executionPlan-connection` + core deps) was compiled. Not compiled: the SQL modules (grammar, transformation, postgres-server, http-api), the dbExtension execution modules beyond H2, the executionPlan module and the PCT modules. Static analysis found nothing in them that javac 17 would reject.
- No tests were executed on 17. That includes Snowflake/Arrow PCT, Kerberos/GSS e2e, testcontainers-based DB tests, the embedded H2 TCP server, and Netty epoll.
- No full `dependency:tree` for scoped modules, so transitive versions (e.g. the Byte Buddy actually resolved with mockito-inline 5.2.0) were not resolved. Mirror and rate-limit constraints applied.
- New dependency-analysis warnings after bumping maven-dependency-plugin to 3.x were not measured.
- Snowflake 4.3.2 `sun.misc.SharedSecrets` references: not confirmed whether they're reachable at runtime on 17.
- Minimum Java for Trino and Testcontainers newer majors was not confirmed from release notes.
- Runtime launch scripts / Dockerfiles (owned by group 3) were not checked for the Arrow add-opens flag.
