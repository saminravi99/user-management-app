################################################################################
# NGINX DOCKERFILE FOR USER MANAGEMENT APPLICATION
#
# WHY: We need a custom Nginx container instead of using the default image
#      because we have specific routing rules and performance optimizations
#
# WHAT: This creates a production-ready Nginx container that:
#       - Acts as a reverse proxy (routes traffic to frontend/backend)
#       - Serves as the entry point for all HTTP requests
#       - Handles SSL termination (in production)
#       - Optimizes performance with caching and compression
#
# HOW: Takes the base Nginx image and customizes it by:
#      1. Removing default configurations
#      2. Adding our custom config files
#      3. Setting up cache directories with correct permissions
#      4. Configuring health checks
#
# WHERE: This runs as a Docker container in your deployment environment
#        Acts as the "front door" - all traffic hits Nginx first
#
# WHEN: Built during CI/CD pipeline and deployed alongside frontend/backend
################################################################################

################################################################################
# BASE IMAGE - Start with official Nginx on Alpine Linux
################################################################################
FROM nginx:1.25-alpine
# Explanation: What are we using as our starting point?
# "nginx:1.25-alpine" breaks down as:
# - nginx = the web server software
# - 1.25 = specific version (stable, tested release)
# - alpine = Linux distribution (minimal, only 5MB vs Ubuntu's 200MB)
#
# Why Alpine:
# - Smaller image size = faster downloads, less storage
# - Fewer packages = smaller attack surface (better security)
# - Same functionality as full Linux, just stripped down
#
# Size comparison:
# nginx:1.25-alpine = ~40MB
# nginx:1.25 (Debian-based) = ~190MB

################################################################################
# METADATA - Labels for documentation and management
################################################################################
LABEL maintainer="your-email@example.com"
# Explanation: Who to contact if there are issues with this image
# Replace with your actual email or team contact
# Useful when image is pushed to registry and others use it

LABEL description="Nginx reverse proxy for User Management Application"
# Explanation: What this container does
# Shows up in Docker registries and container management tools
# Helps identify purpose when you have dozens of containers

################################################################################
# CLEANUP - Remove default Nginx configurations
################################################################################
RUN rm /etc/nginx/nginx.conf
# Explanation: Delete the default main Nginx configuration
# Why: Default config is for serving static websites
# We need custom config for reverse proxy with specific routing rules
#
# The default config includes:
# - Generic performance settings
# - Default server listening on port 80
# - Serves files from /usr/share/nginx/html
# None of this matches our needs, so we remove it

RUN rm /etc/nginx/conf.d/default.conf
# Explanation: Delete the default server block configuration
# Why: Default tries to serve static HTML files
# We need to proxy requests to frontend/backend containers instead
#
# Default behavior we're replacing:
# - Serves /usr/share/nginx/html/index.html
# - No proxy configuration
# - No upstream blocks
# - No special routing logic

################################################################################
# CONFIGURATION - Copy our custom Nginx files
################################################################################
COPY nginx.conf /etc/nginx/nginx.conf
# Explanation: Copy our main configuration file into the container
# Source: nginx.conf (in the same directory as this Dockerfile)
# Destination: /etc/nginx/nginx.conf (where Nginx looks for main config)
#
# This file contains:
# - Worker process settings
# - Performance optimizations (gzip, sendfile, keepalive)
# - Security headers
# - Logging configuration
# - Global settings that apply to all server blocks

COPY conf.d/default.conf /etc/nginx/conf.d/default.conf
# Explanation: Copy our routing configuration into the container
# Source: conf.d/default.conf (relative to this Dockerfile)
# Destination: /etc/nginx/conf.d/default.conf
#
# This file contains:
# - Upstream definitions (backend, frontend)
# - Server blocks (HTTP, HTTPS)
# - Location blocks (routing rules)
# - Proxy settings
# - Caching strategies
#
# Nginx automatically includes all *.conf files from /etc/nginx/conf.d/
# So our file gets loaded when Nginx starts

################################################################################
# CACHE DIRECTORIES - Set up directories for Nginx's internal caching
################################################################################
RUN mkdir -p /var/cache/nginx/client_temp \
    /var/cache/nginx/proxy_temp \
    /var/cache/nginx/fastcgi_temp \
    /var/cache/nginx/uwsgi_temp \
    /var/cache/nginx/scgi_temp \
    && chown -R nginx:nginx /var/cache/nginx
# Explanation: Create directories Nginx uses for temporary storage
# The backslash (\) continues command across multiple lines for readability
# The && chains commands together (only run next if previous succeeded)
#
# Why these directories:
# - client_temp: Stores request bodies too large for memory
# - proxy_temp: Buffers responses from proxied servers (our frontend/backend)
# - fastcgi_temp: For FastCGI proxying (we don't use, but Nginx expects it)
# - uwsgi_temp: For uWSGI proxying (we don't use, but Nginx expects it)
# - scgi_temp: For SCGI proxying (we don't use, but Nginx expects it)
#
# chown -R nginx:nginx:
# Changes ownership to the "nginx" user
# Why: Nginx runs as non-root user "nginx" for security
# The nginx user needs permission to write to these directories
#
# Security context:
# Running as non-root means if Nginx is compromised, attacker has
# limited permissions (can't modify system files, can't access other containers)

