# Domain, kiến trúc, dữ liệu, tích hợp và rủi ro

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

## Business domain

`[Verified]` Một domain duy nhất: **hướng dẫn kiểm thử ứng dụng Spring Boot**. Mọi file trong project đều phục vụ chủ đề này. (`SKILL.md:3`, `SKILL.md:8`)

`[Verified]` Phạm vi phiên bản mà domain nhắm tới: ví dụ viết cho Spring Boot 4, có ghi chú tương thích cho Boot 3. (`SKILL.md:3`, `SKILL.md:8`, `SKILL.md:14-20`)

### Cây domain

```
Hướng dẫn kiểm thử Spring Boot (1 domain)
├── Chọn chiến lược test
│   ├── Phát hiện phiên bản & tương thích Boot 3/4   → SKILL.md:10-22
│   ├── Ma trận chọn test slice (10 loại)            → references/test-slices-overview.md:5-18
│   └── Cây quyết định nhanh                         → SKILL.md:76-96
├── Viết test theo từng slice
│   ├── Test tầng web (@WebMvcTest)                  → references/webmvctest.md
│   ├── Test tầng dữ liệu (@DataJpaTest)             → references/datajpatest.md
│   └── Test REST client (@RestClientTest)           → references/restclienttest.md
├── Chọn API assertion cho tầng web
│   ├── MockMvcTester (khuyến nghị, Boot 3.2+)       → references/mockmvc-tester.md
│   ├── MockMvc cổ điển (codebase cũ)                → references/mockmvc-classic.md
│   └── RestTestClient (Boot 4+)                     → references/resttestclient.md
├── Viết assertion
│   ├── AssertJ cho giá trị đơn                      → references/assertj-basics.md
│   └── AssertJ cho collection                       → references/assertj-collections.md
├── Dựng hạ tầng và dữ liệu test
│   ├── Database thật bằng Testcontainers            → references/testcontainers-jdbc.md
│   ├── Sinh đối tượng test (Instancio)              → references/instancio.md
│   └── Chạy test *IT bằng Failsafe                  → ✗ FILE KHÔNG TỒN TẠI (SKILL.md:63)
├── Tối ưu tốc độ bộ test
│   └── Cache context của Spring TestContext         → references/context-caching.md
└── Nâng cấp phiên bản
    ├── Thay đổi về test khi lên Boot 4.0            → references/sb4-migration.md
    └── Mock dependency (@MockitoBean)               → ✗ FILE KHÔNG TỒN TẠI (SKILL.md:53)
```

### Actor

`[Inferred]` Hai actor, suy ra từ cách văn bản được viết chứ không được tuyên bố ở đâu cả:

- **Agent/AI đọc skill** — là đối tượng của các câu lệnh mệnh lệnh như "Read the project's Spring Boot version and test dependencies before writing tests, and do not upgrade them implicitly". (`SKILL.md:12`)
- **Lập trình viên** — là đối tượng của các câu hướng dẫn quy trình như "If the user agrees to refactor, help identify extraction points". (`SKILL.md:114`)

`[Verified]` Không có cơ chế kiểm soát truy cập nào trong project: không có frontmatter `allowed-tools`, không có hook, không có khai báo quyền. Skill được đọc bởi bất cứ ai/cái gì có quyền đọc file. (`SKILL.md:1-4`)

### Các tính năng (feature) của bộ skill

Mỗi "tính năng" dưới đây là một quyết định mà skill hướng dẫn agent/người thực hiện. Mỗi dòng đã được đi theo từ điểm vào `SKILL.md` tới reference tương ứng và đọc hết reference đó.

