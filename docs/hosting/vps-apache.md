# Hướng dẫn Deploy Sure lên VPS với Apache (Không dùng Docker)

Hướng dẫn này sẽ giúp bạn deploy ứng dụng Sure lên VPS riêng mà không sử dụng Docker, với Apache làm reverse proxy.

## Yêu cầu hệ thống

- **VPS**: Ubuntu 22.04 LTS hoặc Debian 12 (khuyến nghị)
- **RAM**: Tối thiểu 2GB (khuyến nghị 4GB+)
- **CPU**: Tối thiểu 2 cores
- **Storage**: Tối thiểu 20GB
- **Domain name**: Trỏ về IP của VPS

## Bước 1: Cài đặt Dependencies

### Cập nhật hệ thống

```bash
sudo apt update && sudo apt upgrade -y
```

### Cài đặt các package cần thiết

```bash
# Cài đặt các package cơ bản
sudo apt install -y curl git build-essential libssl-dev libreadline-dev \
  zlib1g-dev libyaml-dev libxml2-dev libxslt1-dev libncurses5-dev \
  libffi-dev libgdbm-dev libpq-dev libvips42

# Cài đặt PostgreSQL
sudo apt install -y postgresql postgresql-contrib

# Cài đặt Redis
sudo apt install -y redis-server

# Cài đặt Node.js (cho asset compilation)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

# Cài đặt Apache
sudo apt install -y apache2

# Cài đặt Certbot (cho SSL)
sudo apt install -y certbot python3-certbot-apache
```

### Cài đặt Ruby với rbenv (khuyến nghị)

```bash
# Tạo user cho ứng dụng (khuyến nghị)
sudo adduser --disabled-password --gecos "" sure
sudo usermod -aG sudo sure

# Chuyển sang user sure
sudo su - sure

# Cài đặt rbenv
curl -fsSL https://github.com/rbenv/rbenv-installer/raw/HEAD/bin/rbenv-installer | bash

# Thêm rbenv vào PATH
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init - bash)"' >> ~/.bashrc
source ~/.bashrc

# Cài đặt ruby-build plugin
mkdir -p "$(rbenv root)"/plugins
git clone https://github.com/rbenv/ruby-build.git "$(rbenv root)"/plugins/ruby-build

# Cài đặt Ruby 3.4.7
rbenv install 3.4.7
rbenv global 3.4.7

# Cài đặt Bundler
gem install bundler

# Cài đặt Yarn (cho asset pipeline)
npm install -g yarn
```

## Bước 2: Cấu hình PostgreSQL

```bash
# Chuyển sang user postgres
sudo -u postgres psql

# Trong PostgreSQL shell, tạo database và user
CREATE USER sure_user WITH CREATEDB PASSWORD 'your_secure_password_here';
CREATE DATABASE sure_production OWNER sure_user;
\q
```

**Lưu ý**: Thay `your_secure_password_here` bằng mật khẩu mạnh. Lưu lại mật khẩu này để dùng trong bước sau.

## Bước 3: Cấu hình Redis

```bash
# Kiểm tra Redis đang chạy
sudo systemctl status redis-server

# Nếu chưa chạy, khởi động
sudo systemctl start redis-server
sudo systemctl enable redis-server

# Cấu hình Redis (tùy chọn - cho production)
sudo nano /etc/redis/redis.conf

# Tìm và sửa:
# bind 127.0.0.1  (chỉ cho phép localhost)
# requirepass your_redis_password  (thêm password nếu cần)
```

## Bước 4: Deploy ứng dụng

### Clone repository

```bash
# Vẫn đang ở user sure
cd /home/sure

# Clone repository (thay YOUR_REPO_URL bằng URL repo của bạn)
git clone https://github.com/we-promise/sure.git app
cd app

# Hoặc nếu bạn có code riêng, copy vào /home/sure/app
```

### Cài đặt dependencies

```bash
# Cài đặt Ruby gems
bundle install --deployment --without development test

# Cài đặt Node.js packages (nếu có package.json)
npm install --production
```

### Cấu hình Environment Variables

```bash
# Tạo file .env
nano /home/sure/app/.env
```

Thêm nội dung sau (thay các giá trị phù hợp):

```bash
# Rails Environment
RAILS_ENV=production
SECRET_KEY_BASE=$(openssl rand -hex 64)

# Database
DATABASE_URL=postgresql://sure_user:your_secure_password_here@localhost:5432/sure_production

# Redis
REDIS_URL=redis://localhost:6379/0

# Application
RAILS_MASTER_KEY=your_master_key_here
HOST=yourdomain.com

# Optional: OpenAI (nếu sử dụng AI features)
# OPENAI_ACCESS_TOKEN=your_openai_key

# Optional: Email (nếu cần gửi email)
# SMTP_HOST=smtp.gmail.com
# SMTP_PORT=587
# SMTP_USERNAME=your_email@gmail.com
# SMTP_PASSWORD=your_password
```

