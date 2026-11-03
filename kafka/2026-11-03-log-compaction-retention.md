# Kafka Log Compaction và Retention Policy

- **Date:** 2026-11-03
- **Topic:** kafka
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

"Khi nào dùng log compaction? `retention.ms` và `cleanup.policy` khác nhau thế nào? Ảnh hưởng gì tới disk và consumer lag?"

## 2) Giải thích như trẻ lên 3

Kafka là cuốn sổ ghi tất cả tin nhắn. Có 2 kiểu cuốn sổ:

- **Sổ nhật ký (retention)**: giữ tin cũ trong thời gian rồi xoá. Ai đọc trễ sẽ không còn tin.
- **Sổ tra cứu (compaction)**: giữ lại **phiên bản mới nhất** của mỗi khóa, xoá các phiên bản cũ trùng key. Giống bảng "trạng thái hiện tại" của từng người dùng.

## 3) Giải thích đơn giản cho dev

- `cleanup.policy=delete` (mặc định): xoá theo thời gian (`retention.ms`/`retention.bytes`) hoặc theo đầu segment (`retention.hours` legacy).
- `cleanup.policy=compact`: xoá bản ghi cũ có **cùng key**, giữ bản mới nhất (theo offset). Dùng cho state store: `topic-changelog` của Kafka Streams, config/state topic.
- `cleanup.policy=compact,delete`: vừa compact vừa xoá theo thời gian — bản compact nhất bị xoá khi hết `retention.ms`.
- Compaction chạy nền bởi **Log Cleaner** (dùng `log.cleaner.threads`), không chạy tức thời → dữ liệu cũ vẫn còn tạm thời.
- Consumer bị ảnh hưởng: consumer đọc trễ sẽ gặp "gap" vì record cũ đã bị xoá; với compacted topic, offset có thể nhảy (record bị xoá vẫn giữ offset placeholder).

## 4) Ví dụ code đơn giản

```bash
# Tạo topic compacted (changelog cho Kafka Streams)
kafka-topics.sh --create --topic orders-changelog \
  --partitions 3 --replication-factor 3 \
  --config cleanup.policy=compact \
  --config min.cleanable.dirty.ratio=0.1 \
  --config segment.ms=60000

# Topic thông thường, xoá sau 7 ngày
kafka-topics.sh --create --topic events \
  --partitions 6 --replication-factor 3 \
  --config cleanup.policy=delete \
  --config retention.ms=604800000
```

```java
// Producer: luôn ghi state mới nhất theo key (null key = không compact được)
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

KafkaProducer<String, String> producer = new KafkaProducer<>(props);
producer.send(new ProducerRecord<>("orders-changelog", "order-123", "{\"status\":\"PAID\"}"));
producer.close();
```

## 5) Trả lời Middle+

"`cleanup.policy` quyết định Kafka xoá dữ liệu theo 2 kiểu:

- `delete`: xoá theo tuổi record (`retention.ms`) hoặc theo dung lượng (`retention.bytes`) → topic là log nguồn sự kiện.
- `compact`: giữ bản ghi mới nhất cho mỗi key, xoá bản cũ trùng key → topic là state store, đọc nhanh bằng `seek` tới offset gần nhất.

Dùng compacted topic khi consumer cần **state hiện tại** (Kafka Streams changelog, cấu hình, cache mapping). Nhược điểm: đọc lại toàn bộ lịch sử vẫn tốn thời gian, và compaction chỉ chạy nền nên dữ liệu cũ chưa biến mất ngay.

Kiểm tra: `kafka-topics.sh --describe --topic X` xem `cleanup.policy` và `retention.ms`."

## 6) Trả lời Senior

"**Vì sao compaction**: giảm disk và giữ state hiện tại mà không mất lịch sử trạng thái. Nhưng có cost:
- Log Cleaner cần đọc lại partition → I/O spike, cần tune `log.cleaner.threads`, `log.cleaner.dedupe.buffer.size`.
- Chỉ hoạt động khi partition **dirty** (tỉ lệ record cũ ≥ `min.cleanable.dirty.ratio`) và segment đã đóng (`segment.ms`/`segment.bytes`) → không compact nếu producer ghi liên tục.

**Vận hành**:
- Set `delete.retention.ms` để tombstone được xoá sớm, nếu không tombstone giữ lại key vô hạn.
- **Tombstone** (value = null) là cách xoá key trong compacted topic; nếu không gửi tombstone, key cũ vẫn còn.
- Retention/compaction không cứu được consumer lag: dữ liệu đã xoá → consumer mới không replay được, phải dùng DLQ hoặc snapshot.
- Key phải ổn định và ít cardinality. Key = null → record **không** được compact (log compaction bỏ qua null key), dễ phình disk.
- Nếu cần replay lịch sử dài hạn, dùng `cleanup.policy=delete` với retention dài + tiered storage (Confluent) thay vì compact.

**Trade-off**: `compact,delete` kết hợp để giữ topic gọn nhưng vẫn có cửa sổ dữ liệu; đánh đổi là mất lịch sử để replay sau thời hạn."

## 7) Follow-up / pitfall

**Q:** "Consumer mới join group, offset cũ đã bị xoá thì sao?"

**A:** Nó sẽ đọc từ offset đã lưu nhưng gặp gap — Kafka trả về `OffsetOutOfRange` hoặc bỏ qua record đã xoá. Với compacted topic, consumer thường bị "đẩy" tới offset mới nhất. Xử lý bằng `auto.offset.reset=earliest` + có sẵn snapshot/backfill.

**Q:** "Compaction xoá message còn consumer chưa đọc chưa?"

**A:** Có thể. Compaction chạy theo key, không quan tâm consumer nào đã đọc. Đây là lý do không dùng compacted topic làm event log chuẩn (audit/event sourcing).

**Pitfall:**
- `retention.bytes` trên partition, không phải cả topic → partition này đầy sớm hơn partition khác, gây lệch.
- Nhầm `retention.hours` (legacy) với `retention.ms` — dùng `ms` để rõ đơn vị.
- Compacted topic không có TTL mặc định → nếu không set `delete.retention.ms`, key cũ tồn tại vĩnh viễn.
- `min.compaction.lag.ms` quá lớn → record mới bị giữ lâu hơn cần thiết.

## 8) 30 giây tóm tắt miệng

"Retention xoá record theo thời gian/dung lượng cho event log; compaction giữ bản mới nhất theo key cho state store, chạy nền bởi Log Cleaner nên không xoá tức thời. Dùng compact khi consumer cần state hiện tại như Kafka Streams changelog; cần tombstone + `delete.retention.ms` để xoá key thật sự, và không dùng compacted topic làm log cần replay vì dữ liệu có thể bị xoá trước khi consumer đọc."
