# JWT Access vs Refresh Token

- **Date:** 2026-08-19
- **Topic:** security
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Tại sao hệ thống thường dùng cả **access token** và **refresh token**? Vòng đời của mỗi loại khác nhau như thế nào, và xử lý ra sao khi access token hết hạn hoặc refresh token bị lộ?

## 2) Giải thích như trẻ lên 3

Tưởng tượng bạn vào công viên bằng **vé ngày** (access token) và **thẻ thành viên** (refresh token).

- Vé ngày dùng để đi từng cổng, hết 1 ngày phải đổi vé mới. Mất vé thì người khác dùng được, nhưng chỉ trong ngày.
- Thẻ thành viên đưa ra quầy để đổi vé mới. Nó được cất kỹ, ít khi đưa ra, nên khó bị lộ hơn.

Nhờ vậy, kẻ gian lấy được vé cũng chỉ dùng được ngắn hạn; muốn dùng lâu phải lấy được cả thẻ thành viên.

## 3) Giải thích đơn giản cho dev

| Loại | Mục đích | Lưu trữ phổ biến | TTL điển hình |
|------|----------|------------------|---------------|
| Access token | Gửi kèm mỗi request đến API | Memory client / cookie HttpOnly | 5–30 phút |
| Refresh token | Đổi access token mới | Cookie HttpOnly + DB/Redis server | 7–30 ngày |

- **Access token** thường là JWT (signed, stateless). Server chỉ verify chữ ký + expiry, không cần session.
- **Refresh token** thường là chuỗi random opaque (server so sánh với DB), vì phải thu hồi được.
- Khi access token hết hạn → client gọi endpoint `/auth/refresh` với refresh token (cookie HttpOnly) → server cấp access token mới. Nếu refresh còn hiệu lực và chưa bị thu hồi.

## 4) Ví dụ code đơn giản

```java
// Endpoint đổi access token từ refresh token
@PostMapping("/auth/refresh")
public AuthResponse refresh(HttpServletRequest req) {
    String refresh = readRefreshCookie(req); // HttpOnly cookie
    RefreshToken stored = refreshRepo.findByTokenHash(hash(refresh))
        .orElseThrow(() -> new AuthException("invalid refresh"));

    if (stored.isRevoked() || stored.isExpired()) {
        throw new AuthException("refresh expired/revoked");
    }

    // Rotation: thu hồi refresh cũ, phát refresh mới + access mới
    stored.revoke();
    String newRefresh = randomToken();
    refreshRepo.save(new RefreshToken(userId(stored), newRefresh, Duration.ofDays(14)));

    String access = jwtIssuer.createAccess(userId(stored), Duration.ofMinutes(15));
    return new AuthResponse(access);
}
```

## 5) Trả lời Middle+

- Access token dùng xác thực API, TTL ngắn (5–30 phút) để giảm thiệt hại khi bị lộ.
- Refresh token dùng để cấp access token mới mà không bắt user đăng nhập lại, TTL dài (ngày–tuần).
- Access nên đặt trong memory + cookie HttpOnly; refresh luôn đặt trong cookie HttpOnly + `SameSite=Strict/Lax`, không để JS đọc.
- Hết hạn access → client gọi `/auth/refresh`; nếu refresh cũng hết/thu hồi → buộc đăng nhập lại.
- Refresh phải lưu server-side (DB/Redis) để có thể thu hồi (logout, đổi mật khẩu, phát hiện bất thường).

## 6) Trả lời Senior

- **Rotation + reuse detection**: mỗi lần refresh phát refresh mới, thu hồi cái cũ. Nếu refresh đã thu hồi được dùng lại → nghi ngờ lộ → thu hồi **toàn bộ session** của user đó.
- **Thu hồi chủ động**: logout mọi nơi, đổi mật khẩu, phát hiện device lạ → set `revoked_at`; filter ở bước verify refresh. Access token không thu hồi được thuần stateless, nhưng có thể dùng **denylist Redis** cho token sắp hết hạn nếu cần.
- **Lưu trữ an toàn**: refresh cookie `HttpOnly; Secure; SameSite=Lax|Strict; Path=/auth`. Cân nhắc device-bound refresh (bind với fingerprint/IP/user-agent) nhưng cẩn thận false positive.
- **JWT hardening**: dùng thuật toán asymmetric (RS256/ES256) khi có nhiều service verify, set `aud`, `iss`, `jti`, `exp` chặt; access token JWT nên có clock-skew leeway ≤ 30s.
- **Production concern**: rate-limit `/auth/refresh` (vd 5 lần/phút/user), log + alert khi reuse detection kích hoạt, tách auth domain riêng để dễ rotate signing key. CSRF: vì cookie HttpOnly không tự chống CSRF cho endpoint refresh, dùng `SameSite=Strict` hoặc double-submit token cho endpoint đó.
- **Phân biệt access vs id-token (OIDC)**: access dùng cho API resource server; id-token dành cho client, không gửi lên API.

## 7) Follow-up / pitfall

- Lưu access token trong `localStorage` → dễ mất khi XSS. Ưu tiên memory + cookie HttpOnly.
- Không thu hồi được access token JWT thuần stateless → chấp nhận TTL ngắn hoặc dùng denylist cho token sắp hết.
- Refresh token không rotation → lộ 1 lần dùng được đến hết hạn. Luôn rotation + reuse detection.
- Đặt cùng cookie domain cho refresh và app → không có `SameSite`/CSRF protection → endpoint refresh có thể bị ép gọi chéo site.
- TTL refresh quá dài (vài tháng) + không có device list → user khó logout hết thiết bị lạ.

## 8) 30 giây tóm tắt miệng

"Access token là JWT TTL ngắn, dùng xác thực API; refresh token là opaque string TTL dài, lưu server-side để cấp access mới và có thể thu hồi. Khi access hết hạn, client gọi `/auth/refresh` với refresh trong cookie HttpOnly; server rotation: thu hồi refresh cũ, phát refresh + access mới. Nếu refresh đã thu hồi mà xuất hiện, coi như bị lộ và thu hồi toàn bộ session. Middle+ nhớ TTL và nơi lưu; Senior nhớ rotation, reuse detection, rate-limit, logging và phân biệt access vs id-token."