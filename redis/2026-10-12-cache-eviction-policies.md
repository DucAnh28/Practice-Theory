# Redis Cache Eviction Policies

- **Date:** 2026-10-12
- **Topic:** redis
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

"Khi Redis hết memory, nó xử lý như thế nào? Các eviction policy trong Redis là gì và khi nào dùng từng loại?"

## 2) Giải thích như trẻ lên 3

Tủ đồ chơi đầy rồi, muốn bỏ đồ mới vào phải vứt đồ cũ. Có nhiều cách vứt:
- Vứt đồ nào ít chơi nhất
- Vứt đồ nào lâu không chơi nhất
- Vứt đồ có nhãn "được phép vứt"
- Không vứt, báo "đầy rồi"

## 3) Giải thích đơn giản cho dev

Redis là in-memory database, có giới hạn RAM. Khi đầy, Redis dùng **eviction policy** để quyết định key nào bị xóa tự động.

Config: `maxmemory` (giới hạn RAM) và `maxmemory-policy` (thuật toán xóa).

## 4) Ví dụ code đơn giản

```yaml
# redis.conf
maxmemory 2gb
maxmemory-policy allkeys-lru
```

```java
// Spring Boot application.yml
spring:
  redis:
    host: localhost
    port: 6379
    # Redis server config vẫn cần set maxmemory-policy

// Dùng Redis template bình thường
@Autowired
private RedisTemplate<String, Object> redisTemplate;

public void cacheUser(String userId, User user) {
    redisTemplate.opsForValue().set("user:" + userId, user, 1, TimeUnit.HOURS);
    // Nếu Redis đầy, policy sẽ tự động xóa key cũ
}
```

## 5) Trả lời Middle+

Redis có 8 eviction policies chính:

**1. noeviction (default):**
- Không xóa gì, trả error khi đầy
- Dùng khi: không chấp nhận mất data

**2. allkeys-lru:**
- Xóa key ít dùng nhất (Least Recently Used) trong tất cả keys
- Dùng khi: cache chung, không phân biệt key nào quan trọng

**3. allkeys-lfu:**
- Xóa key ít được truy cập nhất (Least Frequently Used)
- Dùng khi: muốn giữ key "hot" thật sự

**4. allkeys-random:**
- Xóa key ngẫu nhiên
- Dùng khi: tất cả key đều tạm thời, không ưu tiên

**5. volatile-lru:**
- Xóa key LRU chỉ trong các key có TTL (expire set)
- Dùng khi: một phần key là cache (có TTL), một phần là persistent data (không TTL)

**6. volatile-lfu:**
- Xóa key LFU chỉ trong các key có TTL

**7. volatile-random:**
- Xóa ngẫu nhiên trong các key có TTL

**8. volatile-ttl:**
- Xóa key có TTL ngắn nhất trước
- Dùng khi: key sắp hết hạn nên ưu tiên xóa

## 6) Trả lời Senior

**Trade-off:**

| Policy | Ưu điểm | Nhược điểm | Production use |
|--------|---------|------------|----------------|
| **allkeys-lru** | Hiệu quả cho cache tổng quát | Approximate LRU (Redis dùng sampling) | Cache layer phổ biến nhất |
| **allkeys-lfu** | Giữ key thực sự hot | Cần Redis 4.0+, slow start problem | Content delivery, hot data |
| **volatile-lru** | Bảo vệ data không TTL | Nếu không key nào có TTL → error | Mixed workload (cache + session) |
| **noeviction** | An toàn, không mất data | OOM error khi đầy | Queue, critical session |

**Production concerns:**

1. **Memory fragmentation:** Redis dùng jemalloc, actual memory > `INFO memory used_memory`. Monitor `used_memory_rss` và `mem_fragmentation_ratio`.

2. **LRU sampling:** Redis không track access time cho tất cả keys (overhead quá lớn). Nó sample `maxmemory-samples` keys (default 5) → trade-off giữa accuracy và performance. Tăng lên 10 cho accuracy cao hơn.

3. **TTL vs eviction:** Key có TTL vẫn chiếm memory cho đến khi:
   - Lazy deletion (access key đã expire)
   - Active deletion (Redis quét 10 keys/giây)
   - Eviction (khi đầy memory)

4. **Hot key problem:** Nếu 1 key chiếm nhiều memory (big hash/list), eviction không xóa nó → OOM. Giải pháp: sharding, compression.

## 7) Follow-up / pitfall

**Q: Nếu chọn `volatile-lru` nhưng không key nào có TTL thì sao?**
A: Redis trả error `OOM command not allowed when used memory > 'maxmemory'` giống `noeviction`.

**Q: LRU của Redis có chính xác không?**
A: Không. Redis dùng approximate LRU (sample random keys). Không lưu linked list toàn bộ keys → tiết kiệm memory và CPU.

**Q: Khi nào dùng `allkeys-lru` vs `volatile-lru`?**
A: 
- `allkeys-lru`: pure cache, tất cả data đều có thể mất
- `volatile-lru`: mixed (cache có TTL + session/data persistent không TTL)

**Pitfall:**

1. **Quên set `maxmemory`:** Redis mặc định không giới hạn → dùng hết RAM host → OOM killer.

2. **Dùng `noeviction` cho cache:** Khi đầy Redis trả error, app crash nếu không handle. Cache nên dùng LRU/LFU.

3. **Big key không evict:** Key 100MB trong pool 1GB → Redis không xóa được key khác → OOM. Monitor `redis-cli --bigkeys`.

4. **Không monitor eviction:** Track `evicted_keys` metric. Nếu tăng đột ngột → memory không đủ hoặc TTL quá dài.

## 8) 30 giây tóm tắt miệng

"Redis dùng eviction policy khi hết memory. Phổ biến nhất là `allkeys-lru` cho pure cache — xóa key ít dùng nhất. Nếu có cả session data (không được xóa) và cache, dùng `volatile-lru` — chỉ xóa key có TTL. Redis dùng approximate LRU qua sampling 5 keys, không phải true LRU. Production cần monitor `evicted_keys` và `used_memory_rss`, set `maxmemory` hợp lý, tránh big key."
