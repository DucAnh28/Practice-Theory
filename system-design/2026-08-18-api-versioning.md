# API Versioning Strategy

- **Date:** 2026-08-18
- **Topic:** system-design
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Làm cách nào để versioning API khi cần thay đổi contract mà không break client cũ? Các tradeoff khi chọn URL path vs header vs query param?

## 2) Giải thích như trẻ lên 3

Tưởng tượng có một cửa hàng bán đồ. Ngày xưa bạn mua theo cách cũ. Một ngày cửa hàng thay đổi cách bán, nhưng vẫn phục vụ khách cũ theo cách cũ, còn khách mới theo cách mới. Đó là versioning API.

## 3) Giải thích đơn giản cho dev

API versioning cho phép backend thay đổi contract (endpoint, response shape, behavior) mà không làm crash client đang dùng phiên bản cũ. Có 3 chiến lược chính:
- **URL path**: `/v1/users` vs `/v2/users` — dễ debug, rõ ràng, nhưng URL dài hơn.
- **Header**: `Accept: application/vnd.company.v2+json` — clean URL, nhưng khó test qua browser.
- **Query param**: `/users?apiVersion=2` — hiếm dùng, lẫn lộn với filter param.

## 4) Ví dụ code đơn giản

**URL path versioning (Spring Boot):**
```java
@RestController
@RequestMapping("/api")
public class UserController {
    
    @GetMapping("/v1/users/{id}")
    public ResponseEntity<?> getUserV1(@PathVariable Long id) {
        User user = userService.findById(id);
        return ResponseEntity.ok(new {id, name, email}); // V1 response
    }
    
    @GetMapping("/v2/users/{id}")
    public ResponseEntity<?> getUserV2(@PathVariable Long id) {
        User user = userService.findById(id);
        // V2: thêm timestamp, metadata
        return ResponseEntity.ok(new {id, name, email, createdAt, updatedAt});
    }
}
```

**Header-based (content negotiation):**
```java
@GetMapping("/users/{id}")
public ResponseEntity<?> getUser(
    @PathVariable Long id,
    @RequestHeader(value = "Accept", defaultValue = "application/vnd.company.v1+json") String accept
) {
    User user = userService.findById(id);
    if (accept.contains("v2")) {
        return ResponseEntity.ok(new UserResponseV2(user)); // V2
    }
    return ResponseEntity.ok(new UserResponseV1(user)); // V1
}
```

## 5) Trả lời Middle+

**Chọn URL path versioning** là cách tiêu chuẩn, dễ hiểu và debug:
- Mỗi version là một endpoint riêng: `/v1`, `/v2`.
- Client biết chính xác mình dùng version nào.
- Server dễ route và test từng version.
- Tradeoff: URL dài hơn, nhưng đánh đổi được sự rõ ràng.

Quy trình:
1. Khi cần breaking change, tạo endpoint `/v2/users` mới.
2. Giữ `/v1/users` chạy song song (sunset sau N tháng).
3. Client chọn switch version khi sẵn sàng.
4. Deprecated version → gửi warning header, sau đó remove.

## 6) Trả lời Senior

URL path versioning phổ biến nhưng có cost:
- **API bloat**: mỗi version = code nhân đôi. Cân nhắc code reuse (mapper, shared logic).
- **Sunset pressure**: cần strategy khi xóa version cũ. Đặt deadline công khai, monitor adoption, gửi notification.
- **Hybrid approach**: version ở URL nhưng response schema dùng **backward-compatible changes**:
  - Thêm field mới ← OK (client ignores).
  - Xóa field ← breaking (versioning hoặc deprecation warning).
  - Rename field ← breaking.
  - Change type (string → int) ← breaking.

**Giải pháp hybrid**: URL `/api/users` với response schema từng bước forward-compatible:
```json
{
  "id": 123,
  "name": "Alice",
  "email": "alice@example.com",
  "_metadata": {"apiVersion": "2", "deprecated": false}
}
```

Client cũ bỏ qua `_metadata`, không break. Version mới dùng metadata để enable/disable feature.

**Production concern**:
- Monitor client adoption qua `User-Agent` header, endpoint hit rate.
- Metrics: % client dùng v1 vs v2 → decide sunset date.
- API gateway (Kong, AWS API Gateway) giúp versioning ở tầng infra, không cần code mỗi handler.

## 7) Follow-up / pitfall

**Câu xoáy:**
- Baggage: nếu v1 → v2 → v3, có cách reuse logic không? → Custom middleware/mapper, DRY.
- Mobile app mập: nếu user không update app, stuck v1 vĩnh viễn? → Enforce min version ở API.
- Cache busting: URL path versioning có tự invalidate cache? → Yes, `/v1/` vs `/v2/` = khác cache key.

**Pitfall:**
- Quên backward compatibility đến khi breaking. → Hãy xem API response là hợp đồng.
- Sunset không rõ ràng → client nhầm, server vẫn chạy v1. → Công bố timeline cụ thể.
- Version quá nhiều (v1…v10) → debt. → Tối thiểu 2 version, sunset chủ động.

## 8) 30 giây tóm tắt miệu

"Versioning API bằng URL path (`/v1/`, `/v2/`) là cách đơn nhất. Mỗi version là endpoint riêng, client rõ ràng dùng phiên bản nào. Khi breaking change, tạo v2 mới, giữ v1 song song, sau N tháng sunset v1. Giải pháp hybrid: response schema backward-compatible (thêm field không break), version dùng metadata để feature flag. Production: monitor adoption qua metric, enforce min client version nếu cần."
