# Kafka Producer Idempotence và Exactly-Once Semantics

- **Date:** 2026-10-09
- **Topic:** kafka
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

"Khi nào cần bật idempotent producer trong Kafka? Exactly-once semantics khác gì với at-least-once?"

## 2) Giải thích như trẻ lên 3

Bạn gửi thư cho bạn, nhưng sợ thư lạc nên gửi 2 lần. Bưu điện thông minh biết đó là cùng 1 lá thư (dựa vào ID) nên chỉ giao 1 lần thôi. Đó là **idempotence**.

**Exactly-once** = không chỉ bưu điện không giao lặp, mà cả khi bạn nhận thư xong làm việc gì đó (chuyển tiền chẳng hạn), nó cũng chỉ xảy ra đúng 1 lần.

## 3) Giải thích đơn giản cho dev

- **At-least-once**: Producer retry khi không nhận ack → message có thể duplicate.
- **Idempotent producer**: Kafka gán `producerId` + `sequenceNumber` cho từng message. Broker phát hiện duplicate và chỉ lưu 1 lần.
- **Exactly-once (EOS)**: Kết hợp idempotent + transaction. Producer ghi + consumer đọc + xử lý trong 1 transaction, commit atomic.

## 4) Ví dụ code đơn giản

```java
// Idempotent Producer
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("enable.idempotence", "true"); // Bật idempotence
props.put("acks", "all"); // Bắt buộc với idempotence

KafkaProducer<String, String> producer = new KafkaProducer<>(props);
producer.send(new ProducerRecord<>("my-topic", "key", "value"));
producer.close();
```

```java
// Exactly-Once với Transaction
props.put("transactional.id", "txn-order-processor-1");
KafkaProducer<String, String> producer = new KafkaProducer<>(props);

producer.initTransactions();
try {
    producer.beginTransaction();
    producer.send(new ProducerRecord<>("output-topic", "processed-data"));
    // Có thể send nhiều record, hoặc commit consumer offset
    producer.commitTransaction();
} catch (Exception e) {
    producer.abortTransaction();
}
```

## 5) Trả lời Middle+

"**Idempotent producer** bật khi cần đảm bảo không có duplicate message do retry. Kafka tự động gán `producerId` và `sequenceNumber`, broker loại bỏ duplicate.

**Exactly-once semantics** cần thêm transaction. Dùng khi xử lý stream: đọc từ topic A, xử lý, ghi vào topic B, và commit offset trong 1 transaction atomic. Consumer phải config `isolation.level=read_committed` để chỉ đọc message đã commit.

Trade-off: tăng latency, giảm throughput, cần config cẩn thận `transactional.id`."

## 6) Trả lời Senior

"**Idempotence** giải quyết duplicate do producer retry, nhưng không đủ cho end-to-end exactly-once. Cần transaction khi:
- Consumer xử lý message + ghi output + commit offset phải atomic.
- Tránh partial failure (ghi output rồi nhưng không commit offset → reprocess).

**Production concerns:**
- `transactional.id` phải unique per producer instance. Restart cùng ID để recover transaction chưa commit.
- Broker giữ transaction state → overhead lên coordinator.
- `transaction.timeout.ms` ảnh hưởng availability: quá ngắn → abort sớm, quá dài → giữ resource lâu.
- Consumer `isolation.level=read_committed` chỉ đọc committed, nhưng tăng latency.

**Failure mode:** Producer crash giữa transaction → uncommitted message bị abort, consumer không thấy. Cần idempotent retry ở application level nếu muốn đảm bảo delivery cuối cùng.

**When to skip:** Nếu hệ thống tolerate duplicate (idempotent processing ở consumer), không cần EOS, dùng at-least-once đơn giản hơn."

## 7) Follow-up / pitfall

**Q:** "Nếu consumer crash trước khi commit transaction thì sao?"

**A:** Message chưa committed sẽ bị abort sau `transaction.timeout.ms`. Consumer khác sẽ không thấy message đó. Producer cần retry.

**Q:** "Exactly-once có đảm bảo order không?"

**A:** Có, nếu trong cùng partition. Transaction không làm mất order trong partition.

**Pitfall:**
- Quên set `isolation.level=read_committed` ở consumer → vẫn đọc uncommitted message.
- Dùng cùng `transactional.id` cho nhiều producer instance → conflict, abort transaction.
- Không handle abort gracefully → data loss hoặc stuck consumer.

## 8) 30 giây tóm tắt miệng

"Idempotent producer tự động loại duplicate do retry bằng sequence number. Exactly-once thêm transaction để đảm bảo đọc-xử lý-ghi-commit offset atomic. Cần khi không tolerate duplicate hoặc partial processing. Trade-off là latency cao hơn, cần config kỹ transactional.id và timeout, consumer phải đọc read_committed."
