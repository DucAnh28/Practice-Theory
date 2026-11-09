# Database Sharding, Partitioning và Read Replica

- **Date:** 2026-11-09
- **Topic:** system-design
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

DB của bạn đang chậm và sắp đầy ổ cứng, traffic tăng gấp 10. Bạn sẽ chia tải đọc/ghi như thế nào? Khi nào dùng partition, khi nào dùng sharding, khi nào thêm read replica?

## 2) Giải thích như trẻ lên 3

- **Partitioning**: chia một cái kho lớn thành nhiều ngăn nhỏ (theo tháng, theo id) — vẫn là **một** kho, lấy đồ từ đúng ngăn là ra nhanh.
- **Sharding**: chia thành nhiều kho riêng biệt, mỗi kho là một DB vật lý khác nhau, rồi chia khách hàng theo quy tắc để ai đó về kho nào.
- **Read replica**: một bản copy của DB chính, chỉ để đọc. Giống photocopy tờ giấy ra để mọi người cầm xem, bản gốc chỉ dùng để ký.

## 3) Giải thích đơn giản cho dev

- **Vertical scaling** (tăng CPU/RAM/SSD) luôn là bước đầu tiên: rẻ, ít rủi ro, không đụng code.
- **Partitioning (logical)**: cắt bảng theo `created_at` (theo tháng) hoặc hash id. Query phải có điều kiện lọc partition, nếu không sẽ full scan toàn bảng.
- **Sharding (physical)**: tách dữ liệu ra nhiều instance/cluster. Cần **shard key** để định tuyến đúng shard. Shard key tốt = truy vấn đi qua ít shard, đều tải, không hot shard.
- **Read replica**: replica đọc bị **lag** (độ trễ). Dữ liệu vừa ghi chưa thấy ngay ở replica → phải đọc master khi cần đọc-sau-ghi (read-your-write), hoặc dùng routing theo "sticky" session.
- Tách bạch: partitioning là **một** DB; sharding là **nhiều** DB. Read replica là **copy**, không phải chia tải ghi.

## 4) Ví dụ code đơn giản

```java
// Định tuyến tới shard theo tenant_id (hash => shard)
public int shardOf(long tenantId, int shardCount) {
    return (int) Math.floorMod(Long.hashCode(tenantId), shardCount);
}

// Routing: write luôn vào master, read có thể đi replica
public DataSource route(long tenantId, boolean write) {
    Shard shard = shards.get(shardOf(tenantId, shards.size()));
    return write ? shard.master : shard.replica;
}

// Chỉ đọc dữ liệu vừa ghi => ép đọc master để tránh replica lag
public Order findOrderForUpdate(long tenantId, long orderId) {
    // read-after-write: dùng master, không dùng replica
    return jdbcTemplate.queryForObject(
        "SELECT * FROM orders WHERE tenant_id = ? AND id = ?",
        mapper, tenantId, orderId);
}
```

## 5) Trả lời Middle+

- Thứ tự ưu tiên khi DB chậm: **index + query tune → cache (Redis) → vertical scale → partition → replica cho đọc → cuối cùng mới sharding**.
- **Partitioning**: thêm cột `partition_key` (thường là `created_at` theo tháng). Query phải luôn lọc theo partition key; có thể dùng partition pruning để giảm IO.
- **Sharding**: chọn shard key theo tenant/customer id để gom dữ liệu một khách về một shard → truy vấn 1 shard, không cần distributed transaction cho dữ liệu của một tenant.
- **Read replica**: định tuyến các query đọc (report, list, search) sang replica, giữ master cho ghi. Dùng connection pool riêng cho master và replica.

## 6) Trả lời Senior

- **Sharding key là quyết định khó nhất, gần như không đổi lại được**: chọn sai là phải re-shard toàn bộ (resharding + dual-write + backfill). Tiêu chí: phân bố đều, không hot shard, phần lớn truy vấn single-shard, không cần cross-shard transaction thường xuyên.
- **Cross-shard query**: join, `ORDER BY` toàn cục, `COUNT` toàn bảng cần scatter-gather → chậm và tốn tài nguyên. Giải pháp: fan-out từ app layer, denormalize, pre-aggregate (bảng tổng hợp theo giờ), hoặc dùng sharding-aware query engine.
- **Distributed transaction**: tránh 2PC với resource mới. Ưu tiên **local transaction + outbox pattern** + consumer đọc CDC/outbox để đồng bộ sang shard khác; idempotent consumer để an toàn khi retry.
- **Resharding**: dual-write (ghi cả cũ lẫn mới) + backfill dữ liệu lịch sử + verify checksum rồi mới chuyển đọc; cutover bằng feature flag từng shard.
- **Replica lag**: có thể tới vài giây khi có write burst. Xử lý: read-after-write dùng master cho session vừa ghi, sticky routing theo user, đặt max-lag threshold và fallback về master.
- **Failover**: tự động chuyển master khi chết là điểm dễ sai nhất. Cần: HA tool (Patroni/Orchestrator), promotion có thủ tụng khi cần, và luôn hiểu rằng mất dữ liệu vài giây là không tránh khỏi với async replication.
- **Observability**: theo dõi per-shard QPS/latency/p95, độ lệch tải giữa các shard, replication lag, và tỷ lệ query cross-shard. Có cảnh báo khi 1 shard nóng hơn các shard khác — thường là dấu hiệu shard key skew.
- **Kinh tế**: shard tăng chi phí vận hành (nhiều instance, tooling, monitoring). Với lưu lượng < vài nghìn QPS, thường **một DB lớn + replica + cache** rẻ hơn và ít rủi ro hơn.

## 7) Follow-up / pitfall

- **Pitfall 1**: query không lọc partition key → partition pruning không chạy, đọc toàn bảng, nặng hơn table thường.
- **Pitfall 2**: dùng replica cho đọc ngay sau khi ghi → user thấy dữ liệu cũ, báo lỗi "vừa tạo xong mà không thấy".
- **Pitfall 3**: shard key theo `id` auto-increment → tất cả request dồn vào shard cuối (hot shard). Dùng hash hoặc cột phân tán tự nhiên.
- **Pitfall 4**: chọn shard key theo `region` → một vùng hot, một vùng nghẻo.
- **Pitfall 5**: tưởng partition = sharding → vẫn nghĩ DB vẫn là một điểm chết, không scale ngang được.
- **Follow-up**: Có cần shard không, hay chỉ cần partition + read replica là đủ? Khi nào dữ liệu > 1 TB/shard thì mới cân nhắc sharding thật.
- **Follow-up**: Nếu 2 tenant khác nhau cần cùng một transaction thì sao? (trả lời: đưa về cùng shard theo tenant, hoặc dùng saga/outbox, không dùng XA)

## 8) 30 giây tóm tắt miệng

"Trước khi shard, tôi sẽ index, cache, rồi vertical scale, sau đó partition theo tháng để query được prune, và thêm read replica cho các query đọc. Sharding chỉ làm khi thực sự cần, vì nó tốn vận hành và khó resharding. Shard key chọn theo tenant id để hầu hết truy vấn single-shard, cross-shard thì xử lý bằng outbox + idempotent consumer thay vì distributed transaction. Lưu ý replica lag nên đọc-sau-ghi phải ép đọc master."
