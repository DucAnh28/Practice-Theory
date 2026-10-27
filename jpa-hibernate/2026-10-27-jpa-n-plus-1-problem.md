# JPA N+1 Select Problem

- **Date:** 2026-10-27
- **Topic:** jpa-hibernate
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn
"Anh có bao giờ gặp vấn đề N+1 trong JPA/Hibernate chưa? Nó là gì và cách xử lý thế nào?"

## 2) Giải thích như trẻ lên 3
Con có 10 người bạn (bảng Cha), mỗi người có 5 quả bóng (bảng Con). Thay vì hỏi 10 lần (1 lần hỏi cha + 10*5 lần hỏi bóng riêng), con hỏi 1 lần là: "Đưa hết bóng của tất cả 10 bạn đây". N+1 là cách hỏi sai tốn thời gian.

## 3) Giải thích đơn giản cho dev
Hibernate thực hiện 1 truy vấn lấy danh sách thực thể cha, sau đó lại thực hiện thêm N truy vấn con để lấy dữ liệu quan hệ cho từng thực thể cha đó. Tổng là N+1 queries thay vì 1 câu query JOIN.

## 4) Ví dụ code đơn giản
```java
// Trong code gọi:
List<Author> authors = authorRepo.findAll();
for (Author a : authors) {
    System.out.println(a.getBooks().size()); // Trigger truy vấn con mỗi vòng
}
```

## 5) Trả lời Middle+
- Nguyên nhân: Lazy Loading mặc định (`@OneToMany` hoặc `@ManyToMany`).
- Cách sửa: Dùng `JOIN FETCH` trong JPQL, `@EntityGraph`, hoặc `batch size` ở mức `@BatchSize`.

## 6) Trả lời Senior
- Phân tích: N+1 không chỉ là performance, mà còn là vấn đề connection pool starvation.
- Chiến lược: Ưu tiên `JOIN FETCH` cho quy mô nhỏ. Với tập dữ liệu lớn, cân nhắc Projection (DTO projection) hoặc SQL native để tránh load toàn bộ Entity vào Persistence Context (làm tốn bộ nhớ).
- Lưu ý: Cẩn thận `Cartesian Product` khi JOIN FETCH nhiều quan hệ `ToMany` cùng lúc.

## 7) Follow-up / pitfall
- Hỏi: Nếu dùng `JOIN FETCH` cho 2 danh sách `ToMany` cùng lúc?
- Đáp: Sẽ sinh ra Cartesian Product => dữ liệu bị nhân đôi (duplicate). Phải dùng `Set` hoặc truy vấn tách riêng.

## 8) 30 giây tóm tắt miệng
N+1 là lỗi truy vấn dữ liệu quan hệ kiểu lười (lazy load) gây n+1 lần gọi DB. Xử lý bằng `JOIN FETCH` (cho quan hệ đơn) hoặc `DTO Projection` để lấy đúng thứ cần, tránh load thừa entity vào bộ nhớ.
