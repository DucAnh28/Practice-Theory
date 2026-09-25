# Mock vs Stub vs Spy — Khi nào dùng cái nào?

- **Date:** 2026-09-25
- **Topic:** testing
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Phân biệt Mock, Stub và Spy trong unit test. Khi nào dùng cái nào? Có ví dụ thực tế không?

## 2) Giải thích như trẻ lên 3

Khi test một cái hộp (class), ta cần fake các hộp khác bên trong nó:

- **Stub:** Fake hoàn toàn, chỉ trả về giá trị cố định. "Hỏi nó, nó trả lời sẵn."
- **Mock:** Fake và kiểm tra xem có gọi đúng cách không. "Hỏi nó, nó trả lời + ghi nhật ký xem ai hỏi."
- **Spy:** Dùng object thật nhưng theo dõi xem ai gọi nó. "Dùng cái thật nhưng gắn camera."

## 3) Giải thích đơn giản cho dev

**Stub** = dependency trả về dữ liệu cố định, không kiểm tra behavior.

```java
// UserRepositoryStub trả "User exists" mãi
UserRepository stub = new UserRepositoryStub();
stub.findById(1L); // luôn trả User(1, "John")
```

**Mock** = fake + verify được gọi với tham số nào, bao nhiêu lần.

```java
// Kiểm tra PaymentService được gọi đúng
PaymentService mock = mock(PaymentService.class);
when(mock.charge(100)).thenReturn(true);
// test code...
verify(mock).charge(100); // phải gọi đúng 1 lần với 100
```

**Spy** = object thật + track calls (không override behavior mặc định).

```java
// UserService thật, nhưng theo dõi
UserService spy = spy(new UserService());
spy.saveUser(user);
verify(spy).saveUser(user); // gọi được, vẫn chạy code thật
```

## 4) Ví dụ code đơn giản

```java
// Scenario: OrderService gọi UserRepository + PaymentService

// ❌ Dùng Stub khi cần behavior đơn giản, không quan tâm logic
UserRepository userStub = new UserRepository() {
  public User findById(Long id) { return new User(id, "Test"); }
};

// ✅ Dùng Mock khi test OrderService gọi PaymentService đúng cách
PaymentService paymentMock = mock(PaymentService.class);
when(paymentMock.charge(100)).thenReturn(true);

OrderService order = new OrderService(paymentMock);
order.createOrder(user, 100);

verify(paymentMock, times(1)).charge(100); // verify logic

// ✅ Dùng Spy khi test logic của OrderService + track calls
OrderService orderSpy = spy(new OrderService(realPaymentService));
orderSpy.createOrder(user, 100);

verify(orderSpy).createOrder(user, 100); // track nhưng chạy thật
```

## 5) Trả lời Middle+

Chúng ta dùng test double (fake) để isolate unit test. Ba loại chính:

- **Stub:** Chỉ cần trả giá trị. Nhanh, đơn giản, không kiểm tra behavior.
- **Mock:** Cần xác minh có gọi dependency đúng không (verify calls). Dùng khi logic phụ thuộc vào side-effect của dependency (ví dụ ghi log, call API).
- **Spy:** Muốn chạy code thật (thường vì nó phức tạp) nhưng track xem ai gọi. Ít dùng, nhưng hữu ích khi refactor.

**Nguyên tắc:** Mock the things that matter (behavior, calls); Stub the rest (dữ liệu).

## 6) Trả lời Senior

Test double strategy phụ thuộc vào scope và mục tiêu:

**Stub:** Dùng khi dependency chỉ là data source, không có side-effect. Giảm overhead. Nhưng nếu overuse, test mất khả năng phát hiện bug (false negatives).

**Mock:** Phù hợp khi đặt behavior expectation (order của calls, frequency, arguments). Tool: Mockito, JUnit 5 + BDDMockito. Risk: over-specification → brittle test (test fail vì refactor internal call, không phải logic).

**Spy:** Hiếm khi dùng production code. Thường trong refactor. Nhưng có vấn đề: Spy vẫn chạy code thật → có thể trigger real I/O, transaction → test chậm hay fail không mong muốn.

**Best practice:** Dùng Mock + Stub mix. Stub cho dependency non-critical; Mock cho critical path (payment, auth, notification). Tránh spy trừ khi absolutely needed.

**Pitfall:** Over-mock → test kiểm tra implementation detail thay vì behavior → khi refactor, test fail dù code vẫn đúng.

## 7) Follow-up / pitfall

- "Làm sao biết khi nào Mock vs Stub?" → If dependency call affects outcome/business logic, mock it. Else stub.
- "Mock quá nhiều, test dễ break khi refactor?" → Đúng. Mock behavior, not implementation. Use `ArgumentCaptor` thay vì hardcode expected calls.
- "Spy vs Partial Mock?" → Spy = spy(real object); Partial mock = mock(class) + doCallRealMethod(). Spy thường safer vì method not mocked run thật.
- "Mockito + Spring @MockBean vs @SpyBean?" → Tương tự: MockBean = full mock; SpyBean = spy. Dùng spy khi test service logic bên trong, need real dependency.

## 8) 30 giây tóm tắt miệng

"Stub cung cấp dữ liệu, Mock kiểm tra xem code gọi dependency đúng cách không, Spy track calls nhưng chạy code thật. Dùng Mock/Stub mix: Mock critical path, stub the rest. Tránh over-mock vì nó làm test brittle."
