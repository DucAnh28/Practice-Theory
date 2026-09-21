# Idempotency Key Trong Distributed Systems

- **Date:** 2026-09-21
- **Topic:** system-design
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Client gọi API thanh toán (hoặc tạo đơn hàng) nhưng timeout network giữa chừng. Client retry lại request. Làm sao để hệ thống backend không bị trừ tiền hoặc tạo đơn 2 lần (double charge / duplicate order)?

## 2) Giải thích như trẻ lên 3

Giống như đi mua đồ ở siêu thị: Bạn đưa thẻ thanh toán, máy bị đơ. Bạn quẹt lại lần nữa. Thu ngân phải kiểm tra xem giao dịch trước đã đi qua chưa, nếu rồi thì không quẹt trừ tiền lần hai nữa. Cái "mã giao dịch riêng cho mỗi lần bấm" chính là Idempotency Key.

## 3) Giải thích đơn giản cho dev

Idempotency key là một unique identifier (thường là UUID) do client sinh ra và gửi kèm trong header (ví dụ: `Idempotency-Key: 12345-abcde`) của request không an toàn (POST/PUT). Server lưu key này (thường vào Redis kèm TTL) để nhận diện và chặn các request retry trùng lặp.

## 4) Ví dụ code đơn giản

```java
public OrderResponse createOrder(CreateOrderRequest req, String idempotencyKey) {
    // 1. Kiểm tra key trong Redis
    String cachedResponse = redisTemplate.opsForValue().get("idempotency:" + idempotencyKey);
    if (cachedResponse != null) {
        return objectMapper.readValue(cachedResponse, OrderResponse.class);
    }
    
    // 2. Thực hiện logic nghiệp vụ (DB transaction)
    OrderResponse response = orderRepository.saveAndProcess(req);
    
    // 3. Lưu kết quả vào Redis với TTL (ví dụ 24h)
    redisTemplate.opsForValue().set(
        "idempotency:" + idempotencyKey, 
        objectMapper.writeValueAsString(response), 
        Duration.ofHours(24)
    );
    
    return response;
}
```

## 5) Trả lời Middle+

- **Cơ chế:** Client tạo UUID gửi lên qua header mỗi lần gọi API quan trọng (thanh toán, tạo đơn).
- **Lưu trữ:** Server check Redis/DB xem key đã tồn tại chưa. Nếu có rồi, trả về kết quả cũ (cached response) thay vì chạy lại logic. Nếu chưa, thực thi transaction, lưu key + response kèm expiration time.
- **Phạm vi áp dụng:** Các HTTP method không idempotent tự nhiên (như POST).

## 6) Trả lời Senior

- **Race Condition / Concurrent Requests:** Nếu 2 request trùng key gửi đến cùng một microservice tại cùng một thời điểm (do client retry dồn dập), việc check-then-set khôngatomic sẽ gây double execution. Giải pháp: Dùng Redis `SETNX` với lock/expiry ngắn hoặc Unique Constraint ở Database trên cột `idempotency_key`.
- **Transaction Boundary:** Lưu key và business data phải nằm trong cùng một DB transaction hoặc xử lý distributed lock cẩn thận để tránh trạng thái lửng lơ (lưu key nhưng transaction lỗi, hoặc ngược lại).
- **Storage Cleanup:** Dùng Redis TTL hoặc database cleanup job để xóa key cũ sau 24-48 giờ.

## 7) Follow-up / pitfall

- *Pitfall 1:* Quên set TTL cho idempotency key làm đầy bộ nhớ Redis/DB.
- *Pitfall 2:* Không handle trường hợp request đầu tiên đang xử lý mà request thứ hai (retry) ập đến (cần trả về HTTP 409 Conflict hoặc chờ kết quả thay vì chạy song song).

## 8) 30 giây tóm tắt miệng

Idempotency key là mã định danh duy nhất do client gửi lên trong mỗi request quan trọng. Server lưu mã này lại kèm kết quả xử lý, nếu nhận được request trùng mã do retry mạng, server sẽ trả về kết quả cũ mà không thực thi lại logic nghiệp vụ, tránh lặp dữ liệu.
