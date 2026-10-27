# TASKNEST - BACKEND DEVELOPMENT OUTLOOK & PROGRESS TRACKER

> **Tài liệu theo dõi tiến độ và chi tiết các thành phần Backend đã triển khai**  
> *Dự án:* TaskNest (OJT4)  
> *Múi giờ hệ thống:* `Asia/Ho_Chi_Minh` (GMT+7)  
> *Cập nhật lần cuối:* 27/10/2026

---

## 1. TỔNG QUAN KIẾN TRÚC & CÔNG NGHỆ BACKEND

- **Kiến trúc hệ thống:** Microservices kết hợp Spring Cloud.
  - `platform/config-server`: Spring Cloud Config quản lý cấu hình tập trung.
  - `platform/discovery-server`: Netflix Eureka Service Registry.
  - `platform/api-gateway`: Spring Cloud Gateway (Spring WebFlux), xử lý định tuyến tập trung, CORS, CSRF, Correlation ID.
  - `services/identity-service`: Quản lý tài khoản, xác thực, phân quyền, mã hóa mật khẩu, token xác minh email.
  - `services/notification-service`: Xử lý thông báo, gửi email (Brevo/RabbitMQ consumer).
  - `services/work-service`: Quản lý dự án, công việc và tác vụ.
  - `infra/compose`: Docker Compose điều phối MySQL 8.0/8.4, RabbitMQ, Redis, Frontend (Nginx).
- **Chuẩn DTO & Response:**
  - Tất cả DTO dùng class thông thường kết hợp Lombok (`@Getter`, `@Setter`, `@NoArgsConstructor`, `@AllArgsConstructor`, `@Builder`). Không dùng Java record.
  - Cấu trúc response chuẩn: `ApiDataResponse<T>` gồm `(Boolean success, String message, T data, Object errors, HttpStatus httpStatus)`.

---

## 2. NHẬT KÝ CHI TIẾT CÁC TÍNH NĂNG ĐÃ TRIỂN KHAI

### Đợt 1: Đăng ký tài khoản (User Registration - Quy tắc ACC-01 -> ACC-04)

| Mã quy tắc | Yêu cầu nghiệp vụ | Trạng thái | Chi tiết triển khai |
| :--- | :--- | :---: | :--- |
| **ACC-01** | Đăng ký gồm: họ tên, email, mật khẩu và xác nhận mật khẩu. | ✅ Hoàn thành | `RegisterRequest.java` gồm `fullName`, `email`, `password`, `confirmPassword`. |
| **ACC-02** | Họ tên bỏ khoảng trắng đầu/cuối, độ dài 2–100 ký tự. | ✅ Hoàn thành | Xử lý `trimmedFullName = fullName.trim()`, kiểm tra độ dài `[2, 100]`. Ném `ValidationException` nếu vi phạm. |
| **ACC-03** | Email chuẩn định dạng RFC 5322, trim đầu/cuối, chuyển thường, duy nhất trong hệ thống. | ✅ Hoàn thành | `trimmedEmail.toLowerCase(Locale.ROOT)`, regex RFC 5322, kiểm tra qua `userRepository.existsByEmail(...)`. Ném `DuplicateEmailException` (HTTP 409). |
| **ACC-04** | Mật khẩu tối thiểu 8 ký tự, xác nhận mật khẩu phải khớp. Băm bằng BCrypt, không lộ qua API. | ✅ Hoàn thành | Kiểm tra mật khẩu khớp nhau, băm bằng `BCryptPasswordEncoder`. Response chỉ trả `UserResponse` không chứa trường password. |

---

### Đợt 2: Chuẩn hóa kiến trúc phản hồi & xử lý ngoại lệ toàn cục

1. **Class chuẩn hóa phản hồi `ApiDataResponse<T>`**:
   - Vị trí: `vn.tasknest.identityservice.api.dto.common.response.ApiDataResponse<T>`.
   - Các trường: `success`, `message`, `data`, `errors`, `httpStatus`.
