# Hướng dẫn Deploy Sure lên VPS - Quick Start

Hướng dẫn nhanh để deploy ứng dụng Sure lên VPS riêng của bạn.

## Tổng quan

Có 2 cách deploy lên VPS:
1. **Với Docker** (Khuyến nghị - Dễ nhất) - Xem [docker.md](docker.md)
2. **Không dùng Docker** (Chi tiết hơn) - Xem [vps-apache.md](vps-apache.md)

## Yêu cầu VPS

- **OS**: Ubuntu 22.04 LTS hoặc Debian 12
- **RAM**: Tối thiểu 2GB (khuyến nghị 4GB+)
- **CPU**: Tối thiểu 2 cores
- **Storage**: Tối thiểu 20GB
- **Domain**: Trỏ về IP của VPS (tùy chọn, có thể dùng IP trực tiếp)

## Phương án 1: Deploy với Docker (Khuyến nghị)

### Ưu điểm:
- ✅ Dễ setup và maintain
- ✅ Tự động cấu hình tất cả services
- ✅ Dễ update và rollback
- ✅ Isolated environment

### Bước 1: Cài đặt Docker

```bash
# Cập nhật hệ thống
sudo apt update && sudo apt upgrade -y

# Cài đặt Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Cài đặt Docker Compose
sudo apt install -y docker-compose-plugin

# Thêm user vào docker group (thay $USER bằng username của bạn)
sudo usermod -aG docker $USER

# Logout và login lại để áp dụng thay đổi
```

### Bước 2: Tạo thư mục và download config

```bash
# Tạo thư mục cho app
mkdir -p ~/sure
cd ~/sure

# Download Docker Compose config
curl -o compose.yml https://raw.githubusercontent.com/we-promise/sure/main/compose.example.yml
```

### Bước 3: Cấu hình Environment

```bash
# Tạo file .env
nano .env
```

Thêm nội dung:

```bash
# Generate secret key
SECRET_KEY_BASE=$(openssl rand -hex 64)

# Database
POSTGRES_USER=sure_user
POSTGRES_PASSWORD=$(openssl rand -base64 32)
POSTGRES_DB=sure_production

# Optional: OpenAI
# OPENAI_ACCESS_TOKEN=your_key_here

# Optional: Domain (nếu có)
# HOST=yourdomain.com
```

**Lưu ý**: Lưu lại `POSTGRES_PASSWORD` để dùng sau này!

### Bước 4: Chạy ứng dụng

```bash
# Pull images và start services
docker compose up -d

# Xem logs
docker compose logs -f

# Kiểm tra services đang chạy
docker compose ps
```

### Bước 5: Tạo user đầu tiên

```bash
# Vào Rails console
docker compose exec web bundle exec rails console

# Tạo user
user = User.create!(
  email: "admin@example.com",
  password: "SecurePassword123!",
  password_confirmation: "SecurePassword123!",
  first_name: "Admin",
  last_name: "User"
)

# Tạo family
family = Family.create!(
  name: "My Family",
  currency: "VND",
  locale: "vi"
)

family.family_members.create!(
  user: user,
  role: "admin"
)
```

### Bước 6: Cấu hình Reverse Proxy (Nginx)

```bash
# Cài đặt Nginx
sudo apt install -y nginx

# Tạo config
sudo nano /etc/nginx/sites-available/sure
```

Thêm nội dung (thay `yourdomain.com` và `YOUR_VPS_IP`):

```nginx
server {
    listen 80;
    server_name yourdomain.com YOUR_VPS_IP;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```bash
# Enable site
sudo ln -s /etc/nginx/sites-available/sure /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default

# Test và restart
sudo nginx -t
sudo systemctl restart nginx
```

### Bước 7: Cài đặt SSL (Let's Encrypt)

```bash
# Cài đặt Certbot
sudo apt install -y certbot python3-certbot-nginx

# Lấy SSL certificate (chỉ khi có domain)
sudo certbot --nginx -d yourdomain.com

# Auto-renewal đã được cấu hình tự động
```

### Bước 8: Hoàn tất!

Truy cập ứng dụng tại:
- `http://YOUR_VPS_IP` (nếu chưa có domain)
- `https://yourdomain.com` (nếu đã có domain và SSL)

