# Service Discovery và Client-Side Load Balancing

- **Date:** 2026-11-05
- **Topic:** microservices
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

"Khi scale service lên nhiều instance, service A gọi service B bằng địa chỉ nào? Service Discovery giải quyết gì và client-side load balancing khác server-side thế nào?"

## 2) Giải thích như trẻ lên 3

Mỗi cửa hàng có nhiều nhân viên trực. Khách hỏi "quầy 3 còn ai không?" thì nhờ người quản lý tra danh sách nhân viên đang trực rồi chỉ đến quầy còn người. **Service registry** giống bảng danh sách nhân viên đó, còn **client-side load balancing** là khách tự chọn quầy.

Nếu không có bảng tra cứu, khách phải nhớ số điện thoại từng quầy → mỗi lần nhân viên đi làm/nghỉ là khách gọi nhầm.

## 3) Giải thích đơn giản cho dev

- **Registry** (Eureka, Consul, Kubernetes DNS) lưu danh sách instance đang sống + địa chỉ + metadata (zone, version, weight).
- **Heartbeat**: mỗi instance định kỳ gửi heartbeat, quá thời gian không thấy → coi là chết → gỡ khỏi registry.
- **Client-side**: mỗi instance của service A tự tải danh sách B từ registry, tự chọn 1 instance B rồi gọi trực tiếp. Không có hop trung gian.
- **Server-side**: một LB/Proxy đứng trước, client gọi LB, LB mới chọn instance thật. Thêm 1 network hop nhưng client đơn giản.

| | Client-side | Server-side |
|---|---|---|
| Hop | Client → instance thật (thẳng) | Client → LB → instance |
| Extra latency | Không | +1 RTT |
| Load balancer | Logic nằm ở client | Tập trung, dễ đổi policy |
| Overhead client | Mỗi instance tự fetch registry | Không |
| Health check | Phải tự làm | Do LB lo |

Spring Cloud: `spring-cloud-starter-netflix-eureka-client` + `spring-cloud-starter-loadbalancer` → khai báo `@LoadBalanced RestTemplate`/`WebClient` và gọi bằng service name, không cần IP.

## 4) Ví dụ code đơn giản

```yaml
# application.yml
eureka:
  client:
    service-url:
      defaultZone: http://eureka-1:8761/eureka/,http://eureka-2:8761/eureka/
  instance:
    prefer-ip-address: false          # đăng ký bằng hostname cho container
    lease-renewal-interval-in-seconds: 10
    lease-expiration-duration-in-seconds: 30
```

```java
@SpringBootApplication
@EnableDiscoveryClient
public class OrderApp {
    public static void main(String[] args) {
        SpringApplication.run(OrderApp.class, args);
    }

    @Bean
    @LoadBalanced
    RestClient restClient(RestClient.Builder builder) {
        return builder.build();   // gọi "http://inventory-service/api/stock" bằng service name
    }
}

@RestController
class OrderController {
    private final RestClient inventory;

    OrderController(RestClient inventory) { this.inventory = inventory; }

    @GetMapping("/stock")
    String stock() {
        return inventory.get().uri("http://inventory-service/api/stock").retrieve().body(String.class);
    }
}
```

```java
// Tự viết resolver nếu không dùng Spring Cloud LoadBalancer
public List<ServiceInstance> pick(List<ServiceInstance> instances) {
    // round-robin đơn giản, bỏ qua instance unhealthy
    List<ServiceInstance> healthy = instances.stream()
            .filter(i -> i.getMetadata().get("status").equals("UP"))
            .toList();
    int idx = Math.floorMod(counter.getAndIncrement(), healthy.size());
    return List.of(healthy.get(idx));
}
```

## 5) Trả lời Middle+

Service Discovery giải quyết bài toán "instance của service B nằm ở đâu, và cái nào còn sống". Mỗi service đăng ký vào registry bằng heartbeat, client lấy danh sách instance (thường cache local, refresh mỗi 30s) và tự load balance theo round-robin / weighted / zone-aware. Trong Spring Cloud dùng Eureka + `spring-cloud-starter-loadbalancer`, khai báo `@LoadBalanced RestTemplate` rồi gọi bằng service name, thư viện lo phần định tuyến và cân bằng tải.

Điểm cần nhớ: client cache danh sách instance nên khi instance chết, request tới nó vẫn có thể fail trong khoảng thời gian chờ refresh → cần retry với backoff và timeout ngắn.

## 6) Trả lời Senior

