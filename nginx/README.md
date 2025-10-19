# Nginx Configuration for User Management Application

This directory contains the Nginx reverse proxy configuration for the User Management Application.

## 📁 Directory Structure

```
nginx/
├── Dockerfile              # Nginx container image
├── nginx.conf             # Main Nginx configuration
├── conf.d/
│   └── default.conf       # Server blocks and routing rules
└── README.md              # This file
```

## 🎯 How It Works

### Single Nginx Instance for Both Frontend & Backend

```
Internet → Nginx (Port 80) → {
    /api/*     → Backend (Node.js on port 5000)
    /          → Frontend (Next.js on port 3000)
}
```

### Routing Rules

| Request Path                  | Proxied To    | Description                           |
| ----------------------------- | ------------- | ------------------------------------- |
| `/api/*`                      | backend:5000  | All API requests                      |
| `/uploads/*`                  | backend:5000  | Uploaded files from backend           |
| `/_next/static/*`             | frontend:3000 | Next.js static assets (cached 1 year) |
| `/favicon.ico`, `/robots.txt` | frontend:3000 | SEO files (cached 7 days)             |
| `/health`                     | Nginx itself  | Health check endpoint                 |
| `/` (all others)              | frontend:3000 | Frontend pages                        |

## ⚙️ Key Features

### 1. **Reverse Proxy**

- Single entry point for all traffic
- Routes requests to appropriate containers
- Preserves original client IP and headers

### 2. **Load Balancing Ready**

```nginx
upstream backend_api {
    least_conn;  # Use least connections algorithm
    server backend:5000;
    # Easily add more: server backend2:5000;
}
```

### 3. **Caching Strategy**

- **API responses**: No caching (always fresh data)
- **Next.js static files**: 1 year cache (content-hashed)
- **Public assets**: 7 days cache
- **HTML pages**: No caching (always fresh)

### 4. **Security Headers**

- `X-Frame-Options`: Prevents clickjacking
- `X-Content-Type-Options`: Prevents MIME sniffing
- `X-XSS-Protection`: Enables XSS protection
- `server_tokens off`: Hides Nginx version

### 5. **Performance Optimizations**

- **Gzip compression**: Reduces bandwidth by ~70%
- **HTTP/2 support**: Faster page loads (when HTTPS enabled)
- **Keepalive connections**: Reuses connections
- **Connection pooling**: Efficient upstream connections

### 6. **WebSocket Support**

- Enabled for Next.js hot reload in development
- Ready for real-time features (chat, notifications)

## 🚀 Usage with Docker Compose

### Development

```yaml
services:
  nginx:
    build: ./nginx
    ports:
      - "80:80"
    depends_on:
      - frontend
      - backend
```

### Production (with SSL)

```yaml
services:
  nginx:
    build: ./nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /etc/letsencrypt:/etc/letsencrypt:ro # SSL certificates
```

## 🔒 SSL/HTTPS Configuration

### For Production with Domain:

1. **Get SSL Certificate** (Let's Encrypt):

```bash
sudo certbot certonly --standalone -d example.com -d www.example.com
```

2. **Uncomment HTTPS server block** in `conf.d/default.conf`:

```nginx
server {
    listen 443 ssl http2;
    server_name example.com www.example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    # ... rest of configuration
}
```

3. **Update server_name** in both HTTP and HTTPS blocks:

```nginx
server_name example.com www.example.com;
```

## 📊 Monitoring & Logs

### Access Logs

- Location: `/var/log/nginx/access.log`
- Format: Includes response times and upstream timings

### Error Logs

- Location: `/var/log/nginx/error.log`
- Level: `warn` (only warnings and errors)

### Health Check

- Endpoint: `http://your-server/health`
- Returns: `200 OK`
- Used by: Docker, load balancers, monitoring tools

## 🔧 Customization

### Increase Upload Size

In `nginx.conf` or `default.conf`:

```nginx
client_max_body_size 50M;  # Change from 20M to 50M
```

### Add Rate Limiting (prevent DDoS)

In `default.conf`:

```nginx
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

location /api {
    limit_req zone=api_limit burst=20 nodelay;
    # ... rest of config
}
```

### Add IP Whitelisting for Admin Routes

```nginx
location /api/admin {
    allow 1.2.3.4;     # Your office IP
    deny all;          # Deny all others
    proxy_pass http://backend_api;
}
```

## 🧪 Testing Configuration

### Test Nginx config before restarting:

```bash
docker exec nginx nginx -t
```

### Reload Nginx without downtime:

```bash
docker exec nginx nginx -s reload
```

## 📈 Performance Tuning

### For High Traffic:

```nginx
worker_processes auto;           # Use all CPU cores
worker_connections 4096;         # Increase from 1024
keepalive 64;                    # More keepalive connections
```

### For File Uploads:

```nginx
client_body_buffer_size 128k;
client_body_timeout 60s;
```

## 🐛 Troubleshooting

### 502 Bad Gateway

- **Cause**: Backend/Frontend container not running
- **Fix**: Check if containers are healthy: `docker ps`

### 413 Request Entity Too Large

- **Cause**: Upload exceeds `client_max_body_size`
- **Fix**: Increase limit in `nginx.conf`

### Static files not loading

- **Cause**: Wrong proxy_pass or caching issue
- **Fix**: Check browser console, verify upstream is correct

## 📚 References

- [Nginx Official Docs](https://nginx.org/en/docs/)
- [Nginx Reverse Proxy Guide](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)
- [SSL Best Practices](https://ssl-config.mozilla.org/)
