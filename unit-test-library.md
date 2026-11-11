# Danh sách thư viện Unit Test — Java 8 / Spring Boot 2.0.3

**Mục tiêu:** bổ sung Unit Test bằng JUnit 5, Mockito 3, AssertJ 3 và đo coverage, giữ nguyên Java 8, Spring Boot `2.0.3.RELEASE`, Spring Framework `5.0.7.RELEASE`.

> **Lưu ý:** Đây là danh sách phiên bản mục tiêu, **chưa phải** kết quả xác minh từ `trunk/pom.xml` và `mvn dependency:tree` của dự án. `Spring Boot quản lý version` không có nghĩa artifact tự động có trong classpath. Các thư viện `scope=test` chỉ phục vụ kiểm thử.

## 1. Thư viện chính

| # | Maven artifact (`groupId:artifactId`) | Version | Tác dụng | Cách đưa vào dự án |
|---|---|---|---|---|
| 1 | `org.junit.jupiter:junit-jupiter-api` | **5.9.3** | Cung cấp `@Test`, `@BeforeEach`, `@AfterEach`, `Assertions` cho Unit Test JUnit 5. | Dependency `test`, khai báo version/đồng bộ BOM JUnit 5 |
| 2 | `org.junit.jupiter:junit-jupiter-engine` | **5.9.3** | Test engine để JUnit Platform phát hiện và thực thi các test Jupiter. | Dependency `test`, đồng bộ cùng API |
| 3 | `org.mockito:mockito-core` | **3.12.4** | Tạo mock/stub, cấu hình `when(...).thenReturn(...)`, xác minh `verify(...)`, cô lập Service khỏi Repository/API ngoài. | Dependency `test`, **ghi đè Boot 2.0.3 vốn quản lý 2.15.0** |
| 4 | `org.mockito:mockito-junit-jupiter` | **3.12.4** | Tích hợp Mockito với JUnit 5 qua `@ExtendWith(MockitoExtension.class)`; khởi tạo `@Mock`, `@InjectMocks` tự động. | Dependency `test`, khai báo version trực tiếp |
| 5 | `org.assertj:assertj-core` | **3.9.1** | Assert dễ đọc (`assertThat`), so sánh object, collection, exception. | Dependency `test`; Boot 2.0.3 đã quản lý đúng version |
| 6 | `org.springframework:spring-test` | **5.0.7.RELEASE** | `SpringExtension`, Spring TestContext, MockMvc và hỗ trợ kiểm thử tích hợp Spring. **Không bắt buộc** với Unit Test Mockito thuần. | Dependency `test` nếu cần; Boot 2.0.3 đã quản lý đúng version |

### Các thư viện test bổ sung theo nhu cầu

| Artifact | Version | Tác dụng | Khi nào cần? |
|---|---|---|---|
| `org.junit.jupiter:junit-jupiter-params` | **5.9.3** | `@ParameterizedTest`, `@ValueSource`, `@CsvSource` để chạy test với nhiều bộ dữ liệu. | Khi dùng parameterized tests. |
| `org.junit.vintage:junit-vintage-engine` | **5.9.3** | Chạy các bài test JUnit 3/4 cũ qua JUnit Platform. | Nếu dự án đã có test JUnit 4 và cần chạy song song. Có thể cần giữ `junit:junit` 4.12 trong test classpath. |
| `org.junit.platform:junit-platform-launcher` | **1.9.3** | API khởi chạy JUnit Platform, dùng cho một số IDE/runner/công cụ. | Chỉ khai báo trực tiếp nếu tool chạy test thực sự yêu cầu. |
| `org.springframework.boot:spring-boot-test` và `spring-boot-test-autoconfigure` | **2.0.3.RELEASE** | `@SpringBootTest`, test slices và tiện ích của Spring Boot. | Khi viết integration tests dùng Spring Boot; thường có qua `spring-boot-starter-test`. |

## 2. Dependency phụ thuộc của Mockito 3