- **Tính nhất quán của registry**: registry là distributed system, dùng AP theo CAP. Mất vài instance khỏi danh sách lúc đầu ít nguy hiểm hơn mất heartbeat → nhiều instance bị gỡ oan (false negative) dẫn tới thiếu capacity. Vì vậy heartbeat TTL phải lớn hơn nhiều lần chu kỳ heartbeat (ví dụ 30s TTL / 10s renew) để tránh flapping.
- **Full pull vs push**: client pull định kỳ tạo N×M request lên registry khi scale lớn; dùng registry có push (gRPC long-poll, Consul blocking query) hoặc dedupe bằng shared cache.
- **Client-side**: giảm hop và latency, nhưng mỗi client phải cài logic LB, xử lý failover, và phải biết topology. Bù lại: client tự failover được ngay khi instance chết, không phụ thuộc LB.
- **Server-side**: policy tập trung (canary, weighted, blue-green), nhưng LB thành bottleneck và single point of failure → phải scale ngang, có LB ở tầng trên (DNS, cloud LB) hoặc active-active.
- **Sticky session** cần shared store (Redis) vì client-side không có định chỉ session tới instance.
- **Zone-aware**: ưu tiên instance cùng zone để giảm cross-zone traffic (cross-AZ ~1ms+ mỗi call). Dùng metadata `zone` khi đăng ký.
- **Bảo mật**: registry lộ địa chỉ nội bộ → phải auth/mTLS và network isolate. Không để service tự ý gọi IP thẳng (mất toàn bộ lợi ích LB/discovery).
- **Observability**: cần metric theo instance (error rate, latency p99), tracing xuyên service, và alert khi registry rỗng hoặc tỉ lệ heartbeat miss tăng.
- **Kubernetes**: không cần Eureka — dùng `Service` + ClusterIP DNS, DNS-based discovery với `ttl` ngắn (nếu dùng headless + DNS cache dài, sẽ giữ IP cũ).

## 7) Follow-up / pitfall

**Q:** "Client-side hay server-side tốt hơn?"

**A:** Không có đáp án tuyệt đối. Client-side: ít hop, latency thấp, failover nhanh, nhưng logic rải rác và mỗi client đều cần fetch registry. Server-side: policy tập trung, dễ canary/weighted, nhưng thêm hop và LB là SPOF. Thực tế production thường phối hợp: cloud LB (server-side) ở tầng ngoài + client-side service discovery bên trong.

**Q:** "Instance chết giữa chừng thì sao?"

**A:** Client cache danh sách → vẫn gọi instance chết. Xử lý bằng: timeout ngắn, retry có backoff + jitter, và circuit breaker để ngắt hàng loạt. Chỉ "nhìn thấy" instance chết sau khi registry loại bỏ và client refresh.

**Q:** "Có nên dùng DNS thay cho registry không?"

**A:** Được trong Kubernetes, nhưng DNS cache của client/OS có thể giữ IP cũ → failover chậm. Nhiều hệ thống dùng DNS kết hợp headless service + client-side LB (CoreDNS) để vẫn còn tính năng zone-aware.

**Pitfall hay gặp:**
- Đăng ký IP container thay vì hostname → IP đổi mỗi lần restart, registry đầy bản ghi chết.
- `prefer-ip-address: true` với hostname resolution chậm → heartbeat mất, instance bị gỡ oan.
- Không có backoff/jitter khi retry → thundering herd đè vào instance còn sống.
- Health check chỉ ping process, không kiểm tra dependency → traffic vẫn vào instance đang lỗi downstream.
- Registry là SPOF → nhiều instance registry (Eureka peers) hoặc chạy chế độ split-brain với chấp nhận stale data.
- Cho client gọi thẳng IP lấy từ config → mất load balancing và failover, lệch traffic về 1 instance.

## 8) 30 giây tóm tắt miệng

"Service Discovery giải quyết bài toán instance của service B nằm ở đâu và cái nào còn sống: mỗi instance đăng ký vào registry bằng heartbeat, registry gỡ instance khi heartbeat hết hạn. Client-side load balancing là mỗi client tự tải danh sách instance, cache local, tự chọn instance rồi gọi thẳng — ít hop, failover nhanh, nhưng logic rải rác; server-side thì có LB đứng giữa, policy tập trung nhưng thêm RTT và dễ thành SPOF. Trong production cần timeout ngắn, retry có backoff jitter, circuit breaker, zone-aware routing, bảo mật registry bằng mTLS, và luôn nhớ client cache danh sách nên vẫn có thể gọi instance chết."
