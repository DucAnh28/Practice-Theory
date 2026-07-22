# Composite Index — Thứ tự cột có quan trọng không?

- **Date:** 2026-07-22
- **Topic:** sql
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Bảng `orders(user_id, status, created_at)` có ~50 triệu dòng. Bạn tạo composite index `(user_id, status, created_at)`. Câu truy vấn nào sau đây dùng được index, câu nào không? Vì sao thứ tự cột trong composite index lại quan trọng?

```sql
A) SELECT * FROM orders WHERE user_id = 123;
B) SELECT * FROM orders WHERE user_id = 123 AND status = 'PAID';
C) SELECT * FROM orders WHERE status = 'PAID';
D) SELECT * FROM orders WHERE user_id = 123 ORDER BY created_at DESC;
E) SELECT * FROM orders WHERE user_id = 123 AND status = 'PAID'
   ORDER BY created_at DESC;
```

## 2) Giải thích như trẻ lên 3

Tưởng tượng danh bạ điện thoại được sắp theo `(Họ, Tên, Số nhà)`.

- Tìm theo `Họ` → nhanh, vì sắp sẵn.
- Tìm theo `Họ + Tên` → cũng nhanh.
- Tìm chỉ theo `Tên` → chậm, vì sách không xếp theo Tên làm đầu.

Composite index cũng vậy: nó xếp theo thứ tự cột từ trái sang phải. Bỏ cột đầu thì index "không biết đường dùng".

## 3) Giải thích đơn giản cho dev

Composite index `(a, b, c)` tạo cây B-Tree xếp theo `a` trước, trong cùng `a` xếp theo `b`, trong cùng `b` xếp theo `c`. Vì vậy index phục vụ được các truy vấn:

| Truy vấn WHERE | Dùng được index? | Lý do |
|---|---|---|
| `a = ?` | ✅ | Dùng cột đầu |
| `a = ? AND b = ?` | ✅ | Dùng cả 2 cột đầu |
| `a = ? AND b = ? AND c = ?` | ✅ | Đủ 3 cột |
| `a = ? AND c = ?` | ⚠️ một phần | Chỉ `a` dùng index, `c` lọc sau |
| `b = ?` | ❌ | Thiếu cột đầu |
| `c = ?` | ❌ | Thiếu cột đầu |

Ngoài ra `ORDER BY` cũng theo thứ tự: nếu đã lọc bằng `a` rồi `ORDER BY b, c` theo cùng chiều index thì MySQL có thể tránh `filesort`.

## 4) Ví dụ code đơn giản

```sql
-- Index
CREATE INDEX idx_orders_user_status_created
  ON orders (user_id, status, created_at);

-- A: dùng index (ref trên user_id)
EXPLAIN SELECT * FROM orders WHERE user_id = 123;

-- B: dùng index (ref trên user_id, status)
EXPLAIN SELECT * FROM orders WHERE user_id = 123 AND status = 'PAID';

-- C: KHÔNG dùng được index đầu → full scan
EXPLAIN SELECT * FROM orders WHERE status = 'PAID';

-- D: dùng index, tránh sort (index đã xếp created_at)
EXPLAIN SELECT * FROM orders
  WHERE user_id = 123
  ORDER BY created_at DESC;

-- E: dùng index cho cả WHERE lẫn ORDER BY
EXPLAIN SELECT * FROM orders
  WHERE user_id = 123 AND status = 'PAID'
  ORDER BY created_at DESC;
```

Mẹo đọc `EXPLAIN`: cột `key` cho biết index nào được chọn, `Extra` có `Using filesort` / `Using temporary` là dấu hiệu index chưa khớp thứ tự.

## 5) Trả lời Middle+

"Composite index xếp theo thứ tự cột từ trái sang phải, nên chỉ dùng được khi WHERE có cột đầu. Với bảng `orders` và index `(user_id, status, created_at)`:

