# Redis Data Structures nâng cao: Sorted Set, Bitmap, HyperLogLog

- **Date:** 2026-11-04
- **Topic:** redis
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

> "Ngoài String/Hash/List/Set, Redis còn có Sorted Set (ZSET), Bitmap và HyperLogLog. Khi nào dùng từng cái? So sánh độ chính xác và chi phí bộ nhớ với Set thường?"

## 2) Giải thích như trẻ lên 3

- **Sorted Set**: giống bảng xếp hạng có điểm. Mỗi người có tên và điểm, lấy top 3 thì không cần xếp lại tay.
- **Bitmap**: một dãy 0/1 đánh dấu ai đã xem. Ngày nào có 1 triệu người dùng thì chỉ cần 1 triệu bit, rất nhỏ.
- **HyperLogLog**: máy đếm "số lượng khác nhau" xấp xỉ. Đếm 1 triệu user khác nhau vẫn chỉ tốn vài chục KB.

## 3) Giải thích đơn giản cho dev

| Cấu trúc | Dùng khi | Chính xác | Bộ nhớ |
|---|---|---|---|
| `SET` | Tag, thành viên không trùng | Chính xác 100% | Cao, mỗi phần tử ~50-80 byte |
| `ZSET` | Leaderboard, xếp hạng theo điểm/thời gian, top-K | Chính xác 100% | Cao hơn Set, mỗi member ~80-100 byte |
| `BITMAP` | Đánh dấu boolean theo user/day (feature flag, check-in, DAU) | Chính xác 100% | Rẻ nhất: 1 bit / người, 1 triệu user ≈ 128 KB |
| `HLL` | Ước lượng số lượng unique (UV, distinct IP) | Xấp xỉ, sai số ~0.81% | Cố định ~12 KB không tăng theo số lượng |

Điểm mấu chốt: HLL **không lưu danh sách**, chỉ lưa trạng thái đếm → cần biết "user nào" thì dùng Set/ZSET, không dùng HLL.

## 4) Ví dụ code đơn giản

```java
// ZSET: leaderboard, top 10 theo điểm giảm dần
redis.opsForZSet().add("rank:daily", "user:1", 120.0);
redis.opsForZSet().add("rank:daily", "user:2", 300.0);
var top3 = redis.opsForZSet().reverseRange("rank:daily", 0, 2); // [user:2, user:1, ...]
var rankOfUser1 = redis.opsForZSet().reverseRank("rank:daily", "user:1"); // 1
redis.opsForZSet().incrementScore("rank:daily", "user:1", 10.0);

// BITMAP: đánh dấu user đã xem bài viết trong ngày
long day = 20261104L;
redis.opsForValue().setBit("read:20261104", userId, true);
boolean seen = redis.opsForValue().getBit("read:20261104", userId);
// DAU ngày = số bit đang bật
Long dau = redis.opsForValue().bitCount("read:20261104");

// BITOP: user nào hoạt động cả 7 ngày liên tiếp
redis.opsForValue().bitOp(BitOp.AND, "active:7d", "read:20261104", "read:20261103", "read:20261102");
redis.expire("active:7d", Duration.ofDays(30));

// HYPERLOGLOG: ước lượng unique visitor
redis.opsForHyperLogLog().add("uv:20261104", "192.168.1." + rand);
long approxUv = redis.opsForHyperLogLog().size("uv:20261104"); // xấp xỉ
```

## 5) Trả lời Middle+

ZSET dùng cho leaderboard vì score được index sẵn, `ZRANGE`/`ZREVRANGE` lấy top-K O(log N + K), không cần sort toàn bộ. Điểm phải nhớ: member phải unique, nhiều user cùng điểm thì order phụ thuộc lexicographic của member — muốn ổn định thì encode timestamp vào member. Bitmap dùng khi cần đếm/dedupe theo user theo ngày và số lượng lớn; nhớ rằng offset lớn (user id = 1e9) làm key phình to, nên dùng user id thưa hoặc hash. HLL dùng để đếm unique gần đúng, tiết kiệm rất nhiều RAM so với Set.

## 6) Trả lời Senior

- **ZSET**: mỗi member là một entry trong skiplist + dict, `O(log N)` insert/update, `O(log N + K)` range. Với leaderboard > vài triệu user, cân nhắc shard theo range score (bucket theo điểm) để giảm hot key. Cần biết `ZRANGEBYSCORE` + `ZREMRANGEBYRANK` để trim bảng xếp hạng cũ, và TTL để key tự dọn.
- **Bitmap**: offset = user id nên key phình theo id lớn nhất, không theo số user thật. Fix: dùng id thưa hoặc hash 1-1 về index nhỏ, hoặc dùng `SETBIT` trên cảnh giác. `BITCOUNT` trên key rất rộng là O(N/8) → có thể block event loop; dùng `BITPOS`/`BITCOUNT` giới hạn byte range.
- **HLL**: độ chính xác ~0.81% (standard error), dùng `PFADD`/`PFCOUNT`/`PFMERGE`. Không merge cùng key giữa các phiên bản thuật toán khác nhau. Nếu cần exact thì chấp nhận cost của Set, hoặc dùng HLL + bảng chính xác cho phần nghiệp vụ quan trọng.
- **Ops**: phân tách instance riêng cho HLL/Bitmap khỏi cache chính vì dễ consume CPU. Bật `maxmemory-policy` phù hợp và theo dõi `memory_usage`, `keyspace`, latency p99.

## 7) Follow-up / pitfall

- **ZSET score = double**: dùng cho số phân thập phân, mất chính xác; cần số nguyên lớn thì scale thủ công hoặc dùng `ZRANGEBYLEX`.
- **ZSET memory**: member dài làm phình entry; encode member gọn (id thay vì object JSON).
- **Tie-break không ổn định**: cùng điểm order có thể đổi giữa các lần đọc → thêm timestamp vào member.
- **Bitmap offset rỗng**: user id gián đoạn (1, 1000, 100000) → key to hơn dự kiến rất nhiều.
- **BITCOUNT trên key khổng lồ**: chặn Redis, nên chạy off-peak hoặc giới hạn range.
- **HLL không lấy được danh sách member**: cần "user nào" thì HLL vô dụng.
- **PFMERGE** trộn dữ liệu của các phiên khác nhau → số unique sai lệch nếu không cùng cohort.
- **Không dùng HLL để kiểm tra "đã tồn tại"** → false positive dù chỉ 0.81%.

## 8) 30 giây tóm tắt miệng

"Redis có 4 cấu trúc hay hỏi: Set cho membership chính xác, ZSET khi cần xếp hạng theo score, Bitmap khi cần đánh dấu boolean theo user theo ngày, và HyperLogLog khi chỉ cần ước lượng số unique. ZSET cho leaderboard top-K O(log N + K) không cần sort; Bitmap rẻ nhất, 1 bit mỗi user; HLL cố định ~12 KB và sai số ~0.81%. Production cần chú ý ZSET memory và tie-break, Bitmap offset rỗng và BITCOUNT trên key rộng, và tách HLL/Bitmap sang instance riêng vì chúng ngốn CPU."
