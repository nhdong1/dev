# Code Coverage với JaCoCo & Mutation Testing

> Code Coverage (Độ Phủ Code) đo lường phần trăm code được thực thi bởi test suite.
> JaCoCo (Java Code Coverage) là công cụ phổ biến nhất cho Java.
> Mutation Testing (Kiểm Thử Đột Biến) đánh giá CHẤT LƯỢNG của test —
> vì 100% coverage không đảm bảo test thực sự phát hiện bug.

---

## 1. Code Coverage Là Gì?

```
Mã nguồn:
  if (price > 0) {           ← Branch 1: price > 0
      applyDiscount(price);
  } else {                   ← Branch 2: price <= 0
      throw new IllegalArgumentException();
  }

Test chỉ test trường hợp price > 0:
  Line Coverage: 75% (dòng else và throw chưa được thực thi)
  Branch Coverage: 50% (chỉ 1 trong 2 nhánh được test)
  Condition Coverage: 50%
```

### Các Loại Coverage Metrics (Chỉ Số Độ Phủ)

| Metric | Mô Tả | Mức Khuyến Nghị |
|--------|--------|-----------------|
| **Line Coverage** | % dòng code được thực thi | ≥ 80% |
| **Branch Coverage** | % nhánh if/else/switch được kiểm tra | ≥ 70% |
| **Method Coverage** | % method được gọi | ≥ 80% |
| **Class Coverage** | % class được test | ≥ 80% |
| **Instruction Coverage** | % bytecode instruction | Dùng nội bộ JaCoCo |

> **Lưu ý quan trọng:** Coverage cao ≠ Test chất lượng cao.
> Test có thể execute code mà không assert gì → Coverage 100% nhưng vô dụng.

---

## 2. Cài Đặt JaCoCo Với Maven

```xml
<!-- pom.xml -->
<build>
    <plugins>
        <plugin>
            <groupId>org.jacoco</groupId>
            <artifactId>jacoco-maven-plugin</artifactId>
            <version>0.8.12</version>
            <executions>

                <!-- Bước 1: Chuẩn bị JaCoCo agent trước khi chạy test -->
                <execution>
                    <id>prepare-agent</id>
                    <goals>
                        <goal>prepare-agent</goal>
                    </goals>
                </execution>

                <!-- Bước 2: Tạo báo cáo HTML sau khi test -->
                <execution>
                    <id>report</id>
                    <phase>verify</phase>
                    <goals>
                        <goal>report</goal>
                    </goals>
                </execution>

                <!-- Bước 3: Enforce coverage thresholds — build fail nếu không đạt -->
                <execution>
                    <id>check</id>
                    <phase>verify</phase>
                    <goals>
                        <goal>check</goal>
                    </goals>
                    <configuration>
                        <rules>
                            <rule>
                                <element>BUNDLE</element>  <!-- Áp dụng cho toàn bộ project -->
                                <limits>
                                    <limit>
                                        <counter>LINE</counter>
                                        <value>COVEREDRATIO</value>
                                        <minimum>0.80</minimum>  <!-- 80% line coverage -->
                                    </limit>
                                    <limit>
                                        <counter>BRANCH</counter>
                                        <value>COVEREDRATIO</value>
                                        <minimum>0.70</minimum>  <!-- 70% branch coverage -->
                                    </limit>
                                </limits>
                            </rule>
                        </rules>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

### Cài Đặt Với Gradle

```groovy
// build.gradle
plugins {
    id 'jacoco'
}

jacoco {
    toolVersion = "0.8.12"
}

test {
    useJUnitPlatform()
    finalizedBy jacocoTestReport  // Tạo report sau khi test chạy xong
}

jacocoTestReport {
    dependsOn test
    reports {
        xml.required = true    // Cho CI/CD (SonarQube, Codecov)
        html.required = true   // Cho developer xem local
        csv.required = false
    }
}

jacocoTestCoverageVerification {
    violationRules {
        rule {
            limit {
                minimum = 0.80  // 80% line coverage
            }
        }
        rule {
            element = 'CLASS'
            limit {
                counter = 'BRANCH'
                value = 'COVEREDRATIO'
                minimum = 0.70  // 70% branch coverage per class
            }
        }
    }
}