- Câu A, B, D, E dùng index vì có `user_id` đứng đầu.
- Câu C không dùng được vì lọc `status` mà thiếu `user_id`.
- Câu E còn tận dụng được phần `ORDER BY created_at` vì index đã sắp sẵn.

Khi thiết kế index: đặt cột hay lọc nhất / chọn lọc cao nhất lên trước, sau đó mới đến cột dùng cho sắp xếp."

## 6) Trả lời Senior

"Mình bổ sung thêm các điểm thường bị xoáy:

1. **Cột chọn lọc (selectivity) cao nên đặt trước**, nhưng phải thực đo bằng `ANALYZE TABLE` + `EXPLAIN ANALYZE` chứ không đoán. Đôi khi cột ít chọn lọc nhưng luôn xuất hiện trong `WHERE` (vd `tenant_id` trong hệ multi-tenant) vẫn cần đặt trước vì nó cắt giảm tập dữ liệu sớm.
2. **Index không miễn phí**: tốn RAM, chậm write, có thể gây lock contention. Vài index chọn lọc tốt hơn nhiều index trùng cột đầu.
3. **Sắp xếp theo chiều index**: `ORDER BY created_at DESC` vẫn dùng được vì MySQL 8+ hỗ trợ descending index. Cũ với 5.7 thì phải đúng chiều ASC.
4. **Covering index**: nếu SELECT chỉ lấy cột nằm trong index, không cần quay về bảng (`Extra: Using index`). Có thể thêm cột vào cuối index để trở thành covering.
5. **Index merge / skip scan**: MySQL có thể dùng skip scan khi thiếu cột đầu nhưng prefix cardinality thấp — không nên phụ thuộc, vẫn phải đặt cột đầu đúng.
6. **Pitfall hay gặp**: cast sai kiểu (vd `WHERE user_id = '123'` mà `user_id` là bigint) làm index bị bỏ; hàm trên cột (`WHERE DATE(created_at) = ...`) cũng vô hiệu hóa index phần đó."

## 7) Follow-up / pitfall

- **F1.** Nếu cần truy vấn cả `WHERE status = ?` (không có `user_id`) thì sao?
  → Tạo thêm index `(status, created_at)` hoặc `(status)` riêng, hoặc cân nhắc full-text/partial index tùy DB.
- **F2.** Composite index 4–5 cột có ổn không?
  → Không nên. Index càng rộng càng tốn và khó dùng đúng. Ưu tiên 2–3 cột, tạo thêm index phụ cho pattern truy vấn cụ thể.
- **F3.** Sao query chạy nhanh hôm qua, hôm nay chậm?
  → Thường do statistics cũ hoặc skewed data. Chạy `ANALYZE TABLE`; nếu dữ liệu phân bố lệch, cân nhắc histogram (PostgreSQL) hoặc index một phần.
- **F4.** `LIKE 'abc%'` có dùng index không? Còn `LIKE '%abc%'`?
  → `LIKE 'abc%'` dùng được B-Tree index (prefix). `LIKE '%abc%'` thì không; cần full-text search.
- **F5.** `NULL` ảnh hưởng thế nào đến composite index?
  → Trong MySQL, NULL vẫn được index. Nhưng `WHERE col IS NULL` thỉnh thoảng không chọn index nếu optimizer đánh giá tệ — cần `ANALYZE` lại.

## 8) 30 giây tóm tắt miệng

"Composite index xếp theo thứ tự cột từ trái sang phải, nên chỉ phát huy khi WHERE có cột đầu. Với index `(user_id, status, created_at)`, các truy vấn có `user_id` đều dùng được index và có thể tận dụng luôn phần `ORDER BY created_at`. Khi thiết kế, đặt cột hay lọc và có selectivity cao lên trước, đo bằng EXPLAIN, và nhớ index tốn chi phí write cùng với RAM — chỉ tạo khi thật sự cần."
