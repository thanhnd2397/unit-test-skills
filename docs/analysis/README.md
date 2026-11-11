# Phân tích dự án — `awesome-copilot`

## Analysis header

| | |
|--|--|
| Ngày phân tích | 2026-11-10 |
| Scope | Toàn bộ project `D:\new project\Ai\awesome-copilot`, với ưu tiên đầu tiên là rà soát vấn đề bảo mật (theo yêu cầu của người dùng) |
| Shape | **small** — 1 module (một skill package duy nhất), 1 business domain (hướng dẫn kiểm thử Spring Boot), 0 deployable. Dưới ngưỡng 3 domain / 10 module / >1 deployable nên tách `domain.md` và `architecture.md` sẽ khiến cả hai quá mỏng. |
| Commit | **Không có.** Đây không phải git repository (`git`: `Is a git repository: false`; không tồn tại thư mục `.git`). Không thể ghi nhận commit hay trạng thái sạch/bẩn của working tree. |
| Baseline | **Không có baseline.** Đây là lần phân tích đầu tiên (`docs/analysis/` chưa tồn tại trước lần chạy này) và project không có lịch sử git, nên không có gì để diff. Phân tích được thực hiện **đầy đủ**, không có phần nào được coi là "không đổi". |
| Phạm vi đã đọc | **100%** — toàn bộ 14 file markdown (74.449 byte) đã được đọc trọn vẹn, không bỏ dòng nào. |
| Không phân tích | Các skill được tham chiếu nhưng nằm **ngoài** project này: `spring-boot-4-migration` (`references/sb4-migration.md:3`, `:141`) và `java-testcontainers` (`references/testcontainers-jdbc.md:135`). Các thư mục cha của project. Tính đúng đắn kỹ thuật của các API Spring Boot 4 / JUnit 6 / Testcontainers 2.x được nêu trong tài liệu — không thể kiểm chứng từ project này vì project không chứa manifest, lockfile hay mã Java nào. |

## Tóm tắt

