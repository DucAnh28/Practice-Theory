# Kafka idempotent producer

- **Date:** 2026-09-07
- **Topic:** kafka
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Kafka idempotent producer giải quyết lỗi gì? Có thay thế được xử lý trùng ở consumer không?

## 2) Giải thích như trẻ lên 3

Gửi thư có thể mạng chập chờn. Người gửi không biết thư tới chưa nên gửi lại. Idempotent producer đánh số thư; bưu điện thấy số cũ thì không phát thêm lần nữa.

## 3) Giải thích đơn giản cho dev

Khi retry sau lỗi mạng, producer có thể gửi cùng record nhiều lần. Với `enable.idempotence=true`, Kafka dùng producer ID, sequence number và broker state để broker ghi mỗi record một lần trên từng partition trong một producer session.

## 4) Ví dụ code đơn giản

```java
Properties p = new Properties();
p.put("bootstrap.servers", "kafka:9092");
p.put("enable.idempotence", "true");
p.put("acks", "all");

try (var producer = new KafkaProducer<String, String>(p)) {
    producer.send(new ProducerRecord<>("orders", "order-42", "created"))
            .get();
}
```

## 5) Trả lời Middle+

Bật idempotence để retry producer không tạo duplicate do lỗi tạm thời. Nó cần `acks=all` và giữ thứ tự đúng trong cùng partition. Dùng key ổn định để event cùng aggregate vào cùng partition.

## 6) Trả lời Senior

Idempotence chỉ bảo vệ producer-to-broker trong session và partition; không loại duplicate từ producer restart, consumer retry, hay HTTP request lặp. Thiết kế consumer idempotent bằng event ID/dedup store hoặc upsert. Nếu cần atomic consume-process-produce, dùng Kafka transaction; đổi lại throughput, latency và vận hành phức tạp hơn.

## 7) Follow-up / pitfall

- `enable.idempotence` không biến toàn pipeline thành exactly-once.
- Đừng dùng random key nếu cần ordering theo order/user.
- `send().get()` chỉ hợp ví dụ; production nên xử lý callback, retry và metric lỗi.

## 8) 30 giây tóm tắt miệng

Idempotent producer ngăn duplicate khi producer retry vì broker nhận diện sequence number của từng record. Nó bảo đảm theo producer session và partition, không thay thế idempotency ở consumer. Hệ thống production vẫn cần event ID, dedup hoặc upsert; dùng transaction khi cần atomic read-process-write Kafka.
