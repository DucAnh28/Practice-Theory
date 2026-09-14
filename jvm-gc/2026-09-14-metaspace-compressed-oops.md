# JVM Memory Pool: Metaspace vs Compressed Oops

- **Date:** 2026-09-14
- **Topic:** jvm-gc
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

**Metaspace là gì và khác gì PermGen? Compressed Oops hoạt động như thế nào và tại sao giúp tiết kiệm RAM cho ứng dụng 64-bit?**

---

## 2) Giải thích như trẻ lên 3

- **PermGen (ngày xưa):** Là chiếc cặp sách cố định kích thước. Đựng đầy sách giáo khoa (metadata của class) là bị ném lỗi `OutOfMemoryError` dù trong phòng còn nhiều chỗ trống.
- **Metaspace (bây giờ):** Là ba lô co giãn theo dung lượng RAM của máy tính. Cần chỗ nào, hệ điều hành cấp chỗ đó, không lo bị bóp cứng một cục cố định.
- **Compressed Oops:** Giống như viết tắt tên đường trên bản đồ. Thay vì viết địa chỉ 64-bit cực dài tốn giấy, ta nén lại thành 32-bit (dùng offset từ pointer), tiết kiệm một nửa bộ nhớ RAM cho các reference trỏ tới object.

---

## 3) Giải thích đơn giản cho dev

- **Metaspace:** Lưu trữ class metadata, method structures, constant pool. Từ Java 8, PermGen bị xóa bỏ hoàn toàn, Metaspace dùng native memory (RAM hệ thống) thay vì heap, tự động co giãn (`-XX:MaxMetaspaceSize` để giới hạn tránh tràn RAM).
- **Compressed Ordinary Object Pointers (Compressed Oops):** Trên kiến trúc 64-bit, con trỏ chiếm 8 bytes. Nhờ Compressed Oops (bật mặc định khi heap < 32GB), JVM nén con trỏ xuống 4 bytes bằng cách shift right 3-bit (địa chỉ align 8-byte), giúp tiết kiệm ~30-50% heap footprint và tăng cache locality.

---

## 4) Ví dụ code đơn giản

```java
public class CompressedOopsDemo {
    public static void main(String[] args) {
        // Kiểm tra JVM options liên quan
        // Chạy với: java -XX:+PrintFlagsFinal -version | grep UseCompressedOops
        System.out.println("Java Heap Max: " + Runtime.getRuntime().maxMemory() / (1024 * 1024) + " MB");
    }
}
```

---

## 5) Trả lời Middle+

- **Metaspace Management:** Tránh lỗi `java.lang.OutOfMemoryError: PermGen space`. Tuy nhiên, nếu dynamic class loading (như Spring CGLIB proxy, Groovy/JSP compilation) bị leak (tạo class liên tục không unload), Metaspace vẫn sẽ tràn và ăn hết RAM hệ thống.
- **Compressed Oops Constraints:** Hoạt động tốt nhất khi max heap dưới 32GB. Nếu heap vượt quá 32GB, pointer không thể shift 8-byte alignment được nữa (vượt quá 4GB * 8 = 32GB address space), JVM buộc phải chuyển sang uncompressed 64-bit pointers, khiến memory footprint tăng vọt dù data không đổi.

---

## 6) Trả lời Senior

- **Production Tuning & Leak Diagnosis:**
  - Cấu hình `-XX:MetaspaceSize` và `-XX:MaxMetaspaceSize` hợp lý để tránh OS thrashing khi Metaspace grow liên tục.
  - Leak Metaspace thường do classloader leak (ví dụ hot-reload webapp, OSGi bundles, hoặc thư viện third-party tự tạo dynamic proxy vô hạn). Dùng `jcmd <pid> GC.class_stats` hoặc dump heap/metaspace phân tích bằng Eclipse MAT.
- **Compressed Klass Pointers:** Đi đôi với Compressed Oops, Java 8+ còn nén Klass Word trong object header xuống 32-bit (`-XX:+UseCompressedClassPointers`).
- **Zero-based Compressed Oops:** JVM cố gắng map heap bắt đầu từ địa chỉ 0 (`-XX:+UseCompressedOops`) để tối ưu assembly instruction (bỏ qua bước shift addition), giảm CPU overhead khi dereference pointer.

---

## 7) Follow-up / pitfall

- **Pitfall:** Set `-Xmx32g` tròn trĩnh và thấy RAM phình to bất thường. Nguyên nhân: chạm ngưỡng 32GB khiến Compressed Oops bị vô hiệu hóa (pointer dãn thành 8 bytes). Nên đặt heap dưới 32GB (ví dụ 31GB) hoặc lên hẳn 48GB+ nếu RAM server lớn và chấp nhận trade-off memory footprint.
- **Pitfall:** Dynamic proxy generation trong Spring/Hibernate không đóng classloader cũ gây Metaspace OOM.

---

## 8) 30 giây tóm tắt miệng

> "Metaspace thay thế PermGen từ Java 8, lưu class metadata trên native memory và tự động co giãn, tránh OOM tĩnh nhưng vẫn có thể leak nếu sinh class liên tục. Compressed Oops nén con trỏ 64-bit thành 32-bit khi heap dưới 32GB, tiết kiệm đáng kể RAM và tối ưu CPU cache. Vượt ngưỡng 32GB sẽ mất tính năng nén này."
