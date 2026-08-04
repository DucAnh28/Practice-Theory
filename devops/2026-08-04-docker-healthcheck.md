# Docker Healthcheck: kiểm tra container có thực sự khỏe không?

- **Date:** 2026-08-04
- **Topic:** devops
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Docker `HEALTHCHECK` khác gì trạng thái container đang `running`? Nên thiết kế health check thế nào cho service Java?

## 2) Giải thích như trẻ lên 3

Đèn trong cửa hàng còn sáng không có nghĩa nhân viên vẫn phục vụ được. `running` chỉ nói tiến trình còn chạy; health check thử hỏi cửa hàng để biết dịch vụ có phản hồi đúng không.

## 3) Giải thích đơn giản cho dev

- **Liveness (còn sống):** ứng dụng có bị treo và cần khởi động lại không?
- **Readiness (sẵn sàng):** ứng dụng có nhận traffic được không?
- Docker `HEALTHCHECK` chạy lệnh định kỳ rồi gắn trạng thái `starting`, `healthy` hoặc `unhealthy`.
- Docker Engine không tự restart container chỉ vì `unhealthy`; orchestrator hoặc hệ thống giám sát phải xử lý.

## 4) Ví dụ code đơn giản

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=20s --retries=3 \
  CMD wget -qO- http://localhost:8080/actuator/health/readiness || exit 1
```

## 5) Trả lời Middle+

`running` chỉ xác nhận process chính chưa thoát. `HEALTHCHECK` kiểm tra khả năng phục vụ bằng một lệnh thực tế. Với Spring Boot, có thể dùng Actuator health endpoint. Check cần nhanh, timeout ngắn, ít tốn tài nguyên và trả mã lỗi khi service chưa sẵn sàng.

## 6) Trả lời Senior

Tôi tách liveness và readiness: liveness chỉ kiểm tra lỗi nội bộ không thể tự phục hồi; readiness có thể xét dependency cần thiết trước khi nhận traffic. Không nên đưa mọi dependency vào liveness vì database lỗi tạm thời có thể gây restart hàng loạt. Tôi cấu hình `start-period` theo thời gian warm-up, đặt timeout/retry để tránh false positive, giới hạn thông tin health endpoint và theo dõi lịch sử chuyển trạng thái. Trong Kubernetes, tôi ưu tiên probe native thay vì dựa vào Docker `HEALTHCHECK`.

## 7) Follow-up / pitfall

- Check quá sâu hoặc quá thường xuyên làm tăng tải.
- Dùng `curl`/`wget` nhưng image không có binary đó khiến container luôn `unhealthy`.
- Public health endpoint làm lộ tên dependency hoặc trạng thái hạ tầng.
- Readiness fail nên ngừng nhận traffic; không đồng nghĩa phải restart process.

## 8) 30 giây tóm tắt miệng

Container `running` chỉ cho biết process còn tồn tại, còn health check cho biết service có phục vụ được không. Tôi giữ check nhanh, có timeout/retry, tách liveness khỏi readiness và tránh restart dây chuyền khi dependency lỗi. Với Spring Boot dùng Actuator; với Kubernetes dùng probe native và bảo vệ chi tiết health endpoint.
