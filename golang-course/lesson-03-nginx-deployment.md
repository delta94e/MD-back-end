# Lesson 03 — Nginx, Docker Compose & Deployment

> **Mục tiêu:** Triển khai toàn bộ hệ thống app lên server, hiểu cách scale
> multiple instances và vận hành app như một end user thực sự.

---

## Phần 1: Docker Compose — Deep Dive

### 1.1 Các Tính Năng Quan Trọng

```yaml
# docker-compose.yml — full features
version: "3.9"

services:
  app:
    image: myapp:latest
    # Restart policy
    restart: unless-stopped         # always | on-failure | no

    # Environment variables
    environment:
      - APP_ENV=production
      - PORT=8080
    env_file:
      - .env                        # secrets (không commit git)

    # Networking
    networks:
      - frontend-net                # network với nginx
      - backend-net                 # network với DB/Redis

    # Phụ thuộc
    depends_on:
      postgres:
        condition: service_healthy  # chờ postgres healthy mới start
      redis:
        condition: service_healthy

    # Giới hạn tài nguyên
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 256M
        reservations:
          cpus: "0.25"
          memory: 128M

    # Health check
    healthcheck:
      test: ["CMD", "wget", "-q", "-O-", "http://localhost:8080/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s             # chờ app khởi động xong mới check

    # Logging
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

networks:
  frontend-net:
    driver: bridge
  backend-net:
    driver: bridge                  # DB/Redis chỉ trong mạng này, không expose

volumes:
  postgres-data:
    driver: local
  redis-data:
    driver: local
```

### 1.2 Networks — Phân Tách Mạng

```
╔══════════════════════════════════════════════════════════════════════╗
║  NETWORK ISOLATION — Best Practice                                   ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  Internet                                                           ║
║      │                                                               ║
║  [Nginx] ────── frontend-net ────── [App 1]                        ║
║      │                              [App 2]                        ║
║      │                              [App 3]                        ║
║      │                                 │                            ║
║      │                         backend-net                          ║
║      │                                 │                            ║
║      │                          [Postgres]  [Redis]                ║
║      │                                                              ║
║  Postgres và Redis KHÔNG thể truy cập từ Internet!                  ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## Phần 2: Nginx

### 2.1 Nginx Là Gì?

```
Nginx (đọc: "engine-x") là web server / reverse proxy hiệu năng cao.
Trong hệ thống của chúng ta, Nginx đóng vai trò:

1. REVERSE PROXY:    Nhận request từ internet → chuyển đến app
2. LOAD BALANCER:    Phân phối request giữa nhiều instances của app
3. STATIC FILE:      Phục vụ file tĩnh (HTML, CSS, JS) trực tiếp
4. SSL TERMINATION:  Xử lý HTTPS, app chỉ cần HTTP nội bộ
5. RATE LIMITING:    Giới hạn request ở tầng infrastructure
6. CACHING:          Cache response từ upstream app
```

### 2.2 nginx.conf — Cấu Hình Cơ Bản

```nginx
# nginx/nginx.conf
user  nginx;
worker_processes  auto;              # tự động theo số CPU cores

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

