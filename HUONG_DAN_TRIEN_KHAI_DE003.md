# HƯỚNG DẪN TRIỂN KHAI TOÀN DIỆN TỪ ĐẦU ĐẾN CUỐI (STEP-BY-STEP)
## BÀI THI CUỐI KỲ DEVOPS FUNDAMENTALS – ĐỀ 003: QUẢN LÝ CÔNG VIỆC

> **Sinh viên**: Nguyễn Đức Anh | **Mã SV**: B24DTCN184 | **Lớp**: K24CNTT2 (PTIT-HN-105)  
> **Cổng cá nhân (APP_PORT)**: `8082` | **Compose Project**: `ducanh-de003`  
> **Repo GitHub**: [https://github.com/Ducanhhhhh26/CNTT2_KS24_NguyenDucAnh_003.git](https://github.com/Ducanhhhhh26/CNTT2_KS24_NguyenDucAnh_003.git)  

---

> [!CAUTION]
> ### NGUYÊN NHÂN LỖI `invalid hostPort: ______` BẠN VỪA GẶP
> Khi bạn chạy lệnh `docker compose --env-file student.env up -d --build` trên Server bị báo lỗi `invalid hostPort: ______` là do:
> 1. Mã nguồn trên Server là bản clone từ đầu, các file `docker-compose.yml`, `Dockerfile` vẫn là **file khung có dấu `______`**.
> 2. Các file đã sửa hoàn chỉnh mới chỉ nằm trên máy tính cá nhân của bạn, **CHƯA ĐƯỢC PUSH LÊN GITHUB**.
> 👉 **Do đó, bắt buộc phải làm tuần tự theo đúng các bước từ Bước 1 đến Bước 5 dưới đây!**

---

## 🧭 LỘ TRÌNH THỰC HIỆN TỪNG BƯỚC (QUY TRÌNH CHUẨN)

```mermaid
flowchart TD
    A["BƯỚC 1: Hoàn thiện code trên máy tính<br>(student.env, Dockerfile, compose, ci-cd.yml)"] --> B["BƯỚC 2: Commit & Push lên GitHub nhánh main"]
    B --> C["BƯỚC 3: Tạo SSH Key trên Server & Cấu hình GitHub Secrets"]
    C --> D["BƯỚC 4: Chạy thử trên Server (Pull code & Docker compose)"]
    D --> E["BƯỚC 5: Pipeline tự động chạy & Chụp 4 ảnh nộp bài"]
```

---

## 🚀 BƯỚC 1: KIỂM TRA & HOÀN THIỆN CÁC FILE CODE TRÊN MÁY TÍNH

*(Các file này đã được cập nhật chính xác 100% trong thư mục dự án của bạn)*

### 1.1. File `student.env` (Phần 1 - 10 điểm)
Đảm bảo file `student.env` có nội dung sau:
```dotenv
# ---------- 1. Thông tin sinh viên ----------
STUDENT_NAME="Nguyễn Đức Anh"
STUDENT_ID="B24DTCN184"
STUDENT_CLASS="K24CNTT2"
STUDENT_EMAIL="ducanhcualo2403@gmail.com"
GITHUB_USERNAME="Ducanhhhhh26"

# ---------- 2. Thông số triển khai trên server chung ----------
EXAM_CODE="003"
APP_PORT="8082"
COMPOSE_PROJECT_NAME="ducanh-de003"

# ---------- 3. Cơ sở dữ liệu MySQL ----------
MYSQL_DATABASE="ptit_task_db"
MYSQL_USER="app_user"
MYSQL_PASSWORD="AppPass@2026"
MYSQL_ROOT_PASSWORD="RootPass@2026"
```

---

### 1.2. File `Dockerfile` (Phần 2 - 25 điểm - Theo Ảnh `img.png`)
Multi-stage build cho Spring Boot 4:
```dockerfile
# syntax=docker/dockerfile:1
FROM gradle:8.14-jdk21-alpine AS build
WORKDIR /workspace

# Copy file cấu hình build trước để tải dependency tận dụng cache layer
COPY build.gradle settings.gradle ./
RUN gradle dependencies --no-daemon -q

# Copy mã nguồn và đóng gói JAR
COPY src ./src
RUN gradle bootJar -x test --no-daemon -q

# Stage 2: Runtime JRE 21
FROM eclipse-temurin:21-jre-alpine AS runtime

ARG APP_VERSION=dev
ENV APP_VERSION=${APP_VERSION} \
    JAVA_TOOL_OPTIONS="-XX:MaxRAMPercentage=75"

WORKDIR /app

# User thường spring (non-root)
RUN addgroup -S spring && adduser -S spring -G spring
COPY --from=build --chown=spring:spring /workspace/build/libs/app.jar /app/app.jar
USER spring

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

---

### 1.3. File `docker-compose.yml` (Phần 3 - 20 điểm - Theo Ảnh `img_1.png`)
Điền đầy đủ 5 TODO, không còn bất kỳ dấu `______` nào:
```yaml
name: ${COMPOSE_PROJECT_NAME:?Chua dien COMPOSE_PROJECT_NAME trong student.env}

services:
  mysql:
    image: mysql:8.0
    restart: unless-stopped
    command: ["--character-set-server=utf8mb4", "--collation-server=utf8mb4_unicode_ci"]
    environment:
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      TZ: Asia/Ho_Chi_Minh
    volumes:
      - mysql_data:/var/lib/mysql
      - ./database/init.sql:/docker-entrypoint-initdb.d/01-init.sql:ro
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h 127.0.0.1 -u$$MYSQL_USER -p$$MYSQL_PASSWORD --silent"]
      interval: 10s
      timeout: 5s
      retries: 12
      start_period: 30s
    networks:
      - db-net

  app:
    # TODO 1: build từ context . và arg APP_VERSION
    build:
      context: .
      args:
        APP_VERSION: ${APP_VERSION:-local}
    # TODO 2: image <APP_IMAGE>:<APP_TAG>
    image: ${APP_IMAGE:-devops-final-app}:${APP_TAG:-latest}
    restart: unless-stopped
    environment:
      DB_HOST: mysql
      DB_PORT: "3306"
      # TODO 3: lấy từ student.env
      DB_NAME: ${MYSQL_DATABASE}
      DB_USER: ${MYSQL_USER}
      DB_PASSWORD: ${MYSQL_PASSWORD}
      TZ: Asia/Ho_Chi_Minh
    depends_on:
      mysql:
        condition: service_healthy
    networks:
      - db-net
      - proxy-net

  nginx:
    image: nginx:1.27-alpine
    restart: unless-stopped
    # TODO 4: public cổng APP_PORT ra cổng 80 của container
    ports:
      - "${APP_PORT:-8082}:80"
    # TODO 5: mount read-only nginx/default.conf
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - app
    networks:
      - proxy-net

networks:
  proxy-net:
  db-net:

volumes:
  mysql_data:
```

---

### 1.4. File `.github/workflows/ci-cd.yml` (Phần 4 - 35 điểm - Theo Ảnh `img_2.png`)
Đã điền hoàn chỉnh TODO 1 -> 6 (không còn dấu `______`):
- `TODO 1`: `run: docker compose --env-file student.env up -d --build --wait --wait-timeout 300`
- `TODO 2`: `username: ${{ github.actor }}`, `password: ${{ secrets.GITHUB_TOKEN }}`
- `TODO 3`: Tag và push cả 2 tag (`${{ github.sha }}` và `latest`)
- `TODO 4`: `needs: build`
- `TODO 5`: `host: ${{ secrets.SERVER_HOST }}`, `username: ${{ secrets.SERVER_USER }}`, `key: ${{ secrets.SERVER_SSH_KEY }}`
- `TODO 6`: `docker compose --env-file student.env pull app` và `docker compose --env-file student.env up -d --no-build --wait --wait-timeout 300`

---

## 📤 BƯỚC 2: COMMIT & PUSH CODE LÊN GITHUB (NHÁNH `main`)

Mở terminal tại thư mục dự án trên **MÁY TÍNH CỦA BẠN** và chạy các lệnh sau:

```bash
# 1. Đảm bảo nhánh là main (đề bài yêu cầu đưa code lên nhánh main)
git branch -M main

# 2. Thêm tất cả thay đổi
git add .

# 3. Commit code
git commit -m "feat: complete de003 full source code and workflows"

# 4. Đẩy lên repository GitHub
git push -u origin main
```

> [!TIP]
> Sau bước này, toàn bộ code chuẩn (không còn `______`) đã nằm an toàn trên GitHub!

---

## 🔑 BƯỚC 3: THIẾT LẬP SSH KEY TRÊN SERVER VÀ GITHUB SECRETS

### 3.1. Tạo SSH Key trên Server (Ảnh `img_2.png`)
Đăng nhập vào Server (`danh@Server`):
```bash
# 1. Tạo cặp khóa SSH (không đặt passphrase)
ssh-keygen -t ed25519 -f deploy_key -N ""

# 2. Thêm public key vào danh sách authorized_keys
cat deploy_key.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh

# 3. In Private Key ra màn hình để copy
cat deploy_key
```
👉 Hãy bôi đen và **Copy toàn bộ nội dung của private key** (bắt đầu từ `-----BEGIN OPENSSH PRIVATE KEY-----` đến `-----END OPENSSH PRIVATE KEY-----`).

---

### 3.2. Cấu hình GitHub Secrets
Mở trình duyệt vào repo: `https://github.com/Ducanhhhhh26/CNTT2_KS24_NguyenDucAnh_003`
1. Vào **Settings** -> **Secrets and variables** -> **Actions**.
2. Nhấn **New repository secret** và thêm 3 secrets:
   - **`SERVER_HOST`**: Địa chỉ IP của máy chủ Server.
   - **`SERVER_USER`**: `danh` (tài khoản SSH trên Server của bạn).
   - **`SERVER_SSH_KEY`**: Dán toàn bộ nội dung private key `deploy_key` vừa copy ở trên.

---

### 3.3. Bật quyền Workflow Permissions (Bắt buộc để push image lên GHCR)
1. Tại repo GitHub, vào **Settings** -> **Actions** -> **General**.
2. Kéo xuống mục **Workflow permissions**:
   - Chọn **Read and write permissions**.
   - Bấm **Save**.

---

## 💻 BƯỚC 4: KÉO CODE VỀ SERVER VÀ CHẠY THỬ NGHIỆM

Bây giờ bạn quay lại cửa sổ terminal SSH trên Server:
```bash
# 1. Chuyển vào thư mục dự án trên server
cd ~/CNTT2_KS24_NguyenDucAnh_003

# 2. Kéo code mới nhất từ nhánh main vừa push ở Bước 2
git fetch origin main
git checkout main
git reset --hard origin/main

# 3. Khởi động hệ thống (Lúc này chắc chắn KHÔNG còn bị lỗi invalid hostPort nữa!)
docker compose --env-file student.env up -d --build
```

### Kiểm tra kết quả trên Server:
```bash
# 1. Xem trạng thái các container
docker compose --env-file student.env ps
```
*(Kết quả hiển thị 3 service: `mysql` trạng thái healthy, `app` và `nginx` trạng thái Up)*.  
📸 **Chụp màn hình này ngay** để lưu làm ảnh minh chứng số 3 (`screenshots/03-compose-ps.png`).

```bash
# 2. Kiểm tra API trả về dữ liệu danh sách công việc
curl http://localhost:8082/api/tasks
```
*(Trả về chuỗi JSON chứa danh sách 8 công việc)*.

---

## 🌐 BƯỚC 5: KIỂM TRA TRÌNH DUYỆT & LẤY ĐỦ 4 ẢNH MINH CHỨNG (PHẦN 5 - 10 ĐIỂM)

### Danh sách 4 ảnh cần chụp:

| Tên file ảnh | Vị trí chụp | Nội dung cần có trên ảnh |
| :--- | :--- | :--- |
| **`01-actions.png`** | Tab **Actions** trên GitHub repo | Hiển thị workflow CI/CD chạy thành công cả 2 job `build` và `deploy` (đều có tích xanh ✅). |
| **`02-ghcr.png`** | Mục **Packages** trên GitHub | Hiển thị package image có 2 tag: `latest` và commit SHA (`${{ github.sha }}`). |
| **`03-compose-ps.png`** | Terminal SSH Server | Màn hình kết quả lệnh `docker compose --env-file student.env ps` (cả 3 container đều Up). |
| **`04-website.png`** | Trình duyệt web máy tính | Truy cập `http://<IP_SERVER>:8082/api/tasks` (hoặc `http://<IP_SERVER>:8082/`) hiển thị JSON các công việc. |

---

## 📦 HOÀN TẤT & ĐÓNG GÓI BÀI NỘP

1. Lưu 4 file ảnh trên vào thư mục: `screenshots/`
2. Copy 4 ảnh đó sang thư mục: `submission-content/`
3. Commit và push ảnh lên GitHub:
   ```bash
   git add screenshots/ submission-content/
   git commit -m "docs: add exam proof screenshots"
   git push origin main
   ```
4. Nộp link GitHub hoặc nén thư mục dự án theo hướng dẫn của giám thị thi.
