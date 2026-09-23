# Bảo mật API (API Security) & Rate Limiting

- **Date:** 2026-09-23
- **Topic:** security
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Làm sao để bảo vệ REST API khỏi các tấn công phổ biến như brute-force, DDoS quy mô nhỏ và lạm dụng tài nguyên (abuse) ở tầng ứng dụng? Giải thích cơ chế hoạt động của Rate Limiting và các thuật toán phổ biến.

## 2) Giải thích như trẻ lên 3

Hãy tưởng tượng API của bạn là một quầy bán nước ngọt miễn phí. Nếu một người cứ đứng đó hốt cả nghìn chai mỗi giây khiến người khác không mua được, bạn sẽ đặt một quy định: "Mỗi người chỉ được lấy tối đa 5 chai trong 1 phút". Nếu vượt quá, bảo vệ sẽ tạm thời không cho lấy nữa. Đó chính là Rate Limiting.

## 3) Giải thích đơn giản cho dev

Rate Limiting là kỹ thuật giới hạn số lượng request mà một client (IP, User ID, API Key) có thể gửi đến hệ thống trong một khoảng thời gian nhất định (window). Các thuật toán phổ biến:
- **Token Bucket:** Có một cái xô đựng token, token được thêm vào đều đặn với tốc độ cố định. Mỗi request lấy 1 token, hết token thì bị chặn. Cho phép burst (bùng nổ) request ngắn hạn.
- **Leaky Bucket:** Request đổ vào xô, xô rò rỉ ra ngoài với tốc độ không đổi. Xô đầy thì rớt request. Giúp traffic ra đều đặn.
- **Fixed Window Counter:** Đếm request trong một khung thời gian cố định (ví dụ: từ 12:00 đến 12:01). Dễ dính lỗi tràn ở biên thời gian (burst gấp đôi ở phút chuyển giao).
- **Sliding Window Log / Counter:** Khắc phục lỗi ở biên bằng cách tính trượt hoặc kết hợp cửa sổ trước và hiện tại.

## 4) Ví dụ code đơn giản

Ví dụ dùng Redis + Lua script đơn giản cho Fixed Window / Sliding Counter (hoặc tư duy thuật toán):

```java
// Spring Boot Interceptor / Filter pseudo-logic với Redis
public boolean allowRequest(String clientId) {
    String key = "rate_limit:" + clientId;
    long currentCount = redisTemplate.opsForValue().increment(key, 1);
    if (currentCount == 1) {
        redisTemplate.expire(key, Duration.ofMinutes(1));
    }
    return currentCount <= 100; // tối đa 100 req/min
}
```

## 5) Trả lời Middle+

- **Cơ chế:** Dùng Redis để lưu trữ state đếm request theo IP hoặc User ID vì tốc độ nhanh, hỗ trợ TTL tự động xóa key.
- **Phân tầng chặn:** Chặn ở API Gateway (Kong, Nginx, Spring Cloud Gateway) là tốt nhất để giảm tải cho service bên dưới. Nếu cần logic phức tạp (theo user plan, theo role), xử lý ở tầng Application (Interceptor/Filter).
- **Response trả về:** HTTP Status `429 Too Many Requests`, kèm header `Retry-After`, `X-RateLimit-Limit`, `X-RateLimit-Remaining`.

## 6) Trả lời Senior

- **Distributed Rate Limiting:** Khi hệ thống scale-out nhiều instance, Redis cluster hoặc Redis đơn lẻ có thể bị bottleneck nếu mỗi request đều tăng counter qua mạng. Giải pháp: Dùng Local Memory cache (Guava/Caffeine) kết hợp Rate Limiting bậc nhẹ (Token Bucket cục bộ) trước khi gọi Redis, hoặc dùng Redis Lua script để đảm bảo tính atomic (tránh race condition).
- **Granularity (Độ chi tiết):** Rate limit theo IP dễ bị lẩn trốn nếu dùng NAT/Proxy chung hoặc botnet phân tán. Phải kết hợp limit theo User ID, API Key hoặc Device Fingerprint.
- **DDoS Layer:** Rate limiting ứng dụng chỉ chống lạm dụng logic/API abuse. Chống DDoS volume lớn phải dùng WAF (Cloudflare, AWS Shield) ở tầng mạng (Layer 3/4/7).

## 7) Follow-up / pitfall

- **Lỗi Fixed Window Edge:** Người dùng gửi 100 request vào giây cuối của phút thứ 1 và 100 request vào giây đầu của phút thứ 2 -> Gửi 200 request liên tiếp trong 2 giây mà không bị chặn. *Khắc phục:* Dùng Sliding Window Counter.
- **Memory Leak trong Redis:** Quên set TTL cho key rate limit dẫn đến Redis đầy RAM.

## 8) 30 giây tóm tắt miệng

Rate limiting bảo vệ API bằng cách giới hạn request dựa trên Token Bucket hoặc Sliding Window, thường lưu state trên Redis và chặn từ API Gateway với mã lỗi 429, giúp hệ thống không bị sập vì quá tải hoặc bot cào dữ liệu.
