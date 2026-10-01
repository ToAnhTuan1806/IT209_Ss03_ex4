# Bài 4: Chạy song song nhiều cổng dịch vụ (Nginx Virtual Hosts)

Báo cáo cấu hình Nginx chạy song song hai website tĩnh độc lập trên cùng một máy chủ ảo thông qua việc phân chia cổng dịch vụ: **Beta App** (cổng `8080`) và **Internal App** (cổng `8090`).

---

## 1. Các Bước Thực Hiện

### Bước 1: Tạo thư mục lưu trữ mã nguồn cho 2 ứng dụng
Tạo hai thư mục web root riêng biệt và sao chép các tệp `index.html` tương ứng:

```bash
# Tạo thư mục
sudo mkdir -p /var/www/beta-app/html
sudo mkdir -p /var/www/internal-app/html

# Đưa file index.html vào đúng thư mục
sudo cp beta-app/index.html /var/www/beta-app/html/index.html
sudo cp internal-app/index.html /var/www/internal-app/html/index.html

# Phân quyền truy cập cho web server
sudo chown -R www-data:www-data /var/www/beta-app/html /var/www/internal-app/html
sudo chmod -R 755 /var/www/beta-app/html /var/www/internal-app/html
```

---

### Bước 2: Cấu hình Server Block trong Nginx
Tạo file cấu hình `/etc/nginx/sites-available/multi-port.conf`:

```bash
sudo nano /etc/nginx/sites-available/multi-port.conf
```

Nội dung cấu hình chứa 2 khối `server`:
```nginx
# Server Block 1: Beta App (Cổng 8080)
server {
    listen 8080;
    listen [::]:8080;

    server_name _;

    root /var/www/beta-app/html;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }
}

# Server Block 2: Internal App (Cổng 8090)
server {
    listen 8090;
    listen [::]:8090;

    server_name _;

    root /var/www/internal-app/html;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

---

### Bước 3: Mở cổng trên tường lửa UFW và Cloud Firewall
Mở hai cổng `8080/tcp` và `8090/tcp` trên tường lửa máy chủ:

```bash
sudo ufw allow 8080/tcp comment 'Beta App'
sudo ufw allow 8090/tcp comment 'Internal App'
sudo ufw reload
sudo ufw status
```

*(Lưu ý: Nếu sử dụng DigitalOcean hoặc AWS, cần vào mục Cloud Firewall / Security Group để mở Inbound Rule cho 2 cổng TCP 8080 và 8090).*

**Log Terminal:**
```text
devops@ubuntu:~$ sudo ufw allow 8080/tcp comment 'Beta App'
Rule added
Rule added (v6)

devops@ubuntu:~$ sudo ufw allow 8090/tcp comment 'Internal App'
Rule added
Rule added (v6)

devops@ubuntu:~$ sudo ufw status
Status: active

To                         Action      From
--                         ------      ----
2222/tcp                   ALLOW       Anywhere                  
8080/tcp (Beta App)        ALLOW       Anywhere                  
8090/tcp (Internal App)    ALLOW       Anywhere                  
2222/tcp (v6)              ALLOW       Anywhere (v6)             
8080/tcp (Beta App (v6))   ALLOW       Anywhere (v6)             
8090/tcp (Internal App (v6)) ALLOW       Anywhere (v6)
```

---

### Bước 4: Kích hoạt cấu hình và nạp lại Nginx
```bash
sudo ln -sf /etc/nginx/sites-available/multi-port.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
sudo ss -tulpn | grep -E '8080|8090'
```

**Log Terminal:**
```text
devops@ubuntu:~$ sudo nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

devops@ubuntu:~$ sudo systemctl reload nginx

devops@ubuntu:~$ sudo ss -tulpn | grep -E '8080|8090'
tcp   LISTEN 0      128          0.0.0.0:8080      0.0.0.0:*    users:(("nginx",pid=16120,fd=7))
tcp   LISTEN 0      128          0.0.0.0:8090      0.0.0.0:*    users:(("nginx",pid=16120,fd=8))
tcp   LISTEN 0      128             [::]:8080         [::]:*    users:(("nginx",pid=16120,fd=9))
tcp   LISTEN 0      128             [::]:8090         [::]:*    users:(("nginx",pid=16120,fd=10))
```

---

## 2. Kiểm Tra Kết Quả (Từ Máy Cá Nhân)

### Kiểm tra 1: Truy cập cổng 8080 (Beta App)
```bash
curl http://<IP_ADDRESS_DROPLET>:8080
```

**Log Terminal:**
```text
user@laptop:~$ curl http://159.223.45.67:8080
<!DOCTYPE html>
<html lang="vi">
...
        <span class="badge">🚀 Testing Environment</span>
        <h1>Beta Application</h1>
        <p>Trang kiểm thử ứng dụng độc lập dành cho đội ngũ phát triển và kiểm thử sản phẩm.</p>
        <div class="port-info">Listening Port: 8080 (TCP) • Root: /var/www/beta-app/html</div>
...
</html>
```
👉 **Kết quả:** Trả về chính xác nội dung giao diện trang **Beta App**.

---

### Kiểm tra 2: Truy cập cổng 8090 (Internal App)
```bash
curl http://<IP_ADDRESS_DROPLET>:8090
```

**Log Terminal:**
```text
user@laptop:~$ curl http://159.223.45.67:8090
<!DOCTYPE html>
<html lang="vi">
...
        <span class="badge">🔒 Intranet Portal</span>
        <h1>Internal Application</h1>
        <p>Hệ thống cổng thông tin nội bộ doanh nghiệp dành riêng cho nhân viên và quản trị viên.</p>
        <div class="port-info">Listening Port: 8090 (TCP) • Root: /var/www/internal-app/html</div>
...
</html>
```
👉 **Kết quả:** Trả về chính xác nội dung giao diện trang **Internal App**.
