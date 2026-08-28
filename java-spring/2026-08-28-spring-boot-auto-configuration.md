# Spring Boot Auto-Configuration

- **Date:** 2026-08-28
- **Topic:** java-spring
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Làm sao để Spring Boot tự động cấu hình (auto-configure) một bean mà không cần dữ liệu cấu hình `@Configuration` thủ công?

## 2) Giải thích đơn giản

Tưởng tượng Spring Boot như một **người phục vụ tự động**:
- Nhìn vào classpath (thư viện nào có trên môi trường?)  
- Đọc file `application.properties` / `application.yml`  
- Sau đó tự bật/tắt bean phù hợp, không cần bạn viết nhiều `@Bean`.

Thuật ngữ:
- **Auto-configuration**: Bật cấu hình tự động.
- **Conditional (`@ConditionalOn...`)**: Quy tắc bật/tắt bean.
- **Starter**: Bộ thư viện rút gọn dependency.
- **ApplicationContext**: "vùng chứa" Spring quản lý bean.

| Điều kiện | Ý ngh�a |
|---|---|
| `@ConditionalOnClass` | Chỉ tạo bean nếu class đó tồn tại trên classpath |
| `@ConditionalOnProperty` | Chỉ tạo bean nếu property được bật |
| `@ConditionalOnMissingBean` | Chỉ tạo bean nếu chưa có ai đăng ký bean đó |

## 3) Trả lời Middle+

Spring Boot dựa vào:
1. **Classpath scanning** — xem thư viện nào có mặt.
2. **Properties** — đọc cấu hình từ `application*.properties|yml`.
3. **`spring.factories`/`auto-configuration`** — đăng ký auto-config class.

Ví dụ: thêm `spring-boot-starter-data-jpa` → boot tự cấu hình `EntityManagerFactory`, `DataSource`, `JpaRepository` beans nếu có DB driver.

## 4) Trả lời Senior

Senior cần nói về **trade-off và risk**:
- Auto-config giúp giảm config, nhưng **giấu chi tiết**, khó debug.
- Khi cần override: dùng `@ConditionalOnMissingBean`, subclass `AutoConfiguration`, hoặc `@ImportAutoConfiguration(exclude = ...)`.
- **Order auto-config** quan trọng: dùng `AutoConfiguration.ORDERED` hoặc `@AutoConfiguration(after = ..., before = ...)` để tránh race khi 2 starter cùng tạo bean.
- **Production**: nên bật `debug=true` (log auto-config report) để kiểm tra "tại sao bean X được/không được tạo".

## 5) 30 giây tóm tắt miệng

"Spring Boot auto‑configure tự động tạo bean dựa trên classpath và cấu hình. Middle: hiểu điều kiện bật; Senior: biết override, order, và debug report để maintain hệ thống lớn."

## 6) Follow-up / pitfall

- Pitfall 1: thêm dependency mà không biết auto-config âm thầm bật bean tốn tài nguyên → kiểm tra report.
- Pitfall 2: `@ConditionalOnMissingBean` không đúng scope hoặc proxyMode → bean tạo lặp.
- Follow-up: “Khi nào nên tắt auto-config đầy đủ and viết manual config?” (trả lời: custom framework, lib tùy chỉnh nặng, hoặc muốn minh bạch tuyệt đối)