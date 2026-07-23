# Consumer Group & Offset

- **Date:** 2026-07-23
- **Topic:** kafka
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Consumer group là gì? Cách offset được lưu và quản lý? Những vấn đề thường gặp?

## 2) Giải thích như trẻ lên 3

Có một cuốn truyện, mỗi người bạn đọc một phần. Nhóm bạn (consumer group) chia sẻ cuốn truyện thành nhiều phần (partition). Mỗi người chịu trách nhiệm một số phần. Để không đọc lại hoặc bỏ sót, mỗi người dùng một chiếc bìa trang (offset) ghi số trang đã đọc. Khi xong một phần, bạn ghi lại số trang và chuyển sang phần khác.

## 3) Giải thích đơn giản cho dev

- **Consumer group:** Tập hợp các consumer dùng chung một group.id. Kafka phân phối các partition cho từng consumer trong group, mỗi partition chỉ được một consumer xử lý tại một thời điểm.
- **Offset:** Số vị trí (position) consumer đã đọc trong một partition. Kafka lưu offset tại `__consumer_offsets` topic nội bộ (cho group đã commit) hoặc trên local nếu dùng `enable.auto.commit=false` và chưa commit.
- **Commit:** `auto.commit` (mỗi 5s) hoặc `commitSync`/`commitAsync` thủ công.

## 4) Ví dụ code đơn giản

```java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("group.id", "order-processing");
props.put("enable.auto.commit", "false");
props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Collections.singletonList("orders"));

try {
    while (true) {
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
        for (ConsumerRecord<String, String> record : records) {
            // xử lý
            processOrder(record.value());
        }
        // commit sau khi xử lý xong batch
        consumer.commitSync();
    }
} finally {
    consumer.close();
}
```

## 5) Trả lời Middle+

- Consumer group đảm bảo mỗi message chỉ được xử lý bởi một consumer trong group (một partition = một consumer).
- Offset được lưu trong `__consumer_offsets` internal topic, cho phép consumer restart mà không mất vị trí đã đọc.
- Dùng `enable.auto.commit=false` + `commitSync`/`commitAsync` để kiểm soát chính xác thời điểm commit, tránh mất message hoặc xử lý trùng.
- Rebalance xảy ra khi số consumer thay đổi (thêm/xóa consumer, partition thay đổi), lúc này offset có thể bị mất nếu chưa commit.

## 6) Trả lời Senior

- **At-least-once vs at-most-once:** Commit sau khi xử lý (commit-after-process) → at-least-once (có thể trùng lặp). Commit trước khi xử lý (commit-before-process) → at-most-once (có thể mất message). Chọn based on business: tài chính thường chọn at-least-once + idempotency.
- **Rebalance listener:** Dùng `ConsumerRebalanceListener` để commit offset trước khi mất partition, tránh xử lý lại hoặc bỏ sót.
- **Offset reset:** Khi offset không tồn tại (`auto.offset.reset=earliest/latest/none`), cần cấu hình rõ để tránh mất data hoặc xử lý không mong muốn.
- **Monitoring:** Theo dõi `lag` (số message chưa xử lý) qua JMX/Micrometer. Lag tăng = consumer chậm hoặc down.
- **Production concern:** Tránh dùng `enable.auto.commit=true` trong hệ thống quan trọng; luôn commit thủ công sau khi xử lý thành công.

## 7) Follow-up / pitfall

- **Pitfall 1:** Dùng `auto.commit=true` → nếu consumer xử lý lâu hơn 5s (auto.commit interval), message có thể bị xử lý lại sau khi restart.
- **Pitfall 2:** Commit offset trước khi xử lý xong → nếu crash giữa chừng, message bị mất.
- **Pitfall 3:** Không xử lý rebalance → mất offset khi rebalance, xử lý lại toàn bộ.
- **Follow-up:** Cách đảm bảo exactly-once? (Kafka Transactions + idempotent producer)

## 8) 30 giây tóm tắt miệng

Consumer group là tập các consumer cùng chia sẻ công việc đọc từ các partition. Mỗi partition chỉ được một consumer xử lý. Offset là con trỏ đánh dấu vị trí đã đọc, được lưu trong `__consumer_offsets`. Cấu hình `enable.auto.commit=false` và commit thủ công sau khi xử lý để đảm bảo tin nhắn không bị mất hay xử lý trùng. Rebalance có thể gây mất offset nếu chưa commit, nên dùng `ConsumerRebalanceListener` để xử lý.
