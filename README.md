# Bài 4: Cấu hình Reverse Proxy Nginx cho ứng dụng Spring Boot

## 1. Mục tiêu

* Cấu hình Nginx phục vụ trang web tĩnh tại port `80`.
* Cấu hình Reverse Proxy cho API với tiền tố `/api/`.
* Chuyển tiếp request từ Nginx đến ứng dụng Spring Boot tại `127.0.0.1:8082`.
* Kiểm tra cấu hình Nginx và kiểm tra hoạt động bằng `curl`.

---

## 2. Tạo trang web tĩnh

Trang web được đặt tại:

```text
/var/www/html/index.html
```

Nội dung trang web gồm thông tin sinh viên và thông tin bài thực hành.

Nginx sử dụng thư mục:

```nginx
root /var/www/html;
```

---

## 3. Cấu hình Reverse Proxy

Tạo file:

```text
/etc/nginx/sites-available/spring-proxy.conf
```

Nội dung:

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name _;

    root /var/www/html;
    index index.html;

    # Phục vụ trang web tĩnh
    location / {
        try_files $uri $uri/ =404;
    }

    # Reverse Proxy đến Spring Boot
    location /api/ {
        proxy_pass http://127.0.0.1:8082/;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

## 4. Kích hoạt cấu hình

Xóa cấu hình Nginx mặc định:

```bash
sudo rm -f /etc/nginx/sites-enabled/default
```

Tạo symbolic link:

```bash
sudo ln -sf /etc/nginx/sites-available/spring-proxy.conf /etc/nginx/sites-enabled/spring-proxy.conf
```

Kiểm tra:

```bash
ls -l /etc/nginx/sites-enabled/
```

Sau đó kiểm tra cú pháp:

```bash
sudo nginx -t
```

Kết quả:

```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Cấu hình Nginx hợp lệ.

Reload Nginx:

```bash
sudo systemctl reload nginx
```

---

## 5. Kiểm tra trang web

Kiểm tra HTTP Header:

```bash
curl -I http://localhost/
```

Kết quả:

```text
HTTP/1.1 200 OK
Server: nginx
Content-Type: text/html
```

Kiểm tra nội dung:

```bash
curl http://localhost/
```

Kết quả trả về nội dung trang HTML thông tin sinh viên.

Trang web hoạt động thành công tại:

```text
http://221.121.4.73/
```

---

## 6. Kiểm tra Spring Boot Backend

Kiểm tra port `8082`:

```bash
sudo ss -tlnp | grep :8082
```

Kết quả cho thấy ứng dụng Spring Boot đang lắng nghe tại:

```text
127.0.0.1:8082
```

Kiểm tra trực tiếp Backend:

```bash
curl http://127.0.0.1:8082/health
```

Kết quả:

```json
{"status":"UP"}
```

---

## 7. Kiểm tra Reverse Proxy

Kiểm tra thông qua Nginx:

```bash
curl -i http://localhost/api/health
```

Kết quả:

```text
HTTP/1.1 200 OK
Content-Type: application/json
```

Response:

```json
{"status":"UP"}
```

Điều này chứng minh request:

```text
/api/health
```

đã được Nginx chuyển tiếp thành công đến:

```text
http://127.0.0.1:8082/health
```

---

## 8. Kiểm tra từ trình duyệt

Trang web:

```text
http://221.121.4.73/
```

API:

```text
http://221.121.4.73/api/health
```

Kết quả:

* Trang chủ trả về `HTTP 200 OK`.
* API `/api/health` trả về response từ Spring Boot.
* Không xuất hiện lỗi `502 Bad Gateway`.

---

## 9. Sơ đồ hoạt động

```text
                    Client
                       |
                       | HTTP :80
                       v
              +----------------+
              |     Nginx      |
              |    Port 80     |
              +----------------+
                 /          \
                /            \
               v              v
        Trang web tĩnh     /api/health
        /var/www/html/          |
                                |
                                v
                       +----------------+
                       |  Spring Boot   |
                       |    Port 8082   |
                       +----------------+
                                |
                                v
                       {"status":"UP"}
```

---

## 10. Kết luận

Đã hoàn thành cấu hình Reverse Proxy Nginx:

* Nginx hoạt động trên port `80`.
* Website tĩnh được phục vụ từ `/var/www/html/`.
* Request `/api/` được chuyển tiếp đến Spring Boot tại `127.0.0.1:8082`.
* Cấu hình Nginx được kiểm tra thành công bằng `nginx -t`.
* Trang chủ trả về `HTTP 200 OK`.
* API `/api/health` trả về response thành công từ Spring Boot.
