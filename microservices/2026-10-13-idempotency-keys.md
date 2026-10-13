# Idempotency Keys trong Microservices

- **Date:** 2026-10-13
- **Topic:** microservices
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

"Làm sao đảm bảo một request không bị xử lý nhiều lần khi client retry hoặc network timeout trong microservices?"

## 2) Giải thích như trẻ lên 3

Bạn nhờ mẹ mua kem. Mẹ nói "ừ". Bạn sợ mẹ quên nên nhờ lại. Nếu mẹ thông minh, mẹ nhớ lần đầu đã nghe rồi, không mua 2 que kem. Idempotency key như chiếc phiếu: "đã làm rồi thì không làm lại".

## 3) Giải thích đơn giản cho dev

Idempotency key là một ID duy nhất đi kèm request. Service lưu key này sau khi xử lý thành công. Nếu request bị retry với cùng key, service trả response cũ thay vì chạy lại logic.

**Khi nào cần:**
- Thanh toán, chuyển tiền
- Ghi đơn hàng
- Bất kỳ thao tác không được lặp

**Flow cơ bản:**
1. Client tạo UUID, gửi trong header `Idempotency-Key`.
2. Service check DB/cache: key đã tồn tại?
   - Có → trả kết quả cũ (200).
   - Không → xử lý, lưu key + result, trả về.

## 4) Ví dụ code đơn giản

```java
@PostMapping("/orders")
public ResponseEntity<OrderResponse> createOrder(
    @RequestHeader("Idempotency-Key") String idempotencyKey,
    @RequestBody OrderRequest request
) {
    // Check key trong Redis hoặc DB
    String cached = redisTemplate.opsForValue().get("idem:" + idempotencyKey);
    if (cached != null) {
        return ResponseEntity.ok(objectMapper.readValue(cached, OrderResponse.class));
    }

    // Xử lý order
    Order order = orderService.create(request);
    OrderResponse response = new OrderResponse(order);

    // Lưu kết quả với key, TTL 24h
    redisTemplate.opsForValue().set(
        "idem:" + idempotencyKey, 
        objectMapper.writeValueAsString(response),
        Duration.ofHours(24)
    );

    return ResponseEntity.ok(response);
}
```

## 5) Trả lời Middle+

"Dùng idempotency key để đảm bảo request không xử lý nhiều lần. Client gửi UUID trong header `Idempotency-Key`. Service check key trong cache/DB trước khi xử lý. Nếu key đã tồn tại, trả response cũ. Nếu chưa, xử lý logic rồi lưu key cùng kết quả với TTL hợp lý (vài giờ đến vài ngày tùy use case). Key này nên unique per user hoặc per tenant để tránh conflict."

## 6) Trả lời Senior

"Idempotency key là pattern bắt buộc cho distributed systems. Cần chú ý:

**Storage:**
- Redis cho speed, nhưng cần backup hoặc persistent mode nếu data quan trọng.
- DB cho durability, nhưng cần index trên key column.
- Hoặc hybrid: Redis làm L1, DB làm L2.

**Race condition:**
- Nếu 2 request cùng key đến cùng lúc, cần distributed lock hoặc unique constraint ở DB để tránh xử lý song song.
- Pattern: `SET NX` Redis hoặc `INSERT ... ON CONFLICT DO NOTHING` Postgres.

**Phạm vi key:**
- Key phải scope đúng: `userId:action:UUID` hoặc `tenantId:UUID` tránh collision giữa user/tenant.
- Client tạo UUID v4 hoặc v7. Server không tự gen key vì mất tính idempotent khi retry.

**TTL:**
- Quá ngắn: client retry sau TTL → duplicate.
- Quá dài: tốn memory. Thường 24–72h, phụ thuộc SLA retry của client.

**Response handling:**
- Nếu request đang xử lý (in-flight), có thể trả 409 Conflict hoặc đợi kết quả (polling pattern).
- Lưu cả status: `processing`, `completed`, `failed` để client biết retry hay không.

**Failure modes:**
- Logic fail sau khi lưu key → client nghĩ thành công nhưng thật ra fail. Cần lưu key **sau** khi transaction commit, hoặc lưu key + status atomically.
- Network timeout giữa save key và return response → client retry, thấy key, nhận kết quả thành công. Acceptable trade-off."

## 7) Follow-up / pitfall

**Q:** Key nên tạo ở client hay server?  
**A:** Client. Server tạo key không idempotent vì mỗi request tạo key mới.

**Q:** Xử lý fail giữa chừng thì sao?  
**A:** Đừng lưu key nếu chưa commit transaction. Hoặc lưu với status `failed` để client biết retry được.

**Q:** Key bị trùng giữa 2 user?  
**A:** Scope key theo userId hoặc tenantId: `userId:UUID` thay vì chỉ `UUID`.

**Q:** Redis mất data?  
**A:** Dùng AOF/RDB persistence, hoặc fallback sang DB check.

**Q:** Có cần idempotency cho GET request không?  
**A:** GET tự nhiên idempotent (HTTP spec). Không cần key. Chỉ dùng cho POST/PUT/PATCH có side effect.

**Pitfall hay gặp:**
- Lưu key trước khi xử lý xong → fail giữa chừng nhưng key đã tồn tại.
- Key không unique đủ → collision.
- TTL quá ngắn → client retry sau khi key hết hạn → duplicate.

## 8) 30 giây tóm tắt miệng

"Idempotency key đảm bảo request không xử lý nhiều lần khi retry. Client gửi UUID duy nhất, service check key trước khi xử lý. Nếu đã có, trả response cũ. Key lưu Redis hoặc DB với TTL vài ngày. Quan trọng: key phải scope đúng user/tenant, lưu sau khi transaction commit, và xử lý race condition bằng lock hoặc unique constraint."
