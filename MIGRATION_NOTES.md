# Java 8 → 11 Migration Notes

## Summary

This migration moves Singer from Java 8 (`source/target 1.8`) to **Java 11** using the `--release 11` compiler flag. All modules compile and tests pass on JDK 11+.

## Build Changes

### Parent POM (`pom.xml`)
- Replaced `maven.compiler.source`/`maven.compiler.target` (1.8) with `maven.compiler.release` (11)
- Added `pluginManagement` with modern plugin versions:
  - `maven-compiler-plugin` 3.11.0 (release=11)
  - `maven-surefire-plugin` 3.2.5
  - `maven-failsafe-plugin` 3.2.5
  - `maven-javadoc-plugin` 3.6.3
- Added `maven-enforcer-plugin` 3.4.1 requiring JDK ≥ 11

### Module POMs (`singer-commons/pom.xml`, `thrift-logger/pom.xml`)
- Removed `maven.compiler.source` / `maven.compiler.target` properties (now inherited from parent `release`)
- Updated `java.version` property to `11`
- Removed hardcoded old `maven-javadoc-plugin` version 2.9.1 (now inherits from parent pluginManagement)
- Replaced deprecated `<additionalparam>` with `<additionalJOption>` for javadoc plugin

### Singer Module (`singer/pom.xml`)
- Removed hardcoded `maven-surefire-plugin` version 2.12.4 (now inherits 3.2.5 from parent)
- Updated javadoc plugin configuration to use `<additionalJOption>`

## CI Changes

### GitHub Actions (`.github/workflows/maven.yml`)
- Upgraded from `actions/checkout@v1` → `actions/checkout@v4`
- Upgraded from `actions/setup-java@v1` → `actions/setup-java@v4`
- Switched from JDK 1.8 to **JDK 11 (Temurin)**
- Added Maven dependency caching
- Added explicit `verify` and `test` stages
- Added `-Dgpg.skip=true` (GPG signing only needed for releases)

## Removed JDK Module Assessment

| Module | Used? | Action |
|--------|-------|--------|
| JAXB (`javax.xml.bind`) | No | N/A |
| JAX-WS (`javax.xml.ws`) | No | N/A |
| JavaFX | No | N/A |
| CORBA | No | N/A |
| Nashorn | No | N/A |
| `javax.annotation` (JDK module) | No* | N/A |

*Note: `javax.annotation.concurrent.NotThreadSafe` is used in one file, but this annotation comes from `com.google.code.findbugs:jsr305` (transitive via Guava), not from the removed JDK module.

## Known Warnings

### `com.sun.nio.file.SensitivityWatchEventModifier`
- **Location**: `SingerUtils.java:218`
- **Impact**: Compiler warning only; still functional on JDK 11+
- **Reason**: Used for high-sensitivity file system event monitoring
- **Recommendation**: Monitor for removal in future JDKs; consider `java.nio.file.WatchService` polling interval alternatives if deprecated

## Encapsulation / Reflection

- No `--add-opens` or `--add-exports` flags required
- No illegal reflective access warnings observed during testing
- Project runs entirely on the classpath (no JPMS `module-info.java`)

## Security / TLS

- No changes required; Singer uses standard Kafka/Pulsar/S3 clients which handle TLS negotiation
- Default TLS 1.3 support in JDK 11 is transparent and backward-compatible

## GC / Runtime

- Default GC on JDK 11 is G1 (was Parallel in JDK 8); Singer already uses `-Xmx` only, no legacy GC flags
- No obsolete JVM flags to remove

## Follow-ups (Optional)

- Consider adopting `java.net.http.HttpClient` (JDK 11) if custom HTTP logic is added
- Consider `var` for local variable type inference in new code
- Consider `Files.readString`/`Files.writeString` (JDK 11) for file utilities
- Monitor `SensitivityWatchEventModifier` for future deprecation/removal
- Upgrade `logback-core` from 1.1.11 to 1.2.x+ for JDK 11 compatibility improvements
