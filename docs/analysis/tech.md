# Công nghệ, dependency và cách chạy

← Về [README.md](README.md)

## Analysis header

| | |
|--|--|
| Ngày phân tích | 2026-11-10 |
| Scope | Toàn bộ project `D:\new project\Ai\awesome-copilot`, ưu tiên rà soát bảo mật |
| Shape | small (1 module, 1 domain, 0 deployable) |
| Commit | Không có — project không phải git repository |
| Baseline | Không có baseline; phân tích đầy đủ |
| Phạm vi đã đọc | 100% (14/14 file markdown) |
| Không phân tích | Skill `spring-boot-4-migration`, `java-testcontainers` (ngoài project); tính đúng đắn của các API Spring Boot 4 được mô tả |

## Stack của chính project

- `[Verified]` **Không có stack thực thi.** Project gồm 14 file Markdown và không gì khác. Không có Java, không có `pom.xml` hay `build.gradle`, không có `package.json`, không có Dockerfile, không có CI workflow, không có file ẩn. (`find . -type f ! -name "*.md"` → rỗng; `find . -name ".*" -maxdepth 2` → chỉ trả về `.`)
- `[Verified]` Định dạng: Markdown với YAML frontmatter ở `SKILL.md` (hai khóa: `name`, `description`). (`SKILL.md:1-4`)
- `[Inferred]` Định dạng này là định dạng agent skill (một `SKILL.md` + thư mục `references/` được liên kết tiến dần). Suy luận từ cấu trúc frontmatter `name`/`description` và mẫu liên kết từ `SKILL.md` tới `references/*.md`. (`SKILL.md:1-4`, `SKILL.md:35-74`)
- `[Unknown]` Phiên bản của chính bộ skill. Frontmatter không có khóa `version`, project không có manifest và không có lịch sử git. Cần một file `plugin.json`/`CHANGELOG.md` hoặc thông tin từ người bảo trì để xác định.
- `[Unknown]` Giấy phép và nguồn gốc của nội dung. Không có `LICENSE`, `NOTICE`, `README` hay dòng ghi công tác giả nào trong project. Điều kiện tái sử dụng/phân phối lại không xác định được. Cần người dùng cho biết bộ skill này lấy từ đâu.

## Kiểm kê file

Toàn bộ 14 file, đã đọc trọn vẹn:

| Đường dẫn | Byte | Chủ đề |
|---|---:|---|
| `SKILL.md` | 8.503 | Điểm vào: bảng tương thích Boot 3/4, nguyên tắc, bảng chọn test slice, cây quyết định, best practice, dependency |
| `references/test-slices-overview.md` | 5.638 | Ma trận quyết định 10 loại slice, hướng dẫn chọn, lỗi thường gặp, tính năng Java 25 |
| `references/webmvctest.md` | 3.652 | `@WebMvcTest`: GET/POST/validation/query param/path variable/security |
| `references/datajpatest.md` | 5.094 | `@DataJpaTest`: custom query, native query, pagination, lazy loading, cascade, H2 vs Testcontainers |
| `references/restclienttest.md` | 4.736 | `@RestClientTest` + `MockRestServiceServer`: request matching, response, verify |
| `references/resttestclient.md` | 5.042 | `RestTestClient` (Boot 4+): HTTP method, assertion, migration từ `TestRestTemplate` |
| `references/mockmvc-tester.md` | 7.696 | `MockMvcTester` (Boot 3.2+): mẫu `convertTo()`, assertion status/header/body |
| `references/mockmvc-classic.md` | 5.929 | MockMvc cổ điển (`perform`/`andExpect`) cho codebase cũ + bảng migration |
| `references/assertj-basics.md` | 4.384 | AssertJ: scalar, string, number, boolean, date, Optional, exception, custom assert, soft assert |
| `references/assertj-collections.md` | 5.015 | AssertJ: List/Set/Map/array, `extracting`, `filteredOn`, `satisfiesExactly`, recursive comparison |
| `references/testcontainers-jdbc.md` | 7.903 | `@ServiceConnection`, PostgreSQL/MySQL, multi-DB, container reuse, init script, Flyway, wait strategy |
| `references/instancio.md` | 5.032 | Sinh dữ liệu test bằng Instancio |
| `references/context-caching.md` | 2.467 | Tối ưu cache context của Spring TestContext Framework |
| `references/sb4-migration.md` | 3.358 | Thay đổi về test khi nâng từ Boot 3.x lên 4.0 |

## Dependency và phân loại thương mại