2. **Xử lý lỗi tập trung `GlobalExceptionHandler`**:
   - Vị trí: `vn.tasknest.identityservice.exception.GlobalExceptionHandler`.
   - Bắt các ngoại lệ:
     - `ValidationException` -> HTTP 400 BAD_REQUEST.
     - `DuplicateEmailException` -> HTTP 409 CONFLICT.
     - `TokenNotFoundException` -> HTTP 404 NOT_FOUND.
     - `TokenExpiredException` -> HTTP 400 BAD_REQUEST.
     - `CooldownException` -> HTTP 429 TOO_MANY_REQUESTS (kèm `remainingSeconds`).
     - `RateLimitExceededException` -> HTTP 429 TOO_MANY_REQUESTS.
     - `EmailAlreadyVerifiedException` -> HTTP 400 BAD_REQUEST.
     - `MethodArgumentNotValidException` / `ConstraintViolationException` -> HTTP 400 BAD_REQUEST.

---

### Đợt 3: Quy tắc xác thực Email & Quản lý Token (VERIFY-01 -> VERIFY-05)

| Mã quy tắc | Yêu cầu nghiệp vụ | Trạng thái | Chi tiết triển khai |
| :--- | :--- | :---: | :--- |
| **VERIFY-01** | Tài khoản mới đăng ký ở trạng thái `is_email_verified = false`. Sinh token ngẫu nhiên lưu hash SHA-256. | ✅ Hoàn thành | `User.emailVerified = false`. Khi đăng ký tạo 1 bản ghi `AccountToken` (PURPOSE_EMAIL_VERIFICATION), lưu `token_hash BINARY(32)`, hạn dùng 24h. |
| **VERIFY-02** | Tài khoản chưa xác nhận chỉ được thực hiện: xác nhận, gửi lại email, đăng xuất; chặn quyền vào dự án. | ✅ Hoàn thành | Database lưu cờ `is_email_verified`. Các endpoint xác thực được cấu hình `permitAll()` trên Gateway & Identity Service; các logic nghiệp vụ dự án sẽ chặn tài khoản có cờ này là `false`. |
| **VERIFY-03** | Token hiệu lực đúng 24h, chỉ dùng thành công 1 lần (`used_at`). Khi gửi lại, toàn bộ token cũ bị vô hiệu hóa (`revoked_at`). | ✅ Hoàn thành | Kiểm tra `revoked_at == null`, `used_at == null`, `expiresAt.isAfter(now)`. Khi gọi `resend`, dùng `@Modifying` query update `revoked_at = now` cho mọi token active cũ của user. |
| **VERIFY-04** | Rate limit gửi lại: Cooldown tối thiểu 60s giữa 2 lần gửi. Tối đa 5 lần/giờ cho 1 tài khoản. | ✅ Hoàn thành | Query `findTopByUserIdAndPurposeOrderByCreatedAtDesc` để tính cooldown. Nếu < 60s, ném `CooldownException`. Query `countByUserIdAndPurposeAndCreatedAtAfter` trong 1 giờ qua, nếu >= 5 ném `RateLimitExceededException`. |
| **VERIFY-05** | Gửi email thất bại không được rollback tài khoản. Xử lý async qua outbox pattern. | ✅ Hoàn thành | Sự kiện `identity.email-verification.requested` ghi vào bảng `outbox_events` cùng transaction tạo tài khoản. Token gửi mail được mã hóa AES-256-GCM trong `sensitive_payload`. |

---

### Đợt 4: Xác thực người dùng bằng thuật toán bất đối xứng RS256 & Refresh Token Rotation (SCRUM-9)

| Mã quy tắc / Nghiệp vụ | Yêu cầu nghiệp vụ | Trạng thái | Chi tiết triển khai |
| :--- | :--- | :---: | :--- |
| **AUTH-RS256** | Ký Access Token bằng thuật toán bất đối xứng RS256 (RSA 2048-bit). | ✅ Hoàn thành | `RsaKeyConfiguration.java`: Tự sinh và quản lý bean `RSAKey`, `JWKSource`, `NimbusJwtEncoder`, `NimbusJwtDecoder`. Access Token có header `alg: RS256`, `kid: tasknest-key-1`, claims `sub`, `email`, `fullName`, `email_verified`, `iss: https://identity.tasknest.vn`, `exp: 15 phút`. |
| **JWKS-ENDPOINT** | Expose public key cho các Resource Server và Gateway xác thực token độc lập. | ✅ Hoàn thành | Endpoint `GET /api/identity/.well-known/jwks.json` trả về JWK Set chuẩn RFC 7517. |

