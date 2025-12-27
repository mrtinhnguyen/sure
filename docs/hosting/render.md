# Hướng dẫn Deploy Sure lên Render

Hướng dẫn chi tiết này sẽ giúp bạn deploy ứng dụng Sure lên Render một cách hoàn chỉnh và không có lỗi.

## Tổng quan

Render là một nền tảng Platform as a Service (PaaS) hỗ trợ đầy đủ Rails, PostgreSQL, và Redis. Hướng dẫn này sẽ giúp bạn:

- ✅ Deploy web service (Rails app)
- ✅ Deploy background worker (Sidekiq)
- ✅ Setup PostgreSQL database
- ✅ Setup Redis
- ✅ Cấu hình SSL tự động
- ✅ Auto-deploy từ GitHub

## Yêu cầu

- Tài khoản GitHub (để kết nối repository)
- Tài khoản Render (đăng ký tại [render.com](https://render.com))
- Repository Sure đã push lên GitHub

## Bước 1: Chuẩn bị Repository

### 1.1. Đảm bảo code đã được push lên GitHub

```bash
# Kiểm tra remote
git remote -v

# Nếu chưa có, thêm GitHub remote
git remote add origin https://github.com/your-username/sure.git
git push -u origin main
```

### 1.2. Tạo file `render.yaml` (Đã có sẵn)

File `render.yaml` đã được tạo ở root của project với cấu hình đầy đủ. Bạn có thể chỉnh sửa nếu cần:

- **Region**: Thay đổi `singapore` thành region gần bạn nhất
- **Plan**: Thay đổi `starter` thành plan phù hợp (starter, standard, pro)

## Bước 2: Tạo tài khoản và kết nối GitHub

### 2.1. Đăng ký Render

1. Truy cập [render.com](https://render.com)
2. Click "Get Started for Free"
3. Đăng nhập bằng GitHub account

### 2.2. Kết nối GitHub Repository

1. Vào Dashboard → "New" → "Blueprint"
2. Kết nối GitHub account nếu chưa
3. Chọn repository chứa code Sure
4. Render sẽ tự động phát hiện file `render.yaml`

## Bước 3: Deploy với Blueprint (Khuyến nghị)

### 3.1. Sử dụng Blueprint từ render.yaml

1. Trong Render Dashboard, click "New" → "Blueprint"
2. Chọn repository của bạn
3. Render sẽ tự động đọc `render.yaml` và tạo:
   - Web Service (sure-web)
   - Worker Service (sure-worker)
   - PostgreSQL Database (sure-db)
   - Redis (sure-redis)

### 3.2. Cấu hình Environment Variables

Sau khi Blueprint tạo các services, bạn cần cấu hình các biến môi trường:

#### Cho Web Service (sure-web):

1. Vào **sure-web** service → "Environment"
2. Thêm các biến sau:

```bash
# Rails Master Key (bắt buộc)
RAILS_MASTER_KEY=your_master_key_here

# Host (domain của bạn)
HOST=your-app-name.onrender.com

# Optional: OpenAI (nếu sử dụng AI features)
OPENAI_ACCESS_TOKEN=your_openai_key_here

# Optional: Email (nếu cần gửi email)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USERNAME=your_email@gmail.com
SMTP_PASSWORD=your_app_password

# Optional: Stripe (nếu sử dụng payments)
STRIPE_PUBLISHABLE_KEY=pk_live_...
STRIPE_SECRET_KEY=sk_live_...

# Optional: Plaid (nếu sử dụng bank sync)
PLAID_CLIENT_ID=your_plaid_client_id
PLAID_SECRET=your_plaid_secret
PLAID_ENV=sandbox  # hoặc production
```

**Lưu ý quan trọng về RAILS_MASTER_KEY**:
- File `config/master.key` chứa master key
- Copy nội dung file này vào biến `RAILS_MASTER_KEY`
- **KHÔNG** commit file `master.key` lên GitHub (đã có trong .gitignore)

#### Cho Worker Service (sure-worker):

Worker service sẽ tự động kế thừa các biến từ web service. Bạn chỉ cần đảm bảo `RAILS_MASTER_KEY` được set.

### 3.3. Lấy RAILS_MASTER_KEY

Nếu bạn không có file `config/master.key`, tạo mới:

```bash
# Trên máy local
cd /path/to/sure

# Tạo master key mới (nếu chưa có)
EDITOR="code --wait" rails credentials:edit

# Hoặc xem master key hiện tại
cat config/master.key
```

**Lưu ý**: Nếu tạo master key mới, bạn cần cập nhật lại file `config/credentials.yml.enc` hoặc tạo credentials mới.

## Bước 4: Deploy Manual (Nếu không dùng Blueprint)

Nếu bạn muốn tạo từng service thủ công:

### 4.1. Tạo PostgreSQL Database

1. Dashboard → "New" → "PostgreSQL"
2. Tên: `sure-db`
3. Database: `sure_production`
4. User: `sure_user`
5. Region: Chọn region gần bạn
6. Plan: Starter (free tier) hoặc cao hơn
7. Click "Create Database"

### 4.2. Tạo Redis

1. Dashboard → "New" → "Redis"
2. Tên: `sure-redis`
3. Region: Cùng region với database
4. Plan: Starter (free tier) hoặc cao hơn
5. Click "Create Redis"

### 4.3. Tạo Web Service

1. Dashboard → "New" → "Web Service"
2. Connect repository của bạn
3. Cấu hình:
   - **Name**: `sure-web`
   - **Region**: Cùng region với database
   - **Branch**: `main` (hoặc branch bạn muốn)
   - **Root Directory**: (để trống)
   - **Environment**: `Ruby`
   - **Build Command**: `./bin/render-build.sh`
   - **Start Command**: `bundle exec puma -C config/puma.rb`

4. **Environment Variables**:
   - `RAILS_ENV` = `production`
   - `RAILS_MASTER_KEY` = (paste master key)
   - `SECRET_KEY_BASE` = (Render sẽ tự generate)
   - `DATABASE_URL` = (chọn từ sure-db → Internal Database URL)
   - `REDIS_URL` = (chọn từ sure-redis → Internal Redis URL)
   - `HOST` = `your-app-name.onrender.com`
   - `PORT` = `10000` (Render tự set, nhưng cần khai báo)

5. Click "Create Web Service"

### 4.4. Tạo Worker Service

1. Dashboard → "New" → "Background Worker"
2. Connect repository của bạn
3. Cấu hình:
   - **Name**: `sure-worker`
   - **Region**: Cùng region với web service
   - **Branch**: `main`
   - **Environment**: `Ruby`
   - **Build Command**: `./bin/render-build.sh`
   - **Start Command**: `bundle exec sidekiq`

4. **Environment Variables** (giống web service):
   - `RAILS_ENV` = `production`
   - `RAILS_MASTER_KEY` = (cùng với web service)
   - `SECRET_KEY_BASE` = (sync từ sure-web)
   - `DATABASE_URL` = (chọn từ sure-db)
   - `REDIS_URL` = (chọn từ sure-redis)

5. Click "Create Background Worker"

## Bước 5: Chạy Database Migrations

Sau khi deploy, bạn cần chạy migrations:

### 5.1. Sử dụng Render Shell

1. Vào **sure-web** service
2. Click "Shell" tab
3. Chạy lệnh:

```bash
bundle exec rails db:create db:migrate
```

### 5.2. Hoặc sử dụng Render CLI

```bash
# Cài đặt Render CLI
curl -fsSL https://render.com/cli.sh | bash

# Login
render login

# Chạy migrations
render exec sure-web -- bundle exec rails db:create db:migrate
```

## Bước 6: Tạo User đầu tiên

Sau khi migrations hoàn tất, tạo user admin đầu tiên:

### 6.1. Sử dụng Rails Console

1. Vào **sure-web** service → "Shell"
2. Chạy:

```bash
bundle exec rails console
```

3. Trong Rails console:

```ruby
# Tạo user đầu tiên
user = User.create!(
  email: "admin@example.com",
  password: "SecurePassword123!",
  password_confirmation: "SecurePassword123!",
  first_name: "Admin",
  last_name: "User"
)

# Tạo family và gán user làm admin
family = Family.create!(
  name: "My Family",
  currency: "VND",
  locale: "vi"
)

family.family_members.create!(
  user: user,
  role: "admin"
)

puts "User created: #{user.email}"
```

## Bước 7: Cấu hình Custom Domain (Tùy chọn)

### 7.1. Thêm Custom Domain

1. Vào **sure-web** service → "Settings" → "Custom Domains"
2. Click "Add"
3. Nhập domain của bạn (ví dụ: `sure.yourdomain.com`)
4. Render sẽ cung cấp DNS records để thêm vào DNS provider

### 7.2. Cập nhật Environment Variable

Sau khi thêm domain, cập nhật biến `HOST`:

1. Vào **sure-web** → "Environment"
2. Sửa `HOST` thành domain mới: `HOST=sure.yourdomain.com`
3. Restart service

## Bước 8: Kiểm tra và Troubleshooting

### 8.1. Kiểm tra Logs

**Web Service Logs**:
1. Vào **sure-web** service
2. Click "Logs" tab
3. Kiểm tra xem có lỗi không

**Worker Service Logs**:
1. Vào **sure-worker** service
2. Click "Logs" tab
3. Kiểm tra Sidekiq đang chạy

### 8.2. Common Issues và Solutions

#### Lỗi: "RAILS_MASTER_KEY is missing"

**Nguyên nhân**: Chưa set biến `RAILS_MASTER_KEY`

**Giải pháp**:
1. Vào service → "Environment"
2. Thêm `RAILS_MASTER_KEY` với giá trị từ `config/master.key`

#### Lỗi: "Database connection failed"

**Nguyên nhân**: `DATABASE_URL` chưa đúng hoặc database chưa sẵn sàng

**Giải pháp**:
1. Kiểm tra database đã được tạo chưa
2. Vào database service → "Connections" → Copy "Internal Database URL"
3. Paste vào `DATABASE_URL` của web service
4. Đảm bảo dùng "Internal Database URL" (không phải External)

#### Lỗi: "Redis connection failed"

**Nguyên nhân**: `REDIS_URL` chưa đúng

**Giải pháp**:
1. Vào Redis service → "Connections" → Copy "Internal Redis URL"
2. Paste vào `REDIS_URL` của web và worker service

#### Lỗi: "Asset precompilation failed"

**Nguyên nhân**: Build command có vấn đề

**Giải pháp**:
1. Kiểm tra file `bin/render-build.sh` có executable không
2. Đảm bảo Node.js được cài đặt (Render tự động cài)
3. Xem build logs để biết lỗi cụ thể

#### Lỗi: "Port already in use"

**Nguyên nhân**: Puma đang bind sai port

**Giải pháp**:
1. Đảm bảo `PORT` environment variable được set
2. Render tự động set `PORT`, nhưng cần khai báo trong env vars
3. Kiểm tra `config/puma.rb` sử dụng `ENV.fetch("PORT")`

#### Lỗi: "Sidekiq not processing jobs"

**Nguyên nhân**: Worker service chưa chạy hoặc Redis chưa kết nối

**Giải pháp**:
1. Kiểm tra worker service đang "Running"
2. Kiểm tra logs của worker service
3. Đảm bảo `REDIS_URL` đúng
4. Kiểm tra Sidekiq web UI (nếu có)

### 8.3. Health Check

Render tự động health check tại path `/`. Đảm bảo:

1. Route `/` trả về 200 OK
2. Application không crash khi start
3. Database connection thành công

## Bước 9: Auto-Deploy và CI/CD

### 9.1. Auto-Deploy từ GitHub

Mặc định, Render sẽ auto-deploy khi:
- Push code lên branch `main` (hoặc branch bạn chọn)
- Pull request được merge

### 9.2. Manual Deploy

1. Vào service → "Manual Deploy"
2. Chọn commit hoặc branch
3. Click "Deploy"

### 9.3. Deploy Preview (cho Pull Requests)

1. Vào service → "Settings" → "Pull Request Previews"
2. Enable "Create preview deployments for pull requests"
3. Mỗi PR sẽ tạo một preview deployment

## Bước 10: Monitoring và Maintenance

### 10.1. Xem Metrics

1. Vào service → "Metrics"
2. Xem:
   - CPU usage
   - Memory usage
   - Request rate
   - Response time

### 10.2. Setup Alerts

1. Vào service → "Settings" → "Alerts"
2. Thêm email alerts cho:
   - Service down
   - High error rate
   - High memory usage

### 10.3. Backup Database

Render tự động backup PostgreSQL:
- Starter plan: Daily backups (7 days retention)
- Standard plan: Daily backups (30 days retention)
- Pro plan: Continuous backups

Restore backup:
1. Vào database service → "Backups"
2. Chọn backup cần restore
3. Click "Restore"

## Bước 11: Scaling (Khi cần)

### 11.1. Scale Web Service

1. Vào **sure-web** → "Settings" → "Scaling"
2. Tăng số instances nếu cần
3. Tăng plan nếu cần thêm resources

### 11.2. Scale Worker Service

1. Vào **sure-worker** → "Settings" → "Scaling"
2. Tăng số instances để xử lý nhiều jobs hơn

### 11.3. Upgrade Database

1. Vào **sure-db** → "Settings" → "Plan"
2. Upgrade lên plan cao hơn nếu cần:
   - Starter: 1GB storage, shared CPU
   - Standard: 10GB storage, dedicated CPU
   - Pro: 100GB+ storage, high availability

## Bước 12: Cập nhật ứng dụng

### 12.1. Cập nhật Code

```bash
# Trên máy local
git add .
git commit -m "Update application"
git push origin main

# Render sẽ tự động deploy
```

### 12.2. Chạy Migrations sau khi deploy

1. Vào **sure-web** → "Shell"
2. Chạy:

```bash
bundle exec rails db:migrate
```

### 12.3. Restart Services

Nếu cần restart thủ công:

1. Vào service
2. Click "Manual Deploy" → "Clear build cache & deploy"

## Troubleshooting Checklist

Trước khi báo lỗi, kiểm tra:

- [ ] `RAILS_MASTER_KEY` đã được set
- [ ] `DATABASE_URL` đúng và dùng Internal URL
- [ ] `REDIS_URL` đúng và dùng Internal URL
- [ ] `HOST` đúng với domain của service
- [ ] Database migrations đã chạy
- [ ] Build command thành công (xem build logs)
- [ ] Start command đúng
- [ ] Port được set đúng (Render tự động set PORT=10000)
- [ ] Services đang ở trạng thái "Running"
- [ ] Không có lỗi trong logs

## Tài nguyên hữu ích

- [Render Documentation](https://render.com/docs)
- [Render Ruby on Rails Guide](https://render.com/docs/deploy-rails)
- [Render PostgreSQL Guide](https://render.com/docs/databases)
- [Render Redis Guide](https://render.com/docs/redis)

## Kết luận

Sau khi hoàn thành các bước trên, bạn sẽ có:

- ✅ Web service chạy Rails app
- ✅ Worker service chạy Sidekiq
- ✅ PostgreSQL database
- ✅ Redis cho caching và jobs
- ✅ SSL tự động
- ✅ Auto-deploy từ GitHub
- ✅ Monitoring và alerts

Truy cập ứng dụng tại: `https://your-app-name.onrender.com`

Chúc bạn deploy thành công! 🚀
