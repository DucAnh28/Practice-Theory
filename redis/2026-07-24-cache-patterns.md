# Redis Cache Patterns (Cache-Aside, Write-Through, Write-Behind)

- **Date:** 2026-07-24
- **Topic:** redis
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

> "Redis cache có 3 pattern phổ biến: Cache-Aside, Write-Through, Write-Behind. Hãy so sánh ưu/nhược điểm của từng pattern và khi nào nên dùng pattern nào trong production?"

---

## 2) Giải thích đơn giản (cho người không background)

Hãy tưởng tượng Redis như **bộ nhớ đệm (cache)** giống cái **ví tiền mặt** bạn mang theo:
- **Cache-Aside (Lười kiểm ví trước)**: Bạn cần mua cái gì → kiểm ví trước → có tiền thì mua ngay → không có thì ra ATM (DB) rút tiền → đưa vào ví → mua.
- **Write-Through (Viết đồng bộ)**: Bạn mua xong → đồng thời vừa trả tiền (DB) vừa cho tiền vào ví (Redis) cùng lúc.
- **Write-Behind (Viết lười / ghi sau)**: Bạn mua xong → chỉ cho tiền vào ví (Redis) → định kỳ sau đó mới về ATM (DB) nộp tiền một lần.

---

## 3) Giải thích kỹ thuật (cho dev)

| Pattern | Luồng ghi (Write) | Luồng đọc (Read) | Consistency | Latency Write | Phức tạp |
|---------|-------------------|------------------|-------------|---------------|----------|
| **Cache-Aside** (Lazy Loading) | App ghi DB → App tự invalidate/update cache | App đọc cache → miss → đọc DB → ghi cache → trả về | Eventual (có thể stale khi ghi DB mà quên invalidate) | Thấp (chỉ ghi DB) | Thấp |
| **Write-Through** | App ghi cache → cache ghi DB (sync) | App đọc cache → hit thì trả ngay | Strong (cache & DB luôn đồng bộ) | Cao (ghi 2 nơi) | Trung bình |
| **Write-Behind** (Write-Back) | App ghi cache → cache async flush về DB (batch/interval) | App đọc cache → hit thì trả ngay | Eventual (data loss risk nếu cache crash trước khi flush) | Thấp nhất (chỉ ghi cache) | Cao (cần queue/retry/ACK) |

**Key terms:**
- **Cache miss**: Cache không có data → phải đọc DB
- **Cache hit**: Cache có data → trả về ngay
- **Write-through**: Ghi xuyên qua cache xuống DB
- **Write-behind / Write-back**: Ghi cache,延迟 flush DB
- **Invalidation**: Xóa cache cũ khi data thay đổi

---

## 4) Ví dụ code đơn giản (Java Spring Boot + RedisTemplate)

```java
// Cache-Aside pattern
@Service
public class ProductService {
    @Autowired ProductRepository repo;
    @Autowired RedisTemplate<String, Product> redis;

    public Product getProduct(Long id) {
        String key = "product:" + id;
        Product cached = redis.opsForValue().get(key);
        if (cached != null) return cached;           // Cache hit
        Product fromDb = repo.findById(id).orElseThrow(); // Cache miss -> DB
        redis.opsForValue().set(key, fromDb, Duration.ofMinutes(10));
        return fromDb;
    }

    @Transactional
    public Product updateProduct(Product p) {
        Product saved = repo.save(p);                 // 1. Ghi DB
        redis.delete("product:" + p.getId());         // 2. Invalidate cache (lazy)
        return saved;
    }
}
```

```java
// Write-Through pattern (dùng RedisRepository hoặc custom WriteBehind)
@Repository
public class ProductWriteThroughRepo {
    @Autowired ProductRepository db;
    @Autowired RedisTemplate<String, Product> redis;

    public Product save(Product p) {
        Product saved = db.save(p);                   // 1. Ghi DB
        redis.opsForValue().set("product:" + p.getId(), saved); // 2. Ghi cache (sync)
        return saved;
    }
}
```

*Write-Behind thường dùng Redis Streams / RedisGears / custom queue để async flush.*

---

## 5) Trả lời Middle+