| Maven artifact | Version mục tiêu | Tác dụng | Có phải thêm trực tiếp? |
|---|---|---|---|
| `net.bytebuddy:byte-buddy` | **1.11.13** | Sinh/biến đổi bytecode để Mockito tạo mock. | Maven có thể kéo qua `mockito-core`, **nhưng cần kiểm soát version**: Boot 2.0.3 quản lý 1.7.11. |
| `net.bytebuddy:byte-buddy-agent` | **1.11.13** | Hỗ trợ cơ chế Java agent/instrumentation mà Mockito có thể sử dụng. | Tự tải transitively; **kiểm soát version** tương tự. |
| `org.objenesis:objenesis` | **3.2** | Cho phép Mockito khởi tạo một số object không cần gọi constructor thông thường. | Thường tự tải transitively qua `mockito-core`. |

**Quan trọng:** Nếu ép Byte Buddy 1.11.13 bằng `<dependencyManagement>` ở parent POM, thay đổi đó cũng có thể tác động tới dependency runtime (chẳng hạn Hibernate), không chỉ Unit Test. Kiểm tra `dependency:tree` ở cả `compile` và `test`, sau đó chạy regression tests. Không cần thêm `byte-buddy`/`objenesis` thủ công nếu phiên bản resolve đã đúng.

## 3. Dependency phụ thuộc của JUnit 5

| Artifact | Version mục tiêu | Tác dụng | Khai báo trực tiếp? |
|---|---|---|---|
| `org.junit.platform:junit-platform-commons` | **1.9.3** | Các tiện ích/chức năng nền tảng dùng chung của JUnit Platform. | Không, Maven thường tự tải. |
| `org.junit.platform:junit-platform-engine` | **1.9.3** | SPI/API cho test engine của JUnit Platform. | Không, Maven thường tự tải. |
| `org.opentest4j:opentest4j` | Theo dependency của JUnit 5.9.3 | Chuẩn hóa thông tin lỗi assertion/test thất bại. | Không, Maven tự tải. |
| `org.apiguardian:apiguardian-api` | Theo dependency của JUnit 5.9.3 | Annotation mô tả mức độ ổn định của API JUnit. | Không, Maven tự tải. |

**Quan trọng:** Spring Boot 2.0.3 quản lý các artifact `junit-jupiter-api`, `junit-jupiter-engine`, `junit-jupiter-params` ở **5.1.1**. Vì vậy hãy khai báo hoặc quản lý phiên bản **đồng bộ 5.9.3**; không để các module JUnit Jupiter bị trộn 5.1.1 và 5.9.3.

## 4. Maven plugins (không phải thư viện runtime)

| Plugin | Version | Tác dụng | Ghi chú |
|---|---|---|---|
| `org.apache.maven.plugins:maven-surefire-plugin` | **2.22.2** | Phát hiện và chạy Unit Test JUnit 5 trong pha Maven `test`; tạo báo cáo test. | Cần kiểm tra cấu hình hiện tại, nhất là fork/JDK, vì dự án có Maven Toolchains. |
| `org.jacoco:jacoco-maven-plugin` | **0.8.8** | Gắn Java agent để thu thập line/branch coverage và sinh báo cáo HTML/XML khi build. | Dùng `prepare-agent` và `report`; tránh ghi đè `argLine` của JaCoCo. |

## 5. Quy tắc lựa chọn

1. **Unit Test Service/Utility thuần:** JUnit Jupiter API + Engine, Mockito Core + Mockito JUnit Jupiter, AssertJ. Không cần bật Spring Context, MySQL hay Docker.
2. **Test liên quan Spring:** thêm `spring-test`; nếu cần `@SpringBootTest`/test slices thì sử dụng các artifact `spring-boot-test` tương thích Boot 2.0.3.
3. **Test nhiều input:** thêm `junit-jupiter-params`.
4. **Có test JUnit 4 cũ:** thêm Vintage Engine cùng version 5.9.3 và kiểm tra số lượng test cũ thực thi.
5. **Không tự nâng Spring Framework, Spring Security, MySQL JDBC chỉ vì bổ sung JUnit 5.** Giữ nguyên các phiên bản sản xuất đã xác nhận.