| Tính năng | Đầu vào → Đầu ra | Label | Evidence |
|---|---|---|---|
| Phát hiện phiên bản trước khi viết test | Phiên bản Boot + test dependency của dự án → chọn tên annotation/package/artifact đúng cho 3.x hay 4.x | `[Verified]` | `SKILL.md:10-22` |
| Chọn test slice | Loại thành phần cần test → một annotation trong 10 lựa chọn | `[Verified]` | `references/test-slices-overview.md:5-18`, `SKILL.md:31-39` |
| Test controller | Controller + service phụ thuộc → test `@WebMvcTest` với service được mock bằng `@MockitoBean` | `[Verified]` | `references/webmvctest.md:7-19`, `:33-45` |
| Test repository với DB thật | Repository + entity → test `@DataJpaTest` + PostgreSQL container, `Replace.NONE` | `[Verified]` | `references/datajpatest.md:7-23`, `references/testcontainers-jdbc.md:63-78` |
| Test REST client | Lớp client gọi API ngoài → test `@RestClientTest` với `MockRestServiceServer` stub phản hồi | `[Verified]` | `references/restclienttest.md:15-24`, `:39-55` |
| Assert phản hồi HTTP kiểu AssertJ | Kết quả request → `convertTo(T.class)` + `satisfies()` thay vì JSON path | `[Verified]` | `references/mockmvc-tester.md:9-20`, `:307` |
| Assert giá trị và collection | Giá trị thu được → chuỗi assertion AssertJ fluent | `[Verified]` | `references/assertj-basics.md:9-15`, `references/assertj-collections.md:14-22` |
| Sinh dữ liệu test | Lớp có ≥3 thuộc tính → `Instancio.create()` / `Instancio.of().set()` | `[Verified]` | `references/instancio.md:3-10`, `:27-48` |
| Tối ưu cache context | Cấu hình test → nhóm test theo cùng cache key để tái dùng context | `[Verified]` | `references/context-caching.md:7-17`, `:20-46` |
| Migrate test lên Boot 4 | Bộ test Boot 3 → checklist 8 bước đổi annotation/starter/Testcontainers | `[Verified]` | `references/sb4-migration.md:128-137` |
| Chạy test tích hợp `*IT` | Lớp test `*IT` → bind Failsafe vào `<build><plugins>`, chạy `./mvnw verify` | `[Verified]` cho phần hướng dẫn trong `SKILL.md`; `[Unknown]` cho phần chi tiết | `SKILL.md:63`, `:65` — reference chi tiết **không tồn tại**; xem **R2** |
| Mock dependency | Bean phụ thuộc → `@MockitoBean` / `@MockitoSpyBean` | `[Verified]` cho phần hướng dẫn trong `SKILL.md`; `[Unknown]` cho phần chi tiết | `SKILL.md:17`, `:22`, `:53` — reference chi tiết **không tồn tại**; xem **R2** |

`[Unknown]` Mức độ chính xác của các khẳng định kỹ thuật trong bộ skill (ví dụ "Spring Boot 4.0 uses JUnit 6 by default" — `references/sb4-migration.md:76`; "Spring Framework 7 allows mocking prototype-scoped beans" — `references/sb4-migration.md:100`; API `@SpringExtensionConfig(useTestClassScopedExtensionContext = true)` — `references/sb4-migration.md:121`). Project không chứa mã Java, manifest hay lockfile nào để đối chiếu, và các khẳng định này chỉ được dẫn nguồn ở một chỗ duy nhất (`SKILL.md:186`, hai liên kết tới `docs.spring.io` không trỏ tới trang nói về JUnit 6 hay Spring Framework 7). Để xác định, cần đối chiếu với release note chính thức của Spring Boot 4.0 / Spring Framework 7, hoặc dựng một dự án thật dùng các phiên bản đó.

## Kiến trúc

`[Verified]` Kiến trúc là **hub-and-spoke hai tầng, tiết lộ tiến dần (progressive disclosure)**: một điểm vào `SKILL.md` chứa các bảng định tuyến, và 13 reference lá chứa chi tiết. (`SKILL.md:35-74`)

