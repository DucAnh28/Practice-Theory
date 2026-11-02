# JPA/Hibernate

- **Date:** 2026-11-02
- **Topic:** jpa-hibernate
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn
Khi nào nên dùng `@Entity` với Hibernate thay vì `@PersistenceContext`?

## 2) Giải thích đơn giản
`@Entity` là annotation cho class là entity trong database, Hibernate tự động map class này vào table. Dùng `@PersistenceContext` khi bạn cần inject context vào method/constructor.

## 3) Trả lời Middle+
Chọn `@Entity` khi class là model chính của domain (có bảng riêng, được Hibernate map trực tiếp). Dùng `@PersistenceContext` khi bạn cần inject context vào method để xử lý nghiệp vụ cụ thể.

## 4) Trả lời Senior
- **Trade-off:** `@Entity` cho phép Hibernate tự động map, nhưng không hỗ trợ custom DDL, custom query, hoặc integration với frameworks khác (e.g., MyBatis). `@PersistenceContext` cho phép inject context vào code, giúp kiểm soát rõ hơn flow.
- **Production concern:** Khi dùng `@Entity`, Hibernate tự động tạo migration (AQL) và có thể gây xung đột với schema thay đổi. Cần dùng `hibernate.hbm2ddl.auto` cẩn thận.
- **Edge case:** Nếu class có nhiều relation phức tạp, có thể `@PersistenceContext` với custom query (e.g., `@Query` với `JOIN FETCH`) linh hoạt hơn.

## 5) Follow-up / pitfall
- **Pitfall:** Dùng `@Entity` cho class chỉ chứa DTO hoặc helper method sẽ gây overhead không cần thiết. Nên dùng `@Component` hoặc `@Service` cho logic business.
- **Common mistake:** Quên khai báo `@Entity` trên class, dẫn đến Hibernate không map được table.
- **Interview point:** Khi có requirement custom query (e.g., `findByStatusAndCreatedBefore`), `@PersistenceContext` với `@Query` linh hoạt hơn `@Entity`.

## 6) 30 giây tóm tắt miệng
`@Entity` map class vào DB tự động, `@PersistenceContext` inject context vào method. Chọn `@Entity` cho model chính, `@PersistenceContext` cho logic cụ thể.