check.dependsOn jacocoTestCoverageVerification
```

---

## 3. Loại Trừ Code Khỏi Coverage

Không phải tất cả code đều cần test — loại trừ boilerplate, generated code, config:

```xml
<!-- Maven: Loại trừ packages/classes -->
<configuration>
    <excludes>
        <!-- DTOs và Request/Response objects -->
        <exclude>**/dto/**</exclude>
        <exclude>**/request/**</exclude>
        <exclude>**/response/**</exclude>

        <!-- Configuration classes -->
        <exclude>**/config/**</exclude>

        <!-- Main application class -->
        <exclude>**/*Application.class</exclude>

        <!-- Generated code (MapStruct, QueryDSL, Lombok) -->
        <exclude>**/*MapperImpl.class</exclude>
        <exclude>**/Q*.class</exclude>

        <!-- Exception classes (nếu không có logic) -->
        <exclude>**/exception/**</exclude>
    </excludes>
</configuration>
```

### Loại Trừ Bằng Annotation

```java
// Tạo annotation để đánh dấu code không cần coverage
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface ExcludeFromCoverage { }

// Sử dụng
@ExcludeFromCoverage
public class ApplicationConfig {
    // Config boilerplate — không cần test
}
```

```xml
<!-- JaCoCo sẽ bỏ qua classes/methods được đánh dấu @ExcludeFromCoverage -->
<configuration>
    <excludes>
        <exclude>**/*@ExcludeFromCoverage*</exclude>
    </excludes>
</configuration>
```

---

## 4. Xem Coverage Report

### Chạy Report

```bash
# Maven — tạo report
mvn clean verify

# Report được tạo tại:
target/site/jacoco/index.html   ← Mở file này trong browser

# Chạy test và skip coverage check
mvn test -Djacoco.skip=true
```

### Đọc HTML Report

```
JaCoCo Report:
├── Element: com.example.service
│   ├── Coverage: 85% lines, 78% branches
│   │
│   ├── OrderService.java         92% lines ✅
│   ├── PaymentService.java       75% lines ⚠️
│   └── NotificationService.java  45% lines ❌

Màu sắc trong source view:
  Xanh lá  → Dòng được thực thi đầy đủ
  Vàng     → Nhánh được thực thi một phần (partial branch coverage)
  Đỏ       → Dòng chưa được thực thi
```

---

## 5. Tích Hợp CI/CD

### GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Java 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'

      - name: Run tests with coverage
        run: mvn verify

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          files: ./target/site/jacoco/jacoco.xml
          token: ${{ secrets.CODECOV_TOKEN }}

      - name: Publish coverage report
        uses: actions/upload-artifact@v4
        with:
          name: jacoco-report
          path: target/site/jacoco/
```

### SonarQube Integration

```yaml
- name: SonarQube Scan
  run: mvn sonar:sonar
    -Dsonar.projectKey=my-project
    -Dsonar.host.url=${{ secrets.SONAR_URL }}
    -Dsonar.token=${{ secrets.SONAR_TOKEN }}
    -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
```

---

## 6. Mutation Testing với PITest

### Tại Sao Mutation Testing Quan Trọng?

```
Vấn đề với Coverage alone (Độ phủ đơn thuần):

Mã nguồn:
  public boolean isEligibleForDiscount(int age) {
      return age >= 18;
  }

Test:
  @Test
  void testEligible() {
      isEligibleForDiscount(25);  // ← Không assert gì!
  }

Coverage = 100% (dòng code được thực thi)
Test chất lượng = 0% (không có assertion nào)
```

### PITest — Mutation Testing Tool

**PITest** (Pit — Practical Mutation Testing) thay đổi ("đột biến") mã nguồn và kiểm tra xem test có phát hiện ra không:

```
Đột biến 1: return age >= 18  →  return age > 18
  Test fail? Nếu có → Mutation KILLED (test tốt!) ✅
  Test pass? Nếu có → Mutation SURVIVED (test thiếu sót!) ❌

Đột biến 2: return age >= 18  →  return age <= 18
  Test fail? → KILLED ✅
  Test pass? → SURVIVED ❌

Mutation Score = KILLED / (KILLED + SURVIVED) * 100%
Mục tiêu: ≥ 80% mutation score
```

### Các Loại Mutation (Đột Biến)