- `[Verified]` 26 liên kết nội bộ; 22 liên kết đi từ `SKILL.md` ra reference. (đếm theo mẫu `](*.md)`)
- `[Verified]` Chỉ có 4 liên kết đi *giữa* các reference: `assertj-collections.md:10` → `assertj-basics.md`; `mockmvc-classic.md:12` và `:198` → `mockmvc-tester.md`; `testcontainers-jdbc.md:272` → `failsafe-integration-tests.md` (hỏng).
- `[Inferred]` Đồ thị gần như là cây một tầng, tức mỗi reference được thiết kế để đọc độc lập sau khi vào từ `SKILL.md`. Suy luận từ việc chỉ 4/26 liên kết là liên kết ngang và không có chu trình nào. 
- `[Verified]` Không có reference mồ côi: cả 13 file đều được `SKILL.md` liên kết tới.
- `[Verified]` 3 reference có mục "When to Use This Reference" nêu điều kiện kích hoạt rõ ràng (`references/mockmvc-classic.md:5-12`, `references/assertj-collections.md:5-10`, `references/instancio.md:5-10`); 10 reference còn lại không có mục này.
- `[Inferred]` Đây là một bộ skill **đơn lẻ**, không phải một plugin nhiều skill, dù tên thư mục `awesome-copilot` gợi ý điều ngược lại. Suy luận từ việc chỉ có một `SKILL.md` và không có manifest nào khai báo các skill khác. (`find . -type f` → 14 file, 1 `SKILL.md`)
- `[Verified]` Bộ skill phụ thuộc vào **2 skill nằm ngoài project**: `spring-boot-4-migration` (`references/sb4-migration.md:3`, `:141`) và `java-testcontainers` (`references/testcontainers-jdbc.md:135`), thậm chí trỏ tới đường dẫn file bên trong chúng (`references/08-testing.md`, `references/02-starters-and-removed-features.md`).
- `[Unknown]` Hai skill đó có tồn tại trong môi trường của người dùng hay không. Chúng không nằm trong project này. Cần kiểm tra thư mục cài đặt skill của người dùng — nằm ngoài scope đã được giao.

## Dữ liệu

`[Verified]` Project **không lưu trữ hay xử lý dữ liệu nào**: không có schema, không có migration, không có file dữ liệu, không có kết nối database. Mọi thứ liên quan tới dữ liệu đều là mã mẫu trong tài liệu. (`find . -type f ! -name "*.md"` → rỗng)

`[Verified]` Lược đồ dữ liệu *trong các ví dụ* (không phải schema thật, chỉ là domain minh họa): `Order` (id, status, total/amount, createdAt, customer, items), `OrderItem` (product, price, quantity), `Customer` (name, address), `Address` (city), `Weather` (temperature, condition), `Product` (tags), `User` (roles). (`references/datajpatest.md:130-132`, `references/mockmvc-tester.md:81-93`, `references/restclienttest.md:50-54`, `references/assertj-collections.md:152`, `:162`)

`[Verified]` Có một cảnh báo về schema, đã được kiểm chứng bởi tác giả: `Order` là từ khóa SQL dành riêng nên entity tên `Order` cần `@Table(name = "orders")`, nếu không PostgreSQL báo `syntax error at or near "order"`. (`references/testcontainers-jdbc.md:81`)

`[Verified]` Nhưng chính các ví dụ trong bộ skill lại **không** áp dụng cảnh báo đó: `@Table` chỉ xuất hiện duy nhất một lần trong project, và đó là ở trong câu cảnh báo tại `references/testcontainers-jdbc.md:81`; không có khối mã nào thêm `@Table(name = "orders")` vào entity `Order`, trong khi tất cả ví dụ `@DataJpaTest` đều chạy trên PostgreSQL. Xem **R8**.

## Tích hợp

| Tích hợp | Hướng | Label | Evidence |
|---|---|---|---|
| Docker / container runtime | Project chỉ *hướng dẫn* dùng; bản thân nó không gọi | `[Verified]` | `references/testcontainers-jdbc.md:7` |
| Docker Hub (kéo image postgres/mysql khi test chạy) | Chỉ hướng dẫn | `[Verified]` | `references/testcontainers-jdbc.md:71`, `:98` |
| `~/.testcontainers.properties` (file cấu hình trong home directory của người dùng) | Chỉ hướng dẫn — skill bảo người dùng tự thêm `testcontainers.reuse.enable=true` | `[Verified]` | `references/testcontainers-jdbc.md:123` |
| Spring Boot Actuator endpoint | Chỉ hướng dẫn | `[Verified]` | `references/context-caching.md:52-60` |
| Flyway | Chỉ hướng dẫn | `[Verified]` | `references/testcontainers-jdbc.md:150-168` |
| Maven (`./mvnw verify`, Failsafe, Surefire, Jacoco) | Chỉ hướng dẫn | `[Verified]` | `SKILL.md:63`, `:65`, `:171` |
| `docs.spring.io`, `instancio.org` | Liên kết đọc tài liệu | `[Verified]` | `SKILL.md:186`, `references/instancio.md:230-231` |

