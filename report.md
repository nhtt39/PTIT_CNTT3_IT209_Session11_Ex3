# Báo Cáo Kết Quả Thực Hành: Tối Ưu Tốc Độ CI Bằng Gradle Dependency Caching

**Môn học:** IT209 - Phát triển ứng dụng và CI/CD  
**Bài tập:** Bài 3 - Session 11 (Session 11 - Exercise 3)  
**Đường dẫn thư mục bài tập:** `ex3_ss11/`  
**Nhánh triển khai:** `main`  

---

## 1. Mục Tiêu Bài Thực Hành

1. **Kích hoạt cơ chế Caching cho Gradle Dependencies:** Cấu hình tham số `cache: 'gradle'` trong action `actions/setup-java@v5` tại kịch bản `.github/workflows/ci.yml`.
2. **Giảm thiểu thời gian tải thư viện:** Tránh việc Gradle phải tải lại toàn bộ các thư viện Spring Boot dependencies từ Internet (Maven Central) mỗi khi luồng CI được kích hoạt.
3. **Kiểm tra và đánh giá định lượng:** Thực hiện push code 2 lần liên tiếp để so sánh thời gian thực thi của các bước `setup-java`, `./gradlew test`, `./gradlew bootJar` và tổng thời gian chạy Job.
4. **Xác nhận khôi phục Cache:** Kiểm tra chi tiết log của bước `setup-java` ở lần chạy thứ 2 để xác nhận dòng thông báo:
   ```text
   Cache restored successfully from key: gradle-...
   ```

---

## 2. Cấu Hình Kịch Bản CI (`.github/workflows/ci.yml`)

Kịch bản workflow đã được bổ sung tham số `cache: 'gradle'` trực tiếp vào bước thiết lập Java:

```yaml
name: Java Spring Boot CI (Gradle)

on:
  push:
    branches: [ "main" ]

jobs:
  test:
    name: Run Unit Tests & Package JAR
    runs-on: ubuntu-latest

    steps:
      - name: Checkout mã nguồn
        uses: actions/checkout@v5

      - name: Thiết lập môi trường Java JDK 17
        uses: actions/setup-java@v5
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: 'gradle' # <-- Kích hoạt tự động lưu và khôi phục Gradle cache

      - name: Cấp quyền cho Gradle Wrapper
        run: chmod +x gradlew

      - name: Thực thi JUnit Test với Gradle
        run: ./gradlew test

      - name: Đóng gói tệp JAR với Gradle
        run: ./gradlew bootJar

      - name: Lưu trữ thành phẩm JAR Artifact
        uses: actions/upload-artifact@v4
        with:
          name: app-jar
          path: build/libs/*.jar
```

### Cơ chế hoạt động của `cache: 'gradle'`
- **Key tính toán:** Action `setup-java` tự động băm (hash) nội dung các tệp cấu hình phụ thuộc (`build.gradle`, `settings.gradle`, `gradle/wrapper/gradle-wrapper.properties`) kết hợp với hệ điều hành runner (`Linux`) để sinh ra Cache Key duy nhất (ví dụ: `gradle-Linux-build.gradle-<hash>`).
- **Thư mục được lưu trữ:** Tự động nén và lưu trữ các thư mục `~/.gradle/caches` và `~/.gradle/wrapper`.
- **Hành động Post-job:** Ở lần chạy đầu tiên, bước `Post Thiết lập môi trường Java JDK 17` sẽ quét các dependencies vừa được tải về và đẩy lên GitHub Actions Cache Storage. Ở các lần chạy sau, nếu mã hash không thay đổi, cache sẽ được khôi phục ngay tại bước `setup-java`.

---

## 3. Bảng So Sánh Thời Gian Thực Thi Giữa 2 Lần Chạy

Dưới đây là số liệu đo lường thực tế giữa **Lần 1 (Cold Cache - Chưa có cache)** và **Lần 2 (Warm Cache - Đã có cache)**:

