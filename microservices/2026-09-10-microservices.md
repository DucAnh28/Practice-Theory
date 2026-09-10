# API Gateway và BFF trong microservices

- **Date:** 2026-09-10
- **Topic:** microservices
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

API Gateway giải quyết vấn đề gì? Khi nào cần Backend for Frontend (BFF)?

## 2) Giải thích như trẻ lên 3

Gateway như lễ tân tòa nhà. Khách chỉ nói với lễ tân; lễ tân biết phải gọi phòng nào. Mỗi loại khách cần cách phục vụ khác nhau, như điện thoại cần BFF riêng.

## 3) Giải thích đơn giản cho dev

API Gateway là điểm vào chung cho client: định tuyến request, xác thực, rate limit và gom dữ liệu từ nhiều service. BFF là backend riêng cho từng loại client, tối ưu payload và flow cho web hoặc mobile.

## 4) Ví dụ code đơn giản

```java
@GetMapping("/mobile/orders/{id}")
OrderMobileView find(@PathVariable Long id) {
    Order order = orderClient.find(id);
    User user = userClient.find(order.userId());
    return new OrderMobileView(order.id(), order.status(), user.name());
}
```

## 5) Trả lời Middle+

Gateway giảm số endpoint client phải gọi và tập trung concern chung như auth, CORS, rate limit. Không đưa business logic vào Gateway. Chọn BFF khi web và mobile cần payload hoặc flow khác nhau; nếu nhu cầu giống nhau thì một Gateway đủ.

## 6) Trả lời Senior

Gateway/BFF thêm network hop và có nguy cơ thành điểm nghẽn hoặc single point of failure. Cần scale ngang, timeout ngắn, retry có giới hạn, circuit breaker, tracing và rate limit theo client. Tránh fan-out đồng bộ quá lớn; cache read model hoặc chuyển tác vụ không cần phản hồi ngay sang event async. Version API và kiểm soát quyền ở service đích vẫn cần thiết, không chỉ tin Gateway.

## 7) Follow-up / pitfall

- Gateway thành monolith khi chứa business logic.
- Retry fan-out dễ khuếch đại tải khi downstream lỗi.
- BFF không được truy cập trực tiếp database của service khác.

## 8) 30 giây tóm tắt miệng

API Gateway là entry point tập trung routing và concern chung. BFF tối ưu API theo từng client khi nhu cầu web/mobile khác nhau. Trong production, giữ Gateway mỏng, scale ngang, dùng timeout, circuit breaker, tracing; quyền nghiệp vụ vẫn phải được kiểm tra ở service đích.