`[Verified]` **Không có tích hợp đi ra (outbound) nào mà bản thân project thực hiện.** Project không có mã thực thi, nên không có lời gọi mạng nào xuất phát từ nó.

## Reconcile — các chỗ hai nguồn trong project tự mâu thuẫn

1. `[Unknown]` **`@MockBean`: cấm hay cho phép?** `SKILL.md:17` nói về Boot 4: "`@MockitoBean` replaces `@MockBean` (**do not use the old ones**)" — một câu lệnh không điều kiện. Nhưng `references/mockmvc-classic.md:23` dùng `@MockBean` trong ví dụ setup và `:180` import `org.springframework.boot.test.mock.mockito.MockBean`. File đó tự giới hạn phạm vi ở "Spring Boot < 3.2 hoặc legacy codebase" (`references/mockmvc-classic.md:5-8`), nên có thể là cố ý — nhưng `SKILL.md` không nêu ngoại lệ đó. Không có bằng chứng nào trong project cho biết ý định nào là đúng. Cần người bảo trì xác nhận: `SKILL.md:17` nên thêm ngoại lệ cho legacy, hay `mockmvc-classic.md` nên dùng `@MockitoBean`?

2. `[Unknown]` **Phạm vi phiên bản của `mockmvc-classic.md`.** File tự giới thiệu là dành cho "pre-Spring Boot 3.2" (`references/mockmvc-classic.md:3`, `:6`) nhưng `SKILL.md:51` chỉ ghi "Traditional MockMvc (pre-3.2)" trong khi `SKILL.md:29` lại nêu nguyên tắc chung "Prefer MockMvcTester and RestTestClient over legacy alternatives". Không rõ file này được dự định là tài liệu bảo trì code cũ hay là một lựa chọn song hành. Cần người bảo trì xác nhận.

3. `[Unknown]` **`@AutoConfigureTestDatabase` trong ví dụ Instancio.** `references/instancio.md:91` dùng `@AutoConfigureTestDatabase` **không có** `replace = Replace.NONE`, trong khi mọi ví dụ Testcontainers khác đều có (`references/datajpatest.md:9`, `:168`; `references/testcontainers-jdbc.md:65`, `:107`, `:153`, `:197`). Với annotation ở dạng mặc định, Spring sẽ thay DataSource bằng database nhúng, nên container PostgreSQL khai báo ngay bên dưới (`references/instancio.md:98`) có khả năng không được dùng. Không thể xác định đây là lỗi sót hay cố ý mà không chạy test. Cần chạy ví dụ đó trên một dự án thật, hoặc người bảo trì xác nhận.

## Risk — những gì việc đọc mã phơi ra

Đây là những gì lần đọc này **tình cờ phơi ra**, không phải một cuộc audit bảo mật đầy đủ. Không có fix nào được đề xuất ở đây.

Thang severity: `Critical` (khai thác trực tiếp, thiệt hại lớn) › `High` › `Medium` › `Low` (cần nhiều điều kiện phụ, thiệt hại giới hạn).
Verdict: `Confirmed` (đã truy vết tới nơi phát sinh thiệt hại) › `Potential` (nhận ra mẫu hình, chưa truy vết được khả năng tiếp cận).

### Bối cảnh chung của mọi rủi ro dưới đây

`[Verified]` Project không chứa mã thực thi (14/14 file là `.md`, không có frontmatter cấp quyền công cụ). Do đó **không có rủi ro nào dưới đây có thể tự xảy ra**: tất cả đều yêu cầu một con người hoặc một agent sao chép mã mẫu vào một dự án thật rồi chạy nó. Đây là lý do không có mục nào đạt mức `High` hay `Critical`.

---

### R1 — `addFilters = false` được dạy mà không cảnh báo hệ quả bảo mật

