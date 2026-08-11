# JPA N+1 Query

- **Date:** 2026-08-11
- **Topic:** jpa-hibernate
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

N+1 query trong JPA/Hibernate là gì? Phát hiện và xử lý thế nào?

## 2) Giải thích như trẻ lên 3

Có 1 lần lấy danh sách bố, rồi đi hỏi từng người bố về con. 1 lần hỏi danh sách + N lần hỏi từng người = N+1 lần hỏi database.

## 3) Giải thích đơn giản cho dev

Ví dụ lấy 100 `Order`, sau đó gọi `order.getItems()` trong vòng lặp. Nếu association chưa được nạp, Hibernate chạy 1 query lấy orders và thêm 100 query lấy items.

Cách tránh thường dùng:

- `JOIN FETCH` cho use case cần dữ liệu liên quan.
- `@EntityGraph` để khai báo fetch plan.
- Batch fetching (`@BatchSize` hoặc `hibernate.default_batch_fetch_size`) để gom nhiều ID.
- Projection/DTO khi chỉ cần vài cột.

`EAGER` không phải thuốc chữa N+1; có thể vẫn tạo nhiều query và kéo quá nhiều dữ liệu.

## 4) Ví dụ code đơn giản

```java
@Query("""
       select distinct o from Order o
       left join fetch o.items
       where o.status = :status
       """)
List<Order> findOrdersWithItems(OrderStatus status);
```

`distinct` giúp loại bản ghi `Order` trùng trong kết quả Java khi một order có nhiều item.

## 5) Trả lời Middle+

N+1 xảy ra khi query danh sách entity rồi lazy-load association từng entity trong vòng lặp. Tôi kiểm tra SQL log hoặc APM, sau đó dùng `JOIN FETCH`, `EntityGraph`, batch fetching hoặc DTO projection tùy nhu cầu. Tôi không đổi toàn bộ association sang `EAGER` vì cách đó dễ gây query thừa và memory tăng.

## 6) Trả lời Senior

Tôi chọn fetch plan theo từng màn hình hoặc API, không theo mặc định entity. `JOIN FETCH` phù hợp khi dữ liệu nhỏ và cần trả cùng lúc, nhưng nhiều collection có thể tạo Cartesian product, duplicate row và pagination sai. Với list lớn, tôi ưu tiên DTO projection, batch fetching hoặc query hai bước; kiểm tra execution plan, số query, latency, heap và kích thước response. Luôn test với dữ liệu gần production, vì dataset nhỏ có thể che lỗi N+1.

## 7) Follow-up / pitfall

- `JOIN FETCH` nhiều collection: dễ phình số dòng; Hibernate có thể báo multiple bag fetch.
- Pagination cùng collection fetch join: thường không phân trang đúng ở database.
- `@Transactional` không phải cách sửa; nó chỉ giữ session mở để lazy-load.
- Xem SQL log chưa đủ; cần đo query count và latency bằng test/APM.

## 8) 30 giây tóm tắt miệng

N+1 là 1 query lấy danh sách rồi N query lấy association từng phần tử. Tôi phát hiện bằng SQL log, query counter hoặc APM. Tôi chọn `JOIN FETCH` hoặc `EntityGraph` cho dữ liệu nhỏ cần lấy cùng lúc, DTO projection hoặc batch fetching cho list lớn. Không dùng `EAGER` đại trà, và phải kiểm tra pagination, duplicate row, memory cùng performance trên dữ liệu thực tế.
