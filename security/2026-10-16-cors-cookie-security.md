# CORS và Cookie Security

- **Date:** 2026-10-16
- **Topic:** security
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Khi nào CORS fail với cookie? Làm sao fix? Có gì nguy hiểm?

## 2) Giải thích như trẻ lên 3

Bạn A ở trang facebook.com muốn gửi request đến mybank.com để xem balance. Trình duyệt nói: "Không được! Trang khác không được phép lấy data của tớ." Đó là CORS. Nếu bạn A có cookie đăng nhập mybank, thường trình duyệt cũng chặn gửi cookie qua domain khác — điều này bảo vệ bạn khỏi CSRF.

## 3) Giải thích đơn giản cho dev

CORS (Cross-Origin Resource Sharing) kiểm soát request từ domain khác. Mặc định, trình duyệt chặn:
- Request đến domain khác (origin khác)
- Gửi credential (cookie, auth header) qua cross-origin

Để cho phép, server phải trả header:
```
Access-Control-Allow-Origin: https://allowed.com
Access-Control-Allow-Credentials: true
```

Và client phải set `credentials: 'include'` hoặc `withCredentials: true`.

**Cảnh báo:** Nếu bạn set `Access-Control-Allow-Origin: *` + `Access-Control-Allow-Credentials: true`, trình duyệt sẽ reject (mâu thuẫn). Phải whitelist domain cụ thể.

## 4) Ví dụ code đơn giản

**Server (Spring Boot):**
```java
@Configuration
public class CorsConfig {
    @Bean
    public WebMvcConfigurer corsConfigurer() {
        return new WebMvcConfigurer() {
            @Override
            public void addCorsMappings(CorsRegistry registry) {
                registry.addMapping("/api/**")
                    .allowedOrigins("https://trusted.com")
                    .allowedMethods("GET", "POST")
                    .allowCredentials(true)
                    .maxAge(3600);
            }
        };
    }
}
```

**Client (JS):**
```javascript
fetch('https://api.example.com/balance', {
  credentials: 'include',  // Gửi cookie
  headers: { 'Content-Type': 'application/json' }
})
.then(r => r.json());
```

## 5) Trả lời Middle+

CORS bảo vệ user khỏi XSS attack — trang giả không thể tự động gửi request với cookie của bạn. 

Để cross-origin request có cookie: server phải explicitly allow (whitelist origin, set `Credentials: true`), client phải set `credentials: 'include'`.

Sai lầm hay: set `Access-Control-Allow-Origin: *` + `Credentials: true` → trình duyệt reject. Phải whitelist origin cụ thể.

Cookie security: dùng `HttpOnly` (JS không thể truy cập), `Secure` (chỉ HTTPS), `SameSite=Strict` (chặn CSRF).

## 6) Trả lời Senior

CORS là **browser policy**, không phải security chặt. Server có thể bypass bằng proxy. Production cần:

1. **Whitelist origin cụ thể**, không dùng `*`. Có thể dynamic từ config/DB nếu cần.
2. **HttpOnly cookie**: token không bị XSS đánh cắp. Nhưng CSRF vẫn có thể tấn công (payload từ cross-origin, cookie tự động gửi).
3. **SameSite=Strict**: chặn cookie gửi cross-site (tốt nhất). `Lax` = GET safe, POST risky.
4. **CSRF token**: nếu dùng session cookie (không HttpOnly), phải kèm CSRF token trong request body/header.
5. **Preflight request**: OPTIONS request trước POST/PUT — có thể cache (timing attack?), validate origin ở đây.

Trade-off: `SameSite=Strict` an toàn nhưng sẽ break third-party embed (iframe, external link back). `Lax` tốt hơn cho ứng dụng thường.

## 7) Follow-up / pitfall

- **Pitfall 1:** Set `Access-Control-Allow-Origin: *` với credential → error.
- **Pitfall 2:** Quên `credentials: 'include'` ở client → cookie không gửi.
- **Pitfall 3:** Dùng HTTP thay HTTPS → `Secure` flag không work, cookie gửi cleartext.
- **Follow-up:** Nếu mobile app (không browser)? CORS không apply, nhưng vẫn cần HTTPS + token validation.

## 8) 30 giây tóm tắt miệng

"CORS chặn cross-origin request từ trình duyệt để bảo vệ user. Nếu cần gửi cookie qua domain khác, server phải whitelist origin cụ thể và set `Access-Control-Allow-Credentials: true`. Client set `credentials: 'include'`. Dùng HttpOnly + SameSite=Strict để chặn XSS/CSRF. Không bao giờ dùng `Origin: *` với credential."