- **Severity:** Medium
- **Verdict:** Potential
- **Category:** Authorization / test hiệu lực giả
- **Label:** `[Verified]` (nội dung tài liệu), `[Inferred]` (hệ quả)
- **Evidence:** `references/webmvctest.md:162-167`

`[Verified]` `references/webmvctest.md:164` viết `@AutoConfigureMockMvc(addFilters = false) // Skip security filters`, và toàn bộ ngữ cảnh quanh nó chỉ là tiêu đề "Excluding Auto-Configuration" cùng comment `// Tests without security filters`. Không có một dòng nào nói khi nào *không* nên dùng nó.

`[Inferred]` Cờ này loại bỏ toàn bộ chuỗi filter servlet của Spring Security khỏi MockMvc. Hệ quả: hai test ngay phía trên trong cùng file — `adminShouldDeleteOrder` kỳ vọng 204 và `anonymousUserShouldBeForbidden` kỳ vọng 401 (`references/webmvctest.md:136-148`) — sẽ **cùng trả về 2xx** nếu lớp test đó bị thêm `addFilters = false`, tức là test phân quyền trở nên xanh một cách vô nghĩa. Suy luận từ chính chú thích "Skip security filters" của tài liệu kết hợp với các assertion ở `:141` và `:147`; chưa được truy vết bằng cách chạy thật, nên verdict là `Potential` chứ không phải `Confirmed`.

### R2 — 4 liên kết hỏng, trong đó một liên kết chính là cơ chế phòng hộ chống test bị bỏ qua im lặng

- **Severity:** Medium
- **Verdict:** Confirmed
- **Category:** Toàn vẹn tài liệu / độ tin cậy của bộ test
- **Label:** `[Verified]`
- **Evidence:** `SKILL.md:22`, `SKILL.md:53`, `SKILL.md:63`, `references/testcontainers-jdbc.md:272`

`[Verified]` Hai file đích không tồn tại trên ổ đĩa:

| Liên kết tại | Trỏ tới | Tồn tại? |
|---|---|---|
| `SKILL.md:22` | `references/mockitobean.md` | ✗ |
| `SKILL.md:53` | `references/mockitobean.md` | ✗ |
| `SKILL.md:63` | `references/failsafe-integration-tests.md` | ✗ |
| `references/testcontainers-jdbc.md:272` | `failsafe-integration-tests.md` | ✗ |

Verdict `Confirmed` vì thiệt hại đã truy vết được trọn vẹn bằng hệ thống file, không cần giả định: `SKILL.md:63` tự mô tả file bị thiếu là tài liệu về việc "`*IT` classes **silently skipped** unless Failsafe is bound", và `references/testcontainers-jdbc.md:272` nhắc lại "see [failsafe-integration-tests.md] **or they will silently not run**". Tức là chính tài liệu cảnh báo về việc test tích hợp âm thầm không chạy lại không tồn tại. Phần hướng dẫn tóm lược vẫn còn ở `SKILL.md:65` (khối "Maven gotcha"), nên mất mát là một phần, không phải toàn bộ.

`mockitobean.md` bị thiếu để lại một khoảng trống lớn hơn: `SKILL.md:17` ra lệnh dùng `@MockitoBean` nhưng tài liệu giải thích cách dùng nó thì không có, trong khi `@MockitoBean` xuất hiện trong 8 file khác.

### R3 — Docker image ghim bằng tag nổi, không có digest

- **Severity:** Low
- **Verdict:** Potential
- **Category:** Chuỗi cung ứng
- **Label:** `[Verified]` (cách ghim), `[Inferred]` (hệ quả)
- **Evidence:** `references/testcontainers-jdbc.md:71`, `:98`, `:278`

`[Verified]` Toàn bộ 14 lần tham chiếu image đều dùng tag dạng `postgres:18`, `postgres:18-alpine`, `mysql:8.4`, không có `@sha256:` ở đâu cả. Không có registry nào được ghi rõ.

`[Inferred]` Tag Docker có thể thay đổi: `postgres:18` hôm nay và sáu tháng sau có thể là hai image khác nhau, và không có registry tường minh nghĩa là image được kéo từ Docker Hub công khai. Trong CI, điều này khiến bộ test không tái lập được theo thời gian và phụ thuộc vào một bên thứ ba. Suy luận từ ngữ nghĩa của tag Docker; chưa truy vết một trường hợp tag bị repoint cụ thể nào, nên verdict là `Potential`.

