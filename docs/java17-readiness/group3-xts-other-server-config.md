# Java 17 readiness: Group 3, `xts-other-server-config`

Repo: `tlewis984998/legend-engine-M1-Demo` @ `7a87b75794` (master). Scope: 316 `pom.xml` files: the root POM plus every module that is not under `legend-engine-core/`, `legend-engine-xts-relationalStore/` or `legend-engine-xts-sql/`. The repo has 610 POMs in total.
Toolchain used for this assessment: Temurin 17.0.20.1, Maven 3.9.16, `jdeps` from JDK 17.

## 1. Summary and rating: **AMBER**

* **The runtime is already Java 17.** Every CI workflow builds and tests on Zulu JDK 17. Several third-party dependencies already ship Java 11 bytecode (classfile 55): Arrow 18, Deephaven 0.40.7, HikariCP 7, mongo-java-server 1.46, Databricks JDBC and lsp4j. The root enforcer rule has to exclude them explicitly. Java 8 is therefore only a *compile target*. It is no longer a working runtime.
* **The in-scope source code needs no changes to compile at release 17.** There is no Nashorn, JAXB, CORBA, JAX-WS, RMI activation, `sun.misc`, `jdk.internal`, `Unsafe`, `finalize()`, Thread stop/suspend/resume or `java.lang` boxing constructor anywhere in scope. A targeted Maven experiment with `-Dmaven.compiler.release=17` compiled the Kerberos identity module and produced classfile 61. The only new output was 6 deprecation-for-removal *warnings* from the `AccessController`/`AccessControlContext`/`SubjectDomainCombiner` calls.
* **The blockers are build tooling, not code:**
  * The experiment showed that `maven-dependency-plugin` 2.10 (`analyze-only`, which runs in every module) **crashes on Java 17 classfiles**.
  * The `enforceBytecodeVersion` rule has `maxJdkVersion=1.8`.
  * The JDK 11 range is still allowed by `requireJavaVersion`.
  * Javadoc has `<source>8</source>` hard-coded.
  * Checkstyle is 8.25.
  * The server container base image is **JDK 11** (`eclipse-temurin:11.0.17_8`), so release-17 classes would fail to load in production.
* **The server stack is a separate, larger migration.** It is javax-based and end-of-life: Dropwizard 1.3.29, Jetty 9.4, Jersey 2.25.1, HK2 2.5, javax.servlet 3.1, pac4j 4.x, legend-shared 0.37, logback 1.2.3. It runs on JDK 17 today, so it does **not** block `release=17`. That is why the rating is AMBER and not RED. Moving it to current majors would bring in Java 11/17 minimums plus the javax-to-jakarta namespace change; that is an architectural migration (XL) and should be tracked as its own project.

## 2. Findings

### 2.1 Reflection on JDK internals, `setAccessible`, `--add-opens`