################################################################################
# LOG DIRECTORY - Ensure log directory exists and is writable
################################################################################
RUN mkdir -p /var/log/nginx && chown -R nginx:nginx /var/log/nginx
# Explanation: Create log directory and set correct permissions
# mkdir -p: Creates directory if it doesn't exist (usually already exists)
# chown: Give nginx user ownership so it can write log files
#
# Nginx writes two types of logs:
# - access.log: Every request (who, what, when, status code)
# - error.log: Problems, warnings, debugging info
#
# Example log entry:
# 192.168.1.100 - [01/Jan/2024:12:00:00 +0000] "GET /api/users HTTP/1.1" 200 1234
# This shows IP, timestamp, request, response code, bytes sent

################################################################################
# PORT EXPOSURE - Document which ports this container uses
################################################################################
EXPOSE 80 443
# Explanation: Declare that this container listens on ports 80 and 443
# 80 = HTTP (unencrypted web traffic)
# 443 = HTTPS (encrypted web traffic with SSL/TLS)
#
# IMPORTANT: EXPOSE is documentation only!
# It does NOT actually open ports or publish them
# The actual port mapping happens when you run the container:
# docker run -p 8080:80 ...  (maps host port 8080 to container port 80)
#
# In Docker Compose:
# ports:
#   - "80:80"    # Host port 80 -> Container port 80
#   - "443:443"  # Host port 443 -> Container port 443
#
# Why expose both:
# - Port 80: For HTTP traffic (development, or redirect to HTTPS in production)
# - Port 443: For HTTPS traffic (production with SSL certificate)

################################################################################
# HEALTH CHECK - Monitor if Nginx is working properly
################################################################################
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD wget --quiet --tries=1 --spider http://localhost/health || exit 1
# Explanation: Automated health monitoring for this container
# Docker will periodically run this command to check if container is healthy
#
# Parameters explained:
# --interval=30s: Check every 30 seconds
#   Why 30s: Balance between quick detection and not overwhelming the system
#   More frequent = faster failure detection, but more resource usage
#
# --timeout=3s: Each check must complete within 3 seconds
#   Why 3s: Health check should be instant; if it takes >3s, something is wrong
#
# --start-period=10s: Don't check for first 10 seconds
#   Why: Give Nginx time to start up before declaring it unhealthy
#   During startup, configs are loaded, workers spawn - this takes a few seconds
#
# --retries=3: Must fail 3 times in a row before marking unhealthy
#   Why: Prevents false positives (one random timeout doesn't mean container is dead)
#   After 3 consecutive failures, Docker marks container as unhealthy
#
# The actual check command:
# wget --quiet --tries=1 --spider http://localhost/health || exit 1
# - wget: Command-line tool to make HTTP request
# - --quiet: Don't print output
# - --tries=1: Don't retry if it fails
# - --spider: Don't download, just check if URL exists
# - http://localhost/health: Our health check endpoint (defined in nginx config)
# - || exit 1: If wget fails, exit with error code 1 (marks check as failed)
#
# What happens when unhealthy:
# - Docker Compose: Can restart container automatically
# - Kubernetes: Stops sending traffic, restarts pod
# - Load balancers: Remove from pool
# - Monitoring: Triggers alerts

################################################################################
# STARTUP COMMAND - How to run Nginx when container starts
################################################################################
CMD ["nginx", "-g", "daemon off;"]
# Explanation: The command that runs when container starts
# CMD is default command (can be overridden when running container)
#
# Breaking it down:
# nginx: The Nginx executable
# -g: Global directive (pass configuration directly)
# daemon off: Run Nginx in foreground mode
#
# Why "daemon off":
# Normally Nginx runs as a background daemon (starts and detaches)
# Docker containers need a foreground process to stay alive
# If main process exits, container stops
#
# What happens:
# 1. Container starts
# 2. This CMD runs: nginx -g "daemon off;"
# 3. Nginx starts and stays in foreground
# 4. Container stays running as long as Nginx is running
# 5. If Nginx crashes, container stops (Docker can auto-restart)
#
# Alternative startup options we're NOT using:
# CMD ["nginx"] - Would run as daemon, container would exit immediately
# CMD ["/bin/sh", "-c", "nginx"] - Extra shell layer, slower startup
# ENTRYPOINT ["nginx"] - Can't be overridden easily
#
# The square bracket syntax ["nginx", "-g", "daemon off;"] is "exec form"
# Better than shell form "nginx -g 'daemon off;'" because:
# - No extra shell process
# - Signals (SIGTERM for graceful shutdown) go directly to Nginx
# - Faster startup