Đây là các dependency mà bộ skill **khuyến nghị cho dự án đích** — project này không tự cài gì cả. Mỗi dependency bắt buộc có một commercial class.

| Dependency (`artifactId`) | Version | Class | Label | Evidence |
|---|---|---|---|---|
| `spring-boot-starter-test` | `[Unknown]` — do BOM quản lý, không ghi trong file nào | `Open source` | `[Inferred]` | `SKILL.md:190-193`, `references/sb4-migration.md:13-19` |
| `spring-boot-starter-webmvc-test` | `[Unknown]` — BOM | `Open source` | `[Inferred]` | `SKILL.md:197-200`, `references/test-slices-overview.md:179-183` |
| `spring-boot-starter-data-jpa-test` | `[Unknown]` — chỉ được nêu tên trong văn bản, không có khối XML | `Open source` | `[Inferred]` | `SKILL.md:18` |
| `spring-boot-starter-data-jpa` | `[Unknown]` — BOM | `Open source` | `[Inferred]` | `references/test-slices-overview.md:186-189` |
| `spring-boot-starter-restclient-test` | `[Unknown]` — BOM | `Open source` | `[Inferred]` | `references/resttestclient.md:14-18`, `references/sb4-migration.md:34-38` |
| `spring-boot-testcontainers` | `[Unknown]` — BOM | `Open source` | `[Inferred]` | `SKILL.md:204-207`, `references/testcontainers-jdbc.md:24-28` |
| `spring-boot-starter-test-classic` | `[Unknown]` — chỉ nêu tên trong văn bản | `Open source` | `[Inferred]` | `references/sb4-migration.md:141` |
| `testcontainers-postgresql` | `[Unknown]` — BOM, nêu rõ tại `references/testcontainers-jdbc.md:19` | `Open source` | `[Inferred]` | `references/testcontainers-jdbc.md:29-33` |
| `testcontainers-junit-jupiter` | `[Unknown]` — BOM | `Open source` | `[Inferred]` | `references/testcontainers-jdbc.md:34-38` |
| `testcontainers-mysql` | `[Unknown]` — BOM | `Open source` | `[Inferred]` | `references/testcontainers-jdbc.md:86-90`, `:264` |
| `testcontainers-mariadb` | `[Unknown]` — chỉ trong bảng, không có khối XML | `Open source` | `[Inferred]` | `references/testcontainers-jdbc.md:265` |
| `testcontainers-mssqlserver` | `[Unknown]` — chỉ trong bảng | `Open source` | `[Inferred]` | `references/testcontainers-jdbc.md:266` |
| `testcontainers-oracle-free` | `[Unknown]` — chỉ trong bảng | `Needs verification` | `[Unknown]` | `references/testcontainers-jdbc.md:267` |
| `testcontainers-mongodb` | `[Unknown]` — chỉ trong bảng | `Open source` | `[Inferred]` | `references/testcontainers-jdbc.md:268` |
| `instancio-junit` | `5.5.1` | `Open source` | `[Inferred]` | `references/instancio.md:15-20` |

Lý do cho nhãn `[Inferred]` trên cột Class: **project này không khai báo giấy phép của bất kỳ dependency nào** — không có `pom.xml`, không có lockfile, không có file licence. Phân loại `Open source` được suy ra từ việc các artifact này là thư viện mã nguồn mở được công bố rộng rãi (Spring Boot, Testcontainers, Instancio, AssertJ, JUnit), **không** được kiểm chứng từ bất kỳ file nào trong project. Để có kết luận `[Verified]` cần đọc `pom.xml`/lockfile của dự án đích.

`testcontainers-oracle-free` được để `Needs verification` và `[Unknown]`: module Testcontainers thì mã nguồn mở, nhưng image Oracle Database Free đi kèm có điều kiện sử dụng riêng của nhà cung cấp mà project này không nêu. Cần đọc điều khoản của image để xác định.

Thư viện được dùng trong mã mẫu nhưng **không** được khai báo thành dependency riêng (đi kèm trong `spring-boot-starter-test`): AssertJ (`org.assertj.core.api.Assertions`, `references/assertj-basics.md:196`), JUnit Jupiter (`SKILL.md:16`), Mockito (`SKILL.md:86`), Hamcrest (`references/mockmvc-classic.md:184`), Jackson (`references/mockmvc-tester.md:217`). Class: `Open source`, label `[Inferred]`, cùng lý do như trên.

## External service và Docker image

Một service được phân loại thương mại ngay cả khi client library của nó miễn phí.

