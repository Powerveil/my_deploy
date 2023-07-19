# 项目地址

```
https://github.com/pengzhile/pandora
```

# Docker部署

## 安装Docker命令

```shell
curl -fsSL https://get.docker.com |bash -s docker --mirror Aliyun
```

## Docker拉镜像

```shell
docker pull pengzhile/pandora
```

## 运行镜像（注意：命令1和命令2是不同的场景）

该服务占用的端口是8899

### 所有ip都可以访问8899端口

```shell
docker run -e PANDORA_CLOUD=cloud -e PANDORA_SERVER=0.0.0.0:8899 -p 8899:8899 -d pengzhile/pandora
```

### 设置只能本机nginx重定向访问，其他ip不能访问8899端口（有域名了就不让别人直接访问8899）

```shell
docker run -e PANDORA_CLOUD=cloud -e PANDORA_SERVER=0.0.0.0:8899 -p 127.0.0.1:8899:8899 -d pengzhile/pandora
```

# Nginx配置（注意域名那里）

```
# Nginx配置

user  www www;
worker_processes auto;
error_log  /www/wwwlogs/nginx_error.log  crit;
pid        /www/server/nginx/logs/nginx.pid;
worker_rlimit_nofile 51200;

stream {
    log_format tcp_format '$time_local|$remote_addr|$protocol|$status|$bytes_sent|$bytes_received|$session_time|$upstream_addr|$upstream_bytes_sent|$upstream_bytes_received|$upstream_connect_time';
  
    access_log /www/wwwlogs/tcp-access.log tcp_format;
    error_log /www/wwwlogs/tcp-error.log;
    include /www/server/panel/vhost/nginx/tcp/*.conf;
}

events
    {
        use epoll;
        worker_connections 51200;
        multi_accept on;
    }

http
    {
        include       mime.types;
		#include luawaf.conf;

		include proxy.conf;

        default_type  application/octet-stream;

        server_names_hash_bucket_size 512;
        client_header_buffer_size 32k;
        large_client_header_buffers 4 32k;
        client_max_body_size 50m;

        sendfile   on;
        tcp_nopush on;

        keepalive_timeout 60;

        tcp_nodelay on;

        fastcgi_connect_timeout 300;
        fastcgi_send_timeout 300;
        fastcgi_read_timeout 300;
        fastcgi_buffer_size 64k;
        fastcgi_buffers 4 64k;
        fastcgi_busy_buffers_size 128k;
        fastcgi_temp_file_write_size 256k;
		fastcgi_intercept_errors on;

        gzip on;
        gzip_min_length  1k;
        gzip_buffers     4 16k;
        gzip_http_version 1.1;
        gzip_comp_level 2;
        gzip_types     text/plain application/javascript application/x-javascript text/javascript text/css application/xml;
        gzip_vary on;
        gzip_proxied   expired no-cache no-store private auth;
        gzip_disable   "MSIE [1-6]\.";

        limit_conn_zone $binary_remote_addr zone=perip:10m;
		limit_conn_zone $server_name zone=perserver:10m;

        server_tokens off;
        access_log off;

server
    {
        listen 80;
        server_name 你的域名;
        index index.html index.htm index.php;
        root  /www/server/phpmyadmin;

        #error_page   404   /404.html;
        include enable-php.conf;

        # location ~ .*\.(gif|jpg|jpeg|png|bmp|swf)$
        # {
        #     expires      30d;
        # }

        # location ~ .*\.(js|css)?$
        # {
        #     expires      12h;
        # }

        # location ~ /\.
        # {
        #     deny all;
        # }
        
        location / {
        	proxy_pass http://127.0.0.1:8899;
        	proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header X-Forwarded-Proto $scheme;
        }

        access_log  /www/wwwlogs/access.log;
    }
include /www/server/panel/vhost/nginx/*.conf;
}
```

# 出现问题

如果Docker拉取镜像出现问题，请检查Docker服务是否开启

## 检查Docker状态

```shell
systemctl status docker
```

## 开启Docker服务

```shell
systemctl start docker
```

## 开机自启Docker

```shell
systemctl enable docker
```

## 关闭Docker服务

```shell
systemctl stop docker
```

## 重启Docker服务

```shell
systemctl restart docker
```
