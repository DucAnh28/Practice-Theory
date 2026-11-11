# Test Coverage & Mutation Testing — Đo chất lượng test thật

- **Date:** 2026-11-11
- **Topic:** testing
- **Level target:** Middle+ / Senior

## 1) Câu hỏi phỏng vấn

Test coverage cao có nghĩa là test tốt không? Coverage đo được những gì, không đo được những gì? Mutation testing khác coverage ở chỗ nào và khi nào nên dùng?

## 2) Giải thích như trẻ lên 3

- **Coverage**: đếm xem trong 100 dòng code, bao nhiêu dòng được "chạy qua" khi test. Giống xem bạn đã đi qua bao nhiêu con đường trong khu rừng.
- **Đi qua không có nghĩa là đi đúng**: có thể bạn đi qua con đường nhưng không bao giờ nhìn lại. Coverage 100% vẫn có thể bỏ sót lỗi.
- **Mutation testing**: lấy code của bạn và cố tình phá nó (đổi `+` thành `-`, `>` thành `>=`), rồi chạy lại test. Nếu test vẫn xanh → test đó yếu, không bắt được lỗi.
- Giống xem giáo viên cố tình làm sai bài và xem học sinh có phát hiện không.

## 3) Giải thích đơn giản cho dev

- **Coverage đo trong code** (line, branch, instruction, method). Nó là **độ phủ thực thi**, không phải độ phủ logic.
- Công cụ: **JaCoCo** (báo cáo + rule trong Maven/Gradle), **Cobertura** (cũ), **Istanbul** (cho JS, tham khảo cách đo).
- **Mutation testing**: **PITest** (Java), **Stryker** (JS). Sinh nhiều "mutant" — biến thể code — rồi đo **mutation score** = số mutant bị test kill / tổng số mutant.
- **Coverage cao + mutation score thấp** = test chỉ đi qua code mà không kiểm chứng hành vi. Đây là cảnh báo kinh điển: `assertTrue(result != null)` làm line coverage lên 100% nhưng không assert gì cả.
- **Mutation score** là metric chặt hơn coverage về mặt ý nghĩa, nhưng **chạy đắt** (mỗi mutant = 1 lần chạy test lại).
- Báo cáo chất lượng nên dùng **cả hai**: coverage để tìm vùng code chưa chạm, mutation score để đánh giá test có thật sự kiểm chứng không.

## 4) Ví dụ code đơn giản

Code yếu — coverage cao nhưng không có ý nghĩa:

```java
// Test "xanh" nhưng không assert gì cả → line coverage = 100%, mutation score = 0%
@Test
void testApplyDiscount() {
    service.applyDiscount(order, 10);   // chạy xong, không kiểm gì
}
```

Code tốt — assert đúng hành vi:

```java
@Test
void testApplyDiscount() {
    Order order = new Order(100_000L);

    service.applyDiscount(order, 10);

    assertThat(order.getTotal()).isEqualTo(90_000L); // kill được mutant `>` -> `>=`, `-` -> `+`
}
```

Cấu hình JaCoCo trong `pom.xml`:

```xml
<plugin>
  <groupId>org.jacoco</groupId>
  <artifactId>jacoco-maven-plugin</artifactId>
  <version>0.8.12</version>
  <executions>
    <execution>
      <goals><goal>prepare-agent</goal></goals>
    </execution>
    <execution>
      <id>report</id>
      <phase>verify</phase>
      <goals><goal>report</goal></goals>
    </execution>
  </executions>
</plugin>
```

Chạy mutation test với PIT:

```xml
<plugin>
  <groupId>org.pitest</groupId>
  <artifactId>pitest-maven</artifactId>
  <version>1.16.1</version>
  <configuration>
    <targetClasses><param>com.example.service.*</param></targetClasses>
    <targetTests><param>com.example.*Test</param></targetTests>
    <mutationThreshold>60</mutationThreshold> <!-- fail build nếu dưới ngưỡng -->
  </configuration>
</plugin>
```

## 5) Trả lời Middle+

