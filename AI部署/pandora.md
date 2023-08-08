# 项目地址

```
https://github.com/pengzhile/pandora
```

下面几行与本文无关（为了让我快速找到指令，不想往下翻了）

因为我用的服务器短时的，有时候忘记续费，每次连接新的服务器就会出现

**WARNING! The remote SSH server rejected X11 forwarding request.**

看着不舒服

```shell
原因：X11 forwarding依赖xorg-x11-xauth软件包，需要先安装xorg-x11-xauth软件包。
解决方案(只提供这一种)：yum install xorg-x11-xauth
然后重新连接查看效果
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

events {
    use epoll;
    worker_connections 51200;
    multi_accept on;
}

http {
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

    server {
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

# 一些常用指令

## 查看容器

```shell
列出正在运行容器：docker ps
列出所有容器：docker ps -a
```

## 停止容器

```
docker stop [容器ID或容器名称]
```

## 启动容器

```shell
docker start [容器ID或容器名称]
```

## 重启容器

```shell
docker restart [容器ID或容器名称]
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

# 后期的一些场景

## 场景1---每次拉取镜像tag都是latest，想拉取最新的时候，又想把之前的镜像存起来。

查看镜像

```shell
docker images

REPOSITORY          TAG       IMAGE ID       CREATED       SIZE
pengzhile/pandora   latest    afd2b9c7baae   11 days ago   256MB
```

先把原来的镜像重命名存起来

```shell
docker tag 镜像id pengzhile/pandora:标签(TAG)
```

删除本地docker中的这个镜像

```shell
要用的指令：
docker images 镜像名(REPOSITORY):标签(TAG)
我们使用的指令：以下两个指令均可，第一个默认删除标签为latest
docker images 镜像名
docker images 镜像名:latest
```

拉取新镜像

```shell
docker pull pengzhile/pandora
```

如果有需求就让这个新镜像也改个标签（多少版本）

```shell
最终效果
REPOSITORY          TAG       IMAGE ID       CREATED       SIZE
pengzhile/pandora   1.3.0     211f49b77c2c   3 hours ago   255MB
pengzhile/pandora   1.2.8     afd2b9c7baae   11 days ago   256MB
```

**运行的指令也需要修改一下要加上标签，例如下面**

```shell
docker run -e PANDORA_CLOUD=cloud -e PANDORA_SERVER=0.0.0.0:8899 -p 8899:8899 -d pengzhile/pandora:1.3.0


docker run -e PANDORA_CLOUD=cloud -e PANDORA_SERVER=0.0.0.0:8899 -p 127.0.0.1:8899:8899 -d pengzhile/pandora:1.3.0
```

