# Message Queue Patterns

- **Date:** 2026-08-03
- **Topic:** messaging
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Phân biệt giữa **Point-to-Point (Queue)** và **Publish-Subscribe (Topic)** trong message queue. Khi nào dùng cái nào? Trade-off là gì?

## 2) Giải thích như trẻ lên 3

- **Queue (P2P):** Như gửi thư cho một người. Anh em lấy thư rồi xóa đi, người khác không thấy.
- **Topic (Pub-Sub):** Như sơn thông báo công cộng. Ai muốn xem cũng xem được, nhiều người cùng xem.

## 3) Giải thích đơn giản cho dev

| Mẫu | Queue | Topic |
|-----|-------|-------|
| **Người nhận** | 1 consumer duy nhất | Nhiều subscriber độc lập |
| **Xóa tin** | Tin bị xóa sau khi consumer ACK | Tin sống lâu hơn, tùy retention policy |
| **Redelivery** | Nếu consumer crash, tin còn trong queue | Nếu subscriber miss, không có cách lấy lại (trừ replay) |
| **Dùng khi** | Task queue, job processing | Event broadcast, analytics, log aggregation |

## 4) Ví dụ code đơn giản

**Queue (Spring + RabbitMQ):**
```java
// Sender
rabbitTemplate.convertAndSend("order-queue", order);

// Receiver (1 consumer duy nhất lấy tin)
@RabbitListener(queues = "order-queue")
public void processOrder(Order order) {
    // Process, sau đó tin bị xóa
}
```

**Topic (Spring + Kafka):**
```java
// Publisher
kafkaTemplate.send("events-topic", event);

// Subscriber 1
@KafkaListener(topics = "events-topic", groupId = "analytics")
public void logEvent(Event event) { /* log */ }

// Subscriber 2 (cùng lúc)
@KafkaListener(topics = "events-topic", groupId = "notification")
public void notifyUser(Event event) { /* notify */ }
```

## 5) Trả lời Middle+

Point-to-Point dùng khi bạn có một job/task duy nhất cần xử lý: chỉ một service consume, rồi xóa. Ví dụ: order processing, email sending.

Publish-Subscribe dùng khi nhiều service cần phản ứng trên cùng sự kiện: order-created event → inventory service, notification service, analytics service đều nghe. Mỗi service có consumer group riêng.

Trade-off: Queue đơn giản, dễ guarantee "xử lý đúng 1 lần". Topic flexible hơn, nhưng cần care retention policy để replay.

## 6) Trả lời Senior

**Idempotency & exactly-once:**
- Queue: easier để implement exactly-once (offset + ACK).
- Topic: harder; thường là at-least-once → cần idempotent handler ở subscriber.

**Scalability:**
- Queue: 1 consumer xử lý tuần tự, hoặc nhiều instance cùng group chia partition. Throughput bị giới hạn bởi consumer speed.
- Topic: N subscriber group độc lập → throughput cao hơn nếu có nhiều subscriber.

**Operational concern:**
- Queue: storage nhỏ (tin xóa nhanh), nhưng consumer crash → tin stuck. Cần DLQ (Dead Letter Queue).
- Topic: storage lớn (tin sống lâu), replay được. Nhưng broker phải scale storage. Offset management phức tạp hơn nếu nhiều group.

**Production pattern:** Thường dùng **hybrid**: Queue cho task/job; Topic cho event broadcast. Kafka thường chơi cả hai role qua consumer group.

## 7) Follow-up / pitfall

**Q:** Topic có subscriber miss tin không?
**A:** Có. Nếu subscriber subscribe muộn sau khi tin được publish, sẽ miss (trừ khi config retention + replay group từ offset cũ).

**Pitfall:** Không để ý offset management → subscriber xử lý tin cũ lặp lại hoặc skip event.

**Pitfall:** Queue không có DLQ → tin poison khiến consumer crash, rồi stuck forever.

## 8) 30 giây tóm tắt miệng

"Queue là point-to-point, một consumer lấy tin rồi xóa — dùng cho task queue. Topic là broadcast, nhiều subscriber nghe cùng lúc — dùng cho event. Kafka consumer group là topic subscriber, mỗi group offset riêng. Trade-off: Queue đơn giản, Topic flexible nhưng phức tạp offset."
