# JWT vs OAuth2 — đừng nhầm hai khái niệm

- **Date:** 2026-07-30
- **Topic:** security
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

JWT và OAuth2 là cùng một thứ hay khác nhau? Khi nào dùng cái nào? Nếu chỉ làm API nội bộ cho SPA + mobile, bạn chọn giải pháp nào?

## 2) Giải thích đơn giản

| Thuật ngữ | Giống | Ví dụ đời thường |
|---|---|---|
| JWT (JSON Web Token) | Một tờ giấy có chữ ký, đưa đi đâu cũng xác minh được | Thẻ xe buýt đã đục lỗ — lên xe đưa ra là được |
| OAuth2 | Quy trình xin cấp quyền qua bên thứ ba | Cho bạn mượn chìa khóa nhà, nhưng có người gác cổng xác nhận |

- JWT = **token format** (cái tờ giấy).
- OAuth2 = **authorization framework** (quy trình xin cấp/quyền).
- OAuth2 flow thường **dùng JWT làm access token**, nhưng JWT không bắt buộc OAuth2.

## 3) Trả lời Middle+

JWT là một chuẩn mã hóa (RFC 7519) gồm 3 phần `header.payload.signature`, ký bằng HMAC hoặc RSA. Server ký token khi login, client đem đi mỗi request, server verify chữ ký → biết user là ai, không cần session state.

OAuth2 định nghĩa 4 grant type (Authorization Code, Client Credentials, Password, Implicit), chủ yếu giải quyết: ai được ủy quyền truy cập resource của user ở bên thứ ba, scope gì, trong bao lâu.

Với SPA + mobile + backend nội bộ: thường dùng **Authorization Code + PKCE** làm chuẩn, access token có thể là JWT ngắn hạn (15–30 phút), kèm refresh token.

## 4) Trả lời Senior

Trade-off phải nói rõ khi đi phỏng vấn senior:

| Vấn đề | Giải thích senior |
|---|---|
| Stateless JWT nhưng cần revoke | JWT không revoke được khi đã phát hành. Workaround: blacklist ngắn hạn trong Redis, hoặc rotation refresh token + short-lived access token (≤ 15 phút) |
| Token storage ở client | `localStorage` dễ bị XSS. Nên dùng HttpOnly + Secure + SameSite cookie cho refresh token |
| Algorithm confusion | Phải chỉ định rõ `alg` ở verifier, không cho phép `alg: none`. Lỗi kinh điển |
| Secret strength | HMAC secret phải ≥ 256 bit random. Tốt hơn: dùng RS256 với keypair, public key lưu ở resource server |
| Clock skew | Verify `exp` chấp nhận skew ~30s giữa các node |
| OAuth2 vs OpenID Connect | OAuth2 chỉ authorization, cần OpenID Connect (OIDC) nếu muốn có `id_token` xác minh danh tính user |
| Scope vs Role | Scope = quyền trên resource API. Role = quyền trong hệ thống. Hai thứ khác nhau |

Production stack gợi ý: **Authorization Server riêng** (Keycloak / Auth0 / Spring Authorization Server) phát JWT, resource server chỉ verify signature qua JWKS endpoint → có rotation key tự động.

## 5) Ví dụ code đơn giản

Verify JWT ở Spring Boot (cách modern):

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    SecurityFilterChain api(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(a -> a.anyRequest().authenticated())
            .oauth2ResourceServer(o -> o.jwt(j -> j.jwkSetUri("https://auth.example.com/.well-known/jwks.json")));
        return http.build();
    }
}
```

Không cần tự parse `header.payload.signature`. Spring Security xử lý JWKS rotation, expiry, scope mapping.

## 6) Follow-up / pitfall

- **Q:** JWT lưu ở `localStorage` có sao không?
  **A:** Có. XSS đọc được hết. Dùng cookie HttpOnly cho refresh token, access token ngắn hạn.
- **Q:** Tại sao access token nên ≤ 15 phút?
  **A:** Vì không revoke được. Token ngắn → cửa sổ lộ bị hẹp, kết hợp refresh token rotation để phát hiện reuse.
- **Pitfall:** Dùng JWT nhưng không set `exp` → token sống mãi. Hoặc set `exp` mà server tin tưởng `iat` từ client (đừng).
- **Pitfall:** Lưu sensitive data trong payload JWT → base64, không mã hóa, ai cũng đọc được.

## 7) 30 giây tóm tắt miệng

> JWT là **format token**, OAuth2 là **quy trình ủy quyền**. Hai thứ khác nhau nhưng hay đi cùng nhau: OAuth2 flow phát JWT làm access token. Với SPA + mobile nội bộ tôi chọn Authorization Code + PKCE, access token JWT 15 phút + refresh token xoay vòng trong HttpOnly cookie. Quan trọng nhất: secret/keys đủ mạnh, có revoke path (blacklist hoặc rotation), và không bao giờ lưu JWT trong localStorage.
