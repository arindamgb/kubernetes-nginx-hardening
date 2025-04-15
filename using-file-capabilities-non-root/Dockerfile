FROM nginx:1.24.0

# Install libcap2-bin to enable setcap
USER root
RUN apt-get update && \
    apt-get install -y libcap2-bin && \
    setcap 'cap_net_bind_service=+ep' /usr/sbin/nginx && \
    apt-get autoremove -y && \
    rm -rf /var/lib/apt/lists/*

# Create non-root user with UID 1001
RUN useradd -u 1001 -M -s /sbin/nologin nginxuser

# Change ownership of nginx directories
RUN chown -R nginxuser:nginxuser /var/cache/nginx /var/run/ /var/log/nginx

# Use non-root user
USER 1001

# Expose port 80 and start nginx
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