| module | file:line | finding | Java 17 impact | suggested fix |
|---|---|---|---|---|
| legend-engine-application-query | legend-engine-application-query/src/main/java/org/finos/legend/engine/application/query/api/QueryStoreManager.java:518 | `field.setAccessible(true)` on the project's own `Query` class fields (b) | none | none |
| legend-engine-xt-generation (dsl-generation) | legend-engine-xts-generation/legend-engine-language-pure-dsl-generation/src/main/java/org/finos/legend/engine/language/pure/dsl/generation/config/ConfigBuilder.java:53 | `setAccessible(true)` on a public field of the project's generation-config classes (b) | none | none |
| legend-engine-xt-dataquality-api (test) | legend-engine-xts-dataquality/legend-engine-xt-dataquality-api/src/test/java/org/finos/legend/engine/language/dataquality/api/TestDataQualityExecuteReconciliation.java:149 | Test injects a private field of the project's `DataQualityExecute` (b) | none | none |
| root | pom.xml:154 | `surefire.vm.params` contains `--add-opens=java.base/java.nio=ALL-UNNAMED`, which Arrow memory needs (a, third-party) | none (already required on the JDK 17 runtime) | keep; consider also `--add-opens=java.base/java.nio=org.apache.arrow.memory.core` |
| CI | .github/workflows/build.yml:271, :318; .github/workflows/release.yml:212, :256 | `JDK_JAVA_OPTIONS=--add-opens=java.base/java.nio=org.apache.arrow.memory.core,ALL-UNNAMED` on test jobs | none (already in place) | keep |
| legend-engine-server-http-server | legend-engine-config/legend-engine-server/legend-engine-server-http-server/pom.xml:51-56 | Jib `jvmFlags` has **no** `--add-opens` for Arrow. The server pulls in Arrow through the extension collections | runtime_failure on JDK 17 (Arrow `MemoryUtil` init) when Arrow paths run once the image moves to 17. Not verified at runtime | add `--add-opens=java.base/java.nio=ALL-UNNAMED` to `jvmFlags` |
| legend-engine-repl-app-assembly | legend-engine-config/legend-engine-repl/legend-engine-repl-app-assembly/assemble/repl.sh:55 | `java -cp @… ` with no `--add-opens`. The data-cube REPL uses Arrow and DuckDB | runtime_failure on JDK 17 when Arrow paths run (inferred) | add the same `--add-opens` |
| legend-engine-repl-data-cube | legend-engine-config/legend-engine-repl/legend-engine-repl-data-cube/src/main/java/org/finos/legend/engine/repl/dataCube/server/REPLServerHelpers.java | Uses `com.sun.net.httpserver.*`. That package is exported by module `jdk.httpserver` | none | none |
| legend-engine-xt-identity-kerberos | legend-engine-xts-identity/legend-engine-xt-identity-kerberos/src/main/java/org/finos/legend/engine/shared/core/kerberos/LocalLoginConfiguration.java:35 (and UsernamePasswordAccountLoginConfiguration.java:44, SystemAccountLoginConfiguration.java:50) | JAAS config names `com.sun.security.auth.module.Krb5LoginModule` as a string. The package is exported by `jdk.security.auth` | none | none |
| third-party: eclipse-collections 10.2.0, jersey-common 2.25.1, netty-common 4.1.133, protobuf-java 3.25.3, prometheus simpleclient 0.8.1, testcontainers 1.21.4, trino-jdbc 422 | jdeps on jars | `sun.misc.Unsafe` from `jdk.unsupported`, which is still accessible on 17 | none (warning-level in later JDKs) | none for 17 |
| third-party: snowpark 1.16.0 (legend-engine-xt-snowflake-m2mudf-plan-executor) | legend-engine-xts-snowflake/legend-engine-xt-snowflake-m2mudf-plan-executor/pom.xml:114 | jdeps: `sun.security.util.DerInputStream/DerValue` (java.base, **not exported**) and `sun.reflect.ReflectionFactory` | runtime_failure (`IllegalAccessError`) if snowpark's private-key parsing path runs without `--add-exports java.base/sun.security.util=ALL-UNNAMED`. Unchanged by the release flag | upgrade snowpark, or add `--add-exports` where that path is used |
| third-party: wiremock-jre8 2.35.2 (tests) | root pom.xml:236 (`wiremock.version`) | jdeps: `sun.security.x509.*` (not exported), used for HTTPS self-signed cert generation | runtime_failure only if a test enables WireMock HTTPS | move to `org.wiremock:wiremock` 3.x (Java 11) |
| third-party: Immutables `value` 2.8.2 (annotation processor, persistence) | legend-engine-xts-persistence/legend-engine-xt-persistence-component/pom.xml:44 | jdeps: `com.sun.tools.javac.code.*` internals (jdk.compiler, not exported) | none in practice: CI already compiles on JDK 17 and the processor falls back. Not re-verified here | bump to 2.10.x |
| third-party: freemarker 2.3.34 | root pom.xml:189 | jdeps: `com.sun.org.apache.xpath.internal.*`, an optional XPath fallback | none | none |

### 2.2 SecurityManager / AccessController (JEP 411)

| module | file:line | finding | Java 17 impact | suggested fix |
|---|---|---|---|---|
| legend-engine-xt-identity-kerberos | legend-engine-xts-identity/legend-engine-xt-identity-kerberos/src/main/java/org/finos/legend/engine/shared/core/kerberos/ExecSubject.java:28 | `Subject.getSubject(AccessController.getContext())`. **Confirmed** under release 17: 2 "deprecated and marked for removal" warnings | warning_only on 17. On JDK 23+ `Subject.getSubject` throws UOE unless `-Djava.security.manager=allow` | on JDK 17 add `@SuppressWarnings("removal")`. When the runtime reaches 18+, move to `Subject.current()` / `Subject.callAs()` |
| legend-engine-xt-identity-kerberos | legend-engine-xts-identity/legend-engine-xt-identity-kerberos/src/main/java/org/finos/legend/engine/shared/core/kerberos/SubjectTools.java:117-120 | `AccessControlContext`, `AccessController.getContext()`, `SubjectDomainCombiner`. **Confirmed**: 4 removal warnings | warning_only | same as above |
| legend-engine-xt-identity-kerberos | …/shared/core/identity/credential/KerberosUtils.java:28, :34; …/kerberos/ExecSubject.java:46 | `Subject.doAs(...)` | none on 17 (deprecated for removal from JDK 18) | move to `Subject.callAs` on 18+ |
| legend-engine-xt-hostedService-generation | legend-engine-xts-hostedService/legend-engine-xt-hostedService-generation/src/main/java/org/finos/legend/engine/language/hostedService/generation/deployment/HostedServiceDeploymentManager.java:97, :133 | `Subject.doAs` | none on 17 | as above |
| legend-engine-xt-graphQL-http-api | legend-engine-xts-graphQL/legend-engine-xt-graphQL-http-api/src/main/java/org/finos/legend/engine/query/graphQL/api/GraphQL.java:61, :75 | `Subject.doAs` | none on 17 | as above |
| legend-engine-service-post-validation-runner | legend-engine-xts-service/legend-engine-service-post-validation-runner/src/main/java/org/finos/legend/engine/service/post/validation/runner/ServicePostValidationRunner.java:178 | `Subject.doAs` | none on 17 | as above |
| (all in scope) | none found | No `System.setSecurityManager`, `getSecurityManager`, `SecurityManager` subclass, `Policy`, `doPrivileged` or `java.security.manager` property | none | none |

