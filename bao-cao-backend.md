# Báo Cáo Backend — Hệ Thống Quản Lý Nhà Hàng KTHP-LTW

**Môn học:** Lập trình Web (RIPT1307-02-2026)  
**Nhóm:** 12  
**Thành viên:** Đoàn Vũ Phúc · Lê Minh Đạo · Trần Minh Trí

---

## 1. Tổng Quan Dự Án

Hệ thống **KTHP-LTW** là một ứng dụng quản lý nhà hàng full-stack hiện đại, được xây dựng nhằm số hoá toàn bộ quy trình phục vụ — từ lúc khách quét mã QR đặt món cho đến khi thanh toán và đóng phiên. Hệ thống tích hợp:

- **Đặt món QR**: Khách tự quét mã QR tại bàn, xem thực đơn và gửi đơn hàng mà không cần nhân viên hỗ trợ.
- **Kitchen Display System (KDS)**: Màn hình bếp hiển thị hàng đợi theo trạm (GRILL / BAR / COLD) và cập nhật trạng thái món theo thời gian thực.
- **Thanh toán VNPay**: Cổng thanh toán trực tuyến với webhook idempotency, đảm bảo không xử lý trùng giao dịch.
- **Báo cáo & Thống kê**: Dashboard cho Admin/Manager với khả năng xuất báo cáo Excel và gửi email tự động hàng ngày.
- **Real-time Updates**: Sử dụng Socket.IO để đồng bộ trạng thái đơn hàng, bàn, và thông báo giữa tất cả vai trò người dùng.

Backend được phát triển theo kiến trúc phân lớp (Layered Architecture) với Node.js + Express 5, kết hợp PostgreSQL 16 và Redis, triển khai bằng Docker Compose.

---

## 2. Công Nghệ Sử Dụng

### 2.1. Core Framework & Runtime

| Công nghệ | Phiên bản | Vai trò |
|---|---|---|
| **Node.js** | 20+ | JavaScript runtime — nền tảng thực thi backend |
| **Express** | 5.2 | Web framework — xử lý routing, middleware, HTTP request/response |
| **Socket.IO** | 4.8 | Giao tiếp hai chiều thời gian thực qua WebSocket |

**Lý do chọn Express 5:** Express 5 hỗ trợ async/await natively trong route handlers, tự động bắt lỗi async và chuyển đến error handler mà không cần try/catch thủ công hoặc `express-async-errors`. Đây là bước cải tiến lớn so với Express 4.

### 2.2. Cơ Sở Dữ Liệu

| Công nghệ | Phiên bản | Vai trò |
|---|---|---|
| **PostgreSQL** | 16 | RDBMS chính — lưu trữ toàn bộ dữ liệu nghiệp vụ |
| **pg (node-postgres)** | 8.20 | Driver kết nối PostgreSQL từ Node.js, hỗ trợ connection pool |
| **Redis** | 7 (Alpine) | In-memory cache — lưu distributed lock cho webhook idempotency |
| **ioredis** | 5.10 | Redis client cho Node.js với hỗ trợ Promises |

**Lý do dùng PostgreSQL:** Dự án yêu cầu transaction ACID đảm bảo tính nhất quán dữ liệu (đặt hàng, thanh toán, quản lý quota), hỗ trợ JSONB để lưu cấu hình linh hoạt, và khả năng tối ưu hiệu năng bằng 30+ indexes.

**Lý do dùng Redis:** Redis được sử dụng như distributed lock để ngăn xử lý trùng webhook VNPay trong môi trường có nhiều instance. Redis là optional — nếu không có Redis, hệ thống vẫn tiếp tục bằng cơ chế idempotency ở tầng DB.

### 2.3. Xác Thực & Bảo Mật

| Công nghệ | Phiên bản | Vai trò |
|---|---|---|
| **jsonwebtoken** | 9.0 | Tạo và xác thực JWT (access token + session token) |
| **bcrypt** | 6.0 | Hash mật khẩu với salt rounds |
| **helmet** | 8.1 | Thiết lập HTTP security headers |
| **express-rate-limit** | 8.5 | Giới hạn request theo IP, chống brute-force |
| **cors** | 2.8 | Quản lý Cross-Origin Resource Sharing |

### 2.4. Validation & Logging

| Công nghệ | Phiên bản | Vai trò |
|---|---|---|
| **Zod** | 4.4 | Schema validation — kiểm tra toàn bộ input tại API boundary |
| **Winston** | 3.19 | Structured logging với multiple transports |
| **swagger-jsdoc** | 6.2 | Sinh tài liệu OpenAPI từ JSDoc comments |
| **swagger-ui-express** | 5.0 | Tự động render Swagger UI tại `/api/docs` |

