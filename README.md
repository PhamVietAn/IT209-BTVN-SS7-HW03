# Bài 3: Thiết lập cơ sở dữ liệu và dịch vụ Systemd cho Spring Boot

## 1. Tạo cơ sở dữ liệu và tài khoản MySQL

Đăng nhập MySQL bằng tài khoản quản trị:

```bash
sudo mysql
```

Thực hiện các câu lệnh sau:

```sql
CREATE DATABASE IF NOT EXISTS springboot_db
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

CREATE USER IF NOT EXISTS 'spring-admin'@'localhost'
    IDENTIFIED BY 'SpringSecure@123';

ALTER USER 'spring-admin'@'localhost'
    IDENTIFIED BY 'SpringSecure@123';

GRANT ALL PRIVILEGES ON springboot_db.*
    TO 'spring-admin'@'localhost';

FLUSH PRIVILEGES;
SHOW GRANTS FOR 'spring-admin'@'localhost';
EXIT;
```

Kết quả kiểm tra quyền:

```text
+--------------------------------------------------------------------------+
| Grants for spring-admin@localhost                                        |
+--------------------------------------------------------------------------+
| GRANT USAGE ON *.* TO `spring-admin`@`localhost`                         |
| GRANT ALL PRIVILEGES ON `springboot_db`.* TO `spring-admin`@`localhost`  |
+--------------------------------------------------------------------------+
2 rows in set (0.00 sec)
```

Kiểm tra đăng nhập bằng tài khoản vừa tạo:

```bash
mysql -u spring-admin -p -e "SHOW DATABASES LIKE 'springboot_db';"
```

```text
Enter password:
+--------------------------+
| Database (springboot_db) |
+--------------------------+
| springboot_db            |
+--------------------------+
```

## 2. Tạo tài khoản chạy ứng dụng

Tạo thư mục ứng dụng và tài khoản hệ thống không có quyền đăng nhập shell:

```bash
sudo mkdir -p /opt/spring-app
sudo useradd --system \
  --home-dir /opt/spring-app \
  --shell /usr/sbin/nologin \
  spring-runner
```

Sao chép tệp `app.jar` vào thư mục ứng dụng và thiết lập quyền:

```bash
sudo cp app.jar /opt/spring-app/app.jar
sudo chown -R spring-runner:spring-runner /opt/spring-app
sudo chmod 750 /opt/spring-app
sudo chmod 640 /opt/spring-app/app.jar
```

Kiểm tra tài khoản:

```bash
getent passwd spring-runner
```

```text
spring-runner:x:998:998::/opt/spring-app:/usr/sbin/nologin
```

## 3. Cài đặt dịch vụ Systemd

Tệp cấu hình dịch vụ được lưu tại `ex03/spring-app.service`. Sao chép tệp vào Systemd:

```bash
sudo cp spring-app.service /etc/systemd/system/spring-app.service
sudo chmod 644 /etc/systemd/system/spring-app.service
```

Kiểm tra Java trước khi khởi động ứng dụng:

```bash
/usr/bin/java -version
```

Nạp lại cấu hình Systemd, kích hoạt và khởi động dịch vụ:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now spring-app.service
```

Nếu cần theo dõi nhật ký khởi động:

```bash
sudo journalctl -u spring-app.service -n 30 --no-pager
```

```text
Oct 08 10:18:41 devops-server systemd[1]: Started spring-app.service - Ứng dụng Spring Boot.
Oct 08 10:18:43 devops-server java[1842]: Tomcat initialized with port 8082 (http)
Oct 08 10:18:45 devops-server java[1842]: HikariPool-1 - Start completed.
Oct 08 10:18:46 devops-server java[1842]: Started Application in 4.982 seconds
```

## 4. Kiểm tra kết quả

Kiểm tra trạng thái dịch vụ:

```bash
sudo systemctl status spring-app.service --no-pager
```

```text
spring-app.service - Ứng dụng Spring Boot
     Loaded: loaded (/etc/systemd/system/spring-app.service; enabled; preset: enabled)
     Active: active (running) since Thu 2026-10-08 10:18:41 +07; 5min ago
   Main PID: 1842 (java)
      Tasks: 29 (limit: 4557)
     Memory: 186.4M
        CPU: 8.212s
     CGroup: /system.slice/spring-app.service
             └─1842 /usr/bin/java -jar /opt/spring-app/app.jar
```

Kiểm tra tiến trình Java đang chạy bằng tài khoản `spring-runner`:

```bash
ps -o user,pid,cmd -p "$(systemctl show -p MainPID --value spring-app.service)"
```

```text
USER             PID CMD
spring-runner    1842 /usr/bin/java -jar /opt/spring-app/app.jar
```

Kiểm tra cổng `8082`:

```bash
sudo ss -tlnp | grep 8082
```

```text
LISTEN 0      100                *:8082             *:*    users:(("java",pid=1842,fd=17))
```

Các kết quả trên cho thấy dịch vụ đã hoạt động, tiến trình Java chạy dưới quyền `spring-runner` và ứng dụng đang lắng nghe tại cổng `8082`.
