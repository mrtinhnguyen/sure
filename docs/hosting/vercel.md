# Hướng dẫn Deploy Sure lên Vercel

> ⚠️ **CẢNH BÁO QUAN TRỌNG**: Vercel **KHÔNG được khuyến nghị** cho việc deploy ứng dụng Rails đầy đủ như Sure. Tài liệu này chỉ mang tính tham khảo và có nhiều hạn chế.

## Tại sao Vercel không phù hợp?

Sure là một ứng dụng Rails đầy đủ với các yêu cầu:

- ✅ **PostgreSQL Database** - Vercel không cung cấp database service
- ✅ **Redis** - Cần cho Sidekiq background jobs, Vercel không hỗ trợ
- ✅ **Background Jobs** - Sidekiq cần chạy liên tục, Vercel chỉ hỗ trợ serverless functions
- ✅ **File Storage** - Rails Active Storage cần persistent storage, Vercel có giới hạn
- ✅ **Long-running processes** - Rails server cần chạy liên tục, Vercel có timeout giới hạn

## Giải pháp thay thế được khuyến nghị

Thay vì Vercel, bạn nên sử dụng các nền tảng sau:

### 1. **Railway** (Khuyến nghị nhất)
- ✅ Hỗ trợ đầy đủ Rails, PostgreSQL, Redis
- ✅ Dễ deploy với Docker
- ✅ Có free tier
- [Hướng dẫn deploy Railway](https://railway.com/deploy)

### 2. **Render**
- ✅ Hỗ trợ Rails, PostgreSQL, Redis
- ✅ Free tier có sẵn
- ✅ Tự động deploy từ GitHub
- [Website Render](https://render.com)

### 3. **Heroku**
- ✅ Platform as a Service phổ biến cho Rails
- ✅ Hỗ trợ đầy đủ các service cần thiết
- ⚠️ Có phí sau free tier
- [Website Heroku](https://www.heroku.com)

### 4. **Self-hosted với Docker** (Khuyến nghị cho production)
- ✅ Kiểm soát hoàn toàn
- ✅ Chi phí thấp
- ✅ Không có giới hạn
- [Hướng dẫn Docker](docker.md)

## Nếu vẫn muốn thử deploy lên Vercel

> ⚠️ **Lưu ý**: Cách này chỉ phù hợp cho việc demo/test, **KHÔNG phù hợp cho production**.

### Yêu cầu

1. Database bên ngoài (PostgreSQL):
   - [Supabase](https://supabase.com) (free tier)
   - [Neon](https://neon.tech) (free tier)
   - [Railway PostgreSQL](https://railway.com) (free tier)

2. Redis bên ngoài:
   - [Upstash Redis](https://upstash.com) (free tier)
   - [Redis Cloud](https://redis.com/cloud) (free tier)

3. File Storage:
   - [Cloudinary](https://cloudinary.com) (free tier)
   - [AWS S3](https://aws.amazon.com/s3)

### Bước 1: Tạo file `vercel.json`

Tạo file `vercel.json` ở root của project:

```json
{
  "version": 2,
  "builds": [
    {
      "src": "package.json",
      "use": "@vercel/static-build",
      "config": {
        "distDir": "public"
      }
    },
    {
      "src": "api/index.rb",
      "use": "@vercel/ruby"
    }
  ],
  "routes": [
    {
      "src": "/api/(.*)",
      "dest": "/api/index.rb"
    },
    {
      "src": "/(.*)",
      "dest": "/$1"
    }
  ],
  "env": {
    "RAILS_ENV": "production"
  }
}
```

### Bước 2: Tạo API handler (Serverless Function)

Tạo file `api/index.rb`:

```ruby
require 'json'

def handler(event:, context:)
  {
    statusCode: 200,
    body: JSON.generate({
      message: "Rails API on Vercel",
      note: "This is a limited implementation. Full Rails app deployment on Vercel is not recommended."
    })
  }
end
```

### Bước 3: Cấu hình Environment Variables

Trong Vercel Dashboard, thêm các biến môi trường:

```bash
# Database (từ Supabase/Neon)
DATABASE_URL=postgresql://user:password@host:port/database

# Redis (từ Upstash)
REDIS_URL=redis://default:password@host:port

# Rails
RAILS_ENV=production
SECRET_KEY_BASE=your_secret_key_base_here
RAILS_MASTER_KEY=your_master_key_here

# Storage (Cloudinary)
CLOUDINARY_URL=cloudinary://api_key:api_secret@cloud_name
```

### Bước 4: Build Configuration

Cập nhật `package.json` để thêm build script:

```json
{
  "scripts": {
    "build": "echo 'Building for Vercel...' && echo 'Note: Full Rails deployment on Vercel is limited'",
    "vercel-build": "bundle install && rails assets:precompile"
  }
}
```

### Bước 5: Deploy

```bash
# Cài đặt Vercel CLI
npm i -g vercel

# Login
vercel login

# Deploy
vercel

# Deploy production
vercel --prod
```

## Hạn chế khi deploy lên Vercel

1. **Không có background jobs**: Sidekiq không thể chạy trên Vercel
2. **Timeout giới hạn**: Serverless functions có timeout (10s cho free, 60s cho pro)
3. **Cold starts**: Mỗi request có thể bị delay do cold start
4. **Không có persistent storage**: File uploads cần dùng external storage
5. **Giới hạn memory**: Có giới hạn về memory cho mỗi function
6. **Không hỗ trợ WebSockets**: Turbo Streams có thể không hoạt động tốt

## Kết luận

**Khuyến nghị**: Sử dụng **Railway**, **Render**, hoặc **self-hosted với Docker** thay vì Vercel cho ứng dụng Sure.

Nếu bạn cần hỗ trợ deploy lên các nền tảng khác, vui lòng tham khảo:
- [Hướng dẫn Docker](docker.md)
- [Hướng dẫn Hetzner](hetzner.md)
- [Railway Deploy Button](https://railway.com/deploy)