### 2.5. Tích Hợp Bên Thứ Ba

| Công nghệ | Vai trò |
|---|---|
| **VNPay** | Cổng thanh toán trực tuyến — tạo payment URL, xử lý IPN webhook |
| **Nodemailer** | Gửi email báo cáo doanh thu qua SMTP |
| **Mailtrap API** | Alternative email API cho môi trường production (tránh SMTP bị chặn port) |
| **node-cron** | Cron job scheduler — reset quota hàng ngày, gửi email báo cáo tự động |

### 2.6. Upload & File Storage

| Công nghệ | Phiên bản | Vai trò |
|---|---|---|
| **multer** | 2.1 | Middleware xử lý `multipart/form-data` upload ảnh món ăn |
| **ExcelJS** | 4.4 | Tạo file Excel (.xlsx) cho báo cáo xuất vận hành |
| **uuid** | 14.0 | Tạo UUID v4 cho request tracking và transaction reference |

### 2.7. DevOps & Testing

| Công nghệ | Vai trò |
|---|---|
| **Docker** | Container hóa từng service (backend, postgres, redis, seeder, indexer) |
| **Docker Compose** | Orchestrate toàn bộ stack cục bộ |
| **Jest** | Unit testing framework — 22 test files, 27+ test cases |
| **nodemon** | Hot-reload trong môi trường development |
| **node-pg-migrate** | Quản lý database migration phiên bản |

---

## 3. Kiến Trúc Hệ Thống

### 3.1. Tổng Thể

Backend áp dụng **Layered Architecture** (kiến trúc phân lớp):

```
┌─────────────────────────────────────────────────────────┐
│                      Client Layer                        │
│  Customer App │ Staff App │ KDS Screen │ Admin Dashboard │
└───────────────────────────┬─────────────────────────────┘
                            │ HTTP / WebSocket
┌───────────────────────────▼─────────────────────────────┐
│                   Express 5 API Server                   │
│                                                          │
│  helmet → cors → json → requestTracker → rateLimiter     │
│                    ↓                                     │
│              Routes / Controllers                        │
│                    ↓                                     │
│              Service Layer                               │
│   auth │ order │ payment │ kds │ report │ session        │
│                    ↓                                     │
│              Socket.IO Server                            │
│   /customer │ /kitchen │ /staff                          │
└──────────────┬──────────────────────────┬───────────────┘
               │                          │
┌──────────────▼──────────┐  ┌────────────▼──────────────┐
│     PostgreSQL 16        │  │          Redis             │
│  (persistent data)       │  │  (webhook lock/idempotency)│
└─────────────────────────┘  └───────────────────────────┘
               │
┌──────────────▼──────────┐
│     VNPay Gateway        │
│  (payment IPN webhook)   │
└─────────────────────────┘
```

### 3.2. Cấu Trúc Thư Mục

```
src/BE_THLTW/
├── src/
│   ├── app.js               # Express app, middleware chain, Swagger UI
│   ├── server.js            # HTTP server, Socket.IO attach, cron jobs
│   ├── routes/              # Định nghĩa API routes (7 file)
│   ├── controllers/         # Xử lý Request/Response, không chứa business logic
│   ├── services/            # Business logic & SQL queries (12 service)
│   ├── middlewares/         # auth, validation, error, rateLimit, tracking
│   ├── validators/          # Zod schemas (4 validator file)
│   ├── sockets/             # Socket.IO namespaces & auth
│   ├── config/              # DB, Redis, Swagger, VNPay, schema, seed
│   └── utils/               # JWT, VNPay, logger, errors, response util
├── __tests__/               # 22 test files (Jest)
├── Dockerfile
├── docker-compose.yml
├── package.json
└── postman_collection.json
```

### 3.3. Middleware Chain

Mỗi HTTP request được xử lý qua chuỗi middleware theo thứ tự:

```
helmet()
  → cors()
  → express.json()
  → requestTracker (gán req.id UUID v4, log incoming request)
  → rateLimiter
  → Routes
    → authMiddleware (verify JWT / session token)
    → validate(schema) (Zod)
    → controller → service → DB
  → errorHandler (global)
```

**Request Tracking:** Mỗi request được gán một UUID v4 duy nhất (`req.id`) và đo thời gian phản hồi. Thông tin này được log bằng Winston để hỗ trợ debug và monitoring.

---

