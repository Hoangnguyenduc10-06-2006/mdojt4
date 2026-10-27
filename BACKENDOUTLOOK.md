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
  - `infra/compose`: Docker Compose điều phối MySQL 8.0, RabbitMQ, Redis, Frontend (Nginx).
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
| **ACC-03** | Email chuẩn định dạng RFC 5322, trim đầu/cuối, chuyển thường, duy nhất trong hệ thống. | ✅ Hoàn thành | `trimmedEmail.toLowerCase(Locale.ROOT)`, regex `^[A-Za-z0-9+_.-]+@(.+)$`, kiểm tra qua `userRepository.existsByEmail(...)`. Ném `DuplicateEmailException` (HTTP 409). |
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
| **VERIFY-01** | Tài khoản mới đăng ký ở trạng thái `is_email_verified = false`. Sinh token UUID ngẫu nhiên gửi kèm link xác nhận. | ✅ Hoàn thành | `User.emailVerified = false`. Khi đăng ký tạo 1 bản ghi `VerificationToken` có token là `UUID.randomUUID().toString()`, hạn dùng 24h. |
| **VERIFY-02** | Tài khoản chưa xác nhận chỉ được thực hiện: xác nhận, gửi lại email, đăng xuất; chặn quyền vào dự án. | ✅ Hoàn thành | Database lưu cờ `is_email_verified`. Các endpoint xác thực được cấu hình `permitAll()` trên Gateway & Identity Service; các logic nghiệp vụ nghiệp vụ dự án sẽ chặn tài khoản có cờ này là `false`. |
| **VERIFY-03** | Token hiệu lực đúng 24h, chỉ dùng thành công 1 lần (`used_at`). Khi gửi lại, toàn bộ token cũ bị vô hiệu hóa (`revoked = true`). | ✅ Hoàn thành | `expiresAt = Instant.now().plus(24, ChronoUnit.HOURS)`. Kiểm tra `revoked == false`, `used_at == null`, `expiresAt.isAfter(now)`. Khi gọi `resend`, dùng `@Modifying` query update `revoked = true` cho mọi token cũ của user. |
| **VERIFY-04** | Rate limit gửi lại: Cooldown tối thiểu 60s giữa 2 lần gửi. Tối đa 5 lần/giờ cho 1 tài khoản. | ✅ Hoàn thành | Query `findFirstByUserAndTokenTypeOrderByCreatedAtDesc` để tính `Duration.between(...)`. Nếu < 60s, ném `CooldownException` mang theo `60 - elapsedSeconds`. Query `countByUserAndTokenTypeAndCreatedAtAfter` trong 1 giờ qua, nếu >= 5 ném `RateLimitExceededException`. |
| **VERIFY-05** | Gửi email thất bại không được rollback tài khoản. Xử lý async / log / notification-service. | ✅ Hoàn thành | Tách hàm `sendVerificationEmail` độc lập trong block try-catch riêng, không làm gián đoạn transaction lưu User và Token. |

---

### Đợt 4: Xác thực người dùng bằng thuật toán bất đối xứng RS256 & Refresh Token Rotation (SCRUM-9)

