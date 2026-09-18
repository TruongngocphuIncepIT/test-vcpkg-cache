# Test vcpkg Cache

Repo này kiểm thử chiến lược cache vcpkg trong [`.github/workflows/test.yml`](.github/workflows/test.yml).

## Cách hoạt động

- `key` chỉ phụ thuộc vào hệ điều hành và hash của `vcpkg.json`, `CMakeLists.txt`, `CMake/**`; không chứa tên branch hoặc tag.
- GitHub Actions tìm exact key trong scope của branch hiện tại trước. Cache của `development` được dùng làm fallback nhờ cache scope mặc định và `restore-keys`.
- Exact hit (`cache-hit == true`) chỉ dùng cache và không tạo cache mới.
- Miss hoặc partial hit sẽ chạy `vcpkg install`, sau đó cache action lưu key mới cho scope của branch đang chạy.
- Cache lưu binary artifacts của vcpkg tại `.vcpkg-binary-cache`, không lưu mã nguồn từ `development` và không merge branch.

## Cách kiểm thử

1. Chạy workflow một lần trên `development` để tạo cache nền.
2. Chạy lại trên `development`: log phải báo `Exact cache hit`.
3. Tạo branch feature không sửa các file tạo key: branch dùng cache nền và không tạo cache mới.
4. Sửa `vcpkg.json` hoặc `CMakeLists.txt`, chạy branch feature: workflow phải báo miss/partial hit và tạo cache riêng theo key mới.
5. Chạy lại branch feature: workflow phải báo exact hit trong scope của branch đó.
