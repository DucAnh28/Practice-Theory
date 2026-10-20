# Message Queue: At-Least-Once vs Exactly-Once

- **Date:** 2026-10-20
- **Topic:** messaging
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

"Khi dùng message queue (RabbitMQ, Kafka, SQS...), khác gì giữa **at-least-once** và **exactly-once**? Khi nào dùng cái nào? Làm sao đảm bảo exactly-once trong hệ thống thực tế?"

## 2) Giải thích như trẻ lên 3

- **At-least-once**: Như gửi thư có thể bị trùng. Người gửi không chắc đã nhận chưa nên gửi lại, người nhận có thể nhận 2–3 lần cùng lá thư.
- **Exactly-once**: Bưu điện đảm bảo mỗi thư chỉ đến 1 lần duy nhất. Khó hơn, đắt hơn, nhưng không lo trùng.

## 3) Giải thích đơn giản cho dev

### At-Least-Once Delivery

- Message được gửi **ít nhất 1 lần**; có thể bị **duplicate** nếu:
  - Consumer nhận xong nhưng chưa kịp ack → crash → queue gửi lại.
  - Network retry.
- Đơn giản, hiệu năng tốt, phổ biến nhất.
- Consumer phải **idempotent** (xử lý 2 lần cùng message không gây lỗi).

### Exactly-Once Delivery

- Message được xử lý **đúng 1 lần**, không duplicate.
- Cần cơ chế đặc biệt:
  - **Kafka**: transactional producer + consumer đọc committed records.
  - **RabbitMQ**: không có exactly-once native, phải tự build (deduplication table + distributed transaction).
  - **AWS SQS**: FIFO queue + deduplication ID.
- Phức tạp, overhead cao, không phải lúc nào cũng cần.

## 4) Ví dụ code đơn giản

### At-Least-Once với Spring Kafka

```java
@KafkaListener(topics = "orders")
public void consume(OrderEvent event) {
    // Có thể nhận duplicate → phải idempotent
    if (orderRepo.existsById(event.getOrderId())) {
        log.info("Order {} already processed, skip", event.getOrderId());
        return;
    }
    orderRepo.save(new Order(event.getOrderId(), event.getAmount()));
}
```

### Exactly-Once với Kafka Transactions

```java
// Producer
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "tx-producer-1");

producer.initTransactions();
producer.beginTransaction();
producer.send(new ProducerRecord<>("orders", key, value));
producer.commitTransaction(); // Atomic write

// Consumer
props.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed");
// Chỉ đọc message đã commit
```

## 5) Trả lời Middle+

"**At-least-once** là delivery semantic phổ biến: message được gửi ít nhất 1 lần, có thể duplicate nếu consumer chưa ack kịp hoặc retry. Consumer cần idempotent—ví dụ check trước khi insert, dùng unique constraint DB, hoặc lưu message ID đã xử lý.

**Exactly-once** đảm bảo message chỉ được xử lý đúng 1 lần. Kafka hỗ trợ qua transactional producer và `read_committed` consumer. RabbitMQ không có native, phải tự build với deduplication table hoặc distributed lock. Exactly-once tốn performance hơn, chỉ dùng khi nghiệp vụ yêu cầu nghiêm ngặt (financial transaction, billing)."

## 6) Trả lời Senior

"Trong production, **exactly-once end-to-end rất khó**: cần đồng bộ giữa message broker, consumer logic, và database. Kafka transactional chỉ đảm bảo exactly-once trong Kafka ecosystem; nếu consumer ghi vào external DB, vẫn phải dùng **distributed transaction** (2PC) hoặc **outbox pattern** để đảm bảo atomicity.

Thực tế, hầu hết hệ thống dùng **at-least-once + idempotency**:
- Insert với unique key/constraint DB.
- Lưu `processedMessageIds` cache (Redis) hoặc DB dedupe table với TTL.
- Dùng business key thay vì technical message ID.

**Trade-off**:
- At-least-once: đơn giản, nhanh, cần idempotent consumer.
- Exactly-once: phức tạp, chậm hơn, tốn cost (transaction log, coordinator overhead). Chỉ dùng khi duplicate gây vấn đề nghiêm trọng (charge tiền 2 lần, duplicate inventory reservation).

**Pitfall**: Tuyên bố exactly-once nhưng quên check xem consumer logic có atomic với ack không (consume → process → DB commit → ack phải trong 1 transaction scope)."

## 7) Follow-up / pitfall

- **"Kafka exactly-once có đảm bảo consumer của bạn không duplicate không?"**  
  → Không, chỉ đảm bảo trong Kafka. Consumer phải đọc `read_committed` và xử lý transactional nếu ghi ra ngoài.

- **"At-least-once + idempotency có bằng exactly-once không?"**  
  → Về mặt kết quả cuối cùng thì có (functional equivalent), nhưng log/metrics có thể thấy duplicate event.

- **"Nếu consumer xử lý xong nhưng chưa ack được thì sao?"**  
  → Message sẽ redelivery → duplicate. Phải design idempotent hoặc dùng exactly-once semantic.

- **"RabbitMQ có exactly-once không?"**  
  → Không có native. Phải tự implement deduplication (lưu message ID đã xử lý).

- **Lỗi hay gặp**:
  - Consumer xử lý message 2 lần nhưng không idempotent → duplicate data.
  - Dùng auto-commit offset Kafka trước khi xử lý xong → mất message nếu crash.
  - Quên set `isolation.level=read_committed` khi dùng Kafka transactional.

## 8) 30 giây tóm tắt miệng

"At-least-once là message gửi ít nhất 1 lần, có thể duplicate, consumer phải idempotent. Exactly-once đảm bảo chỉ xử lý đúng 1 lần, phức tạp và tốn cost hơn—Kafka hỗ trợ qua transaction, RabbitMQ phải tự build. Thực tế dùng at-least-once + idempotency cho đơn giản; exactly-once chỉ cần khi duplicate gây hậu quả nghiêm trọng như financial transaction."