## 6. Ví dụ cấu hình Maven tối thiểu

> Ghép các đoạn này vào `pom.xml` đang có, **không tạo POM thứ hai**. Với POM đã có `spring-boot-starter-test`, kiểm tra dependencies trùng và các mục `dependencyManagement` hiện tại trước khi thêm.

```xml
<properties>
    <junit5.version>5.9.3</junit5.version>
    <mockito3.version>3.12.4</mockito3.version>
    <bytebuddy.version>1.11.13</bytebuddy.version>
    <jacoco.version>0.8.8</jacoco.version>
</properties>

<!-- Đặt các mục sau vào dependencyManagement hiện có, nếu có -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter-api</artifactId>
            <version>${junit5.version}</version>
        </dependency>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter-engine</artifactId>
            <version>${junit5.version}</version>
        </dependency>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter-params</artifactId>
            <version>${junit5.version}</version>
        </dependency>
        <dependency>
            <groupId>org.mockito</groupId>
            <artifactId>mockito-core</artifactId>
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
    </dependencies>
</dependencyManagement>

<dependencies>
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
        <groupId>org.mockito</groupId>
        <artifactId>mockito-core</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-junit-jupiter</artifactId>
        <version>${mockito3.version}</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.assertj</groupId>
        <artifactId>assertj-core</artifactId>
        <version>3.9.1</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-test</artifactId>
        <version>5.0.7.RELEASE</version>
        <scope>test</scope>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <version>2.22.2</version>
        </plugin>
        <plugin>
            <groupId>org.jacoco</groupId>
            <artifactId>jacoco-maven-plugin</artifactId>
            <version>${jacoco.version}</version>
            <executions>
                <execution>
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
    </plugins>
</build>
```

## 7. Checklist kiểm tra sau tích hợp

```bash
mvn -version
mvn help:effective-pom -Doutput=effective-pom.xml
mvn dependency:tree -Dscope=test
mvn dependency:tree -Dscope=compile
mvn clean verify
```

- [ ] Maven/Surefire thực sự chạy test bằng JDK 8 (không chỉ compiler).
- [ ] JUnit Jupiter đồng bộ **5.9.3**; JUnit Platform đồng bộ **1.9.3**.
- [ ] Mockito Core và Mockito JUnit Jupiter cùng **3.12.4**.
- [ ] Byte Buddy và Agent resolve **1.11.13** hoặc cấu hình tương thích đã được xác minh.
- [ ] Spring Test là **5.0.7.RELEASE**; AssertJ **3.9.1**.
- [ ] Surefire báo **số test thực sự chạy > 0**, kể cả test JUnit 4 cũ nếu có.
- [ ] Báo cáo JaCoCo xuất hiện tại `target/site/jacoco/index.html` (với module Maven tương ứng).
- [ ] Test và regression test không làm thay đổi behavior của các module production.

## 8. Tài liệu tham khảo

- [Spring Boot 2.0.3 — Dependency versions](https://docs.spring.io/spring-boot/docs/2.0.3.RELEASE/reference/html/appendix-dependency-versions.html)
- [Mockito Core 3.12.4 — Maven Central POM](https://central.sonatype.com/artifact/org.mockito/mockito-core/3.12.4)
- [Maven Surefire 2.22.2 — JUnit provider selection](https://maven.apache.org/surefire-archives/surefire-2.22.2/maven-surefire-plugin/examples/junit.html)
- [JaCoCo Maven plugin](https://www.jacoco.org/jacoco/trunk/doc/maven.html)
- [JUnit 5.9.3 User Guide](https://docs.junit.org/5.9.3/user-guide/)

**Trạng thái:** tài liệu đề xuất; chỉ có thể chốt dependency tree cuối cùng sau khi kiểm tra POM thực tế của dự án.
