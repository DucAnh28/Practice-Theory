# OutOfMemoryError: Phân biệt các loại OOM và cách chẩn đoán memory leak

- **Date:** 2026-11-06
- **Topic:** jvm-gc
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

"App bị `OutOfMemoryError` — bạn phân biệt được OOM nào là do Heap, Metaspace, Direct Buffer hay thread? Trong production, quy trình chẩn đoán memory leak của một JVM service thế nào?"

---

## 2) Giải thích như trẻ lên 3

RAM của máy giống một cái vali có nhiều ngăn:

- **Ngăn đồ (Heap):** chỗ chứa mọi đồ vật (`new Object()`). Đầy ngăn này thì báo hết chỗ.
- **Ngăn dụng cụ (Metaspace):** chỗ chứa "hướng dẫn sử dụng" của từng loại đồ (class metadata). Tải app vô, lại tải thêm hàng loạt loại đồ mới → ngăn dụng cụ đầy dù ngăn đồ còn trống.
- **Ngăn ngoài (Direct buffer):** chỗ để vẽ ra ngoài, không tính vào Heap.
- **Chỗ đứng (Thread stack):** mỗi người đứng cần chỗ chân. Đông quá thì không còn ai đứng được → báo "không tạo được người mới".

Memory leak = đồ bị bỏ quên trong vali, không ai dọn, vali đầy dần theo thời gian.

---

## 3) Giải thích đơn giản cho dev

Các biến thể OOM thường gặp — đọc message là biết vùng nào bị đầy:

| Message | Vùng nhớ bị đầy | Nguyên nhân thường |
|---|---|---|
| `OutOfMemoryError: Java heap space` | Heap | Giữ quá nhiều object sống lâu (leak) hoặc 1 request xử lý quá nhiều data |
| `OutOfMemoryError: GC overhead limit exceeded` | Heap | GC chạy liên tục nhưng thu hồi được <2% → thường là **dấu hiệu leak**, không phải thiếu RAM |
| `OutOfMemoryError: Metaspace` | Native memory | Classloader leak, dynamic proxy sinh liên tục (hot-reload, Groovy/JSP, CDI proxy) |
| `OutOfMemoryError: Direct buffer memory` | Off-heap | Netty/direct `ByteBuffer` không release, `maxDirectMemorySize` quá thấp |
| `OutOfMemoryError: unable to create new native thread` | OS/thread | Hết thread, vượt `ulimit -u` hoặc cgroup `pids.max` trong container |
| `Cannot reserve enough space for object heap` | OS | JVM không xin được vùng heap liên tục (swap/overcommit) |

Ba khái niệm hay bị nhầm:

- **Memory pressure**: dùng nhiều RAM nhưng vẫn đúng, GC đi lại và trả lại bộ nhớ → tăng `-Xmx` là hợp lý.
- **Memory leak**: tập object *được giữ lại* (retained) tăng đều theo thời gian, GC không thu hồi được → tăng `-Xmx` chỉ trì hoãn.
- **Unbounded cache**: vốn là "leak" có chủ đích — cần size bound + TTL + metric.

---

## 4) Ví dụ code đơn giản

Quy trình chẩn đoán bằng công cụ đi kèm JDK (không cần thêm dependency):

```bash
# Bật sẵn trước khi lên production: dump heap khi OOM
java -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/var/log/app/heap.hprof \
     -XX:+ExitOnOutOfMemoryError \
     -Xlog:gc*:file=/var/log/app/gc.log:time,level,tags:filecount=5,filesize=20M \
     -jar app.jar

# Xem tình trạng heap lúc đang chạy
jcmd <pid> GC.heap_info

# Top object theo số lượng instance (chẩn đoán nhanh, không cần dừng app)
jcmd <pid> GC.class_histogram

# Bật JFR (Java 11+) để soi allocation theo call-site trong 2 phút
jcmd <pid> JFR.start name=diag settings=profile duration=120s filename=/var/log/app/diag.jfr
jcmd <pid> JFR.summary diag
```

Trong Eclipse MAT mở `heap.hprof` → **Leak Suspects** → **Dominator Tree** → chọn object chiếm nhiều nhất → **Path to GC Roots** (bỏ qua `Weak Reference`/`Soft Reference`) → đọc ngược để tìm chỗ giữ sai.

---

## 5) Trả lời Middle+

Đọc message OOM để xác định vùng nhớ bị đầy, rồi đi theo 3 bước:

1. **Xem GC log**: nếu heap đầy liên tục, allocation rate cao nhưng `used` sau GC không giảm → leak. Nếu `used` sau GC thấp mà vẫn OOM → cấp phát một lần quá lớn (ví dụ đọc file 2GB vào `byte[]`).
2. **Chụp heap dump** (`-XX:+HeapDumpOnOutOfMemoryError` bật trước) → phân tích bằng MAT: Dominator Tree xem object nào giữ nhiều nhất, Path to GC Roots tìm chỗ giữ.
3. **Đối chiếu code**: các nghi phạm kinh điển là `static` collection không giới hạn, cache không có TTL/size bound, listener/subscriber không `unsubscribe`, `ThreadLocal` không `remove()`, `ClassLoader` không đóng.

