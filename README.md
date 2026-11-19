# Đếm ngược Tết Đinh Mùi 2027

Trang tĩnh (HTML/CSS/JS thuần, không cần build) đếm ngược tới giao thừa 0h 06/02/2027 (giờ Việt Nam),
kèm các mốc Tết, phong bao lời chúc và lì xì qua VietQR.

```
tet2027-deploy/
├── site/
│   ├── index.html          # toàn bộ trang (CSS + JS + mã QR nằm sẵn trong file)
│   └── og.jpg              # ảnh xem trước khi chia sẻ link (Zalo, Facebook, Messenger)
├── nginx.conf              # cấu hình nginx: gzip, cache, header bảo mật
├── Dockerfile
├── docker-compose.yml      # chạy ở 127.0.0.1:8027
└── cloudflared-ingress.yml # đoạn cấu hình thêm vào Cloudflare Tunnel
```

Tên miền mặc định là **tet.migor.site**. Nếu dùng tên khác, sửa 2 dòng `og:url` và `og:image` trong `site/index.html`.

---

## Cách 1 — Docker (khuyên dùng)

```bash
# 1. Đưa thư mục lên server
scp tet2027-deploy.zip user@server:~
ssh user@server
unzip tet2027-deploy.zip && cd tet2027-deploy

# 2. Chạy
docker compose up -d --build

# 3. Kiểm tra
curl -I http://127.0.0.1:8027        # phải trả về HTTP/1.1 200 OK
```

Cập nhật nội dung sau này: sửa `site/index.html` rồi chạy lại `docker compose up -d --build`.

## Cách 2 — nginx cài sẵn trên máy (không dùng Docker)

```bash
sudo mkdir -p /var/www/tet2027
sudo cp -r site/* /var/www/tet2027/

# Tạo site nginx nghe ở localhost:8027
sudo tee /etc/nginx/sites-available/tet2027 >/dev/null <<'EOF'
server {
    listen 127.0.0.1:8027;
    root /var/www/tet2027;
    index index.html;
    charset utf-8;
    gzip on;
    gzip_types text/css application/javascript image/svg+xml;
    location = /index.html { expires -1; }
    location ~* \.(jpg|png|svg)$ { expires 7d; }
    location / { try_files $uri /index.html; }
}
EOF
sudo ln -s /etc/nginx/sites-available/tet2027 /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

---

## Đưa ra Internet qua Cloudflare Tunnel (tunnel "home")

```bash
# 1. Tạo bản ghi DNS tet.migor.site trỏ vào tunnel
cloudflared tunnel route dns home tet.migor.site

# 2. Thêm hostname vào file config của tunnel
#    (~/.cloudflared/config.yml hoặc /etc/cloudflared/config.yml — xem cloudflared-ingress.yml)
#    - hostname: tet.migor.site
#      service: http://localhost:8027
#    Đặt TRƯỚC dòng "- service: http_status:404"

# 3. Kiểm tra cú pháp rồi khởi động lại
cloudflared tunnel ingress validate
sudo systemctl restart cloudflared

# 4. Thử
curl -I https://tet.migor.site
```

Cloudflare tự cấp HTTPS, không cần certbot.

> Nếu Cloudflare cache trang cũ sau khi bạn sửa: Cloudflare Dashboard → Caching → **Purge Everything**.

---

## Ghi chú

- **Giờ giao thừa** được tính cố định theo UTC+7, nên người xem ở nước ngoài vẫn thấy đếm đúng giờ Việt Nam.
- **Mã QR lì xì** đã được dựng sẵn trong `index.html` (6 mã: Tùy tâm, 20k, 50k, 100k, 200k, 500k).
  Mã "Tùy tâm" là mã gốc của tài khoản; các mã có số tiền điền sẵn lời nhắn `Li xi Tet Dinh Mui 2027`.
  Nên quét thử mã 20k bằng app ngân hàng trước khi chia sẻ rộng.
- **Font** tải từ Google Fonts (Unbounded, Be Vietnam Pro). Nếu máy không có mạng, trang vẫn chạy với font hệ thống.
- Trang công khai số tài khoản và mã QR — chỉ chia sẻ link cho người bạn muốn.
