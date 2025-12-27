# Khắc phục lỗi 502/503 trên Render - Hướng dẫn nhanh

## Vấn đề chính: External URL vs Internal URL

**⚠️ QUAN TRỌNG**: Nếu bạn đang dùng **External Database/Redis URL**, đây chính là nguyên nhân gây lỗi 502/503!

### Tại sao phải dùng Internal URL?

1. **Bảo mật**: Internal URL chỉ có thể truy cập từ bên trong Render network
2. **Hiệu suất**: Internal URL nhanh hơn, không đi qua internet
3. **Đáng tin cậy**: Không bị ảnh hưởng bởi firewall hoặc network issues
4. **Miễn phí**: Internal traffic không tính phí bandwidth

### External URL sẽ gây ra:
- ❌ Connection timeout
- ❌ 502 Bad Gateway
- ❌ 503 Service Unavailable
- ❌ Slow response times
- ❌ Connection refused errors

## Bước 1: Tạo RAILS_MASTER_KEY

Nếu bạn không có file `config/master.key`, tạo mới:

### Cách 1: Tạo master key mới (Khuyến nghị)

```bash
# Trên máy local
cd ~/agentx/sure

# Tạo master key mới
EDITOR="nano" rails credentials:edit

# Hoặc nếu dùng VS Code
EDITOR="code --wait" rails credentials:edit
```

Lệnh này sẽ:
1. Tạo file `config/master.key` mới (nếu chưa có)
2. Mở editor để chỉnh sửa credentials
3. Lưu và đóng editor

**Lưu ý**: Nếu đây là lần đầu tạo, Rails sẽ tự động generate master key mới.

### Cách 2: Generate master key thủ công

```bash
# Generate một master key ngẫu nhiên
openssl rand -hex 32

# Copy output và lưu vào file
echo "your_generated_key_here" > config/master.key

# Hoặc tạo trực tiếp
rails secret | head -c 32 > config/master.key
```

### Cách 3: Lấy từ credentials hiện có

Nếu bạn đã có `config/credentials.yml.enc` nhưng mất `master.key`:

```bash
# Thử xem credentials (sẽ yêu cầu master key)
rails credentials:show

# Nếu không nhớ, bạn cần tạo lại credentials
# ⚠️ CẢNH BÁO: Sẽ mất tất cả credentials đã lưu
rails credentials:edit
```

### Lấy giá trị master key

Sau khi có file `config/master.key`:

```bash
cat config/master.key
```

Copy toàn bộ nội dung (chỉ có 1 dòng) và dán vào Render Environment Variables.

## Bước 2: Sửa Database URL - Dùng Internal URL

### 2.1. Lấy Internal Database URL

1. Vào Render Dashboard
2. Click vào **sure-db** (PostgreSQL service)
3. Click tab **"Connections"**
4. Tìm **"Internal Database URL"** (KHÔNG phải External!)
5. Copy URL (có dạng: `postgresql://sure_user:password@dpg-xxxxx-a.singapore-postgres.render.com:5432/sure_production`)

### 2.2. Cập nhật trong Render

1. Vào **sure-web** service
2. Click **"Environment"** tab
3. Tìm biến `DATABASE_URL`
4. **XÓA** giá trị cũ (External URL)
5. **PASTE** Internal Database URL mới
6. Click **"Save Changes"**

### 2.3. Làm tương tự cho Worker Service

1. Vào **sure-worker** service
2. Click **"Environment"** tab
3. Cập nhật `DATABASE_URL` với Internal URL
4. Click **"Save Changes"**

## Bước 3: Sửa Redis URL - Dùng Internal URL

### 3.1. Lấy Internal Redis URL

1. Vào Render Dashboard
2. Click vào **sure-redis** (Redis service)
3. Click tab **"Connections"**
4. Tìm **"Internal Redis URL"** (KHÔNG phải External!)
5. Copy URL (có dạng: `redis://red-xxxxx:6379` hoặc `rediss://red-xxxxx:6379`)

### 3.2. Cập nhật trong Render

1. Vào **sure-web** service
2. Click **"Environment"** tab
3. Tìm biến `REDIS_URL`
4. **XÓA** giá trị cũ (External URL)
5. **PASTE** Internal Redis URL mới
6. Click **"Save Changes"**

### 3.3. Làm tương tự cho Worker Service

1. Vào **sure-worker** service
2. Click **"Environment"** tab
3. Cập nhật `REDIS_URL` với Internal URL
4. Click **"Save Changes"**

## Bước 4: Thêm RAILS_MASTER_KEY vào Render

### 4.1. Thêm vào Web Service

1. Vào **sure-web** service
2. Click **"Environment"** tab
3. Click **"Add Environment Variable"**
4. Key: `RAILS_MASTER_KEY`
5. Value: Paste giá trị từ `config/master.key`
6. Click **"Save Changes"**

### 4.2. Thêm vào Worker Service

1. Vào **sure-worker** service
2. Click **"Environment"** tab
3. Thêm `RAILS_MASTER_KEY` với cùng giá trị
4. Click **"Save Changes"**

## Bước 5: Restart Services

Sau khi cập nhật tất cả environment variables:

### 5.1. Restart Web Service