## 4. Phân Tích API

### 4.1. Quy Ước Chung

Backend sử dụng định dạng response thống nhất:

**Response thành công:**
```json
{
  "success": true,
  "message": "OK",
  "data": {}
}
```

**Response lỗi:**
```json
{
  "success": false,
  "message": "Dữ liệu không hợp lệ",
  "errors": [
    { "field": "body.qr_code", "message": "Invalid input: expected string, received undefined" }
  ]
}
```

> **Lưu ý:** Backend dùng PostgreSQL `SERIAL`, nên tất cả ID trong API là số nguyên (integer), không phải UUID.

**Base URL:** `http://localhost:5000/api`  
**Swagger UI:** `http://localhost:5000/api/docs`

---

### 4.2. Module Auth — Xác Thực Nhân Viên

**Prefix:** `/api/auth`

| Method | Endpoint | Mô tả |
|---|---|---|
| `POST` | `/auth/login` | Đăng nhập nhân viên |
| `POST` | `/auth/refresh` | Làm mới access token |
| `POST` | `/auth/logout` | Thu hồi refresh token |

**Luồng xác thực nhân viên (Staff JWT):**

1. **Login:** Client gửi email + password → Server kiểm tra DB, so khớp bcrypt → Trả về `accessToken` (15 phút) + `refreshToken` (7 ngày).
2. **Token Storage:** `refreshToken` được hash SHA-256 trước khi lưu vào bảng `REFRESH_TOKENS`, không bao giờ lưu raw token.
3. **Refresh:** Client gửi `refreshToken` → Server verify signature → Kiểm tra hash trong DB (chưa revoked, chưa hết hạn) → Issue token mới, revoke token cũ (token rotation).
4. **Logout:** Server revoke toàn bộ refresh token active của user đó.

**Request Login:**
```json
{
  "email": "admin@restaurant.com",
  "password": "Password123!"
}
```

**Response Login:**
```json
{
  "success": true,
  "data": {
    "accessToken": "eyJ...",
    "refreshToken": "eyJ...",
    "user": { "id": 1, "full_name": "Nguyễn Admin", "role": "ADMIN" }
  }
}
```

---

### 4.3. Module Customer — Khách Hàng QR

**Prefix:** `/api/customer`

| Method | Endpoint | Auth | Mô tả |
|---|---|---|---|
| `POST` | `/customer/scan` | Rate limit | Quét mã QR để khởi tạo phiên |
| `GET` | `/customer/session` | Session token | Lấy thông tin phiên hiện tại |
| `GET` | `/customer/menu` | Session token | Lấy menu (lọc theo station/category) |
| `POST` | `/customer/orders` | Session token | Tạo đơn hàng mới |
| `GET` | `/customer/orders` | Session token | Xem danh sách đơn của phiên |
| `POST` | `/customer/requests` | Session token | Gọi nhân viên hoặc yêu cầu thanh toán |
| `POST` | `/customer/payment/vnpay` | Session token | Tạo URL thanh toán VNPay |

**Luồng đặt hàng:**

1. Khách quét QR → `POST /customer/scan { qr_code }` → Hệ thống tạo session, trả về `session_token` JWT 24h.
2. Khách xem menu → `GET /customer/menu` → Server trả danh sách món ăn kèm options.
3. Khách gửi đơn → `POST /customer/orders` với `session_version` (dùng cho optimistic locking):

```json
{
  "session_version": 1,
  "items": [
    {
      "menu_item_id": 1,
      "quantity": 2,
      "note": "ít cay",
      "options": [{ "option_id": 1, "quantity": 1 }]
    }
  ]
}
```

**Optimistic Locking trong `createOrder`:**

Hàm `createOrder` thực thi trong một **database transaction** với các bước chặt chẽ:

1. `BEGIN`
2. `SELECT ... FOR UPDATE` — khóa row session theo `id` và `version`; nếu không khớp version → báo lỗi conflict
3. Lock từng `MENU_ITEM` → kiểm tra `is_available` và `daily_quota >= quantity`
4. Trừ quota: `UPDATE MENU_ITEMS SET daily_quota = daily_quota - $1 WHERE daily_quota >= $1` (chống race condition)
5. Insert `ORDERS` + `ORDER_ITEMS` + `ORDER_ITEM_OPTIONS`
6. Tính lại bill session (`calculateSessionBill`)
7. Tăng `session.version`
8. `COMMIT`
9. Sau khi release DB client: emit socket `new_order` → namespace `/kitchen`

