# Spring Bean scope và thread-safety

- **Date:** 2026-08-10
- **Topic:** java-spring
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Spring Bean mặc định là scope gì? Vì sao singleton bean có thể gây lỗi thread-safety trong ứng dụng web?

## 2) Giải thích như trẻ lên 3

Tưởng tượng cả lớp dùng chung một cây bút.

Nếu ai cũng chỉ đọc nhãn cây bút thì ổn. Nếu nhiều bạn cùng lúc ghi tên mình lên cây bút, tên sẽ bị ghi đè lung tung.

Spring singleton bean cũng vậy: nhiều request dùng chung một object. Object đó an toàn nếu không giữ dữ liệu thay đổi theo từng request.

## 3) Giải thích đơn giản cho dev

Mặc định Spring Bean là **singleton**: mỗi bean chỉ có một instance trong `ApplicationContext`.

Singleton bean không tự động thread-safe. Nó chỉ an toàn khi:

- Không có mutable state theo request/user.
- Dependency bên trong cũng thread-safe hoặc stateless.
- Biến local trong method được tạo riêng cho mỗi thread.

Không an toàn khi bean có field kiểu `currentUser`, `lastOrder`, `StringBuilder`, `Map`... và nhiều request cùng ghi vào field đó.

## 4) Ví dụ code đơn giản

```java
@Service
class OrderService {
    private String currentUser; // sai: state dùng chung giữa request

    public void createOrder(String userId) {
        this.currentUser = userId;
        // request khác có thể ghi đè currentUser ở đây
        saveOrderFor(currentUser);
    }
}
```

Cách tốt hơn:

```java
@Service
class OrderService {
    public void createOrder(String userId) {
        saveOrderFor(userId); // state nằm trong biến local/tham số
    }
}
```

## 5) Trả lời Middle+

Spring Bean mặc định là singleton, nghĩa là một instance được dùng lại. Singleton không đồng nghĩa thread-safe. Nếu bean stateless, chỉ dùng biến local và dependency an toàn, thì dùng tốt trong web app. Nếu lưu dữ liệu request vào field, nhiều thread có thể ghi đè nhau và sinh bug khó đoán.

## 6) Trả lời Senior

Trong production, tôi tránh mutable state trong singleton service. State theo request nên đi qua method parameter, DTO, database transaction, hoặc request-scoped bean khi thật sự cần. Nếu phải dùng shared state, cần cơ chế đồng bộ hoặc concurrent collection, nhưng phải cân nhắc contention, memory leak, lifecycle và test concurrency. Với Spring, cần cẩn thận cả proxy như `@Transactional`: self-invocation và scope proxy có thể làm hành vi khác kỳ vọng.

## 7) Follow-up / pitfall

- `@Service` singleton có field `List` cache tạm thời có an toàn không? Không, nếu nhiều thread cùng mutate.
- `prototype` scope có giải quyết mọi thread-safety không? Không, nếu object vẫn bị share thủ công hoặc dependency bên trong không an toàn.
- Dùng `ThreadLocal` được không? Có thể, nhưng phải clear đúng lúc để tránh leak, nhất là thread pool.

## 8) 30 giây tóm tắt miệng

Spring Bean mặc định là singleton, nên nhiều request dùng chung một instance. Singleton bean chỉ an toàn khi stateless hoặc quản lý shared state đúng cách. Tôi không lưu dữ liệu request vào field của service; tôi truyền qua parameter hoặc dùng scope phù hợp. Nếu bắt buộc có shared mutable state, tôi chọn concurrent structure hoặc synchronization và kiểm tra trade-off về performance, lifecycle và leak.
