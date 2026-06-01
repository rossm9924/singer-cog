# Java 8 Baseline — Migration to Java 11

## Build Environment

- **JDK**: OpenJDK 1.8.0_492 (Private Build)
- **Maven**: 3.6.3
- **Build tool**: Maven multi-module (parent POM)
- **Modules**: `singer-commons`, `thrift-logger`, `singer`
- **Compiler config**: `maven.compiler.source=1.8`, `maven.compiler.target=1.8`

## Build Result (JDK 8)

```
BUILD SUCCESS
Reactor Summary:
  Singer Logging Agent ............................... SUCCESS [  0.065 s]
  singer-commons ..................................... SUCCESS [  5.429 s]
  thrift-logger ...................................... SUCCESS [  2.860 s]
  singer ............................................. SUCCESS [  7.440 s]
Total time: 15.883 s
```

### Build Warnings

- Duplicate dependency: `software.amazon.awssdk:s3` (2.21.30 vs 2.17.273) in singer POM
- Missing plugin versions: `maven-source-plugin`, `build-helper-maven-plugin`, `maven-surefire-plugin`

## Test Result (JDK 8)

```
Tests run: 173, Failures: 1, Errors: 0, Skipped: 0
```

### Pre-existing Failure

```
testMultipleConfigsWithDifferentAllowlists(com.pinterest.singer.kubernetes.TestPodAllowlist):
  Should have 2 log paths initialized expected:<2> but was:<3>
```

This failure is **pre-existing on JDK 8** (not migration-related).

## jdeps —jdk-internals (JDK Internal API Usage)

```
singer-1.3.1.jar -> jdk.unsupported
  com.pinterest.singer.utils.SingerUtils ->
    com.sun.nio.file.SensitivityWatchEventModifier  JDK internal API (jdk.unsupported)
```

**Note**: `SensitivityWatchEventModifier` is in `jdk.unsupported` module. It remains accessible on the classpath in JDK 11 but is not part of the public API. No immediate action needed, but consider migration plan.

`singer-commons` and `thrift-logger`: **No JDK internal API usage found**.

## jdeprscan —release 11 (Deprecated API Usage)

### singer-1.3.1.jar

No deprecated API usage found.

### singer-commons-1.3.1.jar & thrift-logger

Deprecated constructor usage in **Thrift-generated code** (not hand-written):

| Deprecated API | Classes (all Thrift-generated) |
|---|---|
| `Integer::<init>(I)V` | `KubeConfig`, `SingerConfig`, `KafkaProducerConfig`, `SingerLogConfig`, `AdminConfig`, `KafkaWriterConfig`, etc. |
| `Long::<init>(J)V` | `LoggingAuditHeaders`, `LogMessage`, `Event`, `AuditMessage`, `ThriftMessage`, `LogPosition`, `LogFile` |
| `Boolean::<init>(Z)V` | `LoggingAuditHeaders`, `LoggingAuditEvent`, `SingerConfig`, `KafkaProducerConfig`, etc. |
| `Double::<init>(D)V` | `AuditConfig` |

**Impact**: Low — these are in Thrift-generated classes. They use deprecated boxed-type constructors (`new Integer(n)` → should be `Integer.valueOf(n)`). The Thrift code generator produces these; fix requires upgrading Thrift or post-processing generated code. Not a blocker for JDK 11.

## javax.* and com.sun.* Import Analysis

| Import | Files | JDK 11 Status |
|---|---|---|
| `javax.annotation.concurrent.NotThreadSafe` | `SimpleRoundRobinPartitioner.java` | **REMOVED** from JDK 11 — needs external dep (`jsr305` or `javax.annotation-api`) |
| `javax.net.ssl.KeyManagerFactory` | SSL config | Safe — still in `java.base` |
| `javax.net.ssl.SSLContext` | SSL config | Safe — still in `java.base` |
| `javax.net.ssl.TrustManagerFactory` | SSL config | Safe — still in `java.base` |
| `com.sun.net.httpserver.HttpExchange` | HTTP utils | Safe — in `jdk.httpserver` module, accessible on classpath |
| `com.sun.net.httpserver.HttpHandler` | HTTP utils | Safe — in `jdk.httpserver` module |
| `com.sun.net.httpserver.HttpServer` | HTTP utils | Safe — in `jdk.httpserver` module |

### Not Used (No Impact)

- JAXB (`javax.xml.bind.*`) — **not imported**
- JAX-WS (`javax.xml.ws.*`) — **not imported**
- CORBA — **not imported**
- JavaFX — **not imported**
- Nashorn — **not imported**

## Reflection Usage