> Tách emit socket ra ngoài transaction đảm bảo lỗi socket không rollback order đã commit.

---

### 4.4. Module KDS — Kitchen Display System

**Prefix:** `/api/kds` — Chỉ role `KITCHEN` và `ADMIN`

| Method | Endpoint | Mô tả |
|---|---|---|
| `GET` | `/kds/orders?station=GRILL` | Lấy hàng đợi theo trạm chế biến |
| `PATCH` | `/kds/items/:id/status` | Cập nhật trạng thái của một món |

**Trạm chế biến hợp lệ:** `GRILL`, `BAR`, `COLD`  
**Trạng thái món hợp lệ:** `PREPARING` → `READY` → `SERVED`

**Luồng cập nhật trạng thái:**

Khi bếp cập nhật `order_item`:
1. Update `ORDER_ITEMS.status`
2. Kiểm tra tất cả items trong cùng `ORDER`:
   - Tất cả `SERVED` → order thành `SERVED`
   - Tất cả `READY/SERVED` → order thành `READY`
   - Có `PREPARING/READY` → order thành `PREPARING`
3. Nếu order status thay đổi → ghi log vào `ORDER_STATUS_LOGS`
4. Emit socket `order_status_updated` → namespace `/customer` (thông báo cho khách)

---

### 4.5. Module Staff — Nhân Viên

**Prefix:** `/api/staff`

| Method | Endpoint | Auth | Mô tả |
|---|---|---|---|
| `GET` | `/staff/tables` | WAITER/CASHIER/MANAGER/ADMIN | Danh sách tất cả bàn |
| `GET` | `/staff/tables/:id/session` | Staff JWT | Phiên đang active của bàn |
| `POST` | `/staff/sessions/:id/checkout` | CASHIER/MANAGER/ADMIN | Thanh toán tiền mặt |
| `GET` | `/staff/requests` | Staff JWT | Danh sách yêu cầu đang mở |
| `PATCH` | `/staff/requests/:id/resolve` | Staff JWT | Đóng một yêu cầu |
| `PATCH` | `/staff/orders/items/:id/cancel` | CASHIER/MANAGER/ADMIN | Hủy một món |
| `POST` | `/staff/sessions/:id/force-close` | MANAGER/ADMIN | Đóng phiên khẩn cấp |

**Checkout tiền mặt:** Backend kiểm tra session còn `ACTIVE` và `amount >= final_amount` trước khi tạo payment `CASH` và đóng phiên.

---

### 4.6. Module Admin — Quản Trị Hệ Thống

**Prefix:** `/api/admin`

#### Báo Cáo & Thống Kê

| Method | Endpoint | Auth | Mô tả |
|---|---|---|---|
| `GET` | `/admin/reports/revenue` | ADMIN/MANAGER | Doanh thu theo ngày/tuần/tháng |
| `GET` | `/admin/reports/menu` | ADMIN/MANAGER | Top 20 món bán chạy nhất |
| `GET` | `/admin/reports/kds` | ADMIN/MANAGER | Thống kê thời gian chế biến |
| `GET` | `/admin/reports/export` | ADMIN/MANAGER | Xuất báo cáo Excel |
| `POST` | `/admin/menu/reset-quota` | ADMIN/MANAGER | Reset quota tất cả món |
| `POST` | `/admin/reports/daily-email/send` | ADMIN | Gửi email báo cáo theo ngày |
| `POST` | `/admin/reports/daily-email/send-now` | ADMIN | Gửi ngay đến email tuỳ chọn |
| `GET` | `/admin/reports/daily-email/status` | ADMIN | Xem cấu hình và lịch sử gửi email |

**Tham số báo cáo doanh thu:**
```
GET /api/admin/reports/revenue?from=2026-05-01&to=2026-05-31&group_by=day
```

**Response mẫu:**
```json
[
  { "date": "2026-05-24T00:00:00.000Z", "method": "CASH", "total": "450000", "order_count": "9" },
  { "date": "2026-05-24T00:00:00.000Z", "method": "VNPAY", "total": "320000", "order_count": "5" }
]
```

#### CRUD Quản Lý Tài Nguyên

