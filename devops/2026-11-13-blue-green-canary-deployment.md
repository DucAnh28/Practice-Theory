# Blue-Green Deployment và Canary Release: deploy không gián đoạn

- **Date:** 2026-11-13
- **Topic:** devops
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Deploy version mới lên production mà không downtime thì làm thế nào? **Blue-Green** khác **Canary** ở chỗ nào? Khi nào chọn cái nào, và làm sao rollback an toàn?

## 2) Giải thích như trẻ lên 3

Có 2 cách chuyển từ xe cũ sang xe mới:

- **Blue-Green**: có sẵn 2 bãi xe — bãi **xanh** (đang chạy) và bãi **xanh lá** (đã dựng sẵn). Chuyển hết khách sang bãi mới trong một lần, rồi bãi cũ giữ lại để lùi.
- **Canary**: lấy 1 phần nhỏ khách (5%) đi thử xe mới trước, thấy ổn mới cho từng phần nhỏ hơn chạy tiếp, cuối cùng mới chuyển hết.

## 3) Giải thích đơn giản cho dev

| | Blue-Green | Canary |
|---|---|---|
| Mô hình | 2 môi trường song song (blue/green), traffic đổi bằng 1 lần | Triển khai dần theo tỉ lệ traffic |
| Phạm vi rủi ro | Toàn bộ user chịu ảnh hưởng nếu bản mới lỗi | Chỉ nhóm nhỏ chịu ảnh hưởng |
| Tốn tài nguyên | Gấp đôi (chạy 2 bản song song) | Thấp hơn (thêm từng phần) |
| Rollback | Đổi traffic về bản cũ — nhanh, atomic | Giảm tỉ lệ dần hoặc bật feature flag tắt |
| Phát hiện bug | Muộn (sau khi đã 100%) | Sớm (sau 5% traffic) |
| Phù hợp | App không stateful, traffic đều | Sửa lỗi nhỏ, muốn thử từng bước |

- **Khác biệt cốt lõi**: Blue-Green = **chuyển đổi một lần** (switch), Canary = **chuyển dần theo tỉ lệ** (ramp-up).
- **Cả hai đều cần**: version mới **tương thích ngược** với version cũ (schema, API contract) và có **feature flag** để tắt tính năng mới mà không cần deploy lại.
- **Migration DB**: phải theo mẫu **expand–migrate–contract** — thêm cột mới (expand), migrate dữ liệu + dual-write/dual-read, rồi mới xoá cột cũ (contract) ở release sau.

## 4) Ví dụ code đơn giản

**Traffic split theo header (nginx) — phục vụ canary 5%:**

```nginx
upstream backend {
    server app-stable:8080;
    server app-canary:8080;
}

map $http_x_canary $upstream {
    default   "app-stable:8080";
    "1"       "app-canary:8080";
}

server {
    listen 80;
    location / {
        proxy_pass http://$upstream;
        proxy_set_header X-Canary $http_x_canary;
    }
}
```

**Blue-Green bằng service selector (Kubernetes):**

```bash
# service trỏ sang label track=blue; đổi label là switch traffic
kubectl patch service myapp -p '{"spec":{"selector":{"track":"green"}}}'
```

**Feature flag để tắt nhanh không cần deploy lại:**

```java
if (featureFlags.isEnabled("new-pricing", userId)) {
    pricingService.calculateV2(order);
} else {
    pricingService.calculate(order);
}
```

**Migrate DB theo expand–migrate–contract:**

```sql
-- Release 1 (expand): thêm cột mới, không xoá cột cũ
ALTER TABLE orders ADD COLUMN total_v2 DECIMAL(19,2) NULL;

-- backfill + dual-write ở application layer
-- Release 2 (contract): chỉ xoá sau khi 100% đọc cột mới và đã rollback window
ALTER TABLE orders DROP COLUMN total;
```

## 5) Trả lời Middle+

Tôi chọn chiến lược deploy theo rủi ro và tốc độ rollback mà bài toán yêu cầu:

- **Blue-Green** khi app không stateful và traffic lớn, cần switch nhanh. Tốn gấp đôi tài nguyên trong thời gian deploy nên cần cân nhắc capacity.
- **Canary** khi muốn giảm rủi ro và phát hiện bug sớm: bắt đầu 5% → 25% → 50% → 100%, mỗi bước theo dõi metric (error rate, latency p95/p99) trước khi tăng tiếp.
- **Cả hai đều cần rollback được**: giữ bản cũ sẵn sàng, và dùng **feature flag** để tắt logic mới ngay lập tức.
- **Migration schema phải tương thích ngược**: thêm cột mới trước, deploy code đọc được cả hai, xoá cột cũ ở release sau.

## 6) Trả lời Senior