- Coverage là **chỉ số phụ trợ**, không phải mục tiêu. Mục tiêu là test có khả năng phát hiện lỗi.
- Dùng **JaCoCo** để có báo cáo coverage trong CI (báo cáo HTML + XML để SonarQube đọc).
- Áp dụng **coverage threshold** trong build (ví dụ line ≥ 80%, branch ≥ 70%) để tránh code mới không có test.
- Phân biệt rõ: **line coverage** (dòng có chạy), **branch coverage** (cả nhánh if/else đều được kiểm), **instruction coverage**. Branch coverage mới là cái bắt được lỗi logic thật sự.
- Đừng viết test chỉ để lên coverage — đó là chỉ số bị "game" rất dễ.
- Với code khó test (legacy, side effect), dùng mutation test có chọn lọc trên package quan trọng thay vì toàn bộ repo.

## 6) Trả lời Senior

- **Coverage là hàm ý, không phải bằng chứng**. Metric tốt hơn: mutation score, số bug lọt qua test, mean time to detect. Ngân sách coverage nên đi kèm chỉ số hiệu quả (mutation score, escaped defects).
- **Mutation testing trong CI**: chạy ở tầng nightly hoặc trên PR với scope hẹp (module đang đổi), với **incremental mode** để chỉ test các mutant liên quan — tránh đốt thời gian build.
- **Kỹ thuật tăng tốc PIT**: `coverageThreshold` để chỉ mutate dòng đã có test, `timeoutFactor`, `threads`, cache report giữa các lần chạy.
- **Quy trình**: mutation score dưới ngưỡng → PR fail, nhưng **giữ exception có chủ đích** qua file cấu hình với comment lý do (ví dụ code sinh tự động, `equals/hashCode` boilerplate).
- **Ưu tiên theo rủi ro**: mutation 100% ở utility class không giá trị bằng mutation score tốt ở module xử lý tiền/authorization. Áp dụng theo vùng, không toàn repo.
- **Code sinh tự động** (Lombok getter, DTO, generated mapper) → loại trừ khỏi scope, nếu không sẽ thành "coverage theater".
- **Quan sát năng suất**: số test, thời gian chạy, flaky rate. Tăng ngân sách coverage mà không tăng mutation score = viết thêm test vô nghĩa, chỉ làm CI chậm.
- **Kết hợp với review**: mutation report là đầu mối để reviewer hỏi "test này có thật sự bảo vệ thay đổi này không?".

## 7) Follow-up / pitfall

- **Pitfall 1**: tối ưu theo coverage → test rác làm coverage lên 95% nhưng không bắt được bug thật.
- **Pitfall 2**: `assertTrue(x != null)` hoặc `assertNotNull` không assert giá trị → kill được 0 mutant. Luôn assert **giá trị quan trọng**, không chỉ kiểm tra khác null.
- **Pitfall 3**: chỉ bật JaCoCo `check` mà không set threshold → báo cáo có nhưng không chặn gì cả.
- **Pitfall 4**: để test tự xác nhận (test chỉ assert điều code vốn đã assert) → tautology, không có giá trị.
- **Pitfall 5**: chạy mutation test toàn repo mỗi commit → pipeline quá chậm, team sẽ bỏ. Dùng incremental/nightly.
- **Pitfall 6**: coverage giảm do test bị xoá hoặc refactor → phải review kỹ thay vì chỉ raise threshold để "xanh".
- **Follow-up**: "100% coverage có đảm bảo không còn bug?" → Không. Coverage = dòng code đã thực thi; mutation score = hành vi có được kiểm chứng hay không.
- **Follow-up**: "Threshold nên đặt bao nhiêu?" → Theo mức rủi ro của module, không đặt một con số chung cho cả repo; đặt ngưỡng cho phần code mới thay vì cả file legacy.
- **Follow-up**: "Coverage có thay thế code review không?" → Không. Coverage nói "dòng này chạy", review nói "logic này đúng".

## 8) 30 giây tóm tắt miệng

"Coverage đo phần code được thực thi khi chạy test, nên 100% coverage vẫn có thể che lỗi logic — vì test chỉ 'chạy qua' chứ không 'kiểm chứng'. Mutation testing giải quyết chỗ đó: nó cố tình phá code rồi xem test có bắt được không; mutation score đo được chất lượng test thật. Trong CI tôi dùng JaCoCo để chặn coverage của code mới và để chạy PITest incremental/nightly trên phần rủi ro cao, và luôn chú ý rằng assert phải kiểm giá trị thật chứ không chỉ khác null."
