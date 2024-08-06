# 使用官方的 Nginx 镜像作为基础镜像
FROM nginx:latest

# 复制你的 Nginx 配置文件到容器中的默认配置文件位置
COPY proxy.conf /etc/nginx/conf.d/default.conf

# 创建日志目录并设置权限
RUN mkdir -p /var/log/nginx && \
    chmod -R 755 /var/log/nginx

# 暴露端口
EXPOSE 80

# 启动 Nginx
CMD ["nginx", "-g", "daemon off;"]