### 2.3 Deprecated-for-removal / removed APIs

| module | file:line | finding | Java 17 impact | suggested fix |
|---|---|---|---|---|
| (all in scope) | none | No Nashorn / `ScriptEngineManager`, `javax.xml.bind`, `javax.activation`, JAX-WS, CORBA, RMI activation, `javax.security.cert`, Applet, Pack200, `sun.misc.BASE64*`, `finalize()`, `Thread.stop/suspend/resume`, `runFinalization`, `ThreadGroup` destroy/daemon, `java.lang.Compiler` | none | none |
| persistence relational-* (ansi, h2, duckdb, memsql, postgres, snowflake) | e.g. legend-engine-xts-persistence/legend-engine-xt-persistence-component/legend-engine-xt-persistence-component-relational-ansi/src/main/java/org/finos/legend/engine/persistence/components/relational/ansi/sql/AnsiDatatypeMapping.java:22-29 | `new Integer()/new Double()/new Boolean()` resolve to project SQL-DOM types (`…sqldom.schema.Integer`), **not** java.lang | none | none |
| legend-engine-xt-protobuf-grammar | legend-engine-xts-protobuf/legend-engine-xt-protobuf-grammar/src/main/java/org/finos/legend/engine/language/protobuf3/grammar/from/Protobuf3GrammarParser.java | `new Double()/new Float()` are protocol metamodel types | none | none |
| root | pom.xml:3583, :4390, :4407-4414 | JAXB: `javax.xml.bind:jaxb-api` is excluded from legend-shared/grpc. `jakarta.xml.bind-api` 2.3.3 and `jakarta.activation-api` 1.2.2 (still the `javax.*` packages) are managed explicitly | none (already explicit dependencies, as Java 11+ needs) | none |
| server / javax EE | 7 files in legend-engine-config/legend-engine-server, dataquality-api (2), graphQL-http-api (6), graphQL-relational-extension (1) import `javax.servlet`; 73 files across 18 top-level dirs import `javax.ws.rs` | javax EE APIs come from Maven artifacts, not the JDK | none for 17. Becomes a mass rename if Jersey 3, Jetty 11+ or Dropwizard 4 is adopted | track as part of the server-stack migration |
| legend-engine-xt-json-http-api; dataquality-api (test) | legend-engine-xts-json/legend-engine-xt-json-http-api/src/main/java/org/finos/legend/engine/external/format/jsonSchema/schema/generations/api/JSONSchemaGenerationService.java; …/dataquality/api/MockPac4jFeature.java | `javax.inject` (artifact `javax.inject:1`) | none | none |
| legend-engine-perf-benchmark; repl-client | legend-engine-config/legend-engine-perf-benchmark/src/main/java/org/finos/legend/engine/perf/PipelineBench.java:93, :99; legend-engine-config/legend-engine-repl/legend-engine-repl-client/src/main/java/org/finos/legend/engine/repl/client/Client.java:451 | `System.exit` (no SecurityManager trap is used to test it) | none | none |

### 2.4 Bytecode generation / javac invocation

