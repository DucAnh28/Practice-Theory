# Retry Strategy: Exponential Backoff, Jitter & DLQ

- **Date:** 2026-11-12
- **Topic:** messaging
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Khi consumer xử lý message bị lỗi, nên retry thế nào? Khác biệt giữa **fixed delay**, **exponential backoff** và **jitter**? Khi nào message nên đẩy sang **Dead Letter Queue (DLQ)**? Retry ở tầng nào — producer, broker hay consumer?

## 2) Giải thích như trẻ lên 3

- **Retry**: làm lại việc bị fail. Giống gọi điện lại cho khách khi họ không nghe.
- **Fixed delay**: chờ 5 giây rồi gọi lại, lần nào cũng 5 giây.
- **Exponential backoff**: lần 1 chờ 1s, lần 2 chờ 2s, lần 3 chờ 4s, lần 4 chờ 8s. Càng thử nhiều càng chờ lâu hơn.
- **Jitter**: thêm một chút ngẫu nhiên vào thời gian chờ, ví dụ 8s thành 6–10s. Như vậy nhiều máy cùng chờ sẽ không "đâm nhau" vào server.
- **DLQ**: một cái kho riêng chứa những message retry mãi không được. Giống "hộp thư lỗi" — không xóa đi, giữ lại để người xem xét.

## 3) Giải thích đơn giản cho dev

| Chiến lược | Công thức | Khi nào dùng |
|---|---|---|
| **Fixed delay** | `delay = 1s` | Lỗi tạm thời, hệ thống bên dưới chỉ hồi phục trong vài giây |
| **Exponential backoff** | `delay = base * 2^attempt` | Lỗi do hạ tầng chập chờn (DB chậm, service đang restart) |
| **Exponential backoff + jitter** | `delay = random(0, base * 2^attempt)` | Nhiều consumer/instance retry đồng thời → tránh thundering herd |
| **Full jitter** (AWS) | `delay = random(0, min(cap, base * 2^attempt))` | Backpressure mạnh, cần dàn đều tải tối đa |

- **Jitter là bắt buộc khi retry ở quy mô lớn**: không có jitter, tất cả consumer fail cùng lúc sẽ retry cùng nhịp → server vừa hồi lại lại bị "đánh" ngay.
- **Cap (max delay)**: không cho delay vô hạn, phải có trần (ví dụ 60s) và **max attempts** (ví dụ 5). Hết số lần → vào DLQ.
- **Retry ở tầng nào**:
  - **Producer retry**: dùng cho lỗi mạng lúc gửi (kafka producer `retries`, `delivery.timeout.ms`). Không áp dụng cho lỗi nghiệp vụ (order không hợp lệ).
  - **Broker/consumer retry**: dùng cho lỗi khi **xử lý** message. RabbitMQ hỗ trợ **delayed retry** qua DLX + TTL queue; Kafka thì dùng **retry topic** (vd `orders-retry` với delay) hoặc Spring Retry trong consumer.
  - **Quy tắc**: retry lỗi **transient** (timeout, connection reset, lock timeout), **không retry** lỗi **permanent** (validation fail, 400, business rule) — vì retry vô hạn một lỗi không bao giờ hết.
- **Jitter implementation**: Spring Retry dùng `Random` theo `BackOffPolicy`; Resilience4j có `intervalFn` với jitter.

## 4) Ví dụ code đơn giản

**Retry với exponential backoff + jitter (Spring Retry):**

```java
// application.yml
spring:
  rabbitmq:
    listener:
      simple:
        retry:
          enabled: true
          initial-interval: 1s
          max-attempts: 5
          multiplier: 2.0
          max-interval: 30s
          # jitter: mỗi lần chờ ngẫu nhiên trong [0, interval]
```

**Tự implement backoff + jitter (không phụ thuộc thư viện):**

```java
private static final long BASE_MS = 1_000;
private static final long CAP_MS  = 30_000;

long delayFor(int attempt, Random rnd) {
    long exp = BASE_MS * (1L << Math.min(attempt, 20)); // tránh overflow
    long cap = Math.min(exp, CAP_MS);
    return rnd.nextLong(cap + 1); // full jitter: 0..cap
}
```

**Consumer: retry rồi đẩy sang DLQ khi hết số lần (Spring AMQP):**

```java
@RabbitListener(queues = "order-queue")
public void handle(Order order) {
    try {
        orderService.process(order);
    } catch (TransientDataAccessException ex) {
        throw new AmqpRejectAndDontRequeueException(ex); // đẩy thẳng sang DLQ
    }
}
```

**RabbitMQ: hàng đợi retry có delay bằng TTL + dead-letter trở lại queue chính:**

```json
{
  "arguments": {
    "x-dead-letter-exchange": "",
    "x-dead-letter-routing-key": "order-queue"
  }
}
```

```java
// Retry queue: TTL = delay mong muốn, hết TTL thì tự đẩy về order-queue
@Bean
Queue orderRetryQueue() {
    return QueueBuilder.durable("order-queue.retry")
        .withArgument("x-message-ttl", 10_000)              // chờ 10s
        .withArgument("x-dead-letter-exchange", "")
        .withArgument("x-dead-letter-routing-key", "order-queue")
        .build();
}
```

**Kafka: retry topic có timestamp để delay (consumer tự poll topic retry khi đủ điều kiện):**

```java
@KafkaListener(topics = "orders", groupId = "order-worker")
public void handle(Order order) {
    try {
        orderService.process(order);
    } catch (TransientException ex) {
        retryTemplate.send("orders-retry",
            new RecordHeader("retry-count", countHeader)); // tăng count, backoff theo count
        // KHÔNG commit offset hiện tại → sẽ redeliver, nhưng cần cẩn thận
    }
}
```