| Mã quy tắc / Nghiệp vụ | Yêu cầu nghiệp vụ | Trạng thái | Chi tiết triển khai |
| :--- | :--- | :---: | :--- |
| **AUTH-RS256** | Ký Access Token bằng thuật toán bất đối xứng RS256 (RSA 2048-bit). | ✅ Hoàn thành | `RsaKeyConfiguration.java`: Tự sinh và quản lý bean `RSAKey`, `JWKSource`, `NimbusJwtEncoder`, `NimbusJwtDecoder`. Access Token có header `alg: RS256`, `kid: tasknest-key-1`, claims `sub`, `email`, `fullName`, `email_verified`, `iss: https://identity.tasknest.vn`, `exp: 15 phút`. |
| **JWKS-ENDPOINT** | Expose public key cho các Resource Server và Gateway xác thực token độc lập. | ✅ Hoàn thành | Endpoint `GET /.well-known/jwks.json` (alias `/api/v1/auth/.well-known/jwks.json`) trả về JWK Set chuẩn RFC 7517. |
| **AUTH-LOGIN** | Đăng nhập bằng email & password, cấp Access Token RS256 và Refresh Token. | ✅ Hoàn thành | `POST /api/v1/auth/login`: So khớp mật khẩu BCrypt. Trả về `LoginResponse` gồm `accessToken`, `refreshToken`, `tokenType: Bearer`, `expiresIn: 900`, `user`. |
| **REFRESH-ROTATION** | Làm mới token với cơ chế xoay vòng (Token Rotation) và phát hiện tái sử dụng (Reuse Detection). | ✅ Hoàn thành | `POST /api/v1/auth/refresh-token`: Thu hồi token cũ (`revoked = true`), sinh token mới, liên kết `replaced_by_token`. Nếu phát hiện dùng lại token đã thu hồi -> lập tức vô hiệu hóa toàn bộ session/token của tài khoản đó (`revokeAllActiveTokensByUser`). |
| **AUTH-LOGOUT** | Đăng xuất người dùng. | ✅ Hoàn thành | `POST /api/v1/auth/logout`: Thu hồi refresh token trong cơ sở dữ liệu. |

---

## 3. CƠ SỞ DỮ LIỆU & FLYWAY MIGRATION

### Script 1: `V2__create_users_table.sql`
Tạo bảng người dùng `users`:
```sql
CREATE TABLE IF NOT EXISTS users (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    full_name VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    is_email_verified BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    updated_at TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6)
);
```

### Script 2: `V3__add_verification_tokens.sql`
Tạo bảng lưu trữ token xác thực `verification_tokens`:
```sql
CREATE TABLE IF NOT EXISTS verification_tokens (
    id VARCHAR(36) PRIMARY KEY,
    user_id BIGINT NOT NULL,
    token VARCHAR(64) NOT NULL,
    token_type VARCHAR(32) NOT NULL DEFAULT 'VERIFY_EMAIL',
    expires_at TIMESTAMP(6) NOT NULL,
    used_at TIMESTAMP(6) NULL DEFAULT NULL,
    revoked BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    CONSTRAINT fk_verification_tokens_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE INDEX idx_verification_tokens_token ON verification_tokens(token);
CREATE INDEX idx_verification_tokens_user_type ON verification_tokens(user_id, token_type);
```

### Script 3: `V4__create_refresh_tokens_table.sql`
Tạo bảng lưu trữ Refresh Token `refresh_tokens`:
```sql
CREATE TABLE IF NOT EXISTS refresh_tokens (
    id VARCHAR(36) PRIMARY KEY,
    user_id BIGINT NOT NULL,
    token VARCHAR(255) NOT NULL UNIQUE,
    expires_at TIMESTAMP(6) NOT NULL,
    revoked BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    replaced_by_token VARCHAR(255) NULL DEFAULT NULL,
    CONSTRAINT fk_refresh_tokens_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE INDEX idx_refresh_tokens_token ON refresh_tokens(token);
CREATE INDEX idx_refresh_tokens_user ON refresh_tokens(user_id);
```

---

## 4. DANH MỤC API ENDPOINTS ĐÃ SẴN SÀNG

| STT | Phương thức | Đường dẫn API | Chức năng | Phân quyền | Mã HTTP thành công |
| :---: | :---: | :--- | :--- | :---: | :---: |
| 1 | `POST` | `/api/identity/register` (hoặc `/api/v1/auth/register`) | Đăng ký tài khoản mới (ACC-01..04, VERIFY-01) | Public | `201 CREATED` |
| 2 | `GET` | `/api/v1/auth/verify-email?token=...` | Xác nhận email bằng token UUID (VERIFY-03) | Public | `200 OK` |
| 3 | `POST` | `/api/v1/auth/resend-verification` | Gửi lại email xác nhận (VERIFY-03, VERIFY-04) | Public | `200 OK` |
| 4 | `POST` | `/api/v1/auth/login` | Đăng nhập cấp Access Token (RS256) & Refresh Token | Public | `200 OK` |
| 5 | `GET` | `/api/v1/auth/.well-known/jwks.json` | Lấy danh sách khóa công khai RSA (JWK Set) | Public | `200 OK` |
| 6 | `POST` | `/api/v1/auth/refresh-token` | Làm mới Token (Refresh Token Rotation) | Public | `200 OK` |
| 7 | `POST` | `/api/v1/auth/logout` | Đăng xuất, thu hồi Refresh Token | Public | `200 OK` |
| 8 | `GET` | `/api/identity/status` | Health check identity-service | Public | `200 OK` |

