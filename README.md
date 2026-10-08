# Bài 3 – Thiết lập Cơ sở dữ liệu và Tự cấu hình dịch vụ Systemd cho Spring Boot

## 1. Khởi tạo cơ sở dữ liệu MySQL

```sql
-- Đăng nhập MySQL với quyền root
mysql -u root -p

-- Tạo database
CREATE DATABASE springboot_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Tạo user và cấp quyền
CREATE USER 'spring-admin'@'localhost' IDENTIFIED BY 'SpringSecure@123';
GRANT ALL PRIVILEGES ON springboot_db.* TO 'spring-admin'@'localhost';
FLUSH PRIVILEGES;

-- Kiểm tra
SHOW DATABASES;
SELECT user, host FROM mysql.user WHERE user = 'spring-admin';
```

## 2. Tạo user hệ thống Linux (non-root, không có shell login)

```bash
# Tạo user spring-runner không có shell login, không có home directory
sudo useradd --system --no-create-home --shell /usr/sbin/nologin spring-runner

# Kiểm tra user đã tạo
id spring-runner
# uid=xxx(spring-runner) gid=xxx(spring-runner) groups=xxx(spring-runner)
```

## 3. Chuẩn bị thư mục ứng dụng

```bash
# Tạo thư mục chứa ứng dụng
sudo mkdir -p /opt/spring-app

# Copy file app.jar vào thư mục
sudo cp app.jar /opt/spring-app/

# Phân quyền cho user spring-runner
sudo chown -R spring-runner:spring-runner /opt/spring-app
sudo chmod 755 /opt/spring-app
sudo chmod 644 /opt/spring-app/app.jar
```

## 4. Tạo file dịch vụ Systemd

```bash
# Copy file service vào systemd
sudo cp spring-app.service /etc/systemd/system/spring-app.service

# Nạp lại daemon
sudo systemctl daemon-reload

# Bật dịch vụ khởi động cùng hệ thống
sudo systemctl enable spring-app.service

# Khởi động dịch vụ
sudo systemctl start spring-app.service
```

### Nội dung file `spring-app.service`

```ini
[Unit]
Description=Spring Boot Application (app.jar) on port 8082
Documentation=https://docs.spring.io/spring-boot/docs/current/reference/html/
After=network.target mysql.service
Wants=mysql.service

[Service]
Type=simple
User=spring-runner
Group=spring-runner

ExecStart=/usr/bin/java -jar /opt/spring-app/app.jar
WorkingDirectory=/opt/spring-app

Restart=on-failure
RestartSec=10

StandardOutput=journal
StandardError=journal

SuccessExitStatus=143

Environment=SERVER_PORT=8082
Environment=SPRING_DATASOURCE_URL=jdbc:mysql://localhost:3306/springboot_db
Environment=SPRING_DATASOURCE_USERNAME=spring-admin
Environment=SPRING_DATASOURCE_PASSWORD=SpringSecure@123

[Install]
WantedBy=multi-user.target
```

**Giải thích các tham số quan trọng:**

| Tham số | Giải thích |
|---------|-----------|
| `After=network.target mysql.service` | Đảm bảo dịch vụ chỉ khởi động sau khi mạng và MySQL đã sẵn sàng |
| `User=spring-runner` | Chạy ứng dụng dưới quyền user giới hạn, không phải root |
| `Restart=on-failure` | Tự động khởi động lại nếu tiến trình thoát với mã lỗi (crash) |
| `RestartSec=10` | Chờ 10 giây trước khi restart |
| `SuccessExitStatus=143` | Coi exit code 143 (SIGTERM) là thoát thành công |
| `WantedBy=multi-user.target` | Dịch vụ sẽ khởi động ở chế độ multi-user (runlevel 3) |

## 5. Kết quả kiểm tra

### 5.1 Trạng thái dịch vụ

```bash
sudo systemctl status spring-app.service
```

**Kết quả mong đợi:**

```
● spring-app.service - Spring Boot Application (app.jar) on port 8082
     Loaded: loaded (/etc/systemd/system/spring-app.service; enabled; vendor preset: enabled)
     Active: active (running) since ...
   Main PID: XXXX (java)
      Tasks: XX (limit: XXXX)
     Memory: XXX.XM
        CPU: Xs
     CGroup: /system.slice/spring-app.service
             └─XXXX /usr/bin/java -jar /opt/spring-app/app.jar
```

### 5.2 Kiểm tra cổng lắng nghe

```bash
ss -tlnp | grep 8082
```

**Kết quả mong đợi:**

```
LISTEN  0  100  *:8082  *:*  users:(("java",pid=XXXX,fd=XX))
```

> ✅ Dịch vụ đang chạy ở trạng thái `active (running)` và cổng 8082 được lắng nghe bởi tiến trình Java dưới quyền user `spring-runner`.