events {
    worker_connections  1024;        # max connections per worker
    use epoll;                       # I/O model tối ưu cho Linux
    multi_accept on;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    # Log format
    log_format  main  '$remote_addr - $request_time '
                      '"$request" $status $body_bytes_sent";

    access_log  /var/log/nginx/access.log  main;

    # Performance
    sendfile        on;
    tcp_nopush      on;
    tcp_nodelay     on;
    keepalive_timeout  65;
    gzip  on;
    gzip_types text/plain application/json application/javascript text/css;

    # Include các vhost configs
    include /etc/nginx/conf.d/*.conf;
}
```

### 2.3 Reverse Proxy + Load Balancer

```nginx
# nginx/conf.d/app.conf

# Upstream = nhóm các backend servers
upstream go_app {
    # Load balancing algorithms:
    # (mặc định) round-robin: phân phối đều lần lượt
    # least_conn:  gửi đến server ít connections nhất
    # ip_hash:     cùng IP → cùng server (sticky session)

    least_conn;

    # Docker service name + port (docker compose tự resolve DNS)
    server app:8080;                 # sẽ resolve đến tất cả containers tên "app"

    # Hoặc khai báo từng instance:
    # server app_1:8080 weight=3;   # weight cao hơn → nhận nhiều request hơn
    # server app_2:8080 weight=1;
    # server app_3:8080 backup;     # chỉ dùng khi các servers kia down

    # Health check (nginx plus) hoặc dùng passive:
    # server app_1:8080 max_fails=3 fail_timeout=30s;
}

server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

    # Redirect HTTP → HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name yourdomain.com;

    # SSL certificates (từ Let's Encrypt)
    ssl_certificate     /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";
    add_header X-XSS-Protection "1; mode=block";

    # Giới hạn kích thước request
    client_max_body_size 10m;

    # Timeout
    proxy_connect_timeout 10s;
    proxy_send_timeout    30s;
    proxy_read_timeout    30s;

    # API endpoints → Go app
    location /api/ {
        proxy_pass          http://go_app;   # chuyển đến upstream
        proxy_http_version  1.1;
        proxy_set_header    Upgrade $http_upgrade;
        proxy_set_header    Connection "upgrade";  # WebSocket support
        proxy_set_header    Host $host;
        proxy_set_header    X-Real-IP $remote_addr;
        proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header    X-Forwarded-Proto $scheme;
    }

    # Static files (frontend build) → phục vụ trực tiếp, không qua Go app
    location / {
        root   /usr/share/nginx/html;
        index  index.html;
        try_files $uri $uri/ /index.html;  # SPA routing (React/Vue)

        # Cache static assets
        location ~* \.(js|css|png|jpg|gif|ico|woff2)$ {
            expires 1y;
            add_header Cache-Control "public, immutable";
        }
    }

    # Health check endpoint (không log)
    location /health {
        proxy_pass http://go_app;
        access_log off;
    }

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    location /api/auth/ {
        limit_req zone=api burst=20 nodelay;
        proxy_pass http://go_app;
    }
}
```

---

## Phần 3: Kiến Trúc Hệ Thống — Infrastructure Diagram

```
╔══════════════════════════════════════════════════════════════════════╗
║  PRODUCTION SYSTEM ARCHITECTURE                                     ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║                        INTERNET                                     ║
║                            │                                         ║
║                     Port 80 / 443                                   ║
║                            │                                         ║
║              ┌─────────────────────────┐                            ║
║              │    NGINX (Reverse Proxy) │                           ║
║              │    - SSL Termination     │                           ║
║              │    - Load Balancing      │                           ║
║              │    - Static Files        │                           ║
║              │    - Rate Limiting       │                           ║
║              └─────────────┬───────────┘                            ║
║                            │                                         ║
║              ┌─────────────┼─────────────┐                          ║
║              │             │             │                           ║
║   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐              ║
║   │  Go App :1   │ │  Go App :2   │ │  Go App :3   │              ║
║   │  (port 8080) │ │  (port 8080) │ │  (port 8080) │              ║
║   └──────┬───────┘ └──────┬───────┘ └──────┬───────┘              ║
║          │                │                 │                        ║
║          └────────────────┴─────────────────┘                       ║
║                           │                                          ║
║              ┌────────────┴──────────────┐                          ║
║              │                           │                           ║
║   ┌──────────────────┐       ┌─────────────────────┐               ║
║   │  PostgreSQL       │       │       Redis          │              ║
║   │  (port 5432)      │       │    (port 6379)       │              ║
║   │  - Primary DB     │       │    - Cache           │              ║
║   │  - Persistent     │       │    - Sessions        │              ║
║   └──────────────────┘       │    - Rate Limit       │              ║
║                               └─────────────────────┘               ║
║                                                                      ║
║  Tất cả trên 1 VPS (Single Server Deployment)                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## Phần 4: docker-compose.yml — Full Production Stack

```yaml
# docker-compose.yml
version: "3.9"

services:
  # ─── NGINX — Entry Point ──────────────────────────────────────────
  nginx:
    image: nginx:1.25-alpine
    container_name: nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./frontend/dist:/usr/share/nginx/html:ro    # frontend build
      - ./certbot/conf:/etc/letsencrypt:ro           # SSL certs
    depends_on:
      - app
    networks:
      - frontend-net
    restart: unless-stopped

  # ─── GO APP — có thể scale ────────────────────────────────────────
  app:
    image: ${REGISTRY}/myapp:${APP_VERSION:-latest}
    env_file: .env
    expose:
      - "8080"                        # expose nội bộ, không ra ngoài
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - frontend-net                  # để nginx gọi được
      - backend-net                   # để gọi DB/Redis
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-q", "-O-", "http://localhost:8080/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 15s
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  # ─── POSTGRESQL ───────────────────────────────────────────────────
  postgres:
    image: postgres:16-alpine
    container_name: postgres
    env_file: .env
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./migrations/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    networks:
      - backend-net
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER} -d ${DB_NAME}"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ─── REDIS ───────────────────────────────────────────────────────
  redis:
    image: redis:7-alpine
    container_name: redis
    command: >
      redis-server
      --requirepass ${REDIS_PASSWORD}
      --maxmemory 256mb
      --maxmemory-policy allkeys-lru
      --save 60 1
    volumes:
      - redis-data:/data
    networks:
      - backend-net
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

networks:
  frontend-net:
    driver: bridge
  backend-net:
    driver: bridge
    internal: true                    # KHÔNG có internet access

volumes:
  postgres-data:
  redis-data:
```

```bash
# .env file (KHÔNG commit vào git — thêm vào .gitignore)
APP_ENV=production
APP_VERSION=v1.0.0
REGISTRY=your-registry.io

DB_HOST=postgres
DB_PORT=5432
DB_NAME=appdb
DB_USER=appuser
DB_PASSWORD=super_secret_password

REDIS_ADDR=redis:6379
REDIS_PASSWORD=redis_secret_password

JWT_SECRET=your_jwt_secret_here
```

---

## Phần 5: Scale Multiple Instances

### 5.1 Scale Với Docker Compose

```bash
# Scale lên 3 instances của app
docker compose up -d --scale app=3

# Kiểm tra:
docker compose ps
# NAME         COMMAND   SERVICE   STATUS    PORTS
# myapp-app-1  ...       app       running   8080/tcp
# myapp-app-2  ...       app       running   8080/tcp
# myapp-app-3  ...       app       running   8080/tcp

# Scale xuống
docker compose up -d --scale app=1

# Zero-downtime deploy (update 1 instance tại 1 thời điểm)
docker compose up -d --no-deps --scale app=3 app
```

### 5.2 Nginx Tự Động Load Balance

```nginx
# Khi scale app=3, Docker DNS tự động resolve "app" thành 3 IPs
# Nginx sẽ phân phối request đều cho 3 instances

upstream go_app {
    least_conn;
    server app:8080;    # Docker tự resolve → round robin to 3 containers
}
```

> **Lưu ý:** Khi dùng `server app:8080` với service name, Docker DNS trả về
> tất cả IP của các containers cùng service. Nginx tự động phân phối.

### 5.3 Stateless App — Điều Kiện Để Scale

```
╔══════════════════════════════════════════════════════════════════════╗
║  ĐỂ SCALE ĐƯỢC, APP PHẢI STATELESS                                  ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  ĐÚNG (stateless):                                                  ║
║  - Session lưu ở Redis (shared giữa tất cả instances)              ║
║  - File upload → S3/Cloud Storage (không lưu local disk)            ║
║  - Config → Environment variables                                   ║
║  - State → PostgreSQL (shared)                                      ║
║                                                                      ║
║  SAI (stateful — không scale được):                                 ║
║  - Lưu session trong memory của app                                 ║
║  - Lưu file upload vào disk của container                           ║
║  - Dùng global variable để lưu state                                ║
║                                                                      ║
║  Request 1 → App Instance 1 (session ở đây)                        ║
║  Request 2 → App Instance 2 (không có session!)  ← BUG             ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## Phần 6: Triển Khai Lên Cloud (VPS)

### 6.1 Chuẩn Bị Server (Ubuntu 22.04)

```bash
# 1. Cập nhật system
sudo apt update && sudo apt upgrade -y

# 2. Cài Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER

# 3. Cài Docker Compose
sudo apt install docker-compose-plugin -y

# Kiểm tra
docker --version
docker compose version
```

### 6.2 Cấu Trúc Thư Mục Trên Server

```
/opt/myapp/
├── docker-compose.yml
├── .env                        # secrets (không trong git)
├── nginx/
│   ├── nginx.conf
│   └── conf.d/
│       └── app.conf
├── migrations/
│   └── init.sql
└── certbot/                    # SSL certificates
    └── conf/
```

### 6.3 Deploy Script

```bash
#!/bin/bash
# scripts/deploy.sh

set -e  # dừng nếu có lỗi

APP_VERSION=${1:-latest}
REGISTRY="your-registry.io"
DEPLOY_DIR="/opt/myapp"

echo "=== Deploying version: $APP_VERSION ==="

# 1. Pull image mới
docker pull $REGISTRY/myapp:$APP_VERSION

# 2. Update APP_VERSION trong .env
sed -i "s/^APP_VERSION=.*/APP_VERSION=$APP_VERSION/" $DEPLOY_DIR/.env

# 3. Rolling update — scale lên trước, rồi xóa cũ
cd $DEPLOY_DIR
docker compose up -d --no-deps --scale app=3 app
sleep 5  # chờ instances mới healthy
docker compose up -d --no-deps --scale app=2 app

echo "=== Deploy thành công ==="
docker compose ps
```

### 6.4 SSL Với Let's Encrypt

```bash
# Cài certbot
sudo apt install certbot -y

# Lấy certificate (tạm dừng nginx nếu đang chạy)
sudo certbot certonly --standalone \
  -d yourdomain.com \
  -d www.yourdomain.com \
  --email your@email.com \
  --agree-tos

# Certificates được lưu ở:
# /etc/letsencrypt/live/yourdomain.com/fullchain.pem
# /etc/letsencrypt/live/yourdomain.com/privkey.pem

# Tự động renew (add vào crontab)
# 0 0 1 * * certbot renew --quiet && docker compose restart nginx
```

### 6.5 Makefile — Deployment Commands

```makefile
REGISTRY       = your-registry.io
APP_NAME       = myapp
VERSION        = $(shell git describe --tags --always)
SERVER_USER    = ubuntu
SERVER_HOST    = your-server-ip
DEPLOY_DIR     = /opt/myapp

.PHONY: build push deploy logs status

# Build và push image
release:
	docker build -t $(REGISTRY)/$(APP_NAME):$(VERSION) .
	docker push $(REGISTRY)/$(APP_NAME):$(VERSION)
	@echo "Released: $(VERSION)"

# Deploy lên server
deploy:
	ssh $(SERVER_USER)@$(SERVER_HOST) \
		"cd $(DEPLOY_DIR) && bash scripts/deploy.sh $(VERSION)"

# Xem logs
logs:
	ssh $(SERVER_USER)@$(SERVER_HOST) \
		"cd $(DEPLOY_DIR) && docker compose logs -f --tail=100 app"

# Kiểm tra trạng thái
status:
	ssh $(SERVER_USER)@$(SERVER_HOST) \
		"cd $(DEPLOY_DIR) && docker compose ps"

# Scale
scale:
	ssh $(SERVER_USER)@$(SERVER_HOST) \
		"cd $(DEPLOY_DIR) && docker compose up -d --scale app=$(N)"
# Dùng: make scale N=3
```

---

## Phần 7: Monitoring — Quan Sát Hệ Thống

```bash
# Xem tất cả containers
docker compose ps

# Resource usage real-time
docker stats

# Xem logs tất cả services
docker compose logs -f

# Xem logs 1 service, 100 dòng cuối
docker compose logs -f --tail=100 app

# Kiểm tra health
docker inspect --format='{{.State.Health.Status}}' myapp-app-1

# Vào container để debug
docker compose exec app sh
docker compose exec postgres psql -U appuser -d appdb
docker compose exec redis redis-cli -a $REDIS_PASSWORD
```

---

## Phần 8: Bài Tập

### Bài 1 — Thêm Frontend Vào Hệ Thống

Build và tích hợp frontend (React/Vite) vào hệ thống:

```
frontend/
  src/
    App.tsx
    pages/
  Dockerfile
  nginx.conf          # đơn giản, chỉ serve static files
```

```dockerfile
# frontend/Dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

Tích hợp vào `docker-compose.yml`:
```yaml
frontend:
  build: ./frontend
  networks:
    - frontend-net
```

Cập nhật nginx config để:
- `/` → phục vụ frontend static files
- `/api/` → proxy đến Go app

### Bài 2 — Scale Và Kiểm Tra Load Balancing

```bash
# 1. Scale app lên 3 instances
docker compose up -d --scale app=3

# 2. Dùng curl để test và verify load balancing
# App nên trả về hostname trong response để biết instance nào xử lý:
# GET /api/debug/instance → {"hostname": "myapp-app-1"}

for i in $(seq 1 9); do
  curl -s http://localhost/api/debug/instance | jq .hostname
done
# Kết quả mong đợi: xen kẽ giữa app-1, app-2, app-3

# 3. Test với wrk (tool benchmark)
wrk -t4 -c100 -d30s http://localhost/api/users
```

### Bài 3 — Diagram Hạ Tầng

Vẽ diagram hệ thống của mình bao gồm:
- Internet → Nginx → App instances → DB/Redis
- Networks (frontend-net, backend-net)
- Ports được expose
- Volumes được mount
- Dùng draw.io, Excalidraw, hoặc ASCII art

### Bài 4 — Deploy Lên Cloud

Thuê VPS (DigitalOcean, Vultr, hoặc AWS EC2 free tier) và thực hiện:
1. Setup server (Docker, UFW firewall)
2. Push image lên Docker Hub (free)
3. Clone config lên server, tạo `.env`
4. Lấy SSL certificate với Certbot
5. Deploy và truy cập app qua domain của mình
6. Scale lên 2 instances và verify load balancing

```bash
# Checklist deployment:
# [ ] docker compose ps — tất cả services Running
# [ ] curl https://yourdomain.com/health — 200 OK
# [ ] curl https://yourdomain.com/api/users — data trả về
# [ ] Mở browser, truy cập domain — frontend hiển thị
# [ ] docker stats — resource usage bình thường
# [ ] docker compose logs — không có ERROR
```

---

## Tổng Kết Lesson 03

```
╔══════════════════════════════════════════════════════════════════════╗
║  LESSON 03 — KEY TAKEAWAYS                                          ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  Docker Compose:                                                    ║
║  + Networks tách biệt frontend/backend → bảo mật tốt hơn           ║
║  + healthcheck → depends_on chờ service thật sự healthy             ║
║  + logging config → tránh disk đầy vì log                          ║
║  + .env file → không bao giờ commit secrets                         ║
║                                                                      ║
║  Nginx:                                                             ║
║  + Reverse proxy: che giấu backend, kiểm soát traffic              ║
║  + Load balancer: phân phối request đều, tự động failover           ║
║  + SSL termination: app chỉ cần HTTP nội bộ                        ║
║  + Static files: phục vụ trực tiếp, không qua app                  ║
║                                                                      ║
║  Scaling:                                                           ║
║  + App PHẢI stateless: session/state ở Redis/DB, không ở memory    ║
║  + docker compose up --scale app=N → horizontal scaling            ║
║  + Nginx tự động phân phối request qua Docker DNS                  ║
║                                                                      ║
║  Deployment:                                                        ║
║  + Multi-stage Dockerfile → image nhỏ, secure                      ║
║  + SSL miễn phí với Let's Encrypt + Certbot                        ║
║  + Rolling deploy → zero-downtime                                   ║
║  + Makefile → deploy 1 lệnh: make release && make deploy           ║
║                                                                      ║
║  Sau lesson này:                                                    ║
║  → Học viên có thể tự deploy app Go lên VPS với domain thật        ║
║  → Có thể scale khi traffic tăng                                    ║
║  → Hiểu toàn bộ luồng từ code → build → deploy → end user          ║
╚══════════════════════════════════════════════════════════════════════╝
```