1. Vào **sure-web** service
2. Click **"Manual Deploy"**
3. Chọn **"Clear build cache & deploy"**
4. Click **"Deploy"**

### 5.2. Restart Worker Service

1. Vào **sure-worker** service
2. Click **"Manual Deploy"**
3. Chọn **"Clear build cache & deploy"**
4. Click **"Deploy"**

## Bước 6: Kiểm tra kết quả

### 6.1. Xem Logs

1. Vào **sure-web** → **"Logs"** tab
2. Kiểm tra xem có lỗi connection không
3. Tìm dòng: `Listening on tcp://0.0.0.0:10000` (hoặc port khác)

### 6.2. Test Application

```bash
# Test health check
curl https://finance.tonyx.dev/up

# Test homepage
curl -I https://finance.tonyx.dev/
```

### 6.3. Kiểm tra Database Connection

1. Vào **sure-web** → **"Shell"** tab
2. Chạy:

```bash
bundle exec rails runner "puts ActiveRecord::Base.connection.execute('SELECT 1').first"
```

Nếu thành công, sẽ thấy: `{"?column?"=>1}`

### 6.4. Kiểm tra Redis Connection

1. Vào **sure-web** → **"Shell"** tab
2. Chạy:

```bash
bundle exec rails runner "puts Redis.new(url: ENV['REDIS_URL']).ping"
```

Nếu thành công, sẽ thấy: `PONG`

## Checklist hoàn chỉnh

Trước khi test, đảm bảo:

- [ ] `RAILS_MASTER_KEY` đã được thêm vào **sure-web**
- [ ] `RAILS_MASTER_KEY` đã được thêm vào **sure-worker**
- [ ] `DATABASE_URL` trong **sure-web** dùng **Internal URL**
- [ ] `DATABASE_URL` trong **sure-worker** dùng **Internal URL**
- [ ] `REDIS_URL` trong **sure-web** dùng **Internal URL**
- [ ] `REDIS_URL` trong **sure-worker** dùng **Internal URL**
- [ ] `HOST` đúng với domain (`finance.tonyx.dev`)
- [ ] Đã restart cả 2 services sau khi cập nhật
- [ ] Services đang ở trạng thái "Live"

## Phân biệt Internal vs External URL

### Internal Database URL
```
postgresql://user:pass@dpg-xxxxx-a.singapore-postgres.render.com:5432/dbname
```
- ✅ Hostname có dạng `dpg-xxxxx-a.singapore-postgres.render.com`
- ✅ Chỉ truy cập được từ bên trong Render network
- ✅ Nhanh và ổn định

### External Database URL
```
postgresql://user:pass@dpg-xxxxx-a.singapore-postgres.render.com:5432/dbname?sslmode=require
```
- ❌ Có thêm `?sslmode=require` hoặc các query params
- ❌ Có thể truy cập từ internet (nhưng chậm và không ổn định)
- ❌ Dễ bị timeout

### Internal Redis URL
```
redis://red-xxxxx:6379
```
hoặc
```
rediss://red-xxxxx:6379  # với SSL
```
- ✅ Hostname có dạng `red-xxxxx`
- ✅ Port thường là 6379
- ✅ Không có password trong URL (Render tự xử lý)

### External Redis URL
```
redis://red-xxxxx:6379?ssl=true
```
hoặc
```
rediss://default:password@red-xxxxx:6379
```
- ❌ Có query params hoặc password trong URL
- ❌ Có thể truy cập từ internet

## Troubleshooting sau khi sửa

### Vẫn gặp lỗi 502?

1. **Kiểm tra logs**: Xem có lỗi gì trong runtime logs
2. **Kiểm tra service status**: Đảm bảo service đang "Live"
3. **Kiểm tra health check**: `curl https://finance.tonyx.dev/up`
4. **Kiểm tra database**: Test connection trong Shell
5. **Kiểm tra Redis**: Test connection trong Shell

### Vẫn gặp lỗi 503?

1. **Kiểm tra memory**: Xem có đủ memory không
2. **Kiểm tra CPU**: Xem có quá tải không
3. **Upgrade plan**: Nếu cần thêm resources
4. **Kiểm tra logs**: Xem có lỗi crash không

### WebSocket vẫn lỗi?

1. **Kiểm tra REDIS_URL**: Đảm bảo dùng Internal URL
2. **Kiểm tra Action Cable config**: Xem `config/cable.yml`
3. **Kiểm tra HOST**: Đảm bảo đúng domain
4. **Restart service**: Sau khi sửa config

## Tài nguyên

- [Render Internal Services](https://render.com/docs/internal-services)
- [Render Database Connections](https://render.com/docs/databases#connecting-from-services)
- [Render Redis Connections](https://render.com/docs/redis#connecting-from-services)

## Kết luận

**Nguyên nhân chính của lỗi 502/503**:
1. ❌ Dùng External URL thay vì Internal URL
2. ❌ Thiếu `RAILS_MASTER_KEY`

**Giải pháp**:
1. ✅ Dùng Internal Database/Redis URL
2. ✅ Thêm `RAILS_MASTER_KEY` vào environment variables
3. ✅ Restart services sau khi cập nhật

Sau khi làm theo hướng dẫn này, lỗi 502/503 sẽ được khắc phục! 🎉