| Mutator | Đột Biến | Ví Dụ |
|---------|---------|-------|
| **ConditionalsBoundary** | Thay đổi điều kiện biên | `>=` → `>` |
| **Negation** | Phủ định giá trị | `return x` → `return -x` |
| **Math** | Thay đổi phép tính | `+` → `-`, `*` → `/` |
| **IncrementsMutator** | Thay đổi increment | `i++` → `i--` |
| **ReturnVals** | Thay đổi giá trị trả về | `return true` → `return false` |
| **VoidMethodCalls** | Xóa void method call | Bỏ `list.add(item)` |
| **NullReturns** | Trả về null | `return user` → `return null` |

### Cài Đặt PITest

```xml
<!-- pom.xml -->
<plugin>
    <groupId>org.pitest</groupId>
    <artifactId>pitest-maven</artifactId>
    <version>1.16.1</version>
    <dependencies>
        <!-- Hỗ trợ JUnit 5 -->
        <dependency>
            <groupId>org.pitest</groupId>
            <artifactId>pitest-junit5-plugin</artifactId>
            <version>1.2.1</version>
        </dependency>
    </dependencies>
    <configuration>
        <!-- Package cần test -->
        <targetClasses>
            <param>com.example.service.*</param>
            <param>com.example.domain.*</param>
        </targetClasses>

        <!-- Test classes để chạy -->
        <targetTests>
            <param>com.example.*Test</param>
            <param>com.example.*Tests</param>
        </targetTests>

        <!-- Ngưỡng mutation score tối thiểu -->
        <mutationThreshold>80</mutationThreshold>
        <coverageThreshold>80</coverageThreshold>

        <!-- Báo cáo định dạng -->
        <outputFormats>
            <param>HTML</param>
            <param>XML</param>
        </outputFormats>

        <!-- Số luồng song song — tăng tốc -->
        <threads>4</threads>

        <!-- Excludes — không test classes này -->
        <excludedClasses>
            <param>**/dto/**</param>
            <param>**/config/**</param>
        </excludedClasses>
    </configuration>
</plugin>
```

### Chạy PITest

```bash
# Chạy mutation testing
mvn pitest:mutationCoverage

# Report tại: target/pit-reports/YYYYMMDDHHMI/index.html

# Chạy chỉ trên files đã thay đổi (nhanh hơn trong CI)
mvn pitest:mutationCoverage -DwithHistory
```

---

## 7. Ví Dụ: Cải Thiện Test Chất Lượng

### Trường Hợp Thực Tế

```java
// Code cần test
public class DiscountService {

    public double calculateDiscount(UserType userType, double orderTotal) {
        if (userType == UserType.PREMIUM && orderTotal >= 1_000_000) {
            return 0.20;  // 20% discount
        } else if (userType == UserType.PREMIUM) {
            return 0.10;  // 10% discount
        } else if (orderTotal >= 500_000) {
            return 0.05;  // 5% discount
        }
        return 0.0;
    }
}
```

### Test Coverage Thấp (Chỉ 1 Trường Hợp)

```java
// ❌ Coverage: 40% — chỉ test 1 nhánh
@Test
void calculateDiscount_premiumUserHighOrder() {
    double discount = discountService.calculateDiscount(UserType.PREMIUM, 2_000_000);
    assertThat(discount).isEqualTo(0.20);
}
```

### Test Coverage Đầy Đủ (Tất Cả Nhánh)

```java
// ✅ Coverage: 100% — test tất cả nhánh
@ParameterizedTest
@MethodSource("provideDiscountCases")
void calculateDiscount_allCases(UserType userType, double total, double expectedDiscount) {
    assertThat(discountService.calculateDiscount(userType, total))
        .isEqualTo(expectedDiscount);
}

static Stream<Arguments> provideDiscountCases() {
    return Stream.of(
        // PREMIUM + order >= 1_000_000 → 20%
        Arguments.of(UserType.PREMIUM, 1_000_000.0, 0.20),
        Arguments.of(UserType.PREMIUM, 2_000_000.0, 0.20),
        // PREMIUM + order < 1_000_000 → 10%
        Arguments.of(UserType.PREMIUM, 500_000.0,   0.10),
        Arguments.of(UserType.PREMIUM, 999_999.0,   0.10),
        // STANDARD + order >= 500_000 → 5%
        Arguments.of(UserType.STANDARD, 500_000.0,  0.05),
        Arguments.of(UserType.STANDARD, 1_000_000.0, 0.05),
        // STANDARD + order < 500_000 → 0%
        Arguments.of(UserType.STANDARD, 100_000.0,  0.0),
        Arguments.of(UserType.STANDARD, 499_999.0,  0.0)
    );
}
```

