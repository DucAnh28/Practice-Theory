# Cache-Aside và cache stampede

- **Date:** 2026-09-08
- **Topic:** redis
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Cache-Aside là gì? Làm sao tránh cache stampede khi nhiều request cùng miss một key hot?

## 2) Giải thích như trẻ lên 3

Redis như tủ đồ gần cửa. Không có món cần tìm thì mọi người cùng chạy vào kho lấy; kho sẽ quá tải. Chỉ một người vào kho, người khác chờ món đó được đặt lại vào tủ.

## 3) Giải thích đơn giản cho dev

Cache-Aside: app đọc Redis trước; miss thì đọc DB, ghi Redis với TTL rồi trả kết quả. Khi key hết hạn cùng lúc, nhiều request có thể cùng đọc DB: đó là cache stampede.

## 4) Ví dụ code đơn giản

```java
var cached = redis.get(key);
if (cached != null) return decode(cached);

if (locks.tryLock("lock:" + key, Duration.ofSeconds(2))) {
  try {
    cached = redis.get(key); // double-check sau khi lấy lock
    if (cached != null) return decode(cached);
    var user = userRepository.findById(id).orElseThrow();
    redis.setex(key, ttlWithJitter(), encode(user));
    return user;
  } finally {
    locks.unlock("lock:" + key);
  }
}
return waitBrieflyThenReadCache(key);
```

## 5) Trả lời Middle+

Dùng Cache-Aside cho dữ liệu đọc nhiều, chấp nhận stale ngắn. Thêm TTL và jitter để key không hết hạn đồng loạt. Với key hot, dùng distributed lock hoặc single-flight; lock holder đọc DB rồi fill cache, request khác đọc lại cache.

## 6) Trả lời Senior

Lock phải có TTL, token ownership khi unlock, và timeout ngắn để tránh treo request. Cân nhắc logical expiry + background refresh cho dữ liệu hot, stale-while-revalidate để giữ latency. Theo dõi cache hit rate, DB QPS lúc expiry, lock contention, refresh failure. Không cache lỗi DB lâu; cần fallback rõ ràng khi Redis hoặc DB lỗi.

## 7) Follow-up / pitfall

- Lock không có TTL: process chết giữ lock mãi.
- Unlock không kiểm tra owner: xóa lock request khác.
- TTL cố định cho mọi key hot: stampede lặp lại.
- Cache null ngắn hạn nếu cần chặn cache penetration.

## 8) 30 giây tóm tắt miệng

Cache-Aside đọc cache trước, miss mới đọc DB rồi ghi lại cache. Cache stampede xảy ra khi nhiều request cùng miss key hot. Em dùng TTL có jitter, double-check sau distributed lock hoặc single-flight. Production cần lock TTL và ownership, fallback khi Redis lỗi, cùng metrics hit rate và DB spike.