Ở production, tôi xem đây là bài toán **risk control**, không chỉ là cách deploy:

- **Canary cần metric gate tự động**, không phải mắt thường. So sánh canary với baseline (traffic stable cùng thời điểm) theo error rate, p99 latency, business metric (tỉ lệ checkout thành công). Vượt ngưỡng → auto rollback, không cần người canh.
- **Canary theo user/cookie, không theo request ngẫu nhiên**: nếu ngẫu nhiên theo request, một user có thể bị đổi phiên bản giữa chừng và gặp lỗi. Gắn cố định theo `user_id` hash để trải nghiệm nhất quán và dễ điều tra.
- **Stateful service** (session lưu trong memory, WebSocket) cần **sticky session** hoặc migrate sang shared store trước khi làm canary, nếu không sẽ mất session và có lỗi rối lung.
- **DB migration là điểm rủi ro lớn nhất** vì không rollback được. Quy tắc: **mọi migration phải tương thích ngược ở cả hai chiều** trong suốt cửa sổ rollback; không bao giờ drop/rename cột ngay trong release deploy code. Có công cụ như Flyway/Liquibase với migration "expand–migrate–contract" và verify tự động.
- **Feature flag** có vòng đời: flag tạm phải được xoá sau khi ổn định, nếu không sẽ thành "flag debt" và nhánh code phình to.
- **Blue-Green với traffic shift ở tầng LB** (ALB weighted target group, Istio VirtualService) tốn capacity gấp đôi. Với hệ thống lớn nên hybrid: **canary nhỏ ở region/tenant nhỏ** trước, blue-green ở release lớn.
- **Observability**: dùng **version header/label** (`X-App-Version`, pod label) để metric tách được theo bản đang chạy. Không có nó thì dashboard gộp chung, không so sánh được canary với stable.
- **Database traffic**: cần connection pool riêng cho canary để không tranh chấp connection với bản stable, tránh canary fail vì hết pool.
- **Rollback không phải lúc nào cũng an toàn**: nếu release đã có side-effect dữ liệu không tương thích ngược, rollback code sẽ tạo dữ liệu sai. Khi đó rollback phải đi kèm script dữ liệu hoặc dùng forward-fix.

## 7) Follow-up / pitfall

- **Pitfall 1**: deploy bản mới rồi mới đổi schema theo kiểu "drop column" → bản cũ đọc lỗi ngay khi rollback. Luôn expand trước, contract sau.
- **Pitfall 2**: canary chia traffic ngẫu nhiên theo request → user gặp nhiều version trong 1 phiên, sinh lỗi khó tái hiện.
- **Pitfall 3**: tăng tỉ lệ canary theo cảm giác, không có metric gate → lỗi lan sang 100% rồi mới phát hiện.
- **Pitfall 4**: blue-green "quay đầu" an toàn nhưng **không** an toàn với migration đã chạy trên dữ liệu — cần dual-read/dual-write.
- **Pitfall 5**: bản canary dùng chung connection pool với stable → hết connection, làm sập cả bản đang chạy tốt.
- **Pitfall 6**: feature flag không dọn → code nhiều nhánh, mới sửa 1 nhánh lại sót nhánh cũ.
- **Pitfall 7**: rollback dựa vào "restart pod" khi image đã bị xoá khỏi registry → không có image để pull. Giữ image bản cũ theo retention policy.
- **Follow-up**: "Blue-Green hay Canary?" → Canary nếu muốn giảm rủi ro và phát hiện lỗi sớm với traffic nhỏ; Blue-Green nếu cần switch atomic, app stateless, chấp nhận nhân đôi capacity. Nhiều team dùng cả hai: canary nhỏ trước, blue-green khi release lớn.
- **Follow-up**: "Làm sao đảm bảo DB không bị bottleneck lúc switch?" → Dùng read replica, chuẩn bị connection pool riêng, đo lag replica trước khi cutover, và giữ nguyên traffic read ở bản cũ trong thời gian ngắn.

## 8) 30 giây tóm tắt miệng

"Để deploy không downtime, tôi chọn giữa hai chiến lược: Blue-Green dựng sẵn môi trường mới rồi chuyển traffic một lần — switch atomic, rollback nhanh, nhưng tốn gấp đôi capacity; Canary cho 5% traffic trước rồi tăng dần theo metric — phát hiện bug sớm, giảm rủi ro, nhưng cần rollout từng bước. Điểm khó nhất không phải cách deploy mà là **migration database**: phải theo expand–migrate–contract để mọi release tương thích ngược, vì rollback code không cứu được dữ liệu đã đổi. Với canary tôi luôn chia traffic theo user chứ không theo request, và có metric gate tự động để rollback khi error rate hoặc p99 vượt ngưỡng."
