# Unit vs Integration Testing

- **Date:** 2026-08-20
- **Topic:** testing
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Phân biệt unit test và integration test. Khi nào dùng cái nào? Trade-off gì?

## 2) Giải thích như trẻ lên 3

- **Unit test**: kiểm tra 1 hàm/class độc lập, không cần DB hay service ngoài. Chạy nhanh, như test một bánh xe.
- **Integration test**: kiểm tra nhiều thành phần chạy cùng nhau (hàm + DB + API). Chậm hơn nhưng gần với thực tế.

## 3) Giải thích đơn giản cho dev

- **Unit**: mock dependency, test logic thuần. Nhanh (ms), dễ debug, phát hiện lỗi logic sớm.
- **Integration**: dùng real DB (testcontainer), real bean Spring. Chậm (s), nhưng bắt lỗi interaction/schema.
- **Pyramid**: nhiều unit, ít integration, rất ít E2E. Lý do: unit nhanh, quy mô tốt, integration chỉ test boundary.

## 4) Ví dụ code đơn giản

**Unit test** (mock):
```java
@Test
void testCalculatePrice() {
  PriceService svc = new PriceService(mock(TaxRepository.class));
  assertEquals(110, svc.priceWithTax(100, 0.1));
}
```

**Integration test** (real Spring + DB):
```java
@SpringBootTest
@Testcontainers
class OrderServiceIT {
  @Container
  static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>();
  
  @Autowired OrderRepository repo;
  @Autowired OrderService svc;
  
  @Test
  void testCreateOrderPersistsToDb() {
    Order o = svc.create("item1", 100);
    assertTrue(repo.existsById(o.getId()));
  }
}
```

## 5) Trả lời Middle+

- Unit test: test đơn vị logic, mock DB/API. Nhanh, dễ bảo trì.
- Integration: test flow end-to-end qua Spring context, dùng testcontainer cho DB.
- Cân bằng: 70% unit, 20% integration, 10% E2E.
- Tool: JUnit 5, Mockito (unit), Testcontainers (integration).

## 6) Trả lời Senior

- **Test pyramid**: nhiều unit (nhanh, reliable), ít integration (tốn resource, flaky nếu environment).
- **Seam model**: unit test không cần mock hết; dùng in-memory DB hoặc fake khi cần, save boot time.
- **Trade-off**: unit phát hiện lỗi logic nhanh → ROI cao. Integration bắt interaction bug (N+1, cascade, transaction boundary) → bắt buộc cho critical path.
- **Production concern**: testcontainer/docker add complexity, slow CI. Consider staged pipeline: unit → fast integration (in-memory H2) → full integration (postgres) ở nhánh.
- **Pitfall**: 100% coverage unit + 0% integration = code chạy đơn lẻ OK nhưng crash khi Spring load bean. Phải test ít nhất một happy path end-to-end.

## 7) Follow-up / pitfall

- Q: Cái nào viết trước? A: unit trước, green trước, rồi tích hợp. Nhưng legacy không có unit → tạo integration test trước để "cage" behavior, rồi refactor.
- Q: Testcontainer quá chậm? A: dùng in-memory (H2, Embedded Kafka) cho tầng CI đầu, full container khi merge.
- Pitfall: quên tear down container → port still bound, test fail random.

## 8) 30 giây tóm tắt miệng

"Unit test kiểm tra logic riêng lẻ, nhanh, dễ debug. Integration test kiểm tra Spring bean + DB thực tế, chậm nhưng bắt lỗi interaction. Cân bằng: 70% unit, 20% integration, 10% E2E. Unit test phát hiện bug logic sớm; integration test bắt N+1 query, transaction boundary issue."
