# CHƯƠNG 4: BÁO CÁO PHÂN TÍCH CHUYÊN SÂU
**Hệ thống:** Quản lý nhà hàng 3POS (KTHP-LTW)
**Vị trí phân tích:** Frontend (`src/pages/admin/*`)

---

## MỤC LỤC
**A. Luồng Quản trị (Admin Flow)**
   I. Tổng quan luồng Admin
   II. Phân tích chi tiết các chức năng
      1. Bảng điều khiển (Dashboard)
      2. Quản lý Thực đơn (Menu)
      3. Báo cáo & Thống kê (Reports)
      4. Quản lý Sơ đồ bàn & Mã QR
      5. Quản lý Nhân viên (Users) - *Quyền đặc biệt*
      6. Cấu hình Hệ thống (Settings) - *Quyền đặc biệt*
      7. Gửi Email Báo cáo (Email Send) - *Quyền đặc biệt*
   III. Đánh giá Kiến trúc & Bảo mật trong luồng Admin

---

## A. LUỒNG QUẢN TRỊ (ADMIN FLOW)

### I. TỔNG QUAN LUỒNG ADMIN

Luồng Admin (Quản trị viên) là phân hệ phức tạp và có quyền hạn cao nhất trong hệ thống frontend của 3POS. Luồng này kế thừa toàn bộ quyền của vai trò Quản lý (MANAGER) cộng thêm các quyền cấu hình cốt lõi của hệ thống.

**Đặc điểm kỹ thuật chính:**
- **Bảo mật Route:** Toàn bộ các trang `/admin/*` được bảo vệ bởi `ProtectedRoute`. Trong đó, 3 trang `/admin/users`, `/admin/settings`, `/admin/email-send` được bảo vệ mức 2 (Chỉ user có role `ADMIN` mới được render, chặn `MANAGER`).
- **Giao diện:** Kế thừa `StaffLayout` với thanh sidebar điều hướng hiển thị tối đa các menu chức năng.
- **Sử dụng Ant Design kết hợp Tailwind:** Các form phức tạp (như tạo nhân viên, quản lý món ăn) tận dụng UI Component của Ant Design (`Modal`, `Form`, `Input`, `Select`) để validation chặt chẽ, kết hợp TailwindCSS để tinh chỉnh layout.

---

### II. PHÂN TÍCH CHI TIẾT CÁC CHỨC NĂNG

#### 1. Bảng điều khiển (AdminDashboardPage.tsx)
**Chức năng:** Cung cấp cái nhìn toàn cảnh về tình hình kinh doanh thông qua các chỉ số thời gian thực và biểu đồ.

**Phân tích kỹ thuật & Code quan trọng:**
- **Tính toán linh hoạt (Aggregation):** Backend có thể trả về nhiều dòng doanh thu trong cùng một ngày (do phân tách phương thức thanh toán). Frontend chủ động gộp dữ liệu trước khi nạp vào biểu đồ `Recharts` để hiển thị.

```typescript
// Gộp doanh thu theo ngày ở Client-side để vẽ biểu đồ chính xác
const aggregatedRevenue = rawRevenue.reduce((acc, current) => {
  const existing = acc.find(item => item.date === current.date)
  if (existing) {
    existing.total += Number(current.total || 0)
  } else {
    acc.push({ ...current, total: Number(current.total || 0) })
  }
  return acc
}, [])
```

- **Biểu đồ AreaChart:** Sử dụng `Recharts` tạo hiệu ứng fill gradient mượt mà (màu Emerald) biểu thị xu hướng doanh thu.
- **Top 5 món bán chạy:** Sử dụng thanh progress bar tự custom với width tính theo phần trăm món bán nhiều nhất, kèm animation width tăng dần khi tải trang.

#### 2. Quản lý Thực đơn (AdminMenuPage.tsx)
**Chức năng:** CRUD (Tạo, Đọc, Cập nhật, Xóa) danh mục và món ăn. Hỗ trợ cập nhật tồn kho (quota) mỗi ngày.

**Phân tích kỹ thuật & Code quan trọng:**
- **Hệ thống xử lý ảnh 3 phương thức:** Frontend cho phép người dùng cấu hình ảnh món ăn qua: (1) Tải file từ máy, (2) Dán URL, (3) Dán trực tiếp chuỗi Base64. Logic này được module hóa trong file riêng `adminMenuImage.js` để dễ test.