| Resource | Endpoints | Hành vi xóa |
|---|---|---|
| Users | `GET/POST /admin/users`, `PUT/DELETE /admin/users/:id` | Soft delete: `is_active = false` |
| Tables | `GET/POST /admin/tables`, `PUT/DELETE /admin/tables/:id` | Hard delete (chỉ khi bàn `AVAILABLE`) |
| QR Codes | `GET/POST /admin/qr_codes`, `PATCH .../toggle`, `DELETE .../:id` | Hard delete |
| Categories | `GET/POST /admin/menu/categories`, `PUT/DELETE .../:id` | Soft delete; chặn nếu còn item active |
| Menu Items | `GET/POST /admin/menu/items`, `PUT/DELETE .../:id` | Soft delete: `is_available = false` |
| Options | `GET/POST /admin/menu/items/:id/options`, `PUT/DELETE /admin/menu/options/:id` | Soft delete |

#### Upload Ảnh Món Ăn

```
POST /api/admin/menu/images
Content-Type: multipart/form-data
```

Hỗ trợ hai phương thức upload:
- **File upload** (`multipart/form-data`): Lưu file lên disk, trả về URL tĩnh
- **Base64 upload** (`/api/admin/menu/images/base64`): Lưu chuỗi Base64 trực tiếp vào DB

Giới hạn: JPEG / PNG / WebP, tối đa 5MB, validate MIME type và nội dung thực tế.

#### Cấu Hình Ngân Hàng (QR Payment)

| Method | Endpoint | Mô tả |
|---|---|---|
| `GET` | `/admin/settings/bank` | Lấy thông tin tài khoản ngân hàng |
| `POST` | `/admin/settings/bank` | Lưu/cập nhật tài khoản ngân hàng |

Thông tin ngân hàng được lưu dưới dạng JSONB trong bảng `RESTAURANT_SETTINGS` với key `'bank_config'`, dùng để sinh QR chuyển khoản động cho khách.

---

### 4.7. Module Webhooks — VNPay IPN

**Prefix:** `/api/webhooks`

| Method | Endpoint | Mô tả |
|---|---|---|
| `GET` / `POST` | `/webhooks/vnpay` | Nhận và xử lý IPN từ VNPay |

**Luồng xử lý webhook idempotent:**

```
VNPay gọi IPN
  → 1. Verify HMAC-SHA512 secure hash
  → 2. Kiểm tra Redis lock webhook:vnpay:{txnRef} (TTL 60s)
        Nếu lock tồn tại → return RspCode: '02' (đã xử lý)
  → 3. Acquire lock
  → 4. BEGIN transaction
  → 5. SELECT PAYMENTS FOR UPDATE (kiểm tra tồn tại, đối chiếu amount)
  → 6. Nếu payment đã COMPLETED/FAILED → return RspCode: '02'
  → 7. Nếu vnp_ResponseCode = '00':
        UPDATE PAYMENTS status = 'COMPLETED'
        UPDATE SESSIONS status = 'CLOSED'
        UPDATE TABLES status = 'AVAILABLE'
        Auto-resolve CUSTOMER_REQUESTS
  → 8. COMMIT
  → 9. Emit socket: session_closed → /customer, table_status_changed → /staff
  → 10. Release Redis lock
```

---

### 4.8. Health Check

| Method | Endpoint | Mô tả |
|---|---|---|
| `GET` | `/health` | Kiểm tra trạng thái server |

```json
{ "status": "UP", "message": "Hệ thống đang hoạt động" }
```

---

## 5. Hệ Thống Real-time (Socket.IO)

### 5.1. Kiến Trúc Namespace

Socket.IO được tổ chức thành 3 namespace riêng biệt, mỗi namespace có cơ chế xác thực khác nhau:

| Namespace | Xác thực khi connect | Vai trò được phép |
|---|---|---|
| `/customer` | Không cần auth khi connect; verify qua `join_session` | Khách hàng |
| `/kitchen` | JWT access token | `KITCHEN`, `ADMIN` |
| `/staff` | JWT access token | `WAITER`, `CASHIER`, `MANAGER`, `ADMIN` |

### 5.2. Danh Sách Events

| Event | Namespace | Chiều | Mô tả |
|---|---|---|---|
| `join_session` | `/customer` | Client → Server | Tham gia phòng phiên sau khi kết nối |
| `order_status_updated` | `/customer` | Server → Client | Thông báo trạng thái đơn hàng thay đổi |
| `session_closed` | `/customer` | Server → Client | Thông báo phiên đã đóng (lý do: PAID) |
| `new_order` | `/kitchen` | Server → Client | Đơn mới vào bếp (gửi theo station) |
| `table_status_changed` | `/staff` | Server → Client | Trạng thái bàn thay đổi |

### 5.3. Ví Dụ Kết Nối Client

