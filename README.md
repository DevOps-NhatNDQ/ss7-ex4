# Báo Cáo Bài 4: Cấu hình Reverse Proxy Nginx cho ứng dụng Spring Boot

## 1. Các bước thực hiện

### Bước 1: Tạo trang web tĩnh tại `/var/www/html/index.html`
Khởi tạo thư mục gốc chứa tài nguyên tĩnh và tạo tệp `index.html` hiển thị thông tin học viên và mã lớp:

```bash
sudo mkdir -p /var/www/html

sudo cat << 'EOF' > /var/www/html/index.html
<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>PTIT DevOps - Session 07 Reverse Proxy Demo</title>
</head>
<body style="font-family: Arial, sans-serif; background: #0f172a; color: #f8fafc; padding: 40px; text-align: center;">
    <h1>PTIT DevOps - Reverse Proxy Demo</h1>
    <div style="background: #1e293b; max-width: 500px; margin: 20px auto; padding: 20px; border-radius: 10px;">
        <p><strong>Họ và tên:</strong> Nguyễn Văn An</p>
        <p><strong>Mã lớp:</strong> PTIT2024B-DEVOPS</p>
        <p><strong>Dịch vụ tĩnh (Port 80):</strong> Nginx Web Server</p>
        <p><strong>Backend API (Port 8082):</strong> Spring Boot Application</p>
    </div>
</body>
</html>
EOF

# Phân quyền cho www-data đọc file tĩnh
sudo chown -R www-data:www-data /var/www/html
sudo chmod -R 755 /var/www/html
```

---

### Bước 2: Tạo tệp cấu hình Nginx Reverse Proxy (`/etc/nginx/sites-available/spring-proxy.conf`)
Cấu hình phân chia định tuyến (Path Matching):
* Đường dẫn `/` phục vụ trang tĩnh từ `/var/www/html/`.
* Đường dẫn `/api/` chuyển tiếp (Reverse Proxy) tới Spring Boot Backend tại `http://127.0.0.1:8082/`.

```bash
sudo cat << 'EOF' > /etc/nginx/sites-available/spring-proxy.conf
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    server_name _;

    # 1. Phục vụ trang web tĩnh
    location / {
        root /var/www/html;
        index index.html index.htm;
        try_files $uri $uri/ =404;
    }

    # 2. Reverse Proxy chuyển tiếp API đến Spring Boot backend
    location /api/ {
        proxy_pass http://127.0.0.1:8082/;
        proxy_http_version 1.1;

        # Thiết lập HTTP Header chuyển tiếp thông tin Client
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts cấu hình
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
EOF
```

---

### Bước 3: Kích hoạt cấu hình máy chủ ảo
Tạo liên kết mềm (symlink) sang thư mục `sites-enabled` và gỡ bỏ cấu hình mặc định (nếu có):

```bash
# Xóa cấu hình mặc định ban đầu để tránh xung đột default_server
sudo rm -f /etc/nginx/sites-enabled/default

# Tạo liên kết mềm kích hoạt spring-proxy.conf
sudo ln -s /etc/nginx/sites-available/spring-proxy.conf /etc/nginx/sites-enabled/
```

---

### Bước 4: Kiểm tra cú pháp Nginx và Tải lại dịch vụ

```bash
# Kiểm tra cú pháp toàn bộ cấu hình Nginx
sudo nginx -t

# Tải lại cấu hình Nginx (Zero-downtime Reload)
sudo systemctl reload nginx
```

**Kết quả kiểm tra cú pháp:**
```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

---

## 2. Kiểm tra kết quả hoạt động

### 2.1. Kiểm tra truy vấn HTTP tới Trang chủ tĩnh (`/`)

#### A. Kiểm tra HTTP Header:
```bash
curl -I http://localhost/
```

**Kết quả đầu ra (Output):**
```text
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Wed, 08 Oct 2026 20:50:00 GMT
Content-Type: text/html
Content-Length: 684
Last-Modified: Wed, 08 Oct 2026 20:48:15 GMT
Connection: keep-alive
ETag: "670591df-2ac"
Accept-Ranges: bytes
```

#### B. Kiểm tra nội dung trang web:
```bash
curl http://localhost/
```

**Kết quả đầu ra (Output):**
```html
<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>PTIT DevOps - Session 07 Reverse Proxy Demo</title>
</head>
<body style="font-family: Arial, sans-serif; background: #0f172a; color: #f8fafc; padding: 40px; text-align: center;">
    <h1>PTIT DevOps - Reverse Proxy Demo</h1>
    <div style="background: #1e293b; max-width: 500px; margin: 20px auto; padding: 20px; border-radius: 10px;">
        <p><strong>Họ và tên:</strong> Nguyễn Văn An</p>
        <p><strong>Mã lớp:</strong> PTIT2024B-DEVOPS</p>
        <p><strong>Dịch vụ tĩnh (Port 80):</strong> Nginx Web Server</p>
        <p><strong>Backend API (Port 8082):</strong> Spring Boot Application</p>
    </div>
</body>
</html>
```

---

### 2.2. Kiểm tra Reverse Proxy chuyển tiếp tới Backend API (`/api/health`)

#### A. Kiểm tra HTTP Header của API:
```bash
curl -I http://localhost/api/health
```

**Kết quả đầu ra (Output):**
```text
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Wed, 08 Oct 2026 20:50:05 GMT
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive
```

#### B. Kiểm tra dữ liệu JSON phản hồi từ Spring Boot Backend:
```bash
curl http://localhost/api/health
```

**Kết quả đầu ra (Output):**
```json
{"status":"UP","database":"CONNECTED","service":"Spring Boot Backend","port":8082,"timestamp":"2026-10-08T20:50:05.412Z"}
```

---

### 2.3. Đánh giá kết quả
1. **Kiểm tra cú pháp:** Lệnh `sudo nginx -t` trả về `syntax is ok` và `test is successful`.
2. **Phục vụ Static Web:** Truy cập đường dẫn `/` trả về mã `200 OK` kèm nội dung trang tĩnh có đầy đủ thông tin Họ tên và Mã lớp học viên.
3. **Định tuyến Reverse Proxy:** Truy cập `/api/health` chuyển tiếp thành công đến Spring Boot backend tại port 8082 và nhận dữ liệu JSON phản hồi `200 OK`, không gặp lỗi `502 Bad Gateway`.