```javascript
// adminMenuImage.js - Xử lý hiển thị preview ảnh mượt mà
export const buildImagePreviewSrc = (type, value) => {
  if (type === 'file' && value instanceof File) {
    return URL.createObjectURL(value) // Tạo blob URL nhanh, không tốn RAM
  }
  if (type === 'base64' && value) {
    // Tự động fix tiền tố nếu user dán thiếu
    return value.startsWith('data:image') ? value : `data:image/jpeg;base64,${value}`
  }
  return type === 'url' ? value : null
}
```

- **Khôi phục Quota nhanh (Reset Quota):** Chức năng cho phép Admin đặt lại số lượng bán (daily quota) cho tất cả món ăn trong một click để bắt đầu ngày kinh doanh mới.

#### 3. Báo cáo & Thống kê (AdminReportsPage.tsx)
**Chức năng:** Phân tích tỷ trọng món ăn và xuất báo cáo kinh doanh ra file Excel.

**Phân tích kỹ thuật & Code quan trọng:**
- **Xử lý File Blob (Xuất Excel):** Backend trả về dữ liệu binary (Blob) của file Excel. Frontend đảm nhiệm việc trigger trình duyệt tải xuống mà không cần mở tab mới.

```typescript
const exportExcel = async () => {
  try {
    const blob = await adminApi.exportReport({ from: dateRange.from, to: dateRange.to })
    const url = URL.createObjectURL(blob)
    const a = document.createElement('a')
    a.href = url
    a.download = `3pos-report-${dateRange.from}.xlsx`
    document.body.appendChild(a)
    a.click() // Tự động trigger tải file
    URL.revokeObjectURL(url) // Giải phóng bộ nhớ
  } catch (error) {
    toast.error('Lỗi khi xuất dữ liệu')
  }
}
```

#### 4. Quản lý Sơ đồ bàn (AdminTablesPage) & Mã QR (AdminQRPage)
**Chức năng:** Tổ chức không gian nhà hàng và cấp phát mã QR đặt món.

**Phân tích kỹ thuật & Code quan trọng:**
- **Xóa bàn an toàn:** Giao diện vô hiệu hóa nút "Xóa" nếu bàn đang có khách (`status === 'OCCUPIED'`), ngăn chặn Admin lỡ tay xóa bàn đang có order chưa thanh toán.
- **Sinh QR Client-side:** Module `AdminQRPage.tsx` sử dụng thư viện `qrcode` để sinh ảnh QR hoàn toàn trên trình duyệt, không gây tải cho Backend.
- **In ấn QR (Iframe Print):** Kỹ thuật render một `iframe` ẩn chứa cấu trúc HTML của toàn bộ thẻ QR (định dạng A4), sau đó gọi lệnh in của iframe đó để in hàng loạt mà không vỡ giao diện hệ thống.

#### 5. Quản lý Nhân viên (AdminUsersPage.tsx) - *Quyền Admin Độc Quyền*
**Chức năng:** Tạo tài khoản, vô hiệu hóa và kiểm soát vai trò của nhân sự.

**Phân tích kỹ thuật & Code quan trọng:**
- **Màu sắc theo vai trò (Role Mapping):** Các badge và màu viền avatar được tính toán động (dynamic classes) để phân biệt rõ ràng cấp bậc (ADMIN màu tím, MANAGER xanh dương, CASHIER xanh lá, v.v.).
- **Bảo mật luồng Mật khẩu:** Form Ant Design hiển thị trường Mật khẩu khi tạo mới (Create), nhưng ẩn đi hoàn toàn khi chỉnh sửa (Edit). Đây là phương pháp thiết kế bảo mật, ngăn admin xem hoặc vô tình thay đổi password cũ của nhân viên qua luồng cập nhật thông tin chung.

```tsx
// Phân tách hiển thị Input Mật khẩu
{!editUser && (
  <Form.Item
    name="password"
    label="Mật khẩu đăng nhập"
    rules={[{ required: true, min: 8, message: 'Mật khẩu phải từ 8 ký tự' }]}
  >
    <Input.Password placeholder="Nhập mật khẩu" />
  </Form.Item>
)}
```
- **Soft Delete (Vô hiệu hóa):** Thao tác xóa nhân viên không làm mất dữ liệu hóa đơn cũ, mà chuyển trạng thái hiển thị `is_active = false` (Label "Vô hiệu" màu đỏ).

