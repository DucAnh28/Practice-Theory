# Circuit Breaker

- **Date:** 2026-07-27
- **Topic:** microservices
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Circuit Breaker là gì? Vì sao cần nó khi service A gọi service B? Ba trạng thái CLOSED / OPEN / HALF_OPEN hoạt động ra sao?

## 2) Giải thích như trẻ lên 3

Tưởng tượng nhà bạn có cầu dao điện. Khi đoản mạch (cháy), cầu dao tự ngắt → nhà tối nhưng cả hệ thống không cháy. Vài phút sau bạn bật thử lại: nếu vẫn lỗi thì ngắt tiếp, nếu ổn thì đóng điện bình thường.

Service gọi service cũng vậy: khi B liên tục lỗi, A phải "ngắt cầu dao" → trả lỗi nhanh thay vì treo 30s, đỡ lây sập.

## 3) Giải thích đơn giản cho dev

Circuit Breaker (CB) là wrapper quanh call tới dependency. Nó đếm số lỗi/thành công trong một cửa sổ thời gian, khi vượt ngưỡng → OPEN (fail-fast). Sau `waitDurationInOpenState` → HALF_OPEN, cho đi qua vài request thử. Nếu OK → CLOSED, nếu vẫn fail → OPEN lại.

| State | Hành vi | Chuyển tiếp |
|-------|---------|-------------|
| CLOSED | Cho request đi bình thường, đếm lỗi | Lỗi > ngưỡng → OPEN |
| OPEN | Reject ngay, không gọi xuống | Hết `waitDuration` → HALF_OPEN |
| HALF_OPEN | Cho N request thử | Đủ OK → CLOSED; còn lỗi → OPEN |

Tham số hay chỉnh:

- `failureRateThreshold` (vd 50%)
- `slowCallDurationThreshold` + `slowCallRateThreshold` (bắt cả call chậm, không chỉ lỗi)
- `slidingWindowSize` (vd 100 call gần nhất)
- `permittedNumberOfCallsInHalfOpenState` (vd 10)

## 4) Ví dụ code đơn giản

Resilience4j — Spring Boot:

```java
CircuitBreaker cb = CircuitBreaker.of("orderService", CircuitBreakerConfig.custom()
    .failureRateThreshold(50)
    .slowCallRateThreshold(50)
    .slowCallDurationThreshold(Duration.ofSeconds(2))
    .slidingWindowSize(100)
    .waitDurationInOpenState(Duration.ofSeconds(10))
    .permittedNumberOfCallsInHalfOpenState(5)
    .build());

Supplier<Response> decorated = CircuitBreaker.decorateSupplier(cb, () -> callDownstream());
try {
    return decorated.get();
} catch (CallNotPermittedException e) {
    return fallback();          // cache / default / queue
}
```

Hoặc annotation:

```java
@CircuitBreaker(name = "orderService", fallbackMethod = "fallback")
public Response placeOrder(OrderRequest req) { return callDownstream(req); }
```

## 5) Trả lời Middle+

"CB là pattern chống cascading failure: khi dependency lỗi liên tục, nó tạm thời chặn call để hệ thống không bị treo/đói thread. Có 3 trạng thái CLOSED → OPEN khi lỗi vượt ngưỡng, OPEN fail-fast và sau khoảng thời gian chờ thì sang HALF_OPEN để thử. Tôi hay dùng Resilience4j: config failure rate, slow call, sliding window, có fallback trả về cache hoặc default. Khi CB mở thì trả lỗi nhanh cho user thay vì chờ timeout."

## 6) Trả lời Senior

"CB chỉ là một lớp trong hệ chống cascading failure, cần kết hợp:

- **Bulkhead** (tách thread pool / connection pool) để một dependency lỗi không kéo hết resource của service khác.
- **Timeout + Retry có giới hạn** (retry không nên quá 2 lần, không retry với idempotent-không-rõ).
- **Fallback** có chiến lược: cache cũ, default value, queue xử lý sau, hoặc degraded UX.
- **Observability**: export metric state (closed/open/half_open), số call bị reject, p99 latency. Alert khi state = OPEN quá lâu → dependency thật sự chết.
- **Failure mode thật**: nếu dependency đang chết thật, OPEN bảo vệ đúng. Nhưng nếu config ngưỡng quá nhạy → false positive chặn traffic; quá lỏng → không bảo vệ được. Cần test bằng chaos test (KillSwitch/Toxiproxy).
- **Half-open sai cũng nguy hiểm**: cho quá nhiều call thử lúc dependency vừa hồi → re-cascade. Dùng `permittedNumberOfCallsInHalfOpenState` nhỏ.
- **Phân biệt lỗi**: 4xx của business thường không nên tính vào failure rate (chỉ 5xx/timeout/IO)."

## 7) Follow-up / pitfall

- CB có chặn được bug do mình không? Không — chỉ chặn lỗi từ dependency.
- Retry với timeout dài + không CB → vẫn treo thread.
- CB mở mà không có fallback → user thấy 500 liên tục. Phải luôn có fallback.
- Stateful CB trong cluster: mỗi instance đếm riêng → tổng số lỗi thật cao hơn nhìn được. Với critical service nên aggregate qua metrics.
- Khi service phụ thuộc nhiều CB cùng loại → cần đặt tên + metric riêng để debug.
- `@CircuitBreaker` không bắt lỗi của `CompletableFuture` chain trừ khi decorate đúng chỗ.

## 8) 30 giây tóm tắt miệng

"Circuit Breaker là cầu dao bảo vệ service khỏi dependency đang lỗi: đếm tỉ lệ lỗi trong sliding window, vượt ngưỡng thì OPEN fail-fast, sau thời gian chờ thử lại ở HALF_OPEN rồi về CLOSED. Tôi dùng Resilience4j kết hợp timeout, retry giới hạn, bulkhead và fallback có chiến lược, kèm metric theo dõi state để alert khi cần."