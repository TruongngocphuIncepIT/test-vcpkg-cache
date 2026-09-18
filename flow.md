# Chiến lược Tối ưu Cache trên GitHub Actions (vcpkg + CMake)

Tài liệu này định nghĩa logic và quy trình quản lý cache tối ưu cho CI/CD nhằm giảm thời gian build, tránh phụ thuộc vào tên nhánh/tag và đảm bảo tính độc lập của mã nguồn.

**Nhánh cache chính:** `development` (đã được đặt làm **Default branch** của repository).

---

## 1. Vai trò của từng loại Cache

| Loại Cache  | Nơi tạo                                      | Phạm vi sử dụng                                   | Mục đích                                                                 |
| :---------- | :------------------------------------------- | :------------------------------------------------ | :----------------------------------------------------------------------- |
| **Cache 1** | Nhánh `development`                          | Toàn bộ repository (mọi nhánh + tag đều đọc được) | Cache nền tảng, ổn định (Base Cache)                                     |
| **Cache 2** | Nhánh feature (ví dụ: `login`, `login-1`…)   | Chỉ trong phạm vi nhánh tạo ra nó                 | Cache riêng của nhánh, dùng khi có sự thay đổi (thiếu) so với `development` |

---

## 2. Quy tắc tạo và sử dụng Cache

### Lần đầu trên nhánh `development`
* Build trên nhánh `development` → tạo ra **Cache 1**.
* Đây đóng vai trò là cache chính (Base Cache) cho toàn bộ dự án.

### Khi build trên nhánh feature (ví dụ: `login`, `login-1`, `login-1-a`, `message-3`…)
* **Nguyên tắc định danh:** Chỉ phụ thuộc vào cấu trúc của `key` (mã băm của các file cấu hình `vcpkg.json` + `CMakeLists.txt` + `CMake/**`). Hoàn toàn **không** phụ thuộc vào `tag name` hay `branch name`.
* **Thứ tự kiểm tra:**
  1. Tìm kiếm `exact key` ngay trên **nhánh feature hiện tại** trước.
  2. Nếu không tìm thấy → tự động tìm kiếm `exact key` hoặc sử dụng `restore-keys` quay về nhánh `development`.

* **Kết quả xử lý dữ liệu:**
  * **`development` có đủ (Exact Hit):** Tải **Cache 1** về sử dụng. Quá trình kết thúc build sẽ **không** tạo thêm cache mới nhằm tiết kiệm dung lượng.
  * **Thiếu hoặc không trùng khóa (Cache Miss / Partial Hit):** Hệ thống tải bản cache gần nhất từ `development` về làm nền tảng → tự động biên dịch/cài đặt thêm phần thư viện còn thiếu → Đóng gói và tạo thành **Cache 2** (chỉ thuộc quyền sở hữu của nhánh feature đang chạy).

### Các lần build sau trên cùng một nhánh feature
* Hệ thống nhận diện và đọc trực tiếp **Cache 2** từ chính nhánh đó.
* Nếu trong quá trình phát triển tiếp tục phát sinh thư viện mới → cài thêm phần thiếu và cập nhật lại **Cache 2**.

---

## 3. Bảng tóm tắt theo từng cấp nhánh

| Khi build ở nhánh     | Có thể lấy cache từ | Có thể tạo cache mới cho          |
| --------------------- | ------------------- | --------------------------------- |
| `development`         | —                   | Cache 1 (cache chính)             |
| `login`               | Cache 1             | Cache riêng của `login`           |
| `login-1`             | Cache 1             | Cache riêng của `login-1`         |
| `login-1-a`           | Cache 1             | Cache riêng của `login-1-a`       |
| `message-3`           | Cache 1             | Cache riêng của `message-3`       |
| Bất kỳ nhánh feature nào khác | Cache 1     | Cache riêng của chính nhánh đó    |

> **Lưu ý quan trọng:** GitHub Actions **không** tạo chuỗi cache theo cây nhánh. Mọi nhánh feature (dù sâu bao nhiêu cấp) đều chỉ có thể lấy trực tiếp từ Cache 1 của `development`, không lấy được từ nhánh cha trung gian.

---

## 4. Nguyên tắc quan trọng về mã nguồn (Source Code)

* Khi push hoặc trigger CI trên nhánh feature, mã nguồn **hoàn toàn** được lấy từ nhánh feature đó.
* Quá trình xử lý cache **không tác động, không merge và không lấy code** từ nhánh `development`. Thứ duy nhất được kế thừa từ `development` là tệp tin nén lưu trữ cache trên Cloud của GitHub.
* Lập trình viên có thể thoải mái thử nghiệm độc lập mà không cần mỗi lần kiểm thử đều phải đưa code lên nhánh `development`.

---

## 5. Cách Trigger Workflow

* **Nhánh `development`:** Chạy tối thiểu 1 lần (hoặc tự động chạy mỗi khi có Pull Request được merge vào) để cập nhật và làm mới **Cache 1**.
* **Nhánh Feature:** Sử dụng cơ chế gắn nhãn tag định dạng riêng (ví dụ: `*github-action*`) hoặc sử dụng tính năng bấm thủ công `workflow_dispatch` để kích hoạt CI mà không cần tạo Pull Request về `development`.

---

## 6. Tóm tắt luồng hoạt động (Workflow Flowchart)

```text
[Nhánh development] Build lần đầu / Cập nhật code
└── Tạo Cache 1 (Cache chính trên hệ thống)

[Nhánh feature bất kỳ] Kích hoạt Build
├── Kiểm tra key trên chính nhánh feature
│   └── CÓ ───────────────────────► Sử dụng Cache 2 (cache riêng của nhánh)
└── KHÔNG CÓ
    └── Kiểm tra key trên nhánh development
        ├── Trùng khớp hoàn toàn ──► Tải Cache 1 (Không tạo cache mới)
        └── Có thay đổi / Thiếu ───► Tải Base Cache từ development
                                    └──► Cài thêm phần thiếu
                                    └──► Đóng gói thành Cache 2 (Gắn với nhánh feature)

[Các lần build sau trên cùng nhánh feature]
└── Hệ thống tự động ưu tiên nhận diện Cache 2 đầu tiên