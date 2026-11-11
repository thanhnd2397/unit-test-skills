# Test Dependencies (Java 8 / Boot 2.0.3)

The exact dependency set this skill assumes, how to pin it, and how to prove it resolved.

## Version table

| Artifact | Version | Why this version |
|---|---|---|
| `org.junit.jupiter:junit-jupiter-api` | 5.9.3 | Boot 2.0.3 manages 5.1.1 — too old for current Surefire/IDE tooling |
| `org.junit.jupiter:junit-jupiter-engine` | 5.9.3 | must match the API exactly |
| `org.junit.jupiter:junit-jupiter-params` | 5.9.3 | only if `@ParameterizedTest` is used |
| `org.junit.platform:junit-platform-commons` | 1.9.3 | transitive; never mix with 1.1.x |
| `org.junit.platform:junit-platform-engine` | 1.9.3 | transitive |
| `org.junit.platform:junit-platform-launcher` | 1.9.3 | declare only if the runner demands it |
| `org.junit.vintage:junit-vintage-engine` | 5.9.3 | only while old JUnit 4 tests still exist (keep `junit:junit:4.12` on the test classpath) |
| `org.mockito:mockito-core` | 3.12.4 | overrides Boot's 2.15.0 |
| `org.mockito:mockito-junit-jupiter` | 3.12.4 | `MockitoExtension`; must equal `mockito-core` |
| `net.bytebuddy:byte-buddy` | 1.11.13 | Mockito 3.12.4 needs it; Boot manages 1.7.11 → `NoSuchMethodError` |
| `net.bytebuddy:byte-buddy-agent` | 1.11.13 | same |
| `org.objenesis:objenesis` | 3.2 | Mockito 3.12.4 expects 3.2; Boot manages 2.6 |
| `org.assertj:assertj-core` | 3.9.1 | the version Boot 2.0.3 manages — this skill's examples stay inside its API |
| `org.springframework:spring-test` | 5.0.7.RELEASE | `SpringExtension`, MockMvc, `ReflectionTestUtils`. Not needed for pure Mockito unit tests |
| `org.springframework.boot:spring-boot-test` / `-test-autoconfigure` | 2.0.3.RELEASE | `@SpringBootTest`, slices, `@MockBean`; normally arrive via `spring-boot-starter-test` |
| `org.apache.maven.plugins:maven-surefire-plugin` | 2.22.2 | first line that runs the JUnit Platform natively |
| `org.jacoco:jacoco-maven-plugin` | 0.8.8 | coverage |

Not in this set, and therefore not used anywhere in this skill: Testcontainers, Instancio, `mockito-inline`, WireMock, REST Assured, Awaitility.

## POM blocks

```xml
<properties>
    <junit5.version>5.9.3</junit5.version>
    <mockito3.version>3.12.4</mockito3.version>
    <bytebuddy.version>1.11.13</bytebuddy.version>
    <objenesis.version>3.2</objenesis.version>
    <jacoco.version>0.8.8</jacoco.version>
</properties>
```

Pin versions in the **existing** `<dependencyManagement>`. If the project imports `spring-boot-dependencies` as a BOM (rather than using `spring-boot-starter-parent`), these entries must come **before** the BOM import — the first declaration wins:

```xml
<dependencyManagement>
    <dependencies>
        <!-- BEFORE the spring-boot-dependencies import -->
        <dependency>
            <groupId>org.junit</groupId>
            <artifactId>junit-bom</artifactId>
            <version>${junit5.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <dependency>
            <groupId>org.mockito</groupId>
            <artifactId>mockito-core</artifactId>
            <version>${mockito3.version}</version>
        </dependency>
        <dependency>
            <groupId>org.mockito</groupId>
            <artifactId>mockito-junit-jupiter</artifactId>
            <version>${mockito3.version}</version>
        </dependency>
        <dependency>
            <groupId>net.bytebuddy</groupId>
            <artifactId>byte-buddy</artifactId>
            <version>${bytebuddy.version}</version>
        </dependency>
        <dependency>
            <groupId>net.bytebuddy</groupId>
            <artifactId>byte-buddy-agent</artifactId>
            <version>${bytebuddy.version}</version>
        </dependency>
        <dependency>
            <groupId>org.objenesis</groupId>
            <artifactId>objenesis</artifactId>
            <version>${objenesis.version}</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```

`junit-bom` keeps `junit-jupiter-*` at 5.9.3 and `junit-platform-*` at 1.9.3 in one place; declaring the three Jupiter artifacts individually works too but drifts easily.

Test dependencies:

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter-api</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter-engine</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter-params</artifactId>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-core</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-junit-jupiter</artifactId>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.assertj</groupId>
        <artifactId>assertj-core</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

`spring-boot-starter-test` on 2.0.3 drags in `junit:junit:4.12`, Hamcrest, JSONassert, JsonPath and `spring-test`. Leave `junit:junit` in place while JUnit 4 tests remain (add `junit-vintage-engine` so they still run); exclude it only once the last JUnit 4 test is gone:

```xml
<exclusions>
    <exclusion><groupId>junit</groupId><artifactId>junit</artifactId></exclusion>
</exclusions>
```

Note that Hamcrest is still needed after that exclusion — `jsonPath("$.items", hasSize(3))` has no AssertJ equivalent.

## Blast radius warning

Byte Buddy and Objenesis are **runtime** dependencies of Hibernate and Spring, not test-only libraries. Pinning them in a parent POM's `<dependencyManagement>` changes the production classpath too.

Before committing a Byte Buddy bump:

```bash
mvn dependency:tree -Dscope=compile | grep -E "byte-buddy|objenesis|hibernate"
mvn dependency:tree -Dscope=test    | grep -E "byte-buddy|objenesis|mockito"
```

Then run the regression suite, not just the new tests. If the compile scope must stay on 1.7.11, scope the override to the test module instead of the parent.

## Post-integration checklist

```bash
mvn -version
mvn help:effective-pom -Doutput=effective-pom.xml
mvn dependency:tree -Dscope=test
mvn dependency:tree -Dscope=compile
mvn clean verify
```

- [ ] Surefire really forks the test JVM on **JDK 8** (`mvn -X test`, look at `Forking command line`) — toolchains must apply to Surefire, not just the compiler
- [ ] All `junit-jupiter-*` resolve to **5.9.3**, all `junit-platform-*` to **1.9.3**, nothing at 5.1.1 / 1.1.x
- [ ] `mockito-core` and `mockito-junit-jupiter` both **3.12.4**
- [ ] `byte-buddy` and `byte-buddy-agent` both **1.11.13**; `objenesis` **3.2**
- [ ] `spring-test` **5.0.7.RELEASE**, `assertj-core` **3.9.1**
- [ ] Surefire reports **tests run > 0** (a Jupiter suite under Surefire < 2.22 reports zero and still passes)
- [ ] Old JUnit 4 tests still execute (count them before and after)
- [ ] JaCoCo report exists at `target/site/jacoco/index.html` in each module
- [ ] Production behavior unchanged — regression suite green

## Checked at the source

Every version above is taken from `unit-test-library.md`. That document is the authority for this skill; if the project POM says otherwise, reconcile the two before writing tests rather than guessing.

A known mismatch to watch: some modules currently resolve JUnit **5.8.2** rather than 5.9.3. Both run on Java 8 and all examples here are valid on either, but the engine, API and params artifacts must share one version inside a module.
