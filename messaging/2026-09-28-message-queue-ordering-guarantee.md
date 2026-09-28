# Message Queue Ordering Guarantee

- **Date:** 2026-09-28
- **Topic:** messaging
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Khi dùng message queue (Kafka, RabbitMQ), làm sao đảm bảo message được xử lý đúng thứ tự? Nếu producer gửi message A, B, C liên tiếp thì consumer phải nhận theo thứ tự đó. Ngoài queue, còn cách nào khác?

## 2) Giải thích như trẻ lên 3

Tưởng như xếp hàng tại quầy: bạn đứng trước, bạn phía sau. Người xếp hàng lần lượt vào. Nếu có nhiều quầy, người cùng hàng không bao giờ xả lẫn nhau.

## 3) Giải thích đơn giản cho dev

Message queue đảm bảo thứ tự **trong một partition/shard/queue logic**. Nếu bạn gửi vào cùng partition, message A luôn trước B, C. Nếu gửi vào nhiều partition khác nhau, không có đảm bảo thứ tự toàn cục. Consumer xử lý **tuần tự** trong một partition = thứ tự được bảo tồn.

## 4) Ví dụ code đơn giản

**Kafka producer — cùng partition key:**
```java
// Cùng khóa → cùng partition → thứ tự
producer.send(new ProducerRecord<>("order-topic", "user:123", "order-1"));
producer.send(new ProducerRecord<>("order-topic", "user:123", "order-2"));
producer.send(new ProducerRecord<>("order-topic", "user:123", "order-3"));

// Consumer — tuần tự
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> r : records) {
        process(r.value()); // order-1, order-2, order-3 → tuần tự
    }
}
```

**RabbitMQ — single consumer binding:**
```java
// Queue đơn, consumer đơn = thứ tự
channel.basicConsume("order-queue", false, (consumerTag, message) -> {
    process(new String(message.getBody()));
    channel.basicAck(message.getEnvelope().getDeliveryTag(), false);
});
```

## 5) Trả lời Middle+

Thứ tự được bảo tồn khi:
1. **Kafka:** Cùng partition key → cùng partition → consumer tuần tự xử lý → thứ tự OK.
2. **RabbitMQ:** Queue đơn, consumer đơn, xử lý tuần tự ACK.
3. **Trade-off:** Một partition/consumer chậm = throughput thấp.

## 6) Trả lời Senior

**Global ordering là anti-pattern.** Production thường chỉ cần ordering **per-entity** (per-user, per-order):

- **Kafka:** Partition key = entity ID. Multi-partition = parallel, không thể guarantee toàn cục.
- **Failure mode:** Rebalance, broker down → consumer offset reset → replay hoặc skip → có thể vi phạm thứ tự nếu không idempotent.
- **Observability:** Monitor consumer lag, check offset commit position.
- **Idempotency:** Nếu message duplicate (retry), consumer xử lý lại không được duplicate side-effect.

```java
// Defensive: check idempotency key
if (alreadyProcessed(message.getId())) {
    return; // Idempotent
}
process(message);
markProcessed(message.getId());
```

**RabbitMQ:** Dead-letter queue (DLQ) khi max-retries → không tự replay → cần manual intervention hoặc poison-pill handler.

## 7) Follow-up / pitfall

1. **Pitfall:** Gửi vào nhiều partition tưởng vẫn order → sai. Phải cùng partition key.
2. **Pitfall:** Batch process trong consumer (aggregate N message rồi xử lý) → có thể vi phạm thứ tự nếu batch không đồng bộ.
3. **Follow-up:** "Nếu cần global ordering thì sao?" → Distribute lock (Redis, DB), hoặc event sourcing + replay từ đầu.
4. **Follow-up:** "Xử lý message mất (message không đến consumer)?" → Retention policy, replication factor, consumer group offset management.

## 8) 30 giây tóm tắt miệng

> Ordering được đảm bảo trong một partition/queue logic. Kafka, dùng cùng partition key để message cùng entity vào cùng partition, consumer tuần tự xử lý. RabbitMQ, binding queue đơn với consumer đơn. Ngoài ra không có ordering toàn cục, và phải tính đến idempotency khi message retry hoặc rebalance xảy ra.