Với OOM do Metaspace → tìm classloader leak. Với `unable to create new native thread` → đếm thread (`jcmd <pid> Thread.print | grep -c "^\""`) và kiểm tra giới hạn OS.

---

## 6) Trả lời Senior

**Chẩn đoán dựa trên metric, không dựa trên cảm giác:**

- Expose `jvm_memory_bytes_used{area="heap"}`, `jvm_memory_bytes_max`, `jvm_gc_pause_seconds`, `jvm_threads_live` (Micrometer/Prometheus). Cảnh báo theo **retained heap sau GC** tăng tuyến tính theo thời gian, không theo đỉnh.
- Phân biệt 3 dạng qua biểu đồ: `used after GC` phẳng + `used` s sawtooth cao = allocation pressure; `used after GC` leo dần = leak; `used after GC` leo rồi đứng = cache size chưa ổn định.

**Trong container:**

- Dùng `-XX:MaxRAMPercentage=70` (không hard-code `-Xmx`) để JVM tự tính theo limit của cgroup.
- Tổng RAM của container phải > heap + metaspace + direct buffer + thread stacks + code cache + native (malloc arena của glibc, off-heap của Netty). `MALLOC_ARENA_MAX=2` giảm RSS do glibc cấp nhiều arena.
- `OutOfMemoryError` và `OOMKilled` là hai thứ khác nhau: OOMKilled = kernel cưỡng chế giết process (dễ xảy ra khi RSS vượt limit do native memory) → phải theo dõi cgroup memory, không chỉ heap.
- `-XX:+ExitOnOutOfMemoryError` để container restart sạch thay vì process treo trong trạng thái hỏng.
- `HeapDumpOnOutOfMemoryError` ghi file — file dump có thể vài GB, ghi vào ephemeral storage của container sẽ làm đầy disk và giết container. Ghi ra volume riêng, giới hạn quota, và phải có quy trình dọn/xoá dump sau khi phân tích.

**Rủi ro bảo mật:** heap dump chứa dữ liệu nhạy cảm (token, PII, mật khẩu trong object). Phải mã hoá khi lưu, hạn chế quyền truy cập, xoá sau khi dùng; không upload lên repo hay service công khai.

**Vận hành:** heap dump lúc OOM làm JVM đứng yên trong lúc ghi → downtime kéo dài hàng chục giây đến vài phút với heap lớn. Với service nhạy cảm, cân nhắc chụp dump định kỳ theo chu kỳ (JFR profile) thay vì chờ tới lúc OOM.

---

## 7) Follow-up / pitfall

**Q:** "Tăng `-Xmx` có chữa được memory leak không?"

**A:** Không. Leak là retained set tăng theo thời gian, tăng heap chỉ đổi lúc chết. Chỉ tăng `-Xmx` khi thấy `used after GC` thấp (allocation pressure).

**Q:** "`GC overhead limit exceeded` nghĩa là GC chạy quá nhiều?"

**A:** Gần đúng nhưng dễ hiểu sai. Nó nghĩa là GC đã cố rất nhiều mà thu hồi được rất ít (thường <2% heap mỗi lần) → không có lối thoát nên JVM tự dừng. Đây là triệu chứng của leak hoặc live set quá lớn.

**Q:** "Dùng `WeakReference`/`SoftReference` để tránh leak?"

**A:** Giảm rủi ro nhưng không miễn nhiễm: cache dựa trên `SoftReference` không có bound → vẫn phình đến khi hết RAM. Cache phải bounded + TTL + metric. `WeakHashMap` cần null-check key, nếu không entry bị clear sớm.

**Pitfall hay gặp:**

- `ThreadLocal` lưu object theo request trong thread pool mà không `remove()` trong `finally` → thread sống mãi giữ object → leak dần theo số request.
- `static` map cache không giới hạn → OOM sau vài giờ dù mỗi entry nhỏ.
- `System.gc()` trong code để "giải quyết" OOM → che triệu chứng, tăng pause, không giải quyết leak.
- Đọc file lớn / `Files.readAllBytes()` vào RAM → một lần xin heap liên tục, GC không cứu được. Dùng streaming.
- Ghi heap dump vào overlay filesystem của container → đầy disk, container chết với lỗi khác, mất dấu vết OOM.
- Chỉ monitor `jvm_memory_bytes_used` (không tách used sau GC) → báo động sai lúc GC đang chạy.

---

## 8) 30 giây tóm tắt miệng

"OutOfMemoryError phải phân loại theo message: Java heap space là đầy heap, GC overhead limit exceeded thường là dấu hiệu leak chứ không phải thiếu RAM, Metaspace là classloader leak, Direct buffer memory là off-heap chưa release, còn unable to create new native thread là vấn đề thread/OS limit. Quy trình chẩn đoán: xem GC log để biết used-after-GC có leo dần không, bật sẵn HeapDumpOnOutOfMemoryError, mở bằng MAT → Dominator Tree → Path to GC Roots để tìm chỗ giữ object. Ở tầng production thì monitor retained heap sau GC, dùng MaxRAMPercentage thay vì Xmx cố định, tính cả native memory trong limit container, phân biệt OOM với OOMKilled, và luôn coi heap dump là dữ liệu nhạy cảm cần mã hoá rồi xoá."