| Service / Image | Class | Label | Evidence |
|---|---|---|---|
| Docker daemon cục bộ (Testcontainers yêu cầu) | `Open source` | `[Inferred]` — Testcontainers cần một container runtime; project không nêu runtime cụ thể nên không loại trừ Docker Desktop (có điều kiện thương mại cho doanh nghiệp lớn) | `references/testcontainers-jdbc.md:7` |
| Docker Hub (registry kéo image về khi chạy test) | `Usage-based` | `[Inferred]` — các tag `postgres:18`, `mysql:8.4` không ghi registry nên mặc định về Docker Hub, nơi áp giới hạn số lần pull theo thời gian; project không cấu hình registry nội bộ hay mirror nào | `references/testcontainers-jdbc.md:71`, `:98`, `:144` |
| `docs.spring.io` (liên kết tài liệu) | `Open source` | `[Verified]` — tài liệu công khai, chỉ là liên kết đọc | `SKILL.md:186` |
| `instancio.org` (liên kết tài liệu) | `Open source` | `[Verified]` — tài liệu công khai, chỉ là liên kết đọc | `references/instancio.md:230-231` |
| `api.weather.com`, `api.example.com` | — | `[Verified]` Là URL **giả lập trong ví dụ**, chỉ dùng làm đối số cho `MockRestServiceServer`; không có lời gọi mạng thật nào | `references/restclienttest.md:42`, `:73`, `:89`, `:96`, `:104`, `:112`, `:121`, `:175` |

Docker image được tham chiếu, kèm cách ghim phiên bản:

| Image | Cách ghim | Label | Evidence |
|---|---|---|---|
| `postgres:18` | Tag nổi (mutable), không có digest | `[Verified]` | `references/datajpatest.md:16`, `:61`, `:173`; `references/testcontainers-jdbc.md:71`, `:113`, `:117`, `:127`, `:144`, `:158`, `:189`, `:203`, `:246`; `references/instancio.md:98`; `references/test-slices-overview.md:61` |
| `postgres:18-alpine` | Tag nổi, không có digest | `[Verified]` | `references/testcontainers-jdbc.md:50` |
| `mysql:8.4` | Tag nổi, không có digest | `[Verified]` | `references/testcontainers-jdbc.md:98`, `:101` |

`[Verified]` Bộ skill có khuyên "Use specific versions (postgres:18) not latest" (`references/testcontainers-jdbc.md:278`) — tức là đã tránh `latest`, nhưng vẫn không đạt mức ghim theo digest. Xem rủi ro **R3** ở [system.md](system.md#risk--những-gì-việc-đọc-mã-phơi-ra).

## Chạy, build, test, CI

- `[Verified]` **Project không có gì để build hay chạy.** Không có build script, không có test runner, không có CI workflow, không có linter config. (`find . -type f ! -name "*.md"` → rỗng)
- `[Inferred]` Cách "dùng" project là cài nó làm agent skill để agent đọc `SKILL.md` rồi đọc tiếp các reference theo liên kết. Suy luận từ định dạng file và các liên kết tiến dần. (`SKILL.md:35-74`)
- `[Unknown]` Cách cài đặt chính thức. Không có `plugin.json`, `.claude-plugin/`, `marketplace.json` hay hướng dẫn cài nào. Một file manifest plugin hoặc tài liệu từ người bảo trì sẽ xác định được điều này.
- `[Verified]` Lệnh duy nhất bộ skill khuyến nghị cho dự án đích: `./mvnw verify` — **không được chạy** trong lần phân tích này. (`SKILL.md:63`, `:65`, ghi nhận là "not run")
- `[Verified]` Bộ skill cũng khuyến nghị bật log debug `logging.level.org.springframework.test.context.cache=DEBUG` (`references/context-caching.md:65`) và truy cập endpoint `GET /actuator/metrics/spring.test.context.cache` (`references/context-caching.md:60`) — **không được thực hiện**, không có hệ thống nào được kết nối.

### Cách kiểm tra chất lượng bộ skill (gợi ý, chưa chạy)

- `[Verified]` Kiểm tra liên kết nội bộ là việc kiểm chứng duy nhất có thể làm hoàn toàn offline, và nó phát hiện 4 liên kết hỏng. Phép kiểm tra đã chạy trong lần phân tích này (chỉ đọc file, không thay đổi gì): đối chiếu mọi mẫu `](*.md)` với sự tồn tại của file trên ổ đĩa.
- `[Unknown]` Có tồn tại eval suite hay harness kiểm thử nào cho bộ skill này không. Project không chứa file test nào. Cần người bảo trì cho biết.