- `[Verified]` Project là **một agent skill duy nhất** dạng tài liệu thuần markdown, tên `spring-boot-testing`, cung cấp hướng dẫn kiểm thử ứng dụng Spring Boot (test slices, Testcontainers, MockMvcTester, AssertJ, Instancio, Failsafe). (`SKILL.md:1-4`)
- `[Verified]` Không có bất kỳ mã thực thi nào trong project: 14/14 file là `.md`, không có script, binary, file ẩn hay file cấu hình. (`find . -type f ! -name "*.md"` không trả về kết quả; `find . -name ".*"` chỉ trả về chính thư mục gốc)
- `[Verified]` Skill không khai báo `allowed-tools` hay bất kỳ quyền truy cập công cụ nào trong frontmatter — frontmatter chỉ có `name` và `description`. (`SKILL.md:1-4`)
- `[Inferred]` Vì vậy skill này **không thể tự thực thi hành động nào**; toàn bộ ảnh hưởng của nó là nội dung văn bản đưa vào ngữ cảnh của agent, và rủi ro bảo mật duy nhất là rủi ro của *lời khuyên sai hoặc thiếu cảnh báo* mà con người/agent có thể sao chép vào mã nguồn thật. Suy luận dựa trên việc không tồn tại file thực thi nào và frontmatter không cấp quyền. (`SKILL.md:1-4`)
- `[Verified]` Tên thư mục là `awesome-copilot` (gợi ý một *bộ* nhiều skill) nhưng chỉ chứa đúng một skill, không có manifest plugin, không có README, không có LICENSE. (danh sách file đầy đủ ở [tech.md](tech.md#kiểm-kê-file))

## Kết luận về bảo mật (ưu tiên số 1 của người dùng)

- `[Verified]` **Không phát hiện prompt injection.** Không có chuỗi nào thuộc nhóm "ignore previous instructions", "disregard", "system prompt", "you are now", "override", "exfiltrate", "jailbreak", hay chỉ thị gửi dữ liệu ra ngoài. (`grep -rniE 'ignore (all |any |previous|prior)|disregard|system prompt|you are now|do not tell|without asking|override|jailbreak|secretly|exfiltrat|...'` trên toàn project → không khớp)
- `[Verified]` **Không có ký tự Unicode ẩn.** Không tồn tại zero-width space, ký tự bidi override hay Unicode tag character. Toàn bộ 16 ký tự non-ASCII trong project là em-dash (`—`) và mũi tên (`→`) trong văn bản thường. (`grep -rnP '[\x{200B}-\x{200F}\x{202A}-\x{202E}\x{2060}-\x{206F}\x{FEFF}\x{E0000}-\x{E007F}]'` → không khớp)
- `[Verified]` **Không có lệnh tải về / thực thi.** Không có `curl`, `wget`, `base64`, `eval`, `exec(`, `subprocess`, `os.system`, `powershell`, `Invoke-WebRequest`, `chmod`, `rm -rf`, `pip install`, `npm i`. Lệnh duy nhất mà skill khuyến nghị là `./mvnw verify` (`SKILL.md:63`, `:65`) — một lệnh build Maven tiêu chuẩn, **không được chạy** trong lần phân tích này.
- `[Verified]` **Không có giá trị bí mật thật.** 9 URL ngoài trong project đều vô hại: `docs.spring.io` (2), `instancio.org` (2), và 5 URL ví dụ thuộc `api.example.com` / `api.weather.com`. Không có endpoint thu thập dữ liệu nào.
- `[Verified]` **Không có hook, không có tool grant, không có MCP server.** Project không chứa `settings.json`, `hooks`, `plugin.json` hay bất kỳ file cấu hình nào.

Kết luận: `[Inferred]` Về mặt *bảo mật chuỗi cung ứng và prompt injection*, bộ skill này **sạch** — nó không có bề mặt tấn công chủ động. Những phát hiện bảo mật còn lại đều thuộc loại **chất lượng lời khuyên**: các đoạn mã mẫu dạy một mẫu hình có thể làm yếu bảo mật nếu được sao chép nguyên văn vào dự án thật. Chi tiết từng phát hiện, kèm mức độ và verdict, nằm ở [system.md § Risk](system.md#risk--những-gì-việc-đọc-mã-phơi-ra). Suy luận dựa trên toàn bộ các kiểm tra `[Verified]` ở trên.

### Những phát hiện mức Medium trở lên

1. `[Verified]` **Medium / Potential** — `@AutoConfigureMockMvc(addFilters = false)` được giới thiệu kèm chú thích "Skip security filters" mà **không có cảnh báo** rằng nó vô hiệu hóa toàn bộ chuỗi filter của Spring Security, khiến mọi assertion về phân quyền trong test đó trở thành vô nghĩa. (`references/webmvctest.md:164`)
2. `[Verified]` **Medium / Confirmed** — 4 liên kết nội bộ trỏ tới 2 file không tồn tại (`references/mockitobean.md`, `references/failsafe-integration-tests.md`). Đáng chú ý: `failsafe-integration-tests.md` chính là tài liệu mà skill nói là để tránh việc test `*IT` **bị bỏ qua trong im lặng** — tức là chính cơ chế phòng hộ lại trỏ vào hư không. (`SKILL.md:22`, `SKILL.md:53`, `SKILL.md:63`, `references/testcontainers-jdbc.md:272`)

## Kiểm kê (counts)

Quy tắc đếm được ghi rõ cho từng số. Phạm vi đếm: toàn bộ project; không loại trừ file nào (không có thư mục dependency, build output hay file sinh tự động để loại trừ).

| Hạng mục | Số lượng | Quy tắc đếm |
|---|---|---|
| Skill | **1** | Số file `SKILL.md` có YAML frontmatter chứa khóa `name`. |
| Module | **1** | Một skill package = một module. Không có manifest nào chia nhỏ hơn. |
| Deployable | **0** | Không có Dockerfile, CI workflow, build script hay package manifest nào. |
| Business domain | **1** | "Hướng dẫn kiểm thử Spring Boot" — chủ đề duy nhất của toàn bộ 14 file. |
| File tổng | **14** | Mọi file thường trong project (`find . -type f`). Gồm 1 `SKILL.md` + 13 reference. |
| File `.md` | **14** | 14/14 — không có file nào thuộc loại khác. |
| Tổng dung lượng | **74.449 byte** | `wc -c` trên cả 14 file. |
| Reference document | **13** | File `.md` trong `references/`, không tính `SKILL.md`. |
| Liên kết nội bộ `.md` | **26** | Số lần khớp mẫu `](<đường dẫn>.md)` trên toàn project. |
| Liên kết bị hỏng | **4** | Trong 26 liên kết trên, số liên kết có file đích không tồn tại trên ổ đĩa. Trỏ tới **2** file đích khác nhau. |
| Reference "mồ côi" | **0** | Reference tồn tại nhưng không được `SKILL.md` liên kết tới. Cả 13 file đều được liên kết. |
| Khối mã (code block) | **170** | Số cặp dấu ``` mở/đóng ở đầu dòng, cộng trên 14 file. |
| Dependency Maven được khai báo | **15** | Số `artifactId` phân biệt xuất hiện trong các khối XML hoặc được nêu tên trong văn bản. Phân loại thương mại ở [tech.md](tech.md#dependency-và-phân-loại-thương-mại). |
| Dependency có version tường minh | **1** | `instancio-junit` `5.5.1`. 14 cái còn lại không ghi version (do BOM quản lý) → `[Unknown]`. |
| Docker image được tham chiếu | **3** | `postgres:18`, `postgres:18-alpine`, `mysql:8.4` — đếm theo chuỗi image:tag phân biệt. |
| Skill ngoài được tham chiếu | **2** | `spring-boot-4-migration`, `java-testcontainers` — không nằm trong project này. |
| Phát hiện rủi ro | **11** | Mọi hạng mục được ghi nhận ở [system.md § Risk](system.md#risk--những-gì-việc-đọc-mã-phơi-ra), gồm cả loại không phải bảo mật. |
| Khoảng trống `[Unknown]` | **14** | Số câu hỏi mở phân biệt mang nhãn `[Unknown]` trong bộ tài liệu này. Loại trừ: 15 ô "version" trong bảng dependency (gộp thành **1** khoảng trống hệ thống — project không có manifest nào) và các lần nhãn được nhắc lại khi tham chiếu chéo. Liệt kê đầy đủ ở tin nhắn bàn giao. |

### Phân bổ theo module

Project chỉ có 1 module, nên mọi con số ở trên đều thuộc module đó. Phân bổ theo file (khối mã / dung lượng):

| File | Khối mã | Byte |
|---|---:|---:|
| `SKILL.md` | 4 | 8.503 |
| `references/mockmvc-tester.md` | 19 | 7.696 |
| `references/restclienttest.md` | 16 | 4.736 |
| `references/resttestclient.md` | 16 | 5.042 |
| `references/instancio.md` | 15 | 5.032 |
| `references/assertj-basics.md` | 14 | 4.384 |
| `references/assertj-collections.md` | 13 | 5.015 |
| `references/mockmvc-classic.md` | 13 | 5.929 |
| `references/testcontainers-jdbc.md` | 13 | 7.903 |
| `references/test-slices-overview.md` | 11 | 5.638 |
| `references/datajpatest.md` | 10 | 5.094 |
| `references/webmvctest.md` | 10 | 3.652 |
| `references/context-caching.md` | 8 | 2.467 |
| `references/sb4-migration.md` | 8 | 3.358 |

## Document map

| File | Nội dung |
|---|---|
| `README.md` (file này) | Header, tóm tắt, kết luận bảo mật, kiểm kê, bản đồ tài liệu |
| [tech.md](tech.md) | Kiểm kê file, dependency và phân loại thương mại, Docker image, cách "chạy"/kiểm tra bộ skill |
| [system.md](system.md) | Domain, actor, các tính năng của skill, kiến trúc tài liệu, dữ liệu & tích hợp, phần Risk đầy đủ |

Không có `features/` nào được viết: người dùng chưa yêu cầu tài liệu cho một tính năng cụ thể.

Không có `risk-register.md` và `questions.md`: hai file này không thuộc shape mặc định. Toàn bộ rủi ro đã được báo cáo trong [system.md § Risk](system.md#risk--những-gì-việc-đọc-mã-phơi-ra) và trong tin nhắn bàn giao; mọi khoảng trống `[Unknown]` nằm trong tài liệu sở hữu chủ đề đó.
