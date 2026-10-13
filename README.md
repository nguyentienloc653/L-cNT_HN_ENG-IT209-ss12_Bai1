# Bài 2: Tự động hóa đóng gói và phát hành Docker Image khi gắn Git Tag (v*.*.*)

## 1. Giới thiệu & Mục tiêu
- Cấu hình luồng Continuous Deployment (CD) tự động đóng gói và phát hành Docker Image chính thức lên **GitHub Container Registry (GHCR)** khi có sự kiện đẩy Git Tag chuẩn phiên bản (`v*.*.*`).
- Tự động trích xuất metadata và gắn các Docker Tag tương ứng với Git Tag nhờ `docker/metadata-action@v5`.

---

## 2. Cấu trúc thư mục
```text
├── .github/
│   └── workflows/
│       └── release.yml          # Workflow chính thức của GitHub Actions
├── homework/
│   └── session_12/
│       └── ex2/
│           ├── release.yml      # Bản nộp bài theo cấu trúc yêu cầu
│           └── README.md        # Hướng dẫn chi tiết và các bước thực hiện
└── Dockerfile                   # File định nghĩa Docker Image mẫu
```

---

## 3. Nội dung Workflow (`release.yml`)

```yaml
name: Release Docker Image

on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  release-image:
    name: Release Image
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v5

      - name: Log in to GitHub Container Registry (GHCR)
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract Docker metadata (tags & labels)
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=ref,event=tag
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push Docker Image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```

---

## 4. Giải thích các thành phần quan trọng

1. **Sự kiện kích hoạt (`on.push.tags`)**:
   - `tags: ['v*.*.*']`: Chỉ chạy khi người dùng gắn và đẩy git tag bắt đầu bằng tiền tố `v` và tuân theo định dạng Semantic Versioning (ví dụ: `v1.0.0`, `v2.1.3`).
2. **Quyền hạn (`permissions`)**:
   - `packages: write`: Cho phép `GITHUB_TOKEN` có quyền đẩy (push) package/image mới lên GitHub Container Registry (`ghcr.io`).
   - `contents: read`: Cho phép đọc source code khi checkout.
3. **Đăng nhập GHCR (`docker/login-action@v3`)**:
   - Đăng nhập vào `ghcr.io` sử dụng biến mặc định `${{ github.actor }}` và `${{ secrets.GITHUB_TOKEN }}` mà không cần tạo Personal Access Token thủ công.
4. **Trích xuất tag (`docker/metadata-action@v5`)**:
   - Tự động lấy tên image dạng `ghcr.io/<owner>/<repo>`.
   - Sinh các tag phù hợp: ví dụ với Git tag `v1.0.0`, sinh ra tag `v1.0.0`, `1.0.0` và `1.0`.
5. **Build và Push (`docker/build-push-action@v5`)**:
   - Đóng gói ứng dụng từ `Dockerfile` và đẩy trực tiếp lên `ghcr.io`.

---

## 5. Hướng dẫn kiểm tra và nộp bài

### Bước 1: Cấu hình quyền Workflow trên GitHub (nếu cần)
1. Vào GitHub Repository -> **Settings** -> **Actions** -> **General**.
2. Cuộn xuống mục **Workflow permissions**:
   - Chọn **Read and write permissions**.
   - Bấm **Save**.

### Bước 2: Commit và gán Git Tag
Chạy các lệnh sau trong Terminal:
```bash
# Thêm file và commit
git add .
git commit -m "feat: setup release docker image workflow on git tag"
git push origin main

# Tạo git tag v1.0.0 và đẩy lên GitHub
git tag v1.0.0
git push origin v1.0.0
```

### Bước 3: Kiểm tra kết quả
1. **Kiểm tra Tab Actions**:
   - Truy cập tab **Actions** trên GitHub repo.
   - Bạn sẽ thấy workflow `Release Docker Image` được kích hoạt bởi tag `v1.0.0`.
2. **Kiểm tra GitHub Packages**:
   - Sau khi job chạy thành công, vào trang chủ repository -> cột bên phải mục **Packages** (hoặc tab **Packages** của tài khoản GitHub).
   - Xác nhận image `ghcr.io/<owner>/<repo>` xuất hiện với tag `v1.0.0`.
   - Chụp ảnh màn hình mục này để nộp bài cùng file workflow.