`[Verified]` Yếu tố giảm nhẹ: bộ skill **đã** khuyên tránh `latest` — "Use specific versions (postgres:18) not latest" (`references/testcontainers-jdbc.md:278`). Vấn đề là nó dừng ở mức tag chứ không tới digest.

### R4 — Mẫu hình hardcode credential trong mã test

- **Severity:** Low
- **Verdict:** Potential
- **Category:** Quản lý secret
- **Label:** `[Verified]` (các chuỗi), `[Inferred]` (hệ quả)
- **Evidence:** `references/restclienttest.md:122-123`, `references/resttestclient.md:171-172`, `references/mockmvc-classic.md:147`, `references/mockmvc-tester.md:209`, `references/testcontainers-jdbc.md:180-181`

`[Verified]` Bộ skill chứa 10 chuỗi trông như credential, tất cả đều là placeholder lộ liễu: header `Authorization` với giá trị `Bearer token123` và `Bearer token`, header `X-Api-Key` với giá trị `secret` (4 lần), và `.withUsername("testuser").withPassword("testpass")` cho container PostgreSQL. **Không có giá trị bí mật thật nào trong project** — và theo nguyên tắc phân tích, không giá trị nào được chép vào bộ tài liệu này ngoài việc nêu rằng chúng là placeholder.

`[Inferred]` Rủi ro không nằm ở các placeholder này mà ở mẫu hình chúng dạy: đặt token và password trực tiếp dưới dạng chuỗi literal trong mã test. Không có ví dụ nào trong bộ skill cho thấy cách lấy secret từ biến môi trường hay từ cấu hình test. Lập trình viên sao chép mẫu này cho một API thật có khả năng điền token thật vào đúng vị trí đó rồi commit. Suy luận từ việc không tồn tại ví dụ thay thế nào trong cả 14 file; chưa truy vết một lần commit secret thật nào, nên `Potential`.

### R5 — Snippet bật Actuator không có cảnh báo chỉ dùng cho test

- **Severity:** Low
- **Verdict:** Potential
- **Category:** Tiết lộ thông tin
- **Label:** `[Verified]` (nội dung), `[Inferred]` (hệ quả)
- **Evidence:** `references/context-caching.md:50-60`

`[Verified]` `references/context-caching.md:52-58` đưa ra một khối YAML bật `management.endpoints.web.exposure.include: metrics`, rồi `:60` hướng dẫn truy cập `GET /actuator/metrics/spring.test.context.cache`. Tiêu đề mục chỉ là "Spring Boot Actuator"; không có dòng nào nói khối YAML này thuộc cấu hình test.

`[Inferred]` Khối YAML không ghi tên file đích, nên người đọc có thể dán vào `application.yml` chung thay vì `application-test.yml`, làm lộ endpoint metrics của ứng dụng production ra HTTP. Suy luận từ việc snippet không nêu file đích và không có cảnh báo phạm vi; mức thiệt hại giới hạn ở metrics (không phải `env` hay `heapdump`) nên severity là Low.

### R6 — `@Rollback(false)` không cảnh báo về database không phải ephemeral

- **Severity:** Low
- **Verdict:** Potential
- **Category:** Toàn vẹn dữ liệu
- **Label:** `[Verified]` (nội dung), `[Inferred]` (hệ quả)
- **Evidence:** `references/datajpatest.md:181-187`

`[Verified]` `references/datajpatest.md:183` giới thiệu `@Rollback(false) // Don't roll back (rarely needed)` và `:186` ghi "Data will remain in database after test".

`[Inferred]` Tài liệu có nói dữ liệu sẽ còn lại, nhưng không nói điều đó nguy hiểm ở đâu: nếu `@DataJpaTest` được trỏ vào một database dùng chung (staging, hoặc DB dev của cả nhóm) thay vì container ephemeral, test sẽ ghi dữ liệu rác vào đó vĩnh viễn. Cụm "rarely needed" là một lời can ngăn nhẹ, không phải cảnh báo. Suy luận từ việc không có câu nào gắn điều kiện "chỉ với DB ephemeral"; `Potential` vì phụ thuộc vào cấu hình DataSource của dự án đích.