**Lưu ý quan trọng**:
- Tạo `SECRET_KEY_BASE`: `openssl rand -hex 64`
- Lấy `RAILS_MASTER_KEY` từ file `config/master.key` hoặc tạo mới
- Thay `your_secure_password_here` bằng mật khẩu PostgreSQL đã tạo ở Bước 2

### Chạy migrations và setup database

```bash
cd /home/sure/app

# Load environment variables
export $(cat .env | xargs)

# Chạy migrations
bundle exec rails db:create db:migrate

# Precompile assets
bundle exec rails assets:precompile

# Tạo user đầu tiên (nếu cần seed data)
# bundle exec rails db:seed
```

## Bước 5: Tạo Systemd Services

### Service cho Rails (Puma)

```bash
sudo nano /etc/systemd/system/sure-web.service
```

Thêm nội dung:

```ini
[Unit]
Description=Sure Rails Application
After=network.target postgresql.service redis-server.service

[Service]
Type=simple
User=sure
WorkingDirectory=/home/sure/app
Environment="RAILS_ENV=production"
EnvironmentFile=/home/sure/app/.env
ExecStart=/home/sure/.rbenv/shims/bundle exec puma -C config/puma.rb
Restart=always
RestartSec=10
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=sure-web

[Install]
WantedBy=multi-user.target
```

### Service cho Sidekiq (Background Jobs)

```bash
sudo nano /etc/systemd/system/sure-worker.service
```

Thêm nội dung:

```ini
[Unit]
Description=Sure Sidekiq Worker
After=network.target postgresql.service redis-server.service

[Service]
Type=simple
User=sure
WorkingDirectory=/home/sure/app
Environment="RAILS_ENV=production"
EnvironmentFile=/home/sure/app/.env
ExecStart=/home/sure/.rbenv/shims/bundle exec sidekiq
Restart=always
RestartSec=10
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=sure-worker

[Install]
WantedBy=multi-user.target
```

### Khởi động services

```bash
# Reload systemd
sudo systemctl daemon-reload

# Khởi động services
sudo systemctl start sure-web
sudo systemctl start sure-worker

# Enable tự động khởi động khi boot
sudo systemctl enable sure-web
sudo systemctl enable sure-worker

# Kiểm tra status
sudo systemctl status sure-web
sudo systemctl status sure-worker
```

## Bước 6: Cấu hình Apache

### Bật các module cần thiết

```bash
sudo a2enmod proxy
sudo a2enmod proxy_http
sudo a2enmod proxy_balancer
sudo a2enmod lbmethod_byrequests
sudo a2enmod rewrite
sudo a2enmod ssl
sudo a2enmod headers
```

### Tạo Virtual Host

```bash
sudo nano /etc/apache2/sites-available/sure.conf
```

Thêm nội dung (thay `yourdomain.com` bằng domain của bạn):

```apache
<VirtualHost *:80>
    ServerName yourdomain.com
    ServerAlias www.yourdomain.com

    # Redirect HTTP to HTTPS
    Redirect permanent / https://yourdomain.com/
</VirtualHost>

<VirtualHost *:443>
    ServerName yourdomain.com
    ServerAlias www.yourdomain.com

    # SSL Configuration (sẽ được cấu hình bởi Certbot)
    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/yourdomain.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/yourdomain.com/privkey.pem

    # Proxy to Puma
    ProxyPreserveHost On
    ProxyPass / http://127.0.0.1:3000/
    ProxyPassReverse / http://127.0.0.1:3000/

    # Headers
    RequestHeader set X-Forwarded-Proto "https"
    RequestHeader set X-Forwarded-Ssl "on"

    # Logging
    ErrorLog ${APACHE_LOG_DIR}/sure_error.log
    CustomLog ${APACHE_LOG_DIR}/sure_access.log combined

    # Security Headers
    Header always set X-Content-Type-Options "nosniff"
    Header always set X-Frame-Options "SAMEORIGIN"
    Header always set X-XSS-Protection "1; mode=block"
</VirtualHost>
```

### Kích hoạt site

```bash
# Disable default site
sudo a2dissite 000-default.conf

# Enable Sure site
sudo a2ensite sure.conf

# Test cấu hình Apache
sudo apache2ctl configtest

# Restart Apache
sudo systemctl restart apache2
```