| Location | Code | Risk for JDK 11 |
|---|---|---|
| `TestKafkaProducerManager.java:34-45` | `KafkaProducer.class.getDeclaredField("totalMemorySize")` + `setAccessible(true)` | **HIGH** — illegal reflective access on Kafka internals |
| `AuditableLogbackThriftLogger.java:111` | `thriftClazz.getDeclaredField("metaDataMap").get(null)` | **MEDIUM** — accessing static field on user-defined Thrift class (likely safe) |
| Various `*.java` | `Class.forName(...)` for dynamic plugin loading | **LOW** — standard classpath-based loading, no JPMS issues |

## JVM Flags & GC Configuration

### `singer/scripts/run_singer_common.sh` (production)

```bash
DAEMON_OPTS="-server -Xmx800M -Xms800M -verbosegc -Xloggc:${LOG_DIR}/gc.log \
-XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=100 -XX:GCLogFileSize=2M \
-XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintGCDateStamps -XX:+PrintClassHistogram \
-XX:+UseG1GC -XX:MaxGCPauseMillis=250 -XX:G1ReservePercent=10 -XX:ConcGCThreads=4 \
-XX:ParallelGCThreads=4 -XX:G1HeapRegionSize=8m -XX:InitiatingHeapOccupancyPercent=70 \
-XX:ErrorFile=${LOG_DIR}/jvm_error.log"
```

### `singer/teletraan/run_singer_foreground.sh` (teletraan deploy)

Same flags as above.

### Flags Requiring Migration for JDK 11

| Flag | Status in JDK 11 | Action |
|---|---|---|
| `-verbosegc` | Deprecated | Replace with `-Xlog:gc` |
| `-Xloggc:gc.log` | **Removed** | Replace with `-Xlog:gc*:file=gc.log:time,uptime,level,tags` |
| `-XX:+UseGCLogFileRotation` | **Removed** | Handled by Unified Logging rotation |
| `-XX:NumberOfGCLogFiles=100` | **Removed** | Use `-Xlog:gc*:file=gc.log::filecount=100` |
| `-XX:GCLogFileSize=2M` | **Removed** | Use `-Xlog:gc*:file=gc.log::filesize=2M` |
| `-XX:+PrintGCDetails` | **Removed** | Covered by `-Xlog:gc*` |
| `-XX:+PrintGCTimeStamps` | **Removed** | Covered by `time,uptime` decorators |
| `-XX:+PrintGCDateStamps` | **Removed** | Covered by `time` decorator |
| `-XX:+PrintClassHistogram` | **Removed** | Use `-Xlog:class+histogram=info` |
| `-XX:+UseG1GC` | Still valid (default in 11) | Can keep or remove (already default) |
| `-XX:MaxGCPauseMillis=250` | Still valid | Keep |
| `-XX:G1ReservePercent=10` | Still valid | Keep |
| `-XX:ConcGCThreads=4` | Still valid | Keep |
| `-XX:ParallelGCThreads=4` | Still valid | Keep |
| `-XX:G1HeapRegionSize=8m` | Still valid | Keep |
| `-XX:InitiatingHeapOccupancyPercent=70` | Still valid | Keep |
| `-XX:ErrorFile=...` | Still valid | Keep |
| `-server` | Still valid | Keep |
| `-Xmx800M` / `-Xms800M` | Still valid | Keep |

## Key Dependencies & Java 11 Compatibility

| Dependency | Version | Java 11 Compatible? |
|---|---|---|
| `mockito-all` | 1.10.19 | **NO** — uses CGLib, incompatible with JDK 11 |
| `junit` | 4.13.1 / 4.12 | Yes (version mismatch across modules) |
| `junit-jupiter` | 5.3.1 | Yes |
| Apache Thrift | 0.12.0 | Yes |
| Apache Kafka clients | 2.3.1 | Yes |
| Netty | 4.1.111.Final | Yes |
| Gson | 2.8.9 | Yes |
| Guava | 16.0.1 | Yes (old but works) |
| AWS SDK v2 | 2.21.30 | Yes |
| Apache Pulsar | 2.3.2 | Yes |

## Summary of Migration Work Required

1. **Build tooling**: Replace `source/target=1.8` with `release=11`; upgrade Maven plugins
2. **Removed modules**: Add `jsr305` or `javax.annotation-api` for `@NotThreadSafe`
3. **Reflection**: Add `--add-opens` for test (`TestKafkaProducerManager`); evaluate `AuditableLogbackThriftLogger`
4. **Test deps**: Upgrade `mockito-all:1.10.19` → `mockito-core:3.x+`
5. **GC/logging flags**: Migrate to Unified Logging in startup scripts
6. **TLS/Security**: Verify SSL connections under TLS 1.3 defaults
7. **CI**: Update `maven.yml` from JDK 1.8 to JDK 11, upgrade action versions
8. **Documentation**: `MIGRATION_NOTES.md`, README updates