---

### Đợt 5: Đồng bộ Schema V2 & Xử lý 7 vấn đề Security P1

| Mã vấn đề | Nội dung lỗi ban đầu | Hướng xử lý & Trạng thái | Chi tiết triển khai kỹ thuật |
| :--- | :--- | :---: | :--- |
| **P1 #1** | Endpoint mới không qua được Gateway (`denyAll()`, lỗi 403 register, 401 jwks, lệch alias `/api/v1/auth/*`). | ✅ Hoàn thành | Thống nhất canonical path `/api/identity/auth/*` trên toàn hệ thống. Cấu hình method-specific `pathMatchers` tại Gateway & Identity. Route predicate gateway cập nhật `Path=/api/identity/**`. |
| **P1 #3 (SEC-11)** | Verification token và link kích hoạt bị ghi rõ vào log khi đăng ký và gửi lại. | ✅ Hoàn thành | Loại bỏ hoàn toàn token và link khỏi log trong `AccountServiceImpl`. Chỉ ghi log định danh `userId` an toàn (`log.info("Email verified successfully for userId={}", user.getId())`). |
| **P1 #4 (AUTH-04, SEC-07)** | Refresh Token trả qua JSON response body, frontend phải lưu và gửi qua request body. | ✅ Hoàn thành | Refresh Token chỉ trao đổi qua Cookie `HttpOnly; Secure; SameSite=Strict; Path=/api/identity/auth`. Loại bỏ hoàn toàn trường `refreshToken` trong `LoginResponse` và `TokenRefreshResponse`. Bật CSRF bảo vệ bắt buộc cho refresh-token và logout. |
| **P1 #5 (AUTH-06)** | Database lưu Refresh Token nguyên dạng trong cột `token` và `replaced_by_token`. | ✅ Hoàn thành | Entity `RefreshToken` chỉ lưu `token_hash BINARY(32)` (SHA-256). Token nguyên dạng chỉ sinh tạm thời trong bộ nhớ để tạo `Set-Cookie`. Ánh xạ cột `replaced_by_id CHAR(36)` thay thế. |
| **P1 #6** | Replay Detection ném RuntimeException làm rollback giao dịch thu hồi phiên. | ✅ Hoàn thành | Tách ranh giới giao dịch: `AuthSessionService` commit việc thu hồi phiên và token trước trong transaction riêng, sau đó `AuthService` mới ném `InvalidRefreshTokenException` cho client. |
| **P1 #7** | Rotation không nguyên tử khi có nhiều request đồng thời dùng cùng một token. | ✅ Hoàn thành | Khóa hàng phiên `auth_sessions` (`SELECT ... FOR UPDATE`) trước khi đọc token. Các truy cập đồng thời với cùng 1 token sẽ bị xếp hàng; yêu cầu thứ hai phát hiện token đã được dùng (`consumed_at != null`) và kích hoạt thu hồi toàn bộ phiên. |
| **P1 #8 (AUTH-02)** | Mỗi lần refresh cấp hạn mới 7 ngày, cho phép kéo dài phiên vô hạn. | ✅ Hoàn thành | Hạn tuyệt đối 7 ngày được lưu tại `auth_sessions.expires_at` từ lúc login và cố định qua mọi lần xoay token. Expiry của Access Token lấy `min(15 phút, session.expires_at)`. |
| **P1 #9 (AUTH-07)** | Đăng xuất chưa thu hồi được quyền của Access Token theo phiên. | ✅ Hoàn thành | Bổ sung claim `sid` (Session ID) vào JWT. Thêm `SessionActiveValidator` vào `JwtDecoder` để kiểm tra trạng thái hoạt động của session phía máy chủ khi xác thực token. Đăng xuất lập tức vô hiệu hóa phiên. |

---

## 3. CƠ SỞ DỮ LIỆU & FLYWAY MIGRATION