## Bước 7: Cấu hình SSL với Let's Encrypt

```bash
# Lấy SSL certificate
sudo certbot --apache -d yourdomain.com -d www.yourdomain.com

# Certbot sẽ tự động cập nhật cấu hình Apache
# Kiểm tra auto-renewal
sudo certbot renew --dry-run
```

## Bước 8: Cấu hình Firewall

```bash
# Cài đặt UFW (nếu chưa có)
sudo apt install -y ufw

# Cho phép SSH, HTTP, HTTPS
sudo ufw allow OpenSSH
sudo ufw allow 'Apache Full'
sudo ufw enable

# Kiểm tra status
sudo ufw status
```

## Bước 9: Kiểm tra và Troubleshooting

### Kiểm tra services đang chạy

```bash
# Kiểm tra Rails app
sudo systemctl status sure-web
curl http://localhost:3000

# Kiểm tra Sidekiq
sudo systemctl status sure-worker

# Kiểm tra logs
sudo journalctl -u sure-web -f
sudo journalctl -u sure-worker -f
```

### Kiểm tra Apache logs

```bash
# Error logs
sudo tail -f /var/log/apache2/sure_error.log

# Access logs
sudo tail -f /var/log/apache2/sure_access.log
```

### Kiểm tra database connection

```bash
sudo -u sure bash
cd /home/sure/app
export $(cat .env | xargs)
bundle exec rails db
```

## Bước 10: Cập nhật ứng dụng

Khi cần cập nhật ứng dụng:

```bash
# Chuyển sang user sure
sudo su - sure
cd /home/sure/app

# Pull code mới
git pull origin main  # hoặc branch của bạn

# Cài đặt dependencies mới
bundle install --deployment --without development test
npm install --production

# Chạy migrations
export $(cat .env | xargs)
bundle exec rails db:migrate

# Precompile assets
bundle exec rails assets:precompile

# Restart services
exit
sudo systemctl restart sure-web
sudo systemctl restart sure-worker
```

## Cấu hình bổ sung

### Tối ưu Puma

Chỉnh sửa `config/puma.rb` để phù hợp với server:

```ruby
# Số workers = số CPU cores
workers ENV.fetch("WEB_CONCURRENCY") { 2 }

# Số threads mỗi worker
threads_count = ENV.fetch("RAILS_MAX_THREADS") { 5 }
threads threads_count, threads_count

# Preload app
preload_app!

# Bind
bind "unix:///home/sure/app/tmp/sockets/puma.sock"
```

Cập nhật Apache config để dùng Unix socket:

```apache
ProxyPass / unix:///home/sure/app/tmp/sockets/puma.sock|http://127.0.0.1/
```

### Backup Database

Tạo script backup:

```bash
sudo nano /home/sure/backup-db.sh
```

```bash
#!/bin/bash
BACKUP_DIR="/home/sure/backups"
DATE=$(date +%Y%m%d_%H%M%S)
mkdir -p $BACKUP_DIR

pg_dump -U sure_user sure_production > $BACKUP_DIR/sure_$DATE.sql

# Xóa backup cũ hơn 7 ngày
find $BACKUP_DIR -name "sure_*.sql" -mtime +7 -delete
```

```bash
chmod +x /home/sure/backup-db.sh

# Thêm vào crontab (chạy mỗi ngày lúc 2h sáng)
crontab -e
# Thêm dòng:
0 2 * * * /home/sure/backup-db.sh
```

## Troubleshooting

### Lỗi "Permission denied"

```bash
# Đảm bảo user sure có quyền
sudo chown -R sure:sure /home/sure/app
```

### Lỗi database connection

```bash
# Kiểm tra PostgreSQL đang chạy
sudo systemctl status postgresql

# Kiểm tra connection
sudo -u postgres psql -c "\l"
```

### Lỗi "Address already in use"

```bash
# Tìm process đang dùng port 3000
sudo lsof -i :3000
# Kill process nếu cần
sudo kill -9 <PID>
```

### Assets không load

```bash
# Recompile assets
cd /home/sure/app
export $(cat .env | xargs)
bundle exec rails assets:precompile
sudo systemctl restart sure-web
```

## Kết luận

Sau khi hoàn thành các bước trên, ứng dụng Sure sẽ chạy trên VPS của bạn với:
- ✅ Rails app chạy trên Puma
- ✅ Sidekiq worker cho background jobs
- ✅ Apache làm reverse proxy
- ✅ SSL certificate từ Let's Encrypt
- ✅ Auto-restart khi server reboot
- ✅ Logging và monitoring

Truy cập ứng dụng tại: `https://yourdomain.com`
