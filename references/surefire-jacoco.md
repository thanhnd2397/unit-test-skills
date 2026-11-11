# Surefire 2.22.2 + JaCoCo 0.8.8 on a Java 8 / Toolchains Build

How the tests actually get executed and measured on this stack. Most "my JUnit 5 tests don't run" reports on Boot 2.0.x are one of the three problems below.

## Why Surefire 2.22.2 is mandatory

| Surefire | Jupiter support |
|---|---|
| < 2.22.0 | none — the JUnit Platform is not detected. Surefire reports **`Tests run: 0`** and the build stays green |
| 2.22.x | native JUnit Platform provider, picked up from `junit-jupiter-engine` on the test classpath |
| 3.x | also fine, but requires Maven ≥ 3.6.3 |

`spring-boot-starter-parent` 2.0.3 pins Surefire **2.21.0**, so the version must be overridden explicitly:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>2.22.2</version>
</plugin>
```

Do **not** add `junit-platform-surefire-provider`. It was the pre-2.22 bridge, is deprecated, and combining it with a modern Jupiter version is a known source of "zero tests" and `NoClassDefFoundError`.

### Verify it after the change

```bash
mvn clean test | grep -E "Tests run|T E S T S"
```

A suite that silently reports `Tests run: 0, Failures: 0, Errors: 0, Skipped: 0` has not run anything. Compare the count against the number of `@Test` methods before trusting the build.

## maven-toolchains-plugin: the test JVM must be JDK 8

The project forks the compiler to JDK 1.8 via toolchains. Surefire forks a **separate** JVM for tests and also honours the toolchain, which is what you want — but it means the failure mode for a library compiled for a newer bytecode level appears at test time, not compile time:

```
java.lang.UnsupportedClassVersionError: org/foo/Bar has been compiled by a more recent
version of the Java Runtime (class file version 55.0), this version of the Java Runtime
only recognizes class file versions up to 52.0
```

Class file 52 = Java 8. If you see 55 (Java 11) or 61 (Java 17), a test dependency is off the supported line — check it against [dependencies.md](dependencies.md).

Confirm which JVM runs the tests:

```bash
mvn -X test | grep -i "Forking command line"
```

It must point at the JDK 1.8 home from `~/.m2/toolchains.xml`, not at the JDK running Maven.

## JaCoCo 0.8.8

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.8</version>
    <executions>
        <execution>
            <id>prepare-agent</id>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>verify</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

Report: `target/site/jacoco/index.html` per module. XML for CI: `target/site/jacoco/jacoco.xml`.

### The argLine conflict

`prepare-agent` works by setting the `argLine` property. If the POM also sets `argLine` for Surefire, the agent is dropped and coverage comes out as 0%:

```xml
<!-- WRONG: overwrites JaCoCo's argLine -->
<configuration>
    <argLine>-Xmx1024m</argLine>
</configuration>

<!-- RIGHT: append to it -->
<configuration>
    <argLine>@{argLine} -Xmx1024m</argLine>
</configuration>
```

`@{argLine}` is late property evaluation — it requires the `prepare-agent` execution to have run, so keep JaCoCo bound to the default `initialize` phase (as above) rather than moving it later.

If coverage is 0% while tests clearly ran, check `argLine` first.

### Coverage threshold (optional)

```xml
<execution>
    <id>check</id>
    <phase>verify</phase>
    <goals>
        <goal>check</goal>
    </goals>
    <configuration>
        <rules>
            <rule>
                <element>BUNDLE</element>
                <limits>
                    <limit>
                        <counter>LINE</counter>
                        <value>COVEREDRATIO</value>
                        <minimum>0.80</minimum>
                    </limit>
                </limits>
            </rule>
        </rules>
    </configuration>
</execution>
```

Introduce a threshold on a legacy codebase only after measuring the current number — starting at 80% on a suite that sits at 12% blocks every build. Raise it in steps, and never lower it to make a build pass.

## Running the suite

```bash
mvn clean test                       # unit + slice tests (Surefire)
mvn clean verify                     # adds the JaCoCo report
mvn test -Dtest=OrderServiceTest     # single class
mvn test -Dtest=OrderServiceTest#shouldCreateOrder   # single method
mvn test -pl trunk/module-a          # one module
```

Tag-based selection with JUnit 5 and Surefire 2.22.2:

```java
@Tag("slow")
class OrderIntegrationTest { }
```

```xml
<configuration>
    <excludedGroups>slow</excludedGroups>
</configuration>
```

## Integration tests (`*IT`)

`maven-failsafe-plugin` is not part of this dependency set. The convention here is that every test runs under Surefire, so name test classes `*Test` and keep them fast enough for the main build.

If `*IT` classes do get introduced later, note the standard trap: Failsafe declared only in `<pluginManagement>` means `*IT` classes **never run** and the build still passes. It must be declared in `<build><plugins>` with the `integration-test` and `verify` goals, and invoked with `mvn verify`.

## Checklist

- [ ] Surefire version is **2.22.2** (or ≥ 2.22) in the effective POM: `mvn help:effective-pom | grep -A2 surefire`
- [ ] `junit-platform-surefire-provider` is **absent**
- [ ] `Tests run:` count matches the number of `@Test` methods
- [ ] `Forking command line` shows the JDK 1.8 home
- [ ] `argLine` either unset or written as `@{argLine} ...`
- [ ] `target/site/jacoco/index.html` exists after `mvn clean verify` and shows non-zero coverage
