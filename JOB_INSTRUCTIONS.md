# Daily Java Interview Practice Job

Mục tiêu: mỗi ngày làm việc (T2–T6, Asia/Saigon) tạo 1 note ngắn để luyện phỏng vấn Java Developer mức Middle+ / Senior, commit và push lên repo này.

## Cách chọn topic

1. Đọc `.state/rotation.json`.
2. Dùng `topics[nextIndex]` nếu hợp lệ.
3. Nếu `nextIndex` lệch với `lastTopic`, vẫn ưu tiên tránh lặp `lastTopic`; chọn topic kế tiếp trong danh sách.
4. Tạo file trong đúng folder topic, ví dụ `kafka/2026-07-20-consumer-group-offset.md`.
5. Sau khi tạo, cập nhật:
   - `lastDate`
   - `lastTopic`
   - `lastFile`
   - `nextIndex` = index kế tiếp, quay vòng nếu hết list.

## Format bắt buộc

Mỗi note ngắn, dễ đọc, không viết lan man:

```markdown
# <Tên chủ đề>

- **Date:** YYYY-MM-DD
- **Topic:** <folder>
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

...

## 2) Giải thích như trẻ lên 3

Giải thích cực đơn giản bằng ví dụ đời thường.

## 3) Giải thích đơn giản cho dev

Nói đúng kỹ thuật nhưng vẫn dễ hiểu.

## 4) Ví dụ code đơn giản

Nếu topic hợp code thì thêm code ngắn, chạy/đọc được. Nếu không hợp code, dùng pseudo-code hoặc flow.

## 5) Trả lời Middle+

Câu trả lời phỏng vấn gọn, đủ ý, thực dụng.

## 6) Trả lời Senior

Câu trả lời có trade-off, production concern, failure mode, performance/security/observability khi phù hợp.

## 7) Follow-up / pitfall

Các câu hỏi xoáy và lỗi hay gặp.

## 8) 30 giây tóm tắt miệng

Một đoạn ngắn có thể nói trực tiếp trong phỏng vấn.
```

## Yêu cầu nội dung

- Tiếng Việt.
- Ngắn thôi: ưu tiên 1 chủ đề nhỏ/ngày.
- Middle+ = hiểu đúng, dùng được trong task thật.
- Senior = biết trade-off, edge case, vận hành production.
- Có ví dụ code khi giúp hiểu nhanh.
- Không nhồi quá nhiều lý thuyết.
- Không tạo dependency/build system mới.
- Sau khi viết: `git add`, `git commit`, `git push origin main`.
- Nếu không có thay đổi mới, không commit rỗng.
