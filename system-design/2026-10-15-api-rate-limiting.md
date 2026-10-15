# API Rate Limiting Strategies

- **Date:** 2026-10-15
- **Topic:** system-design
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Thiết kế một rate limiter cho API của bạn. Yêu cầu: support 10K RPS, fairness giữa client, không làm mất traffic khi limit vượt.

## 2) Giải thích như trẻ lên 3

Tưởng tượng cửa hàng có 1 quầy, mỗi khách được phục vụ 2 phút. Nếu có quá nhiều khách xếp hàng, quầy sẽ nói "xin chờ chút", không đuổi người ra.

## 3) Giải thích đơn giản cho dev

Rate limiter kiểm soát số request từ một client trong một khoảng thời gian. Nếu vượt limit, server trả 429 (Too Many Requests) hoặc queue request. Mục tiêu: bảo vệ backend khỏi overload và đảm bảo fair use.

## 4) Ví dụ code đơn giản

```java
// Token Bucket: trữ token, mỗi request tiêu tốn 1 token
public class RateLimiter {
    private final long capacity;
    private final double refillRate; // token/sec
    private double tokens;
    private long lastRefillTime;
    
    public synchronized boolean allowRequest() {
        refill();
        if (tokens >= 1) {
            tokens--;
            return true;
        }
        return false;
    }
    
    private void refill() {
        long now = System.nanoTime();
        double refilled = (now - lastRefillTime) / 1e9 * refillRate;
        tokens = Math.min(capacity, tokens + refilled);
        lastRefillTime = now;
    }
}
```

## 5) Trả lời Middle+

**Thuật toán phổ biến:**
- **Token Bucket:** refill token theo rate, request tiêu tốn 1 token. Dễ tuning, support burst.
- **Sliding Window:** đếm request trong khung thời gian. Đơn giản, nhưng hiệu năng kém với high throughput.
- **Leaky Bucket:** queue request, drain với fixed rate. Fair, nhưng latency cao.

**Triển khai distributed:**
- Dùng Redis `INCR` + TTL: `rate_limit:{client_id}` increment mỗi request, check counter.
- Token Bucket trong Redis: store `{tokens, last_refill}`, script Lua atomic refill + consume.

**Per-client tracking:**
- By IP, API key, user ID — tùy case.
- Khởi tạo bucket per client khi first request → cleanup sau timeout.

## 6) Trả lời Senior

**Production concerns:**

1. **Distributed clock skew:** trong multi-node, clock khác nhau làm Token Bucket sai. Dùng Redis server time hoặc NTP.

2. **Retry storm:** client retry aggressive khi 429 → làm worse. Implement exponential backoff + `Retry-After` header.

3. **Fairness vs burst:** Token Bucket cho phép burst (good cho spike), nhưng có thể starve slow client. Weighted rate limit (per-tier customer) cần fair queue.

4. **Scale to 10K RPS:** single Redis instance bottleneck. Giải pháp:
   - Shard by client ID hash → multiple Redis nodes.
   - Local token bucket per node + sync periodically (trade accuracy for latency).
   - Consider counter-based (simpler scale).

5. **Observability:** log 429, track rejection rate, client tier. Alert khi rejection > threshold.

6. **Graceful degrade:** nếu Redis down, fallback local rate limit hoặc tạm cho pass (xác định acceptable risk).

## 7) Follow-up / pitfall

**Common mistakes:**
- Không account clock skew → Token Bucket drift.
- Quá strict limit → legitimate client bị block.
- No grace period → retry loop gây spike tệ hơn.

**Follow-up:**
- Nếu need adaptive limit, dùng gì? (tính based on current load, dynamic adjustment)
- Throttling vs rate limiting khác nhau sao? (throttle = slow down, limit = reject)

## 8) 30 giây tóm tắt miệng

"Rate limiter kiểm soát request từ client, dùng Token Bucket: mỗi client có bucket token, hết token thì reject. Distributed thì Redis store bucket state, shard by client ID. Main risk: clock skew, retry storm — cần Retry-After header và exponential backoff."
