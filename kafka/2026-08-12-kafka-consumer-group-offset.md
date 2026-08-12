# Kafka Consumer Group và Offset

- **Date:** 2026-08-12
- **Topic:** kafka
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Consumer Group hoạt động như thế nào trong Kafka? Offset được quản lý ra sao khi consumer bị crash?

## 2) Giải thích đơn giản

Consumer Group = nhóm bạn cùng đọc 1 topic. Mỗi partition chỉ 1 người đọc, tự chia đều. Offset = dấu trang, Kafka lưu đọc đến đâu rồi. Nếu bạn treo, người khác trong group nhận partition đó, offset cũ vẫn giữ nguyên -> đọc lại được.

## 3) Trả lời Middle+

Mỗi Consumer Group có ID riêng. Kafka map partition -> consumer trong group 1:1. Nhiều consumer hơn partition thì thừa, ít hơn thì 1 người đọc nhiều partition. Offset lưu trong broker (`__consumer_offsets`). Mỗi group+partition có offset riêng. Consumer tự commit offset sau khi xử lý xong (auto/commit sync/commit async). Nếu crash trước commit -> lần sau đọc lại từ offset cũ, có duplicate.

## 4) Trả lời Senior

Partition là đơn vị parallel tối đa. Scale consumer = scale partition, không scale consumer group vô hạn. `assign()` thủ công bỏ qua group management -> tự quản lý offset và rebalance. Rebalance dừng toàn bộ consumer (`onPartitionsRevoked` -> `onPartitionsAssigned`) gây stop-the-world. Commit mode quan trọng: `latest` bỏ message, `earliest` replay cũ. Production nên dùng manual sync commit hoặc rebalance listener + in-memory offset + commit sau DB write. Tránh `enable.auto.commit=true` vì không biết message xử lý xong chưa. Offsets expired sau `offsets.retention.minutes` nếu consumer ngừng hoạt động lâu -> group mới khởi động phải re-read từ đầu hoặc dùng `earliest`.

## 5) Follow-up / pitfall

- Rebalance storm: nhiều consumer join/leave liên tục -> session timeout quá ngắn gây loop.
- Duplicate: auto commit trước xử lý -> retry vẫn đọc lại.
- Lost message: auto commit sau xử lý nhưng crash trước commit -> offset cũ, đọc lại OK. Nhưng nếu auto commit trước xử lý -> message mất.
- Consumer group ID trùng giữa app khác nhau -> share partition, race condition.

## 6) 30 giây tóm tắt miệng

Consumer Group chia partition cho từng consumer, mỗi người đọc 1 partition, offset đánh dấu vị trí. Commit xong mới ghi nhận đã đọc. Rebalance khi thay đổi thành viên, tạm dừng toàn bộ. Lưu ý auto commit trước xử lý gây mất message, retry gây duplicate. Production dùng manual commit sau khi xử lý xong.
