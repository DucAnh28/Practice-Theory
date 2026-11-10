# Lưu mật khẩu an toàn — Hash, Salt, BCrypt/Argon2

- **Date:** 2026-11-10
- **Topic:** security
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Làm sao lưu mật khẩu người dùng mà vẫn an toàn nếu DB bị lọt? Hash (one-way) khác mã hóa (two-way) ở chỗ nào? Vì sao cần salt? BCrypt và Argon2 khác nhau thế nào, khi nào nên dùng?

## 2) Giải thích như trẻ lên 3

- **Mã hóa (encryption)**: khóa và nhặt. Có khóa là mở ra xem được → hai chiều.
- **Hash (băm)**: bỏ vào máy xay, ra một mảnh nát không thể lộn ngược thành mìn → một chiều. Đăng nhập thì xay mật khẩu người dùng vừa gõ rồi so với mảnh nát đã lưu.
- **Salt**: rắc muối vào trước khi xay. Hai người cùng mật khẩu "123456" ra hai mảnh nát khác nhau → kẻ trộm có bảng tra sẵn cũng không dùng được.
- **BCrypt/Argon2**: máy xay xay chậm có chủ đích, tốn nhiều thời gian/CPU → làm brute-force tốn kém.

## 3) Giải thích đơn giản cho dev

- **Không bao giờ** lưu mật khẩu dạng rõ (plaintext) hoặc mã hóa có thể đảo ngược (AES). Mất DB = mất toàn bộ mật khẩu.
- **Hash cố định (SHA-256/MD5) là sai**: quá nhanh, GPU hash hàng tỷ mật khẩu/giây → brute-force rẻ.
- **Salt**: chuỗi ngẫu nhiên duy nhất cho từng user, lưu cạnh hash. Salt không cần bí mật, chỉ cần duy nhất.
- **Stretching / work factor**: hash chậm có chủ đích, có **cost factor** để tăng dần theo phần cứng mạnh hơn.
  - **BCrypt**: mặc định cost 10, đo bằng **RAM** (≈ 4 MB × 2^cost), tự mang theo salt + cost trong chuỗi hash.
  - **Argon2id**: dùng cả RAM + CPU, khuyến nghị hiện nay (RFC 9106), chống GPU/ASIC tốt hơn, có thể tune (memory/time/parallelism).
- **Verify bằng `matches`/`verify`**, không so sánh chuỗi hash bằng `equals` (dễ timing attack, và mỗi user một salt khác nhau).
- **Chiến lược khác**: dùng **Password Hashing (pepper)**: secret lưu ở KMS/secret manager, không nằm trong DB → kẻ lấy được DB vẫn không hash được.

## 4) Ví dụ code đơn giản

```java
// Spring Security: bean duy nhất để cấu hình hash mật khẩu
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(12); // cost 12 -> ~250ms/verify
}

// Argon2id (PasswordEncoder tự cấu hình tham số)
@Bean
public PasswordEncoder argon2Encoder() {
    return Argon2PasswordEncoder.defaultsForSpringSecurity_v5_8();
}

// Đăng ký: chỉ lưu hash, không bao giờ log mật khẩu
public void register(String rawPassword) {
    String hash = encoder.encode(rawPassword);
    userRepository.save(new User(hash)); // salt đã nằm trong chuỗi hash
}

// Đăng nhập: verify, không tự so sánh
public boolean check(String rawPassword, String storedHash) {
    return encoder.matches(rawPassword, storedHash);
}
```

```sql
-- BCrypt lưu dạng: $2a$12$<22 ký tự salt><31 ký tự hash>
-- Argon2id lưu dạng: $argon2id$v=19$m=65536,t=3,p=2$<salt>$<hash>
-- => luôn parse tham số từ chính chuỗi hash, đừng hard-code cost ở code
```

## 5) Trả lời Middle+

