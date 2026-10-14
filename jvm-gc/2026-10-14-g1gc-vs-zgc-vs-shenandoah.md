# G1GC vs ZGC vs Shenandoah: Lựa chọn Garbage Collector cho Production

- **Date:** 2026-10-14
- **Topic:** jvm-gc
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Khi nào nên chọn G1GC, ZGC hay Shenandoah cho ứng dụng Java sản phẩm (production)? Sự khác biệt cốt lõi về latency (pause time) và throughput giữa chúng là gì?

## 2) Giải thích như trẻ lên 3

Tưởng tượng việc dọn dẹp nhà cửa:
- **G1GC:** Giống như chia nhà thành nhiều phòng nhỏ. Người dọn dẹp dừng mọi hoạt động trong một phòng ngắn hạn để dọn phòng bẩn nhất trước.
- **ZGC / Shenandoah:** Giống như robot hút bụi thông minh. Nó vừa chạy tự động quét dọn trong khi mọi người vẫn sinh hoạt và đi lại bình thường trong nhà mà không cần bắt mọi người phải đứng yên.

## 3) Giải thích đơn giản cho dev

- **G1GC (Garbage-First GC):** Là GC mặc định từ Java 9+. Chia Heap thành các Region nhỏ (1MB - 32MB). Lựa chọn Region có nhiều rác nhất để thu gom trước. Pause time mục tiêu tùy chỉnh qua `-XX:MaxGCPauseMillis` (thường 100ms - 200ms).
- **ZGC (Z Garbage Collector):** Low-latency GC (đã production-ready từ Java 15, Generational ZGC từ Java 21). Thiết kế cho Heap từ vài GB đến hàng TB với pause time cực thấp (< 1ms), không phụ thuộc vào kích thước Heap.
- **Shenandoah GC:** Low-pause GC giống ZGC (phát triển bởi Red Hat, hỗ trợ từ Java 12). Thực hiện công đoạn đánh dấu (mark) và di chuyển object (evacuation) song song (concurrent) với các application threads để giảm tối đa Stop-The-World (STW).

## 4) Ví dụ code đơn giản

Cấu hình GC qua tham số JVM options khi chạy ứng dụng:

```bash
# 1. Cấu hình G1GC (Default từ Java 9)
java -XX:+UseG1GC -XX:MaxGCPauseMillis=200 -jar app.jar

# 2. Cấu hình ZGC (Low latency < 1ms)
java -XX:+UseZGC -XX:+ZGenerational -jar app.jar

# 3. Cấu hình Shenandoah GC
java -XX:+UseShenandoahGC -jar app.jar
```

## 5) Trả lời Middle+

- **G1GC:** Thích hợp cho hầu hết các ứng dụng Web/Microservices thông thường có Heap từ vài GB đến ~32GB, đòi hỏi sự cân bằng tốt giữa throughput (hiệu năng xử lý) và pause time (khoảng 100-200ms).
- **ZGC:** Lựa chọn tối ưu cho các hệ thống yêu cầu độ trễ cực thấp (P99/P999 latency rất khắt khe như Fintech, Payment Gateways, Real-time Bidding) hoặc các hệ thống chạy Heap rất lớn (vài chục GB đến TB). Pause time luôn dưới 1ms.
- **Shenandoah:** Tương tự ZGC, tập trung giảm tối đa thời gian STW bằng cách evacuated concurrent. Rất phù hợp nếu dùng OpenJDK/Red Hat build trên các máy cấu hình vừa và lớn.

## 6) Trả lời Senior

- **Trade-off cốt lõi (Latency vs Throughput vs CPU footprint):**
  - **ZGC / Shenandoah** đánh đổi CPU overhead và Throughput tổng thể (thường giảm 2-5% throughput so với G1/Parallel GC) để đổi lấy Pause Time siêu thấp. Do GC thread chạy song song đồng thời với App thread (sử dụng Load Barriers / Colored Pointers), CPU phải chia sẻ năng lượng xử lý.
  - **Generational ZGC (Java 21+):** Giải quyết điểm yếu lớn nhất của ZGC cũ bằng cách phân chia Young/Old Generation. Giúp giảm tần suất GC cho các object ngắn hạn, tăng Throughput tiệm cận G1GC trong khi giữ Pause time < 1ms.
- **Production Failure Modes / Monitoring:**
  - **ZGC Allocation Stall:** Nếu tốc độ tạo object mới (Allocation Rate) quá nhanh so với tốc độ thu gom rác của ZGC, ứng dụng sẽ bị stall (đình trệ). Cần monitor `GC Phase Pause` và `Allocation Rate`.
  - **G1GC Humongous Allocation:** Các object lớn hơn 50% kích thước Region sẽ thành Humongous Object, gây phân mảnh Heap và dễ dẫn đến Full GC STW kéo dài.

## 7) Follow-up / pitfall

- **Pitfall 1:** Chuyển sang ZGC trên JVM cũ (trước Java 21) cho app có tốc độ tạo object cao nhưng quên cấp đủ CPU cores, dẫn đến ZGC không thu gom kịp gây Allocation Stall.
- **Pitfalls 2:** Đặt `-XX:MaxGCPauseMillis` quá thấp cho G1GC (ví dụ 10ms), khiến G1GC không đủ thời gian thu gom bớt Young Gen, dẫn đến tích tụ rác và nổ Full GC.
- **Question:** Generational ZGC trong Java 21 có gì cải tiến so với ZGC ban đầu?
  - *Trả lời:* ZGC ban đầu là Single-generation. Java 21 bổ sung Generational ZGC, tách riêng Young Gen để thu gom object ngắn hạn thường xuyên hơn, giúp giảm đáng kể CPU usage và hạn chế Allocation Stall.

## 8) 30 giây tóm tắt miệng

"Chọn GC dựa trên mục tiêu P99 latency và quy mô Heap: G1GC là lựa chọn an toàn, cân bằng tốt giữa throughput và pause time (~100ms) cho hầu hết ứng dụng web. Nếu hệ thống yêu cầu P99 latency siêu thấp (< 1ms) hoặc Heap rất lớn (hàng chục GB - TB), hãy chuyển sang ZGC (đặc biệt là Generational ZGC từ Java 21+) hoặc Shenandoah. Đánh đổi của ZGC/Shenandoah là tốn thêm một ít CPU footprint cho Load Barriers để giữ Pause Time siêu nhỏ."