```javascript
// Customer kết nối namespace /customer
const socket = io('http://localhost:5000/customer');

// Sau khi connect, join vào room của phiên
socket.emit('join_session', {
  session_id: 5,
  session_token: 'eyJ...'
});

// Nhận cập nhật trạng thái đơn hàng
socket.on('order_status_updated', ({ order_id, new_status, changed_at }) => {
  console.log(`Đơn #${order_id} đã ${new_status}`);
});

// Staff/Kitchen kết nối (cần JWT)
const staffSocket = io('http://localhost:5000/staff', {
  auth: { token: accessToken }
});
```

---

## 6. Bảo Mật

### 6.1. Hệ Thống Phân Quyền 6 Roles

| Role | Quyền hạn |
|---|---|
| `ADMIN` | Toàn quyền: CRUD users, tables, menu, reports, settings |
| `MANAGER` | Quản lý vận hành: bàn, QR, menu, báo cáo; không quản lý users |
| `CASHIER` | Checkout, hủy món, xem requests |
| `WAITER` | Xem bàn, xử lý customer requests |
| `KITCHEN` | Xem KDS queue, cập nhật trạng thái món |
| `CUSTOMER` | Tự phục vụ qua QR (dùng session token riêng biệt) |

### 6.2. JWT Token Strategy

- **Access Token:** Thời hạn 15 phút, signed bằng `JWT_ACCESS_SECRET`
- **Refresh Token:** Thời hạn 7 ngày, signed bằng `JWT_REFRESH_SECRET`, lưu DB dạng **SHA-256 hash** (không lưu raw token)
- **Session Token:** Thời hạn 24 giờ, payload `{ session_id, type: 'session' }` — dùng riêng cho khách hàng

### 6.3. Tổng Hợp Biện Pháp Bảo Mật

| Biện pháp | Công cụ | Mô tả |
|---|---|---|
| HTTP Security Headers | `helmet` | Ngăn XSS, clickjacking, MIME sniffing |
| CORS Whitelist | `cors` | Production chỉ chấp nhận origin trong `FRONTEND_URL` |
| Rate Limiting | `express-rate-limit` | Giới hạn request/IP, áp dụng cho `/customer/scan` |
| Input Validation | `Zod` | Validate schema tại mọi API boundary |
| Password Hashing | `bcrypt` | Hash với salt, không lưu plain-text |
| Optimistic Locking | PostgreSQL `FOR UPDATE` + `version` | Tránh race condition khi đặt hàng đồng thời |
| Webhook Idempotency | Redis lock + DB check | Không xử lý trùng giao dịch VNPay |
| Secret Hygiene | `validateEnv.js` | Kiểm tra biến môi trường bắt buộc khi khởi động |
| Error Masking | `error.middleware.js` | Production: ẩn stack trace, chỉ trả "Lỗi hệ thống" |

---

## 7. Cơ Sở Dữ Liệu

### 7.1. Các Bảng Chính

| Bảng | Mô tả |
|---|---|
| `USERS` | Tài khoản nhân viên, role, `is_active` |
| `REFRESH_TOKENS` | Lưu SHA-256 hash của refresh token, `revoked_at`, `expires_at` |
| `TABLES` | Bàn nhà hàng, trạng thái `AVAILABLE` / `OCCUPIED` |
| `MENU_CATEGORIES` | Danh mục món, thuộc trạm `GRILL` / `BAR` / `COLD` |
| `MENU_ITEMS` | Món ăn, giá, `daily_quota`, `daily_quota_default`, `is_available` |
| `MENU_ITEM_OPTIONS` | Tuỳ chọn thêm của món (topping, size...), `extra_price` |
| `QR_CODES` | Mã QR liên kết với bàn, có thể bật/tắt |
| `SESSIONS` | Phiên khách tại bàn, `subtotal`, `final_amount`, `version` (optimistic lock) |
| `ORDERS` | Đơn hàng trong phiên, status |
| `ORDER_ITEMS` | Chi tiết từng món trong đơn, status riêng cho từng món |
| `ORDER_ITEM_OPTIONS` | Options đã chọn của từng order item |
| `ORDER_STATUS_LOGS` | Nhật ký thay đổi trạng thái đơn hàng |
| `PAYMENTS` | Giao dịch thanh toán (CASH / VNPAY), `transaction_id`, `webhook_data` |
| `CUSTOMER_REQUESTS` | Yêu cầu gọi nhân viên / xin thanh toán (`CALL_STAFF`, `REQUEST_BILL`, `OTHER`) |
| `INVOICES` | Hóa đơn chính thức khi đóng phiên (`ISSUED` / `SUPERSEDED` / `CANCELLED`) |
| `INVOICE_LINE_ITEMS` | Bản sao chi tiết món tại thời điểm xuất hóa đơn (cố định giá đối soát) |
| `INVOICE_PRINT_EVENTS` | Nhật ký in hóa đơn (`PRINT` / `REPRINT`) |
| `RESTAURANT_SETTINGS` | Cấu hình hệ thống dạng JSONB (key: `bank_config`) |

### 7.2. Indexes Hiệu Năng (30+)

Hệ thống áp dụng hơn 30 index tối ưu hiệu năng truy vấn:

```sql
-- Chống nhiều session active cùng bàn
CREATE UNIQUE INDEX uniq_sessions_one_active_per_table
ON SESSIONS(table_id) WHERE status = 'ACTIVE';

