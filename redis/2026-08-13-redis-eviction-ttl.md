# Redis Eviction & TTL Strategy

- **Date:** 2026-08-13
- **Topic:** redis
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Redis được dùng làm cache trong hệ thống, nhưng bộ nhớ có giới hạn. Khi Redis hết bộ nhớ, nó xử lý thế nào? TTL (time-to-live) trong Redis hoạt động ra sao? Hãy nêu các chiến lược eviction phổ biến và khi nào nên dùng từng cái.

## 2) Giải thích như trẻ lên 3

Redis giống một cái hộp chứa đồ. Khi hộp đầy, bạn phải bỏ một số cái cũ ra. Redis có cách khác nhau để quyết định cái nào bỏ (ví dụ: bỏ cái cũ nhất, hoặc cái dùng ít nhất). Ngoài ra, bạn có thể đặt hạn giờ cho từng cái đồ: sau giờ hết hạn, nó tự biến mất.

## 3) Giải thích đơn giản cho dev

**TTL (Time-To-Live):**
- Mỗi key trong Redis có thể có TTL (tính bằng giây/miligiây).
- Khi TTL hết, key bị xóa tự động (hoặc bị xóa khi access nếu TTL đã hết).
- Lệnh: `EXPIRE key seconds`, `PEXPIRE key milliseconds`, `TTL key`.

**Eviction Policy:**
Khi `maxmemory` đạt ngưỡng, Redis áp dụng policy để giải phóng bộ nhớ:
- **no-eviction:** không xóa, trả lỗi OOM (out of memory).
- **allkeys-lru:** xóa key ít dùng nhất trong toàn bộ DB.
- **volatile-lru:** xóa key ít dùng nhất **trong những key có TTL**.
- **allkeys-lfu:** xóa key ít được truy cập (tần suất thấp).
- **volatile-lfu:** xóa key ít được truy cập **trong những key có TTL**.
- **allkeys-random:** xóa ngẫu nhiên.
- **volatile-random:** xóa ngẫu nhiên **trong những key có TTL**.
- **volatile-ttl:** xóa key có TTL sắp hết sớm nhất.

## 4) Ví dụ code đơn giản

```java
// Set key với TTL 60 giây
jedis.setex("user:123:cache", 60, jsonData);

// Hoặc dùng pipelined
jedis.set("session:abc", data);
jedis.expire("session:abc", 1800); // 30 phút

// Kiểm tra TTL còn lại
long ttl = jedis.ttl("session:abc"); // trả về giây, -1 nếu không có TTL, -2 nếu key không tồn tại

// Config Redis maxmemory và policy (redis.conf)
// maxmemory 256mb
// maxmemory-policy allkeys-lru
```

## 5) Trả lời Middle+

Redis sử dụng **TTL** để tự động xóa key hết hạn, giúp giải phóng bộ nhớ mà không cần app can thiệp. Khi bộ nhớ đạt `maxmemory`, Redis áp dụng **eviction policy** để xóa key cũ/ít dùng.

Trong hầu hết trường hợp cache, dùng **allkeys-lru** hoặc **allkeys-lfu** đủ tốt:
- **LRU** (Least Recently Used): xóa cái không dùng lâu nhất → tốt cho workload có locality.
- **LFU** (Least Frequently Used): xóa cái dùng hiếm nhất → tốt cho hot/cold data phân rõ.

Ứng dụng nên:
1. Set TTL hợp lý cho session/cache (thường 5–30 phút).
2. Monitor `evicted_keys` metric để phát hiện nếu eviction quá nhiều (dấu hiệu cache quá nhỏ).
3. Cấu hình `maxmemory` phù hợp với khả năng server (để reserved cho OS).

## 6) Trả lời Senior

**Production Concerns:**

1. **TTL Precision & Lazy Deletion:**
   - Redis xóa key hết hạn theo 2 cách: (a) active expiry (background task 10Hz), (b) lazy expiry (khi access).
   - Nếu key hết TTL nhưng chưa được access, nó vẫn chiếm bộ nhớ. Monitor `expired_keys` để detect.
   - Trên high-throughput, TTL chính xác có thể bị trễ vài ms.

2. **Eviction Policy Trade-off:**
   - **allkeys-lru**: tốn CPU để track LRU, phù hợp general cache.
   - **allkeys-lfu**: tracking LFU tốn thêm memory, nhưng cho eviction decision tốt hơn cho workload skewed.
   - **volatile-***: nếu dùng, đảm bảo tất cả key quan trọng đều set TTL, nếu không sẽ bị bỏ lại (potential data loss).
   - **no-eviction**: an toàn nhất nhưng sẽ crash nếu hết bộ nhớ → trigger alert ngay.

3. **Cluster & Replication:**
   - Mỗi node trong cluster có riêng `maxmemory`, eviction policy không replica → slave có thể có data khác master.
   - TTL không guarantee thứ tự trong cluster: replica có thể still có expired key tạm thời.

4. **Monitoring & Alerting:**
   - Theo dõi `used_memory`, `used_memory_rss`, `evicted_keys`, `expired_keys` qua Redis INFO.
   - Alert nếu eviction rate cao (dấu hiệu cache undersized) hoặc `maxmemory_policy=no-eviction` + `used_memory` gần `maxmemory`.

5. **Optimization:**
   - Tune `maxmemory-samples` (default 5) để improve eviction decision accuracy (cao hơn = chính xác hơn nhưng tốn CPU).
   - Nếu dataset warm, dùng LFU + `allkeys` để tối ưu hit rate.
   - TTL nên phù hợp với pattern refresh: session thường 15–30min, nonce 5–10min, metric 1–5min.

## 7) Follow-up / Pitfall

**Follow-up:**
- "Làm sao để guarantee một key không bao giờ bị evict?"  
  → Set TTL = -1 (no expiry) + dùng `volatile-*` policy (chỉ evict những key có TTL). Hoặc upgrade maxmemory.

- "LRU vs LFU, khi nào dùng cái nào?"  
  → LRU nếu access pattern có temporal locality (mới dùng = sắp dùng lại). LFU nếu workload skewed (10% hot, 90% cold).

**Pitfall:**
- ❌ Không set TTL trên cache, dựa toàn bộ vào eviction → khó debug, metric confusing.
- ❌ Dùng `volatile-lru` mà không set TTL trên tất cả key → một số key mọi mãi không bị xóa → memory leak.
- ❌ Config `no-eviction` trong cache scenario → server crash khi hết bộ nhớ.
- ❌ Không monitor eviction rate → phát hiện cache undersized quá muộn.

## 8) 30 giây tóm tắt miệng

"Redis dùng TTL để tự động xóa key hết hạn, giải phóng bộ nhớ. Khi bộ nhớ đầy, eviction policy quyết định xóa cái nào. Phổ biến nhất là allkeys-lru (xóa ít dùng nhất) hoặc allkeys-lfu (xóa dùng hiếm nhất). Trong production, phải monitor eviction rate và set TTL phù hợp với use case (cache 5–30min, session 15–30min). Tránh `no-eviction` nếu là cache, và tránh `volatile-*` nếu không set TTL trên tất cả key."