## Phương án 2: Deploy không dùng Docker

Xem hướng dẫn chi tiết tại: [vps-apache.md](vps-apache.md)

### Tóm tắt các bước:
1. Cài đặt Ruby, PostgreSQL, Redis, Node.js
2. Clone repository
3. Cài đặt dependencies
4. Cấu hình environment variables
5. Chạy migrations
6. Setup systemd services
7. Cấu hình Apache/Nginx reverse proxy
8. Cài đặt SSL

## Cập nhật ứng dụng

### Với Docker:

```bash
cd ~/sure

# Pull code mới
git pull origin main

# Rebuild và restart
docker compose pull
docker compose up -d --build

# Chạy migrations
docker compose exec web bundle exec rails db:migrate
```

### Không dùng Docker:

```bash
cd /home/sure/app

# Pull code mới
git pull origin main

# Cài đặt dependencies mới
bundle install --deployment --without development test
npm install --production

# Chạy migrations
export $(cat .env | xargs)
bundle exec rails db:migrate

# Precompile assets
bundle exec rails assets:precompile

# Restart services
sudo systemctl restart sure-web
sudo systemctl restart sure-worker
```

## Backup Database

### Với Docker:

```bash
# Backup
docker compose exec db pg_dump -U sure_user sure_production > backup_$(date +%Y%m%d).sql

# Restore
docker compose exec -T db psql -U sure_user sure_production < backup_20250101.sql
```

### Không dùng Docker:

```bash
# Backup
pg_dump -U sure_user sure_production > backup_$(date +%Y%m%d).sql

# Restore
psql -U sure_user sure_production < backup_20250101.sql
```

## Troubleshooting

### Kiểm tra logs

**Với Docker:**
```bash
docker compose logs -f web
docker compose logs -f worker
```

**Không dùng Docker:**
```bash
sudo journalctl -u sure-web -f
sudo journalctl -u sure-worker -f
```

### Kiểm tra services

**Với Docker:**
```bash
docker compose ps
docker compose exec web bundle exec rails runner "puts 'OK'"
```

**Không dùng Docker:**
```bash
sudo systemctl status sure-web
sudo systemctl status sure-worker
```

### Lỗi thường gặp

1. **Port 3000 đã được sử dụng**
   - Kiểm tra process: `sudo lsof -i :3000`
   - Kill process: `sudo kill -9 <PID>`

2. **Database connection failed**
   - Kiểm tra PostgreSQL đang chạy: `sudo systemctl status postgresql`
   - Kiểm tra credentials trong `.env` hoặc environment variables

3. **Redis connection failed**
   - Kiểm tra Redis đang chạy: `sudo systemctl status redis-server`
   - Kiểm tra `REDIS_URL` trong config

4. **Assets không load**
   - Precompile lại: `bundle exec rails assets:precompile`
   - Kiểm tra quyền file: `sudo chown -R sure:sure /home/sure/app/public`

## Bảo mật

### Firewall (UFW)

```bash
# Cài đặt UFW
sudo apt install -y ufw

# Cho phép SSH, HTTP, HTTPS
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'  # hoặc 'Apache Full'
sudo ufw enable

# Kiểm tra
sudo ufw status
```

### Fail2Ban (Bảo vệ SSH)

```bash
# Cài đặt
sudo apt install -y fail2ban

# Khởi động
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

## Monitoring

### Setup log rotation

```bash
# Tạo logrotate config
sudo nano /etc/logrotate.d/sure
```

Thêm:
```
/home/sure/app/log/*.log {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    create 0640 sure sure
    sharedscripts
}
```

## Tài nguyên

- [Hướng dẫn Docker chi tiết](docker.md)
- [Hướng dẫn VPS Apache chi tiết](vps-apache.md)
- [Docker Compose Example](https://github.com/we-promise/sure/blob/main/compose.example.yml)

## Kết luận

**Khuyến nghị**: Sử dụng Docker để deploy vì:
- ✅ Dễ setup hơn
- ✅ Dễ maintain hơn
- ✅ Dễ scale hơn
- ✅ Isolated environment

Nếu bạn cần kiểm soát chi tiết hơn, sử dụng phương án không dùng Docker.

Chúc bạn deploy thành công! 🚀