Đã xóa các file migration cũ không tương thích (`V2__create_users_table.sql`, `V3__add_verification_tokens.sql`, `V4__create_refresh_tokens_table.sql`) và sử dụng schema domain chuẩn hóa:
- **`V1__initialize_service.sql`**: Khởi tạo dịch vụ.
- **`V2__create_domain_schema.sql`**: Bảng `users`, `auth_sessions`, `refresh_tokens`, `account_tokens`, `auth_rate_limit_buckets`, `outbox_events`, `processed_events`. Mọi mốc thời gian lưu UTC `DATETIME(6)`. Không dùng `ON DELETE CASCADE`.

---

## 4. DANH MỤC API ENDPOINTS CHUẨN HÓA (CANONICAL)

| STT | Phương thức | Đường dẫn API | Chức năng | Phân quyền | CSRF Protection |
| :---: | :---: | :--- | :--- | :---: | :---: |
| 1 | `POST` | `/api/identity/auth/register` | Đăng ký tài khoản mới (ACC-01..04, VERIFY-01) | Public | Exempt |
| 2 | `GET` | `/api/identity/auth/verify-email?token=...` | Xác nhận email bằng token một lần (VERIFY-03) | Public | GET |
| 3 | `POST` | `/api/identity/auth/resend-verification` | Gửi lại email xác nhận (VERIFY-03, VERIFY-04) | Public | Exempt |
| 4 | `POST` | `/api/identity/auth/login` | Đăng nhập cấp Access Token trong body & Refresh Token trong HttpOnly Cookie | Public | Exempt |
| 5 | `POST` | `/api/identity/auth/refresh-token` | Xoay vòng Refresh Token qua Cookie HttpOnly | Public | Bắt buộc (CSRF) |
| 6 | `POST` | `/api/identity/auth/logout` | Đăng xuất, thu hồi phiên máy chủ, xóa cookie | Public | Bắt buộc (CSRF) |
| 7 | `GET` | `/api/identity/.well-known/jwks.json` | Lấy danh sách khóa công khai RSA (JWK Set) | Public | GET |
| 8 | `GET` | `/api/identity/status` | Health check identity-service | Public | GET |

---

## 5. HỆ THỐNG KIỂM THỬ (TESTING & QUALITY ASSURANCE)

- **`services/identity-service`**: **44/44 Tests Passed (100%)**
  - `AccountServiceImplTest`: 16 tests bao phủ đăng ký, xác nhận email, cooldown 60s, rate limit 5 lần/giờ, xử lý concurrency duplicate email, xóa token nguyên dạng khỏi log.
  - `TokenServiceTest`: 3 tests kiểm tra JWT RS256, claim `sid`, capping hạn Access Token theo hạn phiên 7 ngày tuyệt đối (AUTH-02), và `SessionActiveValidator` từ chối token khi session bị thu hồi (AUTH-07).
  - `AuthServiceTest`: 6 tests kiểm tra luồng login, xoay token, commit thu hồi khi replay (không bị rollback) và logout.
  - `AuthControllerTest`: 16 tests bao phủ toàn bộ canonical endpoints, xác thực Set-Cookie HttpOnly, kiểm tra CSRF filter trên cookie endpoints.
  - `StatusSecurityTest`: 3 tests kiểm tra bảo mật scaffold và từ chối endpoint ngoài danh mục.
- **`platform/api-gateway`**: **8/8 Tests Passed (100%)**
  - `GatewaySecurityTest`: Kiểm tra chuyển tiếp correlation id, chặn endpoint trái phép, permit public auth routes không cần CSRF, và bắt buộc CSRF cho refresh/logout cookie endpoints.
- **Schema Validation Script (`scripts/test-schema.py`)**: **51/51 Assertions Passed (100%)**
  - Đã kiểm tra trực tiếp trên container MySQL 8.4 cô lập: toàn bộ ràng buộc bảng `users`, `auth_sessions`, `refresh_tokens`, `account_tokens`, `outbox_events` hoạt động chính xác.

---

## 6. LƯU Ý KHI KHỞI CHẠY DOCKER CONTAINER

Do cơ sở dữ liệu `tasknest_identity` trong Docker volume trước đây đã từng chạy Flyway với bộ script cũ (V2–V4), khi chạy lại container local cần tạo mới lại schema `tasknest_identity` để Flyway áp dụng script `V2__create_domain_schema.sql` mới sạch sẽ mà không gặp lỗi lệch checksum lịch sử migration.