## 5) Trả lời Middle+

- Phân biệt rõ: lỗi **transient** thì retry, lỗi **permanent** thì reject.
- Dùng **exponential backoff + jitter**, có **max attempts** và **max delay**. Retry không giới hạn sẽ giết chết hệ thống lúc sự cố (retry storm).
- Retry ở **consumer** khi lỗi xảy ra lúc xử lý; retry ở **producer** khi lỗi mạng lúc gửi.
- Hết số lần → **dead-letter** message, không ack/drop im lặng. Tin lỗi phải còn để điều tra.
- Consumer handler phải **idempotent**, vì retry + at-least-once nghĩa là message có thể xử lý nhiều lần.

## 6) Trả lời Senior

- **Retry storm / thundering herd** là rủi ro lớn nhất. Khi hạ tầng lỗi, tất cả consumer retry cùng lúc → tải nhân lên đúng lúc server yếu nhất. Giải pháp: **jitter + cap + circuit breaker** (ngắt retry khi tỉ lệ lỗi tăng) để hệ thống có thở.
- **Retry budget** thay vì retry vô hạn: dùng token bucket — mỗi request được phép retry tối đa X lần trong window Y. Hết budget → fail fast, đẩy sang DLQ. Hạn chế tổng tải retry trên toàn hệ thống.
- **Tách DLQ theo loại lỗi**: lỗi business → DLQ để xử lý thủ công; lỗi hạ tầng → retry dài hơn rồi mới DLQ. DLQ chung một đống sẽ bị "nhiễu" và không ai đọc.
- **Quan sát (observability)**: metric theo từng loại lỗi — `retry_count`, `dlq_depth`, `dlq_oldest_message_age`, `consumer_lag`. Alert khi `dlq_depth` tăng đột biến hoặc message nằm DLQ quá lâu. Cần **alerting** chứ không chỉ dashboard.
- **DLQ không phải nơi để xóa rác**: cần quy trình **replay** — công cụ đọc DLQ, xử lý lại (khi root cause đã fix) với idempotency key để không tạo side-effect trùng. Giữ DLQ **giới hạn TTL** (vài ngày) để không phình storage.
- **Observability cho từng message**: gắn `correlation_id` / `trace_id` vào header để trace từ producer → retry → DLQ. Không có id thì điều tra kiểu mù.
- **Tránh retry ở cả hai đầu**: producer retry + consumer retry cùng lúc → nhân bậc cấp số lần thử (3 × 3 = 9 lần). Chọn một tầng làm chủ, tầng còn lại chỉ log + cảnh báo.
- **Chi phí trễ (latency budget)**: retry ảnh hưởng p99 latency. Với request đồng bộ của user, retry hữu ích; với job nền, nên lùi job sang queue retry có delay dài hơn thay vì giữ worker bận.
- **Ordering**: retry bằng message riêng có thể phá thứ tự (message mới đã xử lý xong trước message cũ đang retry). Giữ ordering bằng cách retry **trong cùng partition** với cùng key, hoặc dùng thứ tự ưu tiên.

## 7) Follow-up / pitfall

- **Pitfall 1**: retry **lỗi validation/business** → tạo "vòng lặp vô hạn" tốn tài nguyên, đồng thời che mất bug thật. Sửa: classify exception trước khi retry.
- **Pitfall 2**: retry **không có jitter** → mọi consumer đồng loạt retry, tạo retry storm giữa lúc hồi phục.
- **Pitfall 3**: retry **không giới hạn số lần** → tin "độc" (poison message) nằm mãi, consumer bị kẹt, DLQ phình to.
- **Pitfall 4**: **ack trước khi xử lý xong** (auto-ack) → mất message khi crash giữa chừng, retry không cứu được.
- **Pitfall 5**: **ack nhưng xử lý lỗi** → mất message, không DLQ, không ai biết. Đây là bug nghiêm trọng nhất.
- **Pitfall 6**: message vào DLQ **không có metadata** (lý do fail, số lần retry, stacktrace) → không debug được, phải đoán.
- **Pitfall 7**: **replay DLQ không idempotent** → xử lý lại gây trùng đơn hàng / trùng thanh toán.
- **Pitfall 8**: dùng **fixed delay** ở quy mô lớn → dồn tải, tạo đỉnh đột ngột.
- **Follow-up**: "Retry ở producer hay consumer?" → Lỗi mạng lúc gửi → producer; lỗi khi xử lý message → consumer. Không retry ở cả hai cùng lúc.
- **Follow-up**: "DLQ khác retry queue thế nào?" → Retry queue là bước trung gian có delay rồi quay lại queue chính; DLQ là điểm dừng cuối, cần người xử lý.
- **Follow-up**: "Có nên retry không giới hạn cho lỗi hạ tầng?" → Không. Dùng retry budget + circuit breaker để hệ thống suy sụp thì còn tệ hơn là không retry.

## 8) 30 giây tóm tắt miệng

"Retry thì phải có backoff: càng thử nhiều càng chờ lâu hơn, và luôn kèm jitter để nhiều consumer không retry cùng nhịp — không jitter thì lúc hệ thống hồi phục sẽ bị retry storm đánh sập lần nữa. Retry chỉ dành cho lỗi transient như timeout, còn lỗi validation thì reject thẳng. Luôn giới hạn số lần và delay tối đa, hết lượt thì đẩy sang DLQ kèm lý do lỗi, thay vì drop im lặng. Ở tầng Senior thì thêm retry budget và circuit breaker để kiểm soát tổng tải retry, và có quy trình replay DLQ chạy idempotent."
