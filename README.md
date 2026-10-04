# Bài 5: Thiết lập thư mục Web an toàn tại phân vùng hệ thống (/opt/)

## 1. Mục tiêu
- Cô lập mã nguồn website và tệp tin log ra khỏi phân vùng mặc định của hệ thống (`/var/www/`) và lưu tại phân vùng tuỳ chọn (`/opt/`).
- Thực hiện phân quyền tệp tin và thư mục nâng cao đảm bảo phân tách vai trò rõ ràng giữa tài khoản quản trị (`devops`) và tiến trình chạy của Nginx (`www-data`).
- Nginx ghi nhận logs trực tiếp vào thư mục chỉ định mới.

## 2. File cấu hình đã đính kèm
- `my-app.conf`: Server Block Nginx, cấu hình chuyển thư mục Root và Log vào `/opt/my-app/`.

---

## 3. Các thao tác thực hiện trên Server ảo

### Bước 1: Khởi tạo các thư mục tại /opt/
Sử dụng quyền sudo để tạo 2 thư mục chứa mã nguồn (`html`) và log (`logs`):
```bash
sudo mkdir -p /opt/my-app/html
sudo mkdir -p /opt/my-app/logs
```
Tạo sẵn 1 file tĩnh để test:
```bash
sudo bash -c 'echo "<h1>Chay Web tu phan vung /opt/</h1>" > /opt/my-app/html/index.html'
```

### Bước 2: Thiết lập Ownership và Permissions an toàn
Cấp quyền sở hữu thư mục mẹ và các mục con cho `devops` (Owner) và `www-data` (Group):
```bash
sudo chown -R devops:www-data /opt/my-app/
```

Thiết lập quyền chung: Owner (7 hoặc 6) để thao tác source, Group (5 hoặc 4) để Nginx đọc, Others (0) chặn truy cập lạ.
```bash
# Thư mục cấp 750 (devops toàn quyền, www-data đọc và thực thi, người khác bị cấm)
sudo find /opt/my-app/ -type d -exec chmod 750 {} \;

# Tập tin cấp 640 (devops đọc/ghi, www-data chỉ đọc)
sudo find /opt/my-app/ -type f -exec chmod 640 {} \;
```

Cấp quyền nâng cao cho Nginx ghi Log:
Vì Nginx (chạy qua group `www-data`) cần quyền tạo mới file access.log/error.log nên phải thêm quyền Write cho group trên thư mục `/logs/`:
```bash
sudo chmod 770 /opt/my-app/logs/
```

### Bước 3: Cấu hình Server Block
Sửa file Nginx tương tự như file `my-app.conf`:
```bash
sudo nano /etc/nginx/sites-available/my-app.conf
```
*(Bổ sung các chỉ thị `root /opt/my-app/html;` và `access_log /opt/my-app/logs/access.log;`)*.

Kích hoạt web và Restart lại:
```bash
sudo ln -s /etc/nginx/sites-available/my-app.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## 4. Kiểm tra (Testing)

**Test 1: Phân quyền đã chính xác chưa?**
```bash
ls -la /opt/my-app/
```
- **Kết quả mong đợi:** Cột user/group sẽ là `devops www-data`. Thư mục `html` có quyền `drwxr-x---`, và thư mục `logs` có quyền `drwxrwx---`.

**Test 2: Ghi file (Write) bằng tài khoản thường**
Với tư cách là user `devops`, chạy lệnh sau (không thêm `sudo`):
```bash
echo "Update Test 123" >> /opt/my-app/html/index.html
```
- **Kết quả mong đợi:** Lệnh chạy thành công, file lưu bình thường do `devops` có quyền ghi trên thư mục thuộc quyền sở hữu của mình.

**Test 3: Nginx ghi log**
Gọi một CURL trực tiếp tới web để Nginx xử lý request:
```bash
curl http://<IP_ADDRESS_DROPLET>/
```
Xác minh bằng cách đọc Log mới:
```bash
tail -f /opt/my-app/logs/access.log
```
- **Kết quả mong đợi:** Màn hình Terminal sẽ xuất ra bản ghi Log truy cập của CURL, chứng minh rằng Nginx (từ user www-data) có quyền ghi tập tin bình thường.
