FROM nginx:1.24.0

# Create non-root user with UID 1001
RUN useradd -u 1001 -M -s /sbin/nologin nginxuser

# Change ownership of nginx directories
RUN chown -R nginxuser:nginxuser /var/cache/nginx /var/run/ /var/log/nginx

# Change the listening port to 8080 (Non-privileged)
COPY default.conf /etc/nginx/conf.d/default.conf

# remove user definition to supress warning
COPY nginx.conf /etc/nginx/nginx.conf
