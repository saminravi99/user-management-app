# Nginx Dockerfile for User Management Application
# This creates an optimized Nginx container for production

# Use official Nginx image based on Alpine Linux (smaller size)
FROM nginx:1.25-alpine

# Maintainer information
LABEL maintainer="your-email@example.com"
LABEL description="Nginx reverse proxy for User Management Application"

# Remove default Nginx configuration
RUN rm /etc/nginx/nginx.conf
RUN rm /etc/nginx/conf.d/default.conf

# Copy our custom Nginx configuration
COPY nginx.conf /etc/nginx/nginx.conf
COPY conf.d/default.conf /etc/nginx/conf.d/default.conf

# Create cache directories with proper permissions
RUN mkdir -p /var/cache/nginx/client_temp \
    /var/cache/nginx/proxy_temp \
    /var/cache/nginx/fastcgi_temp \
    /var/cache/nginx/uwsgi_temp \
    /var/cache/nginx/scgi_temp \
    && chown -R nginx:nginx /var/cache/nginx

# Create log directory (if not exists)
RUN mkdir -p /var/log/nginx && chown -R nginx:nginx /var/log/nginx

# Expose HTTP and HTTPS ports
EXPOSE 80 443

# Health check - verify Nginx is responding
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD wget --quiet --tries=1 --spider http://localhost/health || exit 1

# Start Nginx in foreground mode
CMD ["nginx", "-g", "daemon off;"]
