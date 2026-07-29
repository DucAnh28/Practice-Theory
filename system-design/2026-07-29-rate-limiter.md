# Rate Limiter (Token Bucket)

- **Date:** 2026-07-29
- **Topic:** system-design
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Bạn thiết kế API rate limiter cho hệ thống microservice. Rate limiter cần bảo vệ khỏi DDoS và cung cấp giao diện đơn giản cho các service gọi. Bạn sẽ làm gì?

## 2) Giải thích như trẻ lên 3

Giả sử có một tháp bánh mì. Mỗi giây thì tháp cho 5 viên bánh (tokens). Người ta có thể lấy bánh khi tháp có bánh. Nếu tháp rỗng, người ta phải chờ. Tháp có hạn lượng bánh lớn nhất (bucket size) là 10 viên.

## 3) Giải thích đơn giản cho dev

Rate limiter giảm kbps đầu vào bằng cách giảmthrottle request. Token Bucket vận hành với 2 thông số: `rate` (tokens/giây) và `capacity` (số token tối đa). Mỗi request tiêu tốn 1 token; nếu hết, request bị từ chối hoặc delay.

## 4) Ví dụ code đơn giản

```java
public class TokenBucketRateLimiter {
    private final long capacity;
    private final double tokensPerMillisecond;
    private double tokens;
    private long lastRefillTimestamp;

    public TokenBucketRateLimiter(long capacity, long refillPeriod, long tokensPerPeriod) {
        this.capacity = capacity;
        this.tokens = capacity;
        this.tokensPerMillisecond = (double) tokensPerPeriod / refillPeriod;
        this.lastRefillTimestamp = System.nanoTime();
    }

    private void refill() {
        long now = System.nanoTime();
        double elapsed = (now - lastRefillTimestamp) / 1_000_000.0; // ms
        tokens = Math.min(capacity, tokens + elapsed * tokensPerMillisecond);
        lastRefillTimestamp = now;
    }

    public boolean tryAcquire() {
        refill();
        if (tokens >= 1) {
            tokens -= 1;
            return true;
        }
        return false;
    }
}
```

## 5) Trả lời Middle+

- **Cơ chế**: Token Bucket hoặc Leaky Bucket.
- **Triển khai**: Redis + INCR/EXPIRE cho distributed rate limiting; cache local trong memory cho single instance.
- **Key**: rate limit key = service + endpoint + user-id hoặc IP.
- **Edge case**: clock drift, concurrent access, burst traffic.

## 6) Trả lời Senior

- **Distributed**: Redis cluster, Circuit breaker pattern, Consistent hashing cho key distribution.
- **Trade-off**: accuracy vs performance; sliding window log chính xác nhưng tốn bộ nhớ; sliding window counter gần đúng nhưng nhẹ.
- **Observability**: metrics (thống kê rate limit), alerts khi vượt threshold.
- **Fail-open/fail-safe**: Redis unavailable → có thể cho phép hoặc từ chối tùy mức độ coi chậm.

## 7) Follow-up / pitfall

- **Pitfall**: không xét case concurrent → race condition khi multiple instance đồng thời INCR.
- **Cách bypass**: client có thể đặt lại header `X-Forwarded-For` để dùng IP khác.
- **Memory leak**: sliding window log không bao giờ xoá entry → cần cleanup TTL.

## 8) 30 giây tóm tắt miệng

Rate limiter giúp bảo vệ API khỏi spam và DDoS. Cách phổ biến nhất là Token Bucket: mỗi giây sinh X token, mỗi request tiêu tốn 1 token nếu có. Triển khai distributed dùng Redis INCR/EXPIRE, trong memory dùng ScheduledExecutorService refill. Lưu ý về race condition, clock drift, và cần monitor metrics để phát hiện abusive users sớm.