-- Tìm kiếm nhanh refresh token
CREATE INDEX idx_refresh_tokens_token ON REFRESH_TOKENS(token);

-- Query order theo session
CREATE INDEX idx_orders_session_id ON ORDERS(session_id);
CREATE INDEX idx_orders_status ON ORDERS(status);
CREATE INDEX idx_orders_created_at ON ORDERS(created_at);

-- KDS query theo station
CREATE INDEX idx_menu_items_category ON MENU_ITEMS(category_id);

-- Payment lookup
CREATE INDEX idx_payments_transaction_id ON PAYMENTS(transaction_id);
CREATE INDEX idx_payments_session_id ON PAYMENTS(session_id);
CREATE INDEX idx_payments_paid_at ON PAYMENTS(paid_at);
```

---

## 8. Cron Jobs & Tác Vụ Nền

### 8.1. Daily Quota Reset

```javascript
cron.schedule('0 0 * * *', async () => {
  await db.query('UPDATE MENU_ITEMS SET daily_quota = daily_quota_default');
}, { timezone: 'Asia/Ho_Chi_Minh' });
```

Mỗi ngày lúc 00:00 giờ Việt Nam, tất cả `daily_quota` của món ăn được reset về giá trị mặc định `daily_quota_default`.

### 8.2. Daily Revenue Email Report

```javascript
// Mặc định chạy lúc 00:05 hàng ngày (cấu hình qua REPORT_EMAIL_CRON)
cron.schedule('5 0 * * *', async () => {
  await dailyRevenueEmailService.sendDailyReport();
}, { timezone: 'Asia/Ho_Chi_Minh' });
```

Service `dailyRevenueEmail.service.js` tổng hợp doanh thu ngày hôm trước và gửi email đến danh sách recipients. Hỗ trợ:
- **SMTP** (qua Nodemailer) cho môi trường local/development
- **Mailtrap API** cho production (tránh bị chặn port SMTP trên cloud hosting)

Admin có thể gửi thủ công qua API `POST /api/admin/reports/daily-email/send` hoặc kiểm tra lịch sử gửi qua `GET /api/admin/reports/daily-email/status`.

### 8.3. Keepalive Bot

Service `keepalive.service.js` ping định kỳ đến các URL trong `KEEPALIVE_TARGETS` (mặc định mỗi 10 phút) để giữ cho các dịch vụ trên Render.com không rơi vào trạng thái "idle sleep" sau 15 phút không hoạt động.

---

## 9. Testing

### 9.1. Tổng Quan

| Thống kê | Giá trị |
|---|---|
| Test framework | Jest 29 |
| Số file test | 22 |
| Phạm vi | Unit tests + Integration tests |
| Chạy lệnh | `npm test` (trong thư mục `src/BE_THLTW`) |

### 9.2. Danh Sách Test Suites

| File Test | Phạm vi kiểm thử |
|---|---|
| `auth.test.js` | Luồng login, refresh token, logout |
| `auth-middleware.test.js` | Xác thực JWT, session token |
| `order.test.js` | Tạo đơn hàng, optimistic locking, quota check |
| `session.test.js` | Scan QR, tạo session |
| `kds.test.js` | KDS queue, cập nhật status món |
| `vnpay.test.js` | Webhook idempotency, checksum verification |
| `validation.test.js` | Zod schema validation tất cả endpoints |
| `manager-permissions.test.js` | Kiểm tra phân quyền Manager vs Admin |
| `invoice.test.js` | Tạo và quản lý hóa đơn |
| `daily-revenue-email.test.js` | Gửi email báo cáo tự động |
| `dish-image-upload.test.js` | Upload ảnh món (file + base64) |
| `keepalive-bot.test.js` | Keepalive service ping logic |
| `socket-session.test.js` | Socket.IO join session, auth |
| `socket-notification.test.js` | Emit và nhận socket events |
| `checkout-bank.test.js` | Checkout và cấu hình ngân hàng |
| `admin-report-sync.test.js` | Đồng bộ báo cáo admin |
| `admin-email-send.test.js` | Gửi email thủ công từ admin |
| `error-response.test.js` | Format response lỗi |
| `secret-hygiene.test.js` | Không lộ secret trong response/log |
| `staff-table-contract.test.js` | Contract API bàn cho staff |
| `migration-history.test.js` | Kiểm tra migration database |
| `app-static.test.js` | Phục vụ file tĩnh ảnh món |

---

## 10. Triển Khai (Deployment)

### 10.1. Docker Compose (Môi Trường Local)

```bash
cd src/BE_THLTW
docker compose up -d --build
```

Docker Compose tự động khởi động 5 service theo thứ tự phụ thuộc:

```
postgres (healthy)
  → redis (healthy)
    → seeder (hoàn thành)
      → indexer (hoàn thành)
        → backend (port 5001:5000)
```

| Service | Image | Port |
|---|---|---|
| `postgres` | postgres:16-alpine | 5433:5432 |
| `redis` | redis:7-alpine | 6379:6379 |
| `seeder` | (build từ Dockerfile) | — |
| `indexer` | (build từ Dockerfile) | — |
| `backend` | (build từ Dockerfile) | 5001:5000 |

### 10.2. Triển Khai Cloud (Render + Vercel)

- **Backend + PostgreSQL + Redis:** Triển khai trên **Render.com**
- **Frontend (React + Vite):** Triển khai trên **Vercel**

**Lưu ý khi deploy trên Render:**
- Health check: `GET /api/health` → 200 OK — Render dùng để biết app đã sẵn sàng
- Cold start: Cấu hình `idleTimeoutMillis` hợp lý để tránh drop connection sau 15 phút idle
- Stateless: Không lưu state trong memory — mọi dữ liệu đều từ PostgreSQL/Redis
- Timezone: Cron job dùng `Asia/Ho_Chi_Minh` — đảm bảo DB và app cùng timezone

### 10.3. Biến Môi Trường

| Biến | Mô tả |
|---|---|
| `DATABASE_URL` | Connection string PostgreSQL |
| `REDIS_URL` | Connection string Redis (optional) |
| `JWT_ACCESS_SECRET` | Secret ký access token (≥32 ký tự) |
| `JWT_REFRESH_SECRET` | Secret ký refresh token (≥32 ký tự) |
| `VNPAY_TMNCODE` | Mã merchant VNPay |
| `VNPAY_HASHSECRET` | Secret hash VNPay |
| `FRONTEND_URL` | Whitelist CORS (nhiều URL cách nhau bằng dấu phẩy) |
| `REPORT_EMAIL_RECIPIENTS` | Danh sách email nhận báo cáo |
| `SMTP_HOST` / `SMTP_PORT` | Cấu hình SMTP server |
| `MAILTRAP_API_TOKEN` | Mailtrap API cho production |

---

## 11. Kết Luận

Backend của hệ thống KTHP-LTW được thiết kế và triển khai với các tiêu chí rõ ràng:

- **Tính nhất quán dữ liệu:** Mọi thao tác nghiệp vụ quan trọng đều thực hiện trong PostgreSQL transaction với cơ chế optimistic locking, đảm bảo không xảy ra race condition khi nhiều khách đặt món cùng lúc.
- **Bảo mật đa lớp:** JWT với token rotation, bcrypt, Helmet, CORS whitelist, rate limiting, Zod validation, webhook idempotency — mỗi lớp giải quyết một nhóm rủi ro khác nhau.
- **Real-time Experience:** Socket.IO với 3 namespace riêng biệt tạo trải nghiệm cập nhật tức thì cho toàn bộ vai trò người dùng — từ khách hàng đến bếp và thu ngân.
- **Khả năng mở rộng:** Cấu trúc layered architecture, Docker Compose, 30+ indexes, cron jobs timezone-aware — sẵn sàng cho môi trường production.
- **Tự động hoá vận hành:** Báo cáo doanh thu email hàng ngày, keepalive bot, daily quota reset — giảm thiểu can thiệp thủ công của quản trị viên.
- **Chất lượng code:** 22 test suites với coverage rộng từ auth, order, KDS, payment đến permissions và socket events.

---

*Tài liệu được tổng hợp từ source code tại `src/BE_THLTW/` — Nhóm 12, RIPT1307-02-2026*