| module | file:line | finding | Java 17 impact | suggested fix |
|---|---|---|---|---|
| legend-engine-xt-deephaven-executionPlan-test | legend-engine-xts-deephaven/legend-engine-xt-deephaven-executionPlan-test/src/main/java/org/finos/legend/engine/plan/execution/stores/deephaven/test/JavaSourceCompiler.java:39-63 | `ToolProvider.getSystemJavaCompiler()` with options `-classpath` only. No `-source`/`--release`, so it emits the running JDK's level (61 on 17) | none | none |
| flatdata/json/mongodb/deephaven/snowflake compiler extensions | e.g. legend-engine-xts-flatdata/legend-engine-xt-flatdata-runtime/src/main/java/org/finos/legend/engine/external/format/flatdata/FlatDataJavaCompilerExtension.java:47 | These plug into the core `EngineJavaCompiler`/`ExecutionPlanJavaCompilerExtension`. The `-source`/`-target` options live in legend-engine-core (another worker's scope) | unknown (see core report) | check against the core worker's finding on `EngineJavaCompiler` options |
| legend-engine-perf-benchmark | legend-engine-config/legend-engine-perf-benchmark/src/main/java/org/finos/legend/engine/perf/Environment.java:40 | Records `java.version` and does not assert on it | none | none |

## 3. Library inventory (root `dependencyManagement` + in-scope module overrides)

"Pinned by Java 8?" asks whether the *next* major line needs Java 11 or later, so that the Java 8 target is what blocks the upgrade. "Broken on 17?" refers to the current version on a JDK 17 runtime. Newer-major Java minimums come from upstream release notes and were not re-verified in this session (see §7).

| artifact | current | defined in | newer major | newer-major min Java | pinned by Java 8? | broken on 17? | notes |
|---|---|---|---|---|---|---|---|
| io.dropwizard:dropwizard-* | 1.3.29 | pom.xml:185 `dropwizard.version` | 2.1.x / 3.0 / 4.0 | 2.1: 8, 3.0: 11, 4.0: 11 (jakarta, Jetty 11/12, Jersey 3) | partly (2.1 still supports 8; the real pin is legend-shared + Jersey 2.25/HK2 coupling) | no (runs in CI on 17) | 1.3 is EOL. **javax→jakarta in 4.x** |
| org.eclipse.jetty:* | 9.4.44.v20210927 | pom.xml:209 | 10 / 11 / 12 | 10: 11, 11: 11 (jakarta), 12: 17 | yes | no | 9.4 is EOL. Server-image risk |
| org.glassfish.jersey.*:* | 2.25.1 | pom.xml:208 | 2.4x / 3.1 | 2.4x: 8, 3.1: 11 (jakarta) | no for 2.4x, yes for 3.x | no | 2.25.1 uses `sun.misc.Unsafe` (OK) |
| javax.ws.rs:javax.ws.rs-api | 2.0.1 | pom.xml:207 `jaxrs.version` | jakarta.ws.rs-api 3.x/4.x | 3.1: 11 | n/a | no | namespace change |
| javax.servlet:javax.servlet-api | 3.1.0 | pom.xml:206 | 4.0.1 / jakarta.servlet 5/6 | 6.0: 11 | n/a | no | namespace change |
| org.glassfish.hk2:* | 2.5.0-b32 | pom.xml:197 | 2.6 / 3.x | 3.x: 11 (jakarta) | no | no | beta build line |
| io.swagger:swagger-annotations | 1.5.20 | pom.xml:232 | io.swagger.core.v3 2.2.x (`-jakarta` variant) | 8 / 11 | no | no | |
| com.smoketurner:dropwizard-swagger | 1.3.17-1 | pom.xml:183 | 2.x / 4.x | follows Dropwizard | coupled to DW | no | |
| org.pac4j:pac4j-core | 4.5.8 | pom.xml:227 | 5.x / 6.x | 5: 11, 6: 17 (jakarta) | **yes** | no | |
| org.pac4j:jersey-pac4j, org.pac4j.jax-rs:core | 4.0.0 / 4.0.0 | pom.xml:225-226 | 5.x / 6.x | 11 / 17 | yes | no | |
| org.finos.legend.shared:legend-shared-* | 0.37.0 | pom.xml:126 | (FINOS) | unknown | coupled to DW 1.3 | no | owner of the auth/pac4j/Kerberos server glue |
| org.hibernate:hibernate-validator | 5.4.3.Final | pom.xml:196 | 6.2 / 8.0 / 9.0 | 6.2: 8, 8: 11 (jakarta), 9: 17 | no for 6.2 | no | |
| ch.qos.logback:* | 1.2.3 | pom.xml:217 | 1.3 / 1.5 | 1.3: 8, 1.5: 11 | partly (1.3 needs slf4j 2) | no | CVEs; also slf4j 1.7.36 (pom.xml:230) → 2.0 (8) |
| com.fasterxml.jackson.*:* | 2.10.5 / databind 2.10.5.1 | pom.xml:202-203 | 2.18+ / 3.0 | 2.x: 8, 3.0: 17 | no | no | very old 2.x; DW 1.3 coupling |
| org.yaml:snakeyaml | 1.33 | pom.xml:231 | 2.x | 8 | no | no | CVE fixed in 2.0 |
| org.eclipse.collections:* | 10.2.0 | pom.xml:186 | 11.1 / 12 / 13 | 11: 8, 12+: 11 | partly | no | `sun.misc.Unsafe` (OK) |
| com.google.guava:guava | 33.4.6-jre | pom.xml:194 | n/a | 8 | no | no | current |
| io.netty:* | 4.1.133.Final | pom.xml:201 | 4.2 | 8 | no | no | |
| io.grpc:* / com.google.protobuf:* | 1.65.1 / 3.25.3 | pom.xml:192, :229 | grpc 1.7x / protobuf 4.x | 8 | no | no | protobuf/deephaven modules |
| org.apache.arrow:* | 18.0.0 | pom.xml:172 | 19+ | 11 already | n/a (already Java 11 bytecode, excluded at pom.xml:~650) | **needs `--add-opens java.base/java.nio`** | arrow, deephaven, bigqueryFunction, data-cube |
| io.deephaven:* | 0.40.7 | pom.xml:181 | — | 11+ already (excluded from bytecode rule) | n/a | no | |
| org.apache.iceberg:iceberg-* | 1.3.0 | legend-engine-xts-iceberg/pom.xml (`iceberg.version`) | 1.7+ | 11 (1.7 dropped Java 8) | **yes** | no (not verified at runtime) | the persistence module has no Hadoop/Parquet/Avro/Spark/Flink; Pure's hadoop-common is excluded at pom.xml:3279 |
| software.amazon.awssdk:* | 2.17.129 | pom.xml:170 | 2.2x | 8 | no | no | iceberg, authentication |
| org.mongodb:mongodb-driver-* | 5.3.1 | pom.xml:221 | 5.x current | 8 | no | no | mongodb, shared-mongo |
| de.bwaldvogel:mongo-java-server-* | 1.46.0 | pom.xml:233 | — | 11 already (excluded) | n/a | no | test |
| org.testcontainers:* | 1.21.4 | pom.xml:234 | 2.x | 17 (unverified) | likely | no | |
| org.mockito:mockito-core / mockito-inline | 4.4.0 / **5.2.0** | pom.xml:219-220 | 5.x | 11 | **yes** | no, but core 4.4 vs inline 5.2 is a version mismatch | align on 5.x with byte-buddy ≥1.14 |
| net.bytebuddy:byte-buddy | 1.11.20 | pom.xml:174 | 1.15/1.17 | 8 | no | no for 17 (needs newer for 21+) | |
| junit:junit / org.junit.jupiter | 4.13.1 / 5.11.0 | pom.xml:214, :213; persistence-component pom.xml:45 | JUnit 6 | 17 | yes (JUnit 6) | no | |
| com.github.tomakehurst:wiremock-jre8 | 2.35.2 | pom.xml:236 | org.wiremock:wiremock 3.x | 11 (Jetty 11, jakarta) | **yes** | uses `sun.security.x509` (HTTPS only) | |
| net.javacrumbs.json-unit:* | 2.17.0 | pom.xml:212 | 3.x/4.x | 17 (unverified) | yes | no | xml/json tests |
| org.immutables:value | 2.8.2 | persistence-component pom.xml:44 | 2.10.x | 8 | no | no (javac internals with fallback) | |
| org.antlr:antlr4(-runtime) | 4.8-1 | pom.xml:171 | 4.13.x | tool 11, runtime 8 | partly | no | |
| io.github.classgraph:classgraph | 4.8.25 | pom.xml:190 | 4.8.17x | 7+ | no | not verified (very old) | changetoken tests |
| org.codehaus.janino:janino | 3.1.0 | pom.xml:204 | 3.1.12 | 8 | no | no | |
| org.bouncycastle:bcprov/bcpkix-jdk15on | 1.67 | pom.xml:173 | *-jdk18on 1.78+ | 8 | no | no | artifact rename; CVEs |
| io.prometheus:simpleclient | 0.8.1 | pom.xml:228 | 1.x | 8 | no | no | |
| org.freemarker:freemarker | 2.3.34 | pom.xml:189 | — | 8 | no | no | |
| org.apache.httpcomponents:httpclient/httpcore | 4.5.13 / 4.4.13 | pom.xml:199-200 | httpclient5 | 8 | no | no | elasticsearch, hostedService |
| commons-io / commons-lang3 / commons-text | 2.7 / 3.18.0 / 1.10.0 | pom.xml:178-180 (lang3 3.10 in some persistence modules) | — | 8 | no | no | |
| io.opentracing:* / zipkin reporter | 0.32.0 / 2.15.0 | pom.xml:223-224, :237 | OpenTelemetry | 8 | no | no | opentracing is archived |
| io.dropwizard.metrics:* | 4.1.16 | pom.xml:184 | 4.2 | 8 | no | no | |
| jakarta.xml.bind-api / jakarta.activation-api | 2.3.3 / 1.2.2 | pom.xml:4407-4414 | 3.x/4.x | 11 (jakarta packages) | n/a | no | |
| org.jline:jline | 3.26.3 | pom.xml:238 | — | excluded from bytecode rule | n/a | no | REPL |
| com.snowflake:snowpark | 1.16.0 | legend-engine-xts-snowflake/legend-engine-xt-snowflake-m2mudf-plan-executor/pom.xml:115 | 1.1x latest | 8/11/17 | no | uses non-exported `sun.security.util` | |
| com.google.cloud:google-cloud-bigquery | 2.29.0 | bigqueryFunction module pom | 2.4x | 8 | no | no | |
| io.trino:trino-jdbc | 422 | in-scope module pom | 4xx | 8 for the JDBC driver (unverified) | no | no | |
| com.microsoft.sqlserver:mssql-jdbc | 9.4.1.jre8 | in-scope module pom | 12.x.jre11 | 11 | **yes** (classifier) | no | |
| org.eclipse.microprofile.openapi:microprofile-openapi-api | 3.1 | legend-engine-xts-deephaven/legend-engine-xt-deephaven-executionPlan/pom.xml:255 | 4.x | 11 | — | no | |
| DB drivers (h2 2.1.214, postgres 42.7.4, snowflake 4.3.2, databricks 3.4.1, duckdb 1.3.0.0, mariadb 3.0.6, clickhouse 0.9.6, HikariCP 7.0.2) | — | pom.xml:158-167 | — | HikariCP 7 and databricks already Java 11 | — | no | covered in detail by the relational worker |
| MCP / Python / Elasticsearch / Avro / GraalVM / Nashorn | — | — | — | — | — | — | No MCP SDK (the mcp modules use Jackson only), no GraalVM/Jython/JEP (python modules are Pure + testcontainers), no Elasticsearch client (Pure + httpclient), no `org.apache.avro` runtime, no Spark/Flink/Kafka/Hadoop in scope |

## 4. Build/tooling issues

| item | where | status for release 17 | fix |
|---|---|---|---|
| `maven.compiler.source/target=1.8`, `release=8` | pom.xml:148-150; plugin config `<source>/<target>` at pom.xml:498-499 | must change | set `maven.compiler.release=17` and drop source/target (plugin config: `<release>${maven.compiler.release}</release>`) |
| `maven-compiler-plugin` 3.8.0 | pom.xml:248 | works (the experiment produced classfile 61) | optional bump to 3.13.x |
| **`maven-dependency-plugin` 2.10 `analyze-only`** (failOnWarning) | pom.xml:249, :556-570 | **BREAKS**: `IllegalArgumentException` at `org.objectweb.asm.ClassReader.<init>`. ASM 5 cannot read classfile 61. Confirmed by experiment | bump to ≥3.3.0 (ASM 9); recommend 3.8.x. Expect new used/unused warnings from the newer analyzer |
| **`enforceBytecodeVersion maxJdkVersion=1.8`** (extra-enforcer-rules 1.12.0) | pom.xml:592 (+ excludes ~:595-670) | will fail as soon as one in-reactor module depends on another compiled at 61. This is inferred: the experiment only consumed released 4.114.2 jars | set `maxJdkVersion` to 17 (or `${maven.compiler.release}`) and delete the Java 11 excludes |
| `requireJavaVersion [11.0.10,12),[17,18)` | pom.xml:151 (used at :686, :733, :782) | JDK 11 cannot run `--release 17`; this also blocks JDK 21 | `[17,)` or `[17,22)` |
| Duplicate `maven.enforcer.plugin.version` (3.0.0-M1 and 3.5.0) | pom.xml:250 and :258 | the last one wins (3.5.0 ran) | remove line 250 |
| Javadoc `<source>8</source>` hard-codes | pom.xml:312; legend-engine-xts-persistence/legend-engine-xt-persistence-component/pom.xml:57 | javadoc will reject Java 9+ syntax once it is used (release profile) | `<source>${maven.compiler.release}</source>` or remove |
| `maven-javadoc-plugin` 3.3.1 | pom.xml:252 | OK | optional bump |
| `maven-surefire-plugin` 2.22.2 | pom.xml:262 | works on 17 (the experiment phase ran) | bump to 3.2+/3.5 (better module-path and JDK 17 support); no failsafe in scope |
| JaCoCo 0.8.10 | pom.xml:245 | supports classfile 61 (≥0.8.7) | bump to ≥0.8.11 before any JDK 21 move |
| Checkstyle plugin 3.1.1 / checkstyle 8.25 | pom.xml:247 | OK for Java 8 syntax. **Cannot parse** records, text blocks, switch expressions or `instanceof` patterns | bump to checkstyle 10.x / plugin 3.4+ before adopting Java 17 syntax |
| `maven-shade-plugin` 3.4.1 | pom.xml:260 | OK (ASM 9.x) | none |
| `maven-jar-plugin` 3.1.2, build-helper 3.2.0, flatten 1.6.0, resources 3.2.0 | pom.xml:241-261 | ran fine in the experiment | none |
| legend-pure maven plugins (`legend-pure-maven-compiler`, generation) | pom.xml:417 | not tested (core/pure scope). The generated Java is compiled by legend-pure tooling | verify with the core worker |
| **Server Jib base image `eclipse-temurin:11.0.17_8-jdk-jammy`** | legend-engine-config/legend-engine-server/legend-engine-server-http-server/pom.xml:39 (Jib 3.4.3 at :36) | **runtime_failure**: classfile 61 → `UnsupportedClassVersionError` on JDK 11 | `eclipse-temurin:17-jre-jammy` (or 17-jdk if runtime javac is needed for plan compilation, which is likely); add the Arrow `--add-opens` to `jvmFlags` |
| Perf benchmark container | legend-engine-config/legend-engine-perf-benchmark/scripts/run-in-container.sh:28 | `eclipse-temurin:17-jdk`: OK | none |
| REPL launcher | legend-engine-config/legend-engine-repl/legend-engine-repl-app-assembly/assemble/repl.sh:55 | relies on the user's `java`. Needs ≥17 and `--add-opens` for Arrow | document the JDK 17 requirement; add the flag |
| CI JDK | .github/workflows/build.yml:71-72, :136-137; release.yml:96-97; code-quality.yml:66; legend-stack-release.yml:46-47; performance.yml:84-85 | Zulu 17 everywhere | none |
| Dockerfiles | none in scope (`find` found no Dockerfile/compose) | — | — |
| Toolchains | no `maven-toolchains-plugin` / `toolchains.xml` in scope | — | none |
| `CLAUDE.md` says JDK 11 / enforcer `[11.0.10,12)` | CLAUDE.md | stale documentation | update after the flip |

## 5. Top blockers (ranked by effort × risk)

1. **`maven-dependency-plugin` 2.10 crashes on Java 17 classfiles** (confirmed). Files: pom.xml:249, :556-570. Fix: bump to 3.8.x and triage any new analyzer warnings, which fail the build because `failOnWarning=true`. Effort S/M, risk medium.
2. **Server container runs JDK 11** (`eclipse-temurin:11.0.17_8-jdk-jammy`). Release-17 classes cannot load. File: legend-engine-config/legend-engine-server/legend-engine-server-http-server/pom.xml:39, :51-56. Fix: 17 base image plus `--add-opens=java.base/java.nio=ALL-UNNAMED`, then smoke-test the image. Effort S, risk medium (production runtime change).
3. **Enforcer bytecode/JDK rules pinned to 8/11.** Files: pom.xml:151, :592 (+ exclude list), :250 (duplicate property). Fix: `maxJdkVersion=17`, `requireJavaVersion=[17,)`, prune excludes. Effort S, risk low.
4. **Compiler properties/config.** Files: pom.xml:148-150, :498-499. Fix: `release=17` only. Effort S, risk low.
5. **Javadoc `<source>8</source>` hard-codes.** Files: pom.xml:312, legend-engine-xts-persistence/legend-engine-xt-persistence-component/pom.xml:57. Fix: parameterise. Effort S, risk low.
6. **Kerberos `AccessController`/`AccessControlContext`/`SubjectDomainCombiner` removal warnings** (confirmed; 6 warnings). Files: legend-engine-xts-identity/legend-engine-xt-identity-kerberos/src/main/java/org/finos/legend/engine/shared/core/kerberos/ExecSubject.java:28, SubjectTools.java:117-120; plus `Subject.doAs` sites (KerberosUtils.java:28/34, HostedServiceDeploymentManager.java:97/133, GraphQL.java:61/75, ServicePostValidationRunner.java:178). Fix: `@SuppressWarnings("removal")` on 17; plan `Subject.current()`/`callAs()` when moving to a 18+ runtime (mandatory before JDK 23+). Effort M, risk medium (auth path).
7. **Arrow `java.nio` `--add-opens` missing from non-CI launchers.** Files: repl.sh:55, server Jib `jvmFlags`. Fix: add the flag. Effort S, risk medium.
8. **Checkstyle 8.25 cannot parse Java 14-17 syntax.** File: pom.xml:247. Fix: checkstyle 10.x. Effort S, risk low (do it before using new syntax).
9. **Test stack pinned/mismatched.** Mockito core 4.4.0 vs inline 5.2.0, byte-buddy 1.11.20, wiremock-jre8 (`sun.security.x509`), surefire 2.22.2. Files: pom.xml:174, :219-220, :236, :262. Fix: Mockito 5.x + byte-buddy 1.15+, wiremock 3.x, surefire 3.5. Effort M, risk medium.
10. **javax-based, end-of-life server stack** (not required for release 17): Dropwizard 1.3.29, Jetty 9.4, Jersey 2.25.1, HK2 2.5, javax.servlet 3.1, pac4j 4.x, legend-shared 0.37, hibernate-validator 5.4, logback 1.2.3. Files: pom.xml:126, :183-186, :196-197, :206-209, :217, :225-227. 73 `javax.ws.rs` and 16 `javax.servlet` source files in scope. Fix: a separate program (Dropwizard 4 / Jetty 12 / Jersey 3 / pac4j 6 → jakarta.*, which needs legend-shared upgraded first). Effort XL, risk high.

## 6. Commands run and outcomes

* `java -version` / `mvn -v` / `which jdeps`: Temurin 17.0.20.1, Maven 3.9.16; JDK 8 also present.
* `find . -name pom.xml` with the three excluded trees: 316 in scope (610 total).
* `rg` scans (in scope, excluding `target/`) for `setAccessible`, `sun.*`, `com.sun.*`, `jdk.internal`, `Unsafe`, `--add-opens/--add-exports/--illegal-access`, SecurityManager/AccessController/`doPrivileged`/`Policy`, Nashorn/ScriptEngine/GraalVM, JAXB/activation/JAX-WS/CORBA/RMI, `finalize`, Thread stop/suspend/resume, boxing constructors, `javax.servlet`/`javax.ws.rs`/`javax.inject`, `ToolProvider`/`-source`/`--release`, `java.version`: results as tabulated above.
* `grep` over the in-scope POMs and `.github/workflows/*.yml` for compiler, plugin and JDK settings; `find` for Dockerfiles (none).
* `curl` Maven Central directly gave HTTP 429. The configured mirror (`maven-central.storage-download.googleapis.com`, `~/.m2/settings.xml`) gave 200.
* Scratch POM outside the repo (`~/j17scratch/pom.xml`) + `mvn dependency:copy-dependencies -DexcludeTransitive=true`: 51 key jars downloaded. Then `jdeps --multi-release 17 --jdk-internals` on each (results in §2.1) and a classfile-major scan: Arrow 18, HikariCP 7 and mongo-java-server are major 55; the rest are 49-52.
* **EXPERIMENT (no files edited; command-line overrides only):** `mvn -B clean verify -DskipTests -Drevision=4.114.2 -Dmaven.compiler.source=17 -Dmaven.compiler.target=17 -Dmaven.compiler.release=17 -pl legend-engine-xts-identity/legend-engine-xt-identity-kerberos`. Compile succeeded with 6 "marked for removal" warnings; the build then **FAILED** in `dependency:2.10:analyze-only` with `IllegalArgumentException` at `org.objectweb.asm.ClassReader.<init>` (confirmed with `-e`). `-Drevision=4.114.2` was used so the released 4.114.2 `legend-engine-identity-core` from `~/.m2` resolves without `-am`. `verify` was used instead of `install` (the AGENTS.md `clean` rule was kept) so that `~/.m2` is not overwritten.
* Same command + `-Dmdep.analyze.skip=true`: BUILD SUCCESS through enforcer, jacoco, jar and checkstyle. `ExecSubject.class` header has major **61**.
* Baseline (no overrides, release 8): BUILD SUCCESS, 0 removal warnings.
* `git status --short` afterwards: clean (only gitignored `target/` output). No commits, branches or pushes.

## 7. Not verified

* Effect of `maxJdkVersion=1.8` on in-reactor release-17 artifacts. This is inferred from how the rule works; it was not reproduced, because that needs a multi-module `-am` build.
* Any module other than `legend-engine-xt-identity-kerberos` compiled at release 17. Static scans found no removed-API use, but Pure-generated sources (built by the legend-pure plugins) were not compiled at 17.
* Unit tests under release 17 (`-DskipTests` everywhere). In particular, whether Mockito-inline/byte-buddy 1.11.20 can mock classfile-61 classes.
* Runtime of the server image on JDK 17 (Jib build, Arrow `--add-opens`, Kerberos/pac4j login flows).
* `mvn dependency:tree` for the server/extension collections (it needs the SNAPSHOT reactor). Transitive versions, such as the exact Jackson and Jetty that end up in the server, come from the root properties and were not resolved.
* Newer-major minimum-Java values in §3 come from upstream release notes / general knowledge and were not looked up in this session (network restricted). Least certain: testcontainers 2.x, JsonUnit 3+, trino-jdbc, legend-shared.
* Snowpark/WireMock non-exported-package paths are jdeps-static only; it is not confirmed whether the code paths are exercised.
* `legend-pure-maven-*` and `EngineJavaCompiler` `-source`/`-target` options (core scope).