### R7 — Container reuse để lại database sống với credential mặc định trên máy lập trình viên

- **Severity:** Low
- **Verdict:** Potential
- **Category:** Bề mặt tấn công cục bộ
- **Label:** `[Verified]` (nội dung), `[Inferred]` (hệ quả)
- **Evidence:** `references/testcontainers-jdbc.md:121-133`

`[Verified]` `references/testcontainers-jdbc.md:123` hướng dẫn bật `testcontainers.reuse.enable=true` trong `~/.testcontainers.properties` và `:127-133` hướng dẫn khởi động container thủ công trong khối `static` **cố ý không** để extension dừng container.

`[Inferred]` Hệ quả là một container PostgreSQL tồn tại sau khi test kết thúc, với credential mặc định của Testcontainers và một cổng được map ra host. Trên máy lập trình viên trong mạng không tin cậy, đó là một database nghe trên cổng mà chẳng ai còn nhớ là đang chạy. Suy luận từ chính mô tả "without extension-managed shutdown" ở `:123`.

`[Verified]` Yếu tố giảm nhẹ mạnh: tài liệu nói rõ ngay câu đầu "Cross-run reuse is **local-dev only**" (`references/testcontainers-jdbc.md:123`), tức phạm vi đã được giới hạn đúng.

### R8 — Cảnh báo về từ khóa SQL `order` không được áp dụng trong chính các ví dụ

- **Severity:** Low
- **Verdict:** Confirmed
- **Category:** Tính nhất quán nội bộ / mã mẫu không chạy được
- **Label:** `[Verified]`
- **Evidence:** `references/testcontainers-jdbc.md:81`

`[Verified]` `references/testcontainers-jdbc.md:81` ghi: "`Order` is a reserved word in SQL; an entity named `Order` needs `@Table(name = "orders")` or PostgreSQL fails with `syntax error at or near "order"` (verified)."

`[Verified]` Nhưng `@Table` chỉ xuất hiện đúng một lần trong toàn project — chính trong câu cảnh báo ở `:81` — và không có khối mã nào áp dụng nó, trong khi entity `Order` được dùng trong mọi ví dụ JPA chạy trên PostgreSQL (`references/datajpatest.md:42-44`, `:60-61`, `:94-96`; `references/testcontainers-jdbc.md:214-215`; `references/instancio.md:106-111`). Verdict `Confirmed` vì mâu thuẫn đã được truy vết trọn vẹn giữa hai vị trí trong cùng project — người sao chép ví dụ `@DataJpaTest` sẽ gặp đúng lỗi mà `:81` cảnh báo.

### R9 — Assertion 401 trong test phân quyền không khớp tên test và phụ thuộc cấu hình

- **Severity:** Low
- **Verdict:** Potential
- **Category:** Độ tin cậy của test bảo mật
- **Label:** `[Verified]` (nội dung), `[Inferred]` (hệ quả)
- **Evidence:** `references/webmvctest.md:144-148`

`[Verified]` Test tên `anonymousUserShouldBeForbidden` (`references/webmvctest.md:145`) lại assert `.hasStatus(HttpStatus.UNAUTHORIZED)` (`:147`) — tức 401, trong khi "Forbidden" là 403.

`[Inferred]` Mã trạng thái thực tế mà Spring Security trả về cho một request không xác thực phụ thuộc vào `AuthenticationEntryPoint` được cấu hình (form login thường redirect/401; stateless API thường 401; người dùng đã xác thực nhưng thiếu quyền thì 403). Tài liệu đưa ra assertion tuyệt đối mà không nêu điều kiện đó. Hệ quả thực tế: test này sẽ đỏ trên nhiều cấu hình, và cách "sửa" nhanh nhất mà một agent hay lập trình viên hay chọn là nới assertion ra (`is4xxClientError()`) hoặc tắt filter — tức dẫn thẳng tới **R1**. Suy luận từ sự lệch nhau giữa tên test và assertion; `Potential` vì chưa chạy trên một cấu hình cụ thể nào.