---

## 8. Coverage Targets Theo Loại Code

| Loại Code | Target Coverage | Ghi Chú |
|-----------|-----------------|---------|
| **Business Logic** (Service) | ≥ 90% | Core của ứng dụng |
| **Domain Model** | ≥ 85% | Entity, Value Objects |
| **Repository** | ≥ 75% | Custom queries cần test |
| **Controller** | ≥ 80% | Validation, error handling |
| **DTO / Request / Response** | Loại trừ | Boilerplate, no logic |
| **Configuration** | ≤ 30% | Khó test, ít logic |
| **Exception Classes** | Loại trừ | Thường không có logic |
| **Main Application** | Loại trừ | Boilerplate |

---

## 9. Coverage Trong Pull Request (Yêu Cầu Xem Xét)

```yaml
# Codecov — chặn PR nếu coverage giảm
# codecov.yml
coverage:
  status:
    project:
      default:
        target: 80%           # Coverage tổng cần đạt
        threshold: 2%         # Cho phép giảm tối đa 2%
    patch:                    # Coverage của code mới trong PR
      default:
        target: 80%           # Code mới phải có ≥ 80% coverage
        threshold: 0%         # Không cho phép giảm coverage của patch
```

---

## 10. Anti-Patterns Coverage (Lỗi Thường Gặp)

### ❌ Test Không Có Assertion — Coverage Ảo

```java
// ❌ Sai — coverage tăng nhưng test vô nghĩa
@Test
void testCreateOrder() {
    orderService.createOrder(request);  // ← Không assert gì!
    // Test này không bao giờ fail
}

// ✅ Đúng — assert kết quả
@Test
void createOrder_withValidRequest_returnsOrderWithCorrectStatus() {
    OrderResponse result = orderService.createOrder(request);
    assertThat(result.getStatus()).isEqualTo(OrderStatus.PENDING);
    assertThat(result.getId()).isNotNull();
    verify(orderRepository).save(any(Order.class));
}
```

### ❌ Test Chỉ Để Tăng Coverage

```java
// ❌ Sai — test getter/setter không có giá trị
@Test
void testGettersAndSetters() {
    Order order = new Order();
    order.setId(UUID.randomUUID());
    assertThat(order.getId()).isNotNull();  // Luôn pass, không có ý nghĩa
}
```

### ❌ Hardcode Expected Values Không Rõ Ý Nghĩa

```java
// ❌ Sai — Magic number 0.2 nghĩa là gì?
assertThat(discount).isEqualTo(0.2);

// ✅ Đúng — rõ ràng
double PREMIUM_LARGE_ORDER_DISCOUNT = 0.20;
assertThat(discount).isEqualTo(PREMIUM_LARGE_ORDER_DISCOUNT);
```

---

## 11. Checklist Coverage

- [ ] Cấu hình JaCoCo trong pom.xml/build.gradle với minimum thresholds
- [ ] Loại trừ DTOs, config classes, generated code khỏi coverage
- [ ] Coverage report được tạo trong CI/CD pipeline
- [ ] Coverage không được phép giảm khi merge PR
- [ ] Mutation testing chạy định kỳ (không nhất thiết mỗi commit)
- [ ] Focus vào Branch Coverage, không chỉ Line Coverage
- [ ] Không viết test chỉ để tăng số coverage

---

## 📚 Tài Liệu Tham Khảo

- [JaCoCo Documentation](https://www.jacoco.org/jacoco/trunk/doc/)
- [PITest Documentation](https://pitest.org/quickstart/maven/)
- [Codecov Documentation](https://docs.codecov.com/)
- [SonarQube Quality Gates](https://docs.sonarqube.org/latest/user-guide/quality-gates/)

---

## 🎯 Câu Hỏi Phỏng Vấn

- [ ] Code coverage 100% có đảm bảo code không có bug không? Tại sao?
- [ ] Sự khác biệt giữa Line Coverage và Branch Coverage?
- [ ] Mutation Testing là gì? Giải thích khái niệm "mutation survived" và "mutation killed"
- [ ] Nên đặt coverage threshold ở mức nào? Phụ thuộc vào yếu tố gì?
- [ ] Những loại code nào nên loại trừ khỏi coverage calculation?
- [ ] Làm thế nào tích hợp JaCoCo vào CI/CD để chặn PR có coverage thấp?
