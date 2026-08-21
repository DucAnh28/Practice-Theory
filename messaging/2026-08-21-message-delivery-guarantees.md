# Message Delivery Guarantees trong Messaging Systems

- **Date:** 2026-08-21
- **Topic:** messaging
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

"Hãy giải thích 3 mức delivery guarantee trong message queue: at-most-once, at-least-once, và exactly-once. Khi nào dùng mức nào?"

## 2) Giải thích như trẻ lên 3

Tưởng tượng bạn gửi thư cho bạn:

- **At-most-once**: Bỏ thư vào hòm thư rồi đi luôn, không quan tâm có đến không. Nhanh nhưng có thể mất.
- **At-least-once**: Gửi thư, bạn phải nhắn "đã nhận". Nếu không thấy nhắn thì gửi lại. Chắc chắn đến nhưng có thể nhận 2 lần.
- **Exactly-once**: Gửi thư có mã số, bạn check mã trước khi đọc. Đảm bảo đúng 1 lần nhưng phức tạp hơn.

## 3) Giải thích đơn giản cho dev

Trong messaging system (Kafka, RabbitMQ, SQS...), delivery guarantee quyết định message có thể:

| Guarantee | Ý nghĩa | Cách hoạt động |
|-----------|---------|----------------|
| **At-most-once** | Gửi 1 lần, mất thì thôi | Producer gửi không chờ ack; consumer đọc xong xóa ngay |
| **At-least-once** | Đảm bảo đến, có thể trùng | Producer retry đến khi ack; consumer xử lý xong mới ack |
| **Exactly-once** | Đúng 1 lần, không trùng | Cần idempotency key + transactional outbox hoặc dedup |

**Thuật ngữ:**
- **Ack (acknowledgment)**: tín hiệu xác nhận đã nhận
- **Idempotency**: xử lý nhiều lần cùng 1 message cho kết quả giống nhau
- **Transactional outbox**: pattern lưu message + business logic trong 1 transaction DB

## 4) Ví dụ code đơn giản

```java
// At-least-once với manual ack (RabbitMQ/Spring)
@RabbitListener(queues = "order-queue", ackMode = "MANUAL")
public void processOrder(Order order, Channel channel, 
                         @Header(AmqpHeaders.DELIVERY_TAG) long tag) {
    try {
        // Xử lý business logic
        orderService.createOrder(order);
        
        // Ack sau khi xử lý xong
        channel.basicAck(tag, false);
    } catch (Exception e) {
        // Nack để retry
        channel.basicNack(tag, false, true);
    }
}

// Exactly-once với idempotency key
@Transactional
public void processPayment(PaymentMessage msg) {
    String idempotencyKey = msg.getMessageId();
    
    // Check đã xử lý chưa
    if (processedMessages.exists(idempotencyKey)) {
        log.info("Message already processed: {}", idempotencyKey);
        return; // Skip
    }
    
    // Xử lý + lưu key trong cùng transaction
    paymentService.charge(msg.getAmount());
    processedMessages.save(idempotencyKey);
}
```

## 5) Trả lời Middle+

"Ba mức delivery guarantee là:

**At-most-once**: Message gửi 1 lần, nếu mất thì mất. Dùng khi performance quan trọng hơn độ tin cậy, ví dụ logging, metrics không quan trọng.

**At-least-once**: Message chắc chắn được deliver, nhưng có thể trùng lặp. Producer retry đến khi nhận ack, consumer xử lý xong mới ack. Đây là mode phổ biến nhất. Cần logic idempotent để handle duplicate.

**Exactly-once**: Message được xử lý đúng 1 lần. Rất khó implement, thường cần:
- Idempotency key để dedup
- Transactional outbox pattern
- Hoặc feature built-in như Kafka transactions

Tôi thường dùng at-least-once kết hợp idempotency key, vì exactly-once phức tạp và có performance cost cao."

## 6) Trả lời Senior

"Delivery guarantee là trade-off giữa reliability, performance và complexity.

**At-most-once** (fire-and-forget):
- Use case: telemetry, audit log không critical
- Pros: latency thấp, throughput cao
- Cons: data loss khi network fail hoặc consumer crash

**At-least-once** (retry until ack):
- Use case: 90% hệ thống production (order, payment, notification)
- Pros: không mất message, implementation đơn giản
- Cons: duplicate messages, cần idempotent consumer
- Implementation: producer retry + manual ack sau xử lý xong

**Exactly-once**:
- True exactly-once chỉ có trong scope hẹp (Kafka transactions trong single cluster)
- End-to-end exactly-once không tồn tại thực tế vì distributed system
- Best practice: at-least-once + application-level idempotency
  - Lưu message ID trong DB cùng transaction với business logic
  - Dùng unique constraint hoặc check before insert
  - TTL cho bảng dedup để không phình to

**Production concerns:**
- Kafka: `acks=all` + `enable.idempotence=true` cho producer, manual commit offset cho consumer
- RabbitMQ: publisher confirms + consumer manual ack
- SQS: visibility timeout + delete message sau khi xử lý
- Retry policy: exponential backoff, dead letter queue cho poison messages
- Monitoring: track duplicate rate, ack rate, retry rate

Câu hỏi follow-up thường là 'làm sao đảm bảo idempotency?' → dùng unique constraint DB hoặc distributed lock (Redis/Zookeeper) nếu cần check state phức tạp."

## 7) Follow-up / pitfall

**Follow-up:**
- "Làm sao implement idempotency khi không có unique key từ producer?"
  → Tạo hash từ message content làm dedup key
- "Exactly-once trong Kafka hoạt động thế nào?"
  → Kafka transactions dùng 2PC, chỉ đảm bảo trong phạm vi Kafka (producer → broker → consumer), không cover external systems
- "Nếu consumer xử lý xong nhưng chưa ack thì crash, sẽ ra sao?"
  → Message quay lại queue, được consume lại → duplicate → cần idempotency

**Pitfalls:**
1. **Ack quá sớm**: Ack trước khi xử lý xong → mất data nếu crash
2. **Không có timeout**: Consumer giữ message mãi không xử lý → block queue
3. **Quên idempotency**: At-least-once mà không handle duplicate → tạo order trùng, charge tiền 2 lần
4. **Dedup table phình to**: Lưu message ID mãi mãi → cần TTL/cleanup job
5. **Network partition**: Ack bị mất trên đường → producer retry → duplicate dù consumer đã xử lý

## 8) 30 giây tóm tắt miệng

"Có 3 mức delivery guarantee. At-most-once gửi 1 lần không retry, nhanh nhưng có thể mất message. At-least-once retry đến khi ack, đảm bảo deliver nhưng có thể trùng. Exactly-once rất khó implement, thực tế mình thường dùng at-least-once kết hợp idempotency key: lưu message ID trong DB cùng transaction với business logic, check trước khi xử lý. Điều quan trọng là luôn ack sau khi xử lý xong, và có dead letter queue cho message lỗi nhiều lần."