### R10 — Không có giấy phép và không có thông tin nguồn gốc

- **Severity:** Low
- **Verdict:** Confirmed
- **Category:** Pháp lý / tuân thủ
- **Label:** `[Verified]`
- **Evidence:** `find . -type f` → 14 file, không có `LICENSE`, `NOTICE`, `README` hay `COPYING`

`[Verified]` Project không chứa file giấy phép, không có dòng ghi công tác giả, không có trường `version` hay `author` trong frontmatter (`SKILL.md:1-4`), và không có lịch sử git. Verdict `Confirmed` vì sự thiếu vắng đã được kiểm chứng trọn vẹn bằng hệ thống file. Thiệt hại: điều kiện tái sử dụng, phân phối lại hay sửa đổi bộ skill này không xác định được, và không truy được nguồn gốc nội dung.

### R11 — `@MockBean` đã deprecated xuất hiện trong ví dụ, ngược với lệnh ở điểm vào

- **Severity:** Low
- **Verdict:** Confirmed
- **Category:** Tính nhất quán nội bộ
- **Label:** `[Verified]`
- **Evidence:** `SKILL.md:17` so với `references/mockmvc-classic.md:23`, `:180`

`[Verified]` `SKILL.md:17` ra lệnh không điều kiện cho Boot 4: "`@MockitoBean` replaces `@MockBean` (do not use the old ones)". `references/mockmvc-classic.md:23` dùng `@MockBean` và `:180` import `org.springframework.boot.test.mock.mockito.MockBean`. Verdict `Confirmed` vì mâu thuẫn văn bản đã truy vết được giữa hai dòng cụ thể. File legacy có tự giới hạn phạm vi (`:5-8`) nên có thể là cố ý — nhưng khi đó chính `SKILL.md:17` là chỗ thiếu ngoại lệ. Xem [§ Reconcile](#reconcile--các-chỗ-hai-nguồn-trong-project-tự-mâu-thuẫn) mục 1 cho khoảng trống `[Unknown]` đi kèm.

---

### Những gì đã kiểm tra và **không** tìm thấy

Nêu rõ ở đây để sự im lặng không bị đọc thành "đã kiểm tra mọi thứ":

| Đã kiểm tra | Kết quả | Label |
|---|---|---|
| Prompt injection (ignore previous / disregard / system prompt / you are now / override / jailbreak / exfiltrate / secretly / do not tell) | Không khớp | `[Verified]` |
| Ký tự Unicode ẩn (zero-width, bidi override, tag character) | Không khớp; 16 ký tự non-ASCII đều là `—` và `→` | `[Verified]` |
| Lệnh tải về / thực thi (`curl`, `wget`, `base64`, `eval`, `exec(`, `subprocess`, `os.system`, `powershell`, `Invoke-WebRequest`, `chmod`, `rm -rf`, `pip install`, `npm i`) | Không khớp | `[Verified]` |
| File thực thi, script, binary, file ẩn | Không có; 14/14 file là `.md` | `[Verified]` |
| Frontmatter cấp quyền công cụ (`allowed-tools`) hoặc hook | Không có; frontmatter chỉ có `name`, `description` | `[Verified]` |
| Endpoint thu thập dữ liệu trong 9 URL ngoài | Không có; toàn bộ là tài liệu công khai hoặc URL ví dụ | `[Verified]` |
| Giá trị secret thật | Không có; 10 chuỗi tìm thấy đều là placeholder (xem **R4**) | `[Verified]` |
| Chỉ thị dành cho agent được nhúng ngoài mục đích hợp pháp của skill | Không có | `[Verified]` |

`[Unknown]` Những gì **chưa** được kiểm tra: tính đúng đắn kỹ thuật của các khẳng định về Spring Boot 4 / JUnit 6 / Testcontainers 2.x / Java 25 (không có manifest hay mã nguồn trong project để đối chiếu, và không có hệ thống thật nào được kết nối); các lỗ hổng của chính các thư viện được khuyến nghị (cần đọc lockfile của dự án đích); và nội dung của hai skill ngoài được tham chiếu. Để lấp các khoảng trống này cần chạy một audit riêng theo `security-and-hardening` trên một dự án thật có áp dụng bộ skill.