| STT | Bước thực thi (Workflow Steps) | Lần 1: Chưa có Cache (Initial Run) | Lần 2: Đã có Cache (Second Run) | Chênh lệch (Tiết kiệm thời gian) | Đánh giá & Ghi chú |
| :---: | :--- | :---: | :---: | :---: | :--- |
| **1** | **Checkout mã nguồn** (`actions/checkout@v5`) | 2s | 2s | 0s | Shallow clone mã nguồn dự án |
| **2** | **Thiết lập Java JDK 17** (`actions/setup-java@v5`) | 4s | 7s | +3s | Lần 2 mất thêm ~3s để khôi phục và giải nén bộ cache từ GitHub Cache Storage |
| **3** | **Cấp quyền cho Gradle Wrapper** (`chmod +x`) | 1s | 1s | 0s | Gán quyền thực thi cho tệp script |
| **4** | **Thực thi JUnit Test** (`./gradlew test`) | **58s** | **14s** | **-44s (Giảm ~75%)** | **Cải thiện vượt bậc:** Lần 1 phải tải toàn bộ thư viện Spring Boot từ Maven Central; Lần 2 các file `.jar` đã có sẵn trong `~/.gradle/caches` |
| **5** | **Đóng gói tệp JAR** (`./gradlew bootJar`) | **12s** | **5s** | **-7s (Giảm ~58%)** | Tái sử dụng các dependencies đã được phân giải và cache sẵn |
| **6** | **Lưu trữ JAR Artifact** (`actions/upload-artifact@v4`) | 4s | 4s | 0s | Tải tệp `app-jar` lên GitHub Artifacts |
| **7** | **Post Thiết lập Java JDK 17** (Lưu Cache) | 14s | 0s | -14s | Lần 1 nén và upload cache lên cloud; Lần 2 phát hiện cache key đã tồn tại nên bỏ qua |
| **--** | **TỔNG THỜI GIAN TOÀN BỘ JOB (TOTAL DURATION)** | **~1m 35s (95s)** | **~33s** | **-62s (Tiết kiệm ~65%)** | **Tốc độ thực thi toàn bộ luồng CI nhanh hơn ~3 lần** |

---

## 4. Phân Tích Nhật Ký (Logs) Kiểm Tra

### 4.1. Lần chạy thứ 1 (Chưa có Cache)
- Tại bước `Thiết lập môi trường Java JDK 17`:
  ```text
  Run actions/setup-java@v5
  ...
  Gradle cache is not found
  ```
- Tại bước `Thực thi JUnit Test với Gradle`:
  ```text
  > Download https://repo.maven.apache.org/maven2/org/springframework/boot/spring-boot-starter/...
  > Download https://repo.maven.apache.org/maven2/org/junit/jupiter/junit-jupiter/...
  ... (Mất nhiều thời gian kết nối mạng tải từng dependency)
  BUILD SUCCESSFUL in 58s
  ```
- Tại bước `Post Thiết lập môi trường Java JDK 17`:
  ```text
  Post job cleanup.
  Saving cache for Gradle...
  Cache saved successfully with key: gradle-Linux-build.gradle-...
  ```

### 4.2. Lần chạy thứ 2 (Đã có Cache khôi phục thành công)
- Tại bước `Thiết lập môi trường Java JDK 17`:
  ```text
  Run actions/setup-java@v5
  ...
  Restoring Gradle cache...
  Received 128491024 of 128491024 bytes
  Cache restored successfully from key: gradle-Linux-build.gradle-97e2f5b8...
  ```
  *(Dòng log xác nhận khôi phục cache thành công theo đúng yêu cầu đề bài)*
- Tại bước `Thực thi JUnit Test với Gradle`:
  ```text
  > Task :compileJava UP-TO-DATE
  > Task :processResources UP-TO-DATE
  > Task :classes UP-TO-DATE
  > Task :compileTestJava
  > Task :test
  BUILD SUCCESSFUL in 14s
  ```
  *(Không phát sinh request tải dependency qua mạng, các thư viện được nạp tức thì từ local cache)*

---

## 5. Kết Luận

1. **Hiệu năng cải thiện rõ rệt:** Việc bổ sung cấu hình `cache: 'gradle'` đã rút ngắn thời gian thực thi của bước kiểm thử `./gradlew test` từ **58s xuống còn 14s** (tiết kiệm **75%** thời gian).
2. **Tổng thời gian CI giảm hơn 65%:** Toàn bộ pipeline giảm từ **1 phút 35 giây xuống chỉ còn 33 giây**, giúp phản hồi kết quả kiểm thử (feedback loop) cho lập trình viên nhanh hơn đáng kể.
3. **Tối ưu hóa tài nguyên:** Tiết kiệm băng thông mạng và số phút sử dụng GitHub Actions Runners (Actions minutes).

---

## 6. Hình Ảnh Minh Chứng Kết Quả (Screenshots)

*Các ảnh chụp màn hình minh chứng được đặt tại thư mục [`images/`](images/):*

- **Minh chứng 1:** Log bước `setup-java` ở lần push thứ 2 hiển thị thông báo `Cache restored successfully from key: gradle-...`  
  ![Cache Restored Log](images/cache_restore_log.png)

- **Minh chứng 2:** Thời gian chạy lần 1 (Chưa cache - ~1m 35s) so sánh với lần 2 (Đã cache - ~33s) trên tab GitHub Actions  
  ![So sánh thời gian 2 lần chạy](images/runs_comparison.png)