- Lưu mật khẩu bằng **hash + salt**: `BCryptPasswordEncoder` (cost 10–12) hoặc `Argon2PasswordEncoder`; kiểm tra bằng `matches()`.
- Salt tự sinh theo user, lưu cùng hash → không cần cột riêng, không cần migration khi đổi thuật toán.
- Không bao giờ tự viết vòng hash bằng `MessageDigest`; luôn dùng thư viện có sẵn trong Spring Security.
- **Rehash khi login**: nếu `matches` trả về `true` nhưng tham số của stored hash cũ yếu hơn mức hiện hành → encode lại và update. Đây là cách nâng cost từ từ mà không khóa user.

## 6) Trả lời Senior

- **Chọn thuật toán theo mô hình đe doạ**: Argon2id (memory-hard) chống GPU/ASIC tốt hơn; BCrypt vẫn ổn nhưng tốn RAM chứ không chịu được tăng nhiều. Argon2id cần tune theo RAM của server (`m`, `t`, `p`), oversubscribe CPU thì throughput sập.
- **Đặt ngân sách hash theo tài nguyên, không theo cảm tính**: cost phải được chọn sao cho p99 verify < vài trăm ms và vẫn chịu được traffic đăng nhập hàng loạt; đo bằng benchmark trên chính instance thật.
- **Login endpoint là mục tiêu tấn công số 1**: không tạo user mới khi tài khoản không tồn tại (user enumeration), trả lỗi chung "sai tài khoản hoặc sai mật khẩu", thêm rate limit + lockout có backoff, và đừng dùng lockout cứng (DoS account) — dùng exponential backoff hoặc CAPTCHA.
- **Pepper**: nếu thêm pepper thì phải xử lý rotation (pepper cũ / mới song song) và không được lưu cùng chỗ với DB.
- **Migrating hash**: đừng migrate hàng loạt offline cả DB (nghẽn + rủi ro). Chiến lược chuẩn: hash theo format hiện tại, đổi sang format mới **khi user đăng nhập** (lazy rehash).
- **Observability**: đo tỷ lệ login fail, số account bị khóa, thời gian verify p99; alert khi spike (dấu hiệu credential stuffing). Không log mật khẩu/token; mask PII trong log.
- **Yêu cầu mật khẩu**: dùng chính sách dài hơn thay vì phức tạp (NIST 800-63B), hỗ trợ passkey/WebAuthn, MFA cho admin, và cơ chế quên mật khẩu với **token một lần, hạn ngắn, không tiết lộ user tồn tại**.

## 7) Follow-up / pitfall

- **Pitfall 1**: lưu plaintext hoặc mã hóa AES có thể giải mã → mất DB là mất tất cả.
- **Pitfall 2**: dùng MD5/SHA-1/SHA-256 không có work factor → bị rainbow table / GPU brute-force.
- **Pitfall 3**: dùng salt chung cho tất cả user → 1 bảng tra precompute đánh bại hết.
- **Pitfall 4**: so sánh hash bằng `String.equals` → sai logic (mỗi user salt khác) và dễ rò timing.
- **Pitfall 5**: tăng cost quá cao → đăng nhập chậm, CPU cháy, tăng rủi ro DoS login.
- **Pitfall 6**: reset mật khẩu bằng cách gửi link tĩnh → bị dùng lại nhiều lần.
- **Follow-up**: hai người có cùng mật khẩu, DB lộ, hacker có biết họ dùng chung mật khẩu không? (Không — salt ngẫu nhiên + hash chậm làm 2 hash giống nhau là ngẫu nhiên; nhưng vẫn nên có breach check như Have I Been Pwned.)
- **Follow-up**: tăng cost lên gì? Cần rehash dần — dùng `upgradeEncoding` của Spring Security.

## 8) 30 giây tóm tắt miệng

"Mật khẩu tôi chỉ lưu dạng hash một chiều kèm salt riêng cho từng user, dùng thuật toán chậm có chủ đích như BCrypt hoặc Argon2id chứ không dùng SHA-256 trần. Verify bằng `matches` của PasswordEncoder, và nếu tham số cũ yếu hơn thì rehash lúc user đăng nhập để nâng cost dần. Tôi cũng chặn user enumeration, rate limit endpoint đăng nhập và dùng MFA cho tài khoản admin."
