# Phantom read và transaction isolation

- **Date:** 2026-09-04
- **Topic:** sql
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Phantom read là gì? Chọn isolation level nào khi báo cáo cần tập dữ liệu ổn định trong một transaction?

## 2) Giải thích như trẻ lên 3

Đang đếm 3 quả táo. Người khác bỏ thêm 1 quả vào giỏ. Đếm lại thành 4 quả, dù chưa rời khỏi giỏ. Quả táo mới là “phantom”.

## 3) Giải thích đơn giản cho dev

Một transaction chạy cùng điều kiện `WHERE` hai lần nhưng thấy thêm hoặc mất dòng vì transaction khác đã `INSERT`/`DELETE` rồi commit. `READ COMMITTED` có thể gặp. `REPEATABLE READ` hoặc cơ chế snapshot phù hợp thường giữ tập đọc ổn định trong transaction, tùy DB.

## 4) Ví dụ code đơn giản

```sql
-- Transaction A
BEGIN;
SELECT COUNT(*) FROM orders WHERE status = 'PENDING'; -- 10

-- Transaction B
INSERT INTO orders(status) VALUES ('PENDING');
COMMIT;

-- Transaction A
SELECT COUNT(*) FROM orders WHERE status = 'PENDING'; -- có thể thành 11
COMMIT;
```

## 5) Trả lời Middle+

Phantom read là thay đổi tập dòng khớp cùng điều kiện giữa hai lần query. Dùng isolation level DB hỗ trợ, hoặc đọc qua snapshot nhất quán, khi nghiệp vụ cần số liệu ổn định. Không tăng isolation mặc định chỉ để xử lý một màn hình báo cáo.

## 6) Trả lời Senior

Kiểm tra semantics theo DB: PostgreSQL `REPEATABLE READ` dùng snapshot; MySQL InnoDB còn phụ thuộc consistent read, locking read và index. `SERIALIZABLE` giảm anomaly nhưng có thể lock, abort hoặc serialization failure. Ưu tiên constraint, transaction ngắn, index đúng điều kiện, retry lỗi serialization có giới hạn. Gắn metric retry/lock wait để phát hiện contention.

## 7) Follow-up / pitfall

- Phantom read khác non-repeatable read: non-repeatable là cùng một dòng đổi giá trị; phantom là tập dòng đổi.
- `SELECT ... FOR UPDATE` không thay thế hiểu biết isolation; phạm vi lock phụ thuộc DB, index và query plan.
- Đừng dùng isolation cao cho luồng đọc không cần tính nhất quán mạnh.

## 8) 30 giây tóm tắt miệng

Phantom read xảy ra khi query cùng điều kiện hai lần trong một transaction nhưng số dòng đổi do transaction khác thêm hoặc xóa dữ liệu đã commit. Chọn snapshot hoặc isolation level theo semantics DB và yêu cầu nghiệp vụ. Với production, giữ transaction ngắn, index tốt, xử lý retry khi serializable abort và theo dõi lock wait.