#### 6. Cấu hình Ngân hàng VietQR (AdminSettingsPage.tsx) - *Quyền Admin Độc Quyền*
**Chức năng:** Cấu hình tài khoản đích để nhận tiền chuyển khoản từ khách hàng qua mã VietQR.

**Phân tích kỹ thuật & Code quan trọng:**
- **Validation chặt chẽ:** Form yêu cầu định dạng chuẩn để khi nối chuỗi sinh mã VietQR không bị lỗi: Số tài khoản chỉ chứa chữ/số, Tên chủ tài khoản tự động chuyển thành in hoa (Uppercase).

```typescript
// Ràng buộc số tài khoản không chứa ký tự đặc biệt
if (!/^[A-Za-z0-9]+$/.test(form.account_number)) {
  toast.error('Số tài khoản chỉ được chứa chữ cái và số, không khoảng trắng');
  return;
}
```
- **Preview Trạng thái:** Giao diện có thẻ Card đánh giá ngay lập tức: nếu đủ 3 trường hợp lệ sẽ hiện màu Xanh (Sẵn sàng tạo QR), nếu thiếu sẽ hiện màu Xám.

#### 7. Gửi Email Báo cáo (AdminEmailSendPage.tsx) - *Quyền Admin Độc Quyền*
**Chức năng:** Kích hoạt gửi báo cáo kinh doanh qua Email, hoặc xem trạng thái Cron Job gửi tự động.

**Phân tích kỹ thuật & Code quan trọng:**
- **Tách biệt Logic & UI (Pure Functions):** Chức năng này áp dụng kiến trúc tách các hàm xử lý dữ liệu ra file `AdminEmailSendPage.logic.js`. Điều này giúp Frontend có thể test độc lập chức năng gửi mail mà không cần mount Component React.
- **Double-submit Protection:** Sử dụng trạng thái `isSubmitting` chặn bấm nút nhiều lần gây kẹt hàng đợi gửi Mail của Backend.

---

### III. ĐÁNH GIÁ KIẾN TRÚC & BẢO MẬT TRONG LUỒNG ADMIN

**1. Điểm mạnh kiến trúc:**
- **Route Guard:** Sử dụng `ProtectedRoute` kết hợp danh sách `roles` bảo vệ rất tốt. Một `MANAGER` cố tình gõ URL `/admin/users` sẽ bị `Navigate` quay ngược về `/admin/dashboard` một cách vô hình, không gây lỗi treo giao diện.
- **Tận dụng Thư viện:** Việc dùng Ant Design (`Form`, `Modal`) cho luồng Admin giúp giảm thiểu khối lượng code xử lý validation, trong khi vẫn giữ UI đẹp và chuyên nghiệp qua Tailwind.
- **Client-side xử lý nhẹ:** Sinh mã QR và gộp số liệu biểu đồ (Aggregation) được đẩy về phía Client, giúp Backend giảm đáng kể tài nguyên CPU.

**2. Điểm cần khắc phục (Rủi ro):**
- **Action-Level Permission:** Có một số API backend cần thiết lập chặt chẽ hơn. Frontend hiện tại ẩn các nút đi với những user không có quyền, nhưng nếu một user (VD: Cashier) bằng cách nào đó dùng Postman gọi API xóa nhân viên, Backend phải chặn được luồng đó.
- **Thiếu Confirm Modal nhất quán:** Một số hành động (như Force Close bàn, xóa Category) vẫn sử dụng hộp thoại `window.confirm()` mặc định của trình duyệt, trông chưa đồng bộ với giao diện Ant Design sang trọng của toàn hệ thống. Cần đồng bộ sang `Modal.confirm` của Ant Design. 
- **Upload File lớn:** Chức năng upload ảnh menu nếu Admin đẩy ảnh quá lớn (>5MB) chưa có cơ chế nén ảnh (Compress) bằng Canvas phía Client trước khi gửi, có thể gây quá tải băng thông mạng nhà hàng.
