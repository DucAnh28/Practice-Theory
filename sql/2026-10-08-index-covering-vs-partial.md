# Covering Index vs Partial Index trong SQL

- **Date:** 2026-10-08
- **Topic:** sql
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

> Covering index là gì? Khi nào dùng Partial index? Trade-off của từng loại?

## 2) Giải thích như trẻ lên 3

Tưởng tượng bạn tìm số điện thoại trong danh bạ:
- **Covering index** = danh bạ đã có sẵn đủ thứ bạn cần (tên + số + địa chỉ) — không cần lật sang trang khác.
- **Partial index** = danh bạ chỉ in những người đang hoạt động — mỏng hơn, tìm nhanh hơn, nhưng không tìm được người đã nghỉ.

## 3) Giải thích đơn giản cho dev

**Covering index:** Index chứa đủ tất cả column mà query cần (`SELECT` + `WHERE` + `ORDER BY`). Database không cần đọc thêm row từ heap/table — gọi là *index-only scan*.

**Partial index:** Index chỉ đánh trên một tập con của row (theo `WHERE` condition). Nhỏ hơn, nhanh hơn cho query thường gặp nhất, nhưng vô dụng nếu query không match condition.

## 4) Ví dụ code đơn giản

```sql
-- Bảng orders
-- Covering index: query chỉ cần 3 cột này, không cần đụng vào heap
CREATE INDEX idx_orders_covering
  ON orders (user_id, status, created_at);

-- Query hưởng lợi: index-only scan
SELECT status, created_at
FROM orders
WHERE user_id = 42;

-- -------------------------------------------------------

-- Partial index: chỉ index đơn hàng chưa xử lý (status = 'PENDING')
-- 99% query là tìm đơn PENDING → index nhỏ, nhanh
CREATE INDEX idx_orders_pending
  ON orders (created_at)
  WHERE status = 'PENDING';

-- Query hưởng lợi
SELECT * FROM orders
WHERE status = 'PENDING'
  AND created_at < NOW() - INTERVAL '1 day';

-- Query KHÔNG hưởng lợi (status != 'PENDING')
SELECT * FROM orders WHERE status = 'COMPLETED';
```

## 5) Trả lời Middle+

- **Covering index** giúp query tránh *heap fetch* (table lookup) vì index đã có đủ data cần trả về. Dùng khi query lặp đi lặp lại trên một vài column cố định và performance là ưu tiên.
- **Partial index** giúp index nhỏ hơn khi chỉ một tập con row được query thường xuyên (ví dụ: `status = 'ACTIVE'`, `deleted_at IS NULL`). Tiết kiệm storage, write overhead thấp hơn.
- Kiểm tra bằng `EXPLAIN (ANALYZE, BUFFERS)` để xác nhận *Index Only Scan* hay *Index Scan*.

## 6) Trả lời Senior

**Covering index trade-off:**
- Mỗi column thêm vào index → write amplification tăng (INSERT/UPDATE chậm hơn).
- Index phình to → tốn RAM buffer pool, có thể đẩy hot data ra khỏi cache.
- `INCLUDE` clause (PostgreSQL 11+) cho phép thêm column vào index mà không tăng tree key size — giảm write cost so với đưa hết vào key:
  ```sql
  CREATE INDEX idx_orders_uid ON orders (user_id) INCLUDE (status, created_at);
  ```

**Partial index trade-off:**
- Planner chỉ dùng được khi query predicate khớp *chính xác* condition của index (PostgreSQL kiểm tra literal match, không suy luận).
- Dễ mồ côi: nếu logic nghiệp vụ thay đổi (`PENDING` → `QUEUED`), index trở nên vô dụng mà không có lỗi báo.
- Trong môi trường multi-tenant, partial index trên `tenant_id = X` không scale — mỗi tenant một index là anti-pattern.

**Production concern:**
- `CREATE INDEX CONCURRENTLY` để không lock bảng.
- Monitor `pg_stat_user_indexes` → `idx_scan = 0` sau vài tuần = index chết, drop đi.
- Bloat: chạy `REINDEX CONCURRENTLY` định kỳ hoặc dùng `pg_squeeze`.

## 7) Follow-up / pitfall

- *"Index có column A, B, C — query chỉ filter B thì có dùng được không?"* → Không (nếu không có A trong WHERE). B-tree index tuân theo **left-prefix rule**.
- *"Covering index thì SELECT \* có hưởng lợi không?"* → Không, vì `*` kéo thêm column nằm ngoài index.
- *"Partial index trên `deleted_at IS NULL` có dùng được khi query `WHERE deleted_at IS NULL AND id = 5`?"* → Có, PostgreSQL nhận ra predicate bao gồm condition của index.
- Pitfall phổ biến: tạo covering index nhưng quên `ORDER BY` column → query vẫn sort thêm, không phải index-only sort.

## 8) 30 giây tóm tắt miệng

> "Covering index giúp database trả kết quả thẳng từ index mà không cần đọc lại bảng — hiệu quả khi query lặp nhiều trên vài column cố định, nhưng tốn chi phí write và RAM. Partial index thì nhỏ gọn hơn vì chỉ đánh trên tập row thường dùng, ví dụ đơn hàng đang chờ xử lý — phù hợp khi 90% traffic chỉ quan tâm một trạng thái cụ thể. Senior cần lưu ý left-prefix rule, write amplification, và dùng `EXPLAIN ANALYZE` để xác nhận index-only scan thực sự xảy ra."