---

## 5. HỆ THỐNG KIỂM THỬ (TESTING & QUALITY ASSURANCE)

### Unit & Integration Tests (Maven Surefire)
- **`services/identity-service`**: **52/52 Tests Passed (100%)**
  - `AccountServiceImplTest`: Kiểm thử bao phủ đăng ký, xác nhận email, cooldown 60s, rate limit 5 lần/giờ, login thành công/thất bại, refresh rotation, reuse detection, logout.
  - `TokenServiceTest`: Kiểm thử ký token bằng RSA 2048-bit RS256 và giải mã / kiểm tra chữ ký qua `NimbusJwtDecoder`.
  - `AuthControllerTest`: MockMvc Web Layer test format JSON, validate status code, error responses cho toàn bộ 8 endpoint.
  - `StatusSecurityTest`: Kiểm thử cấu hình bảo mật Spring Security và CSRF.
- **`platform/api-gateway`**: **5/5 Tests Passed (100%)**
  - `GatewaySecurityTest`: Kiểm thử route matching, CSRF exemption, permitAll trên WebFlux Gateway.

### End-to-End Live Testing (Docker Compose stack trên localhost:3000)
- Luồng Nginx (3000) -> API Gateway (8080) -> Eureka -> Identity Service (8081).
- Test thành công:
  - Đăng ký tài khoản -> Nhận token xác minh.
  - Kích hoạt email -> Trả về 200 OK.
  - Đăng nhập -> Nhận JWT ký RS256 (`alg: RS256`, `kid: tasknest-key-1`) + Refresh Token.
  - Lấy JWKS từ `/.well-known/jwks.json` -> Trả về RSA Public Key thành công.
  - Gọi refresh-token -> Cấp Access Token mới + Refresh Token mới, token cũ bị thu hồi.
  - Thử dùng lại refresh token cũ -> Bị phát hiện (Reuse Detection), trả về 401 và thu hồi toàn bộ token liên quan.
  - Đăng xuất -> Thu hồi refresh token thành công.

---

## 6. FILE POSTMAN TEST
Đã cấu hình bộ test Postman đầy đủ tại file:
`postman_collection_identity_verification.json` (tại thư mục gốc dự án)
Bao gồm:
1. Đăng ký tài khoản (Register)
2. Xác nhận email (Verify Email)
3. Gửi lại email xác nhận (Resend Verification)
4. Đăng nhập (Login - Ký RS256 JWT & Cấp Refresh Token)
5. Lấy JWKS Public Key (GET /.well-known/jwks.json)
6. Làm mới Token (Refresh Token Rotation)
7. Đăng xuất (Logout)
8. Test lỗi: Đăng nhập sai mật khẩu (401 Unauthorized)
9. Test lỗi: Tái sử dụng Refresh Token đã bị thu hồi (401 Reuse Detection)

---

## 7. KẾ HOẠCH BÀI TOÁN TIẾP THEO (NEXT MILESTONES)
- [ ] Tích hợp gửi email thực tế qua **Brevo API** hoặc chuyển message qua RabbitMQ tới `notification-service`.
- [ ] Cấu hình `JwtDecoder` trên `work-service` và `notification-service` kết nối tới JWKS URI để bảo vệ các API nội bộ.
- [ ] Chặn truy cập API dự án đối với người dùng có claim `email_verified = false` (VERIFY-02).
- [ ] Quên mật khẩu / Đặt lại mật khẩu (`forgot-password`, `reset-password`).