**Cache-Aside (phổ biến nhất):**
- Ưu: Đơn giản, ghi DB nhanh, cache chỉ là optimization.
- Nhược: **Cache stampede** (thundering herd) khi key hot expire → nhiều request cùng truy DB. Giải pháp: **mutex/lock** (Redis SETNX) hoặc **probabilistic early expiration**.
- Phù hợp: Read-heavy, tolerate stale data vài giây/phút.

**Write-Through:**
- Ưu: Data luôn consistent giữa cache và DB.
- Nhược: Write latency cao (gấp 2), cache size lớn nếu write nhiều key ít đọc.
- Phù hợp: Write-heavy nhưng read cũng nhiều, cần strong consistency (ví dụ: inventory, balance).

**Write-Behind:**
- Ưu: Write latency thấp nhất, giảm load DB (batch write).
- Nhược: **Data loss risk** nếu Redis crash trước khi flush; cần retry queue, idempotency, reconciliation job.
- Phù hợp: Write-heavy, high throughput, tolerate eventual consistency (logging, analytics, session).

---

## 6) Trả lời Senior

| Concern | Cache-Aside | Write-Through | Write-Behind |
|---------|-------------|---------------|--------------|
| **Consistency model** | Eventual (read-your-write không đảm bảo) | Strong (linearizable nếu DB + cache transactional) | Eventual (window = flush interval) |
| **Failure mode** | Stale read (acceptable) | Write latency spike / partial failure | **Data loss** nếu Redis down trước flush |
| **Recovery** | Tự heal khi TTL expire / next read | Idempotent write + retry | Cần **replay log / WAL** (Redis Stream / Kafka) + reconciliation job |
| **Observability** | Cache hit/miss rate, latency p99 | Write latency p99, cache size | Flush lag, queue depth, reconciliation drift |
| **Production tips** | - TTL + jitter (tránh thundering herd)<br>- Cache-aside + **read-through** wrapper<br>- Lua script cho atomic check-and-set | - Dùng **Redis transaction / Lua** cho atomic write-through<br>- Circuit breaker cho DB write | - **Redis Streams** làm write-behind log<br>- Idempotent key (UUID) + exactly-once semantics<br>- Scheduled reconciliation job (cron) so sánh Redis vs DB |

**Senior tips:**
- **Cache-Aside + Read-Through**: wrap logic vào `CacheManager` hoặc `@Cacheable` + `@CachePut`/`@CacheEvict` (Spring Cache abstraction).
- **Hybrid**: Hot key → Write-Through; Cold key → Cache-Aside.
- **Cache warming** cho hot key sau deploy/restart Redis.
- **Monitoring**: `redis-cli INFO stats` → `keyspace_hits`, `keyspace_misses`, `instantaneous_ops_per_sec`.

---

## 7) Follow-up / Pitfall

| Pitfall | Mô tả | Giải pháp |
|---------|-------|-----------|
| **Cache Stampede** | Key hot expire → hàng trăm request cùng truy DB | Mutex lock (SETNX), probabilistic early expiration, `random TTL + jitter` |
| **Cache Penetration** | Truy key không tồn tại → luôn miss DB | Cache null/empty object với TTL ngắn, Bloom filter |
| **Cache Avalanche** | Nhiều key expire cùng lúc (TTL giống nhau) | TTL random jitter (±10-20%) |
| **Write-Behind Data Loss** | Redis crash trước khi flush | Redis Stream / AOF + RDB + replication, reconciliation job |
| **Split-brain (Write-Through)** | Ghi cache thành công, ghi DB fail | 2PC / saga pattern, hoặc accept eventual + compensation job |

---

## 8) 30 giây tóm tắt miệng

> "Ba pattern cache Redis phổ biến: **Cache-Aside** (lười, đọc cache miss thì load DB, ghi DB xong invalidate cache) — đơn giản, phù hợp read-heavy, chấp nhận stale data; **Write-Through** (ghi cache và DB cùng lúc) — strong consistency, write chậm hơn, dùng cho inventory/balance; **Write-Behind** (ghi cache, async flush DB) — write nhanh nhất, giảm load DB, nhưng rủi ro data loss nếu Redis crash, cần write-behind log (Redis Stream) + reconciliation job. Production: Cache-Aside phổ biến nhất, thêm mutex chống stampede, TTL jitter chống avalanche, monitor hit/miss rate. Write-Behind cần idempotency key và replay log